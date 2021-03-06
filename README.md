# PredictionIO classification app

Predictive classification powered by [PredictionIO](https://predictionio.incubator.apache.org), machine learning on [Heroku](http://www.heroku.com).

This is a demo application of PredictionIO version 0.9.5, already customized for the smoothest experience possible. **Custom PredictionIO engines** may be deployed as well, see [CUSTOM documentation](CUSTOM.md).

## How To 📚

✏️ Throughout this document, code terms that start with `$` represent a value (shell variable) that should be replaced with a customized value, e.g `$eventserver_name`, `$engine_name`, `$postgres_addon_id`…

### Deploy to Heroku

Please follow steps in order.

1. [Requirements](#1-requirements)
1. [Eventserver](#2-eventserver)
  1. [Create the eventserver](#create-the-eventserver)
  1. [Deploy the eventserver](#deploy-the-eventserver)
1. [Classification engine](#3-classification-engine)
  1. [Create the engine](#create-the-engine)
  1. [Connect the engine with the eventserver](#connect-the-engine-with-the-eventserver)
  1. [Import data](#import-data)
  1. [Deploy the engine](#deploy-the-engine)

### Usage

Once deployed, how to work with the engine.

* [Scale-up](#scale-up)
* 🎯 [Query for predictions](#query-for-predictions)
* [Diagnostics](#diagnostics)


# Deploy to Heroku 🚀

## 1. Requirements

* [Heroku account](https://signup.heroku.com)
* [Heroku CLI](https://toolbelt.heroku.com), command-line tools
* [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

## 2. Eventserver

### Create the eventserver

```bash
git clone \
  https://github.com/heroku/predictionio-buildpack.git \
  pio-eventserver-classification

cd pio-eventserver-classification

heroku create $eventserver_name
heroku addons:create heroku-postgresql:hobby-dev
heroku buildpacks:add -i 1 https://github.com/heroku/predictionio-buildpack.git
heroku buildpacks:add -i 2 heroku/scala
```

### Deploy the eventserver

We delay deployment until the database is ready.

```bash
heroku pg:wait && git push heroku master
```


## 3. Classification Engine

### Create the engine

```bash
git clone \
  https://github.com/heroku/predictionio-engine-classification.git \
  predictionio-engine-classification

cd predictionio-engine-classification

heroku create $engine_name
heroku buildpacks:add -i 1 https://github.com/heroku/heroku-buildpack-jvm-common.git
heroku buildpacks:add -i 2 https://github.com/heroku/predictionio-buildpack.git
```

### Connect the engine with the eventserver

First, collect a few configuration values.

#### Get the eventserver's database add-on ID

```bash
heroku addons:info heroku-postgresql --app $eventserver_name
#
# Use the returned Postgres add-on ID
# to attach it to the engine.
# Example: `postgresql-aerodynamic-00000`
#
heroku addons:attach $postgres_addon_id --app $engine_name
```

#### Get an access key for this engine's data

```bash
heroku run 'pio app new classi' --app $eventserver_name
#
# Use the returned access key for `$pio_app_access_key`
#
heroku config:set \
  PIO_EVENTSERVER_HOSTNAME=$eventserver_name.herokuapp.com \
  PIO_EVENTSERVER_PORT=80 \
  PIO_EVENTSERVER_ACCESS_KEY=$pio_app_access_key \
  PIO_EVENTSERVER_APP_NAME=classi
```

### Import data

🚨 Mandatory: data is required for training to succeed and then to serve predictive queries.

* `pip install predictionio` may be required for the import script to run; see [how-to install pip](https://pip.pypa.io/en/stable/installing/)

```bash
python ./data/import_eventserver.py \
  --url https://$eventserver_name.herokuapp.com \
  --access_key $pio_app_access_key
```

### Deploy the engine

```bash
git push heroku master
#
# Follow the logs to see training 
# and then start-up of the engine.
#
heroku logs -t --app $engine_name
```


# Usage ⌨️

## Scale up

Once deployed, scale up the processes to avoid memory issues:

```bash
heroku ps:scale \
  web=1:Standard-2X \
  release=0:Performance-L \
  train=0:Performance-L \
  --app $engine_name
```


## Query for predictions

When deployed, the model was auto-trained to predict the best fitting service plan for a mobile phone user based on their voice, data and text usage.

Submit queries containing these three user properties to get a prediction for the best fitting service plan:

```bash
curl -X POST https://$engine_name.herokuapp.com/queries.json \
     -H 'Content-Type: application/json; charset=utf-8' \
     -d '{ "voice_usage":10, "data_usage":26, "text_usage":30 }'

curl -X POST https://$engine_name.herokuapp.com/queries.json \
     -H 'Content-Type: application/json; charset=utf-8' \
     -d '{ "voice_usage":58, "data_usage":26, "text_usage":300 }'
```

This model is simplified for demonstration. For a real-world model, more aspects of a user account should be taken into consideration:

  * account type: individual, business, or family
  * how frequently do they roam
  * do they roam internationally
  * kind of device:  smart phone or feature phone
  * age of device

The training dataset is crucial to the  relevancy of predictions. Expirementation with your unique data is a must.

See [usage details for this classification engine](http://predictionio.incubator.apache.org/templates/classification/quickstart/#6.-use-the-engine) in the PredictionIO docs.


## Diagnostics

If you hit any snags with the engine serving queries, check the logs:

```bash
heroku logs -t --app $engine_name
```

If errors are occuring, sometimes a restart will help:

```bash
heroku restart --app $engine_name
```


# Going Deeper 🔬

This is a demo application of PredictionIO, already customized for the smoothest experience possible.

**Custom PredictionIO engines** may be deployed with this buildpack too. See [CUSTOM documentation](CUSTOM.md).

More details including [training](CUSTOM.md#training), [evaluation](CUSTOM.md#evaluation), & [configuration](CUSTOM.md#configuration) are explained there to.

