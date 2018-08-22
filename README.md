bottled-water
===

This repo serves as an example of how you can run Kafka Connect or KSQL on Heroku.
Specifically, it demonstrates how fundamental Kafka components integrate well with the
Heroku Runtime.

Kafka Connect and KSQL, though not yet officially supported by Heroku, runs well on Heroku 
and works under both single- and multi-tenant Kafka plans.

## Architecture

This infrastructure effectively leverages the Heroku Runtime as a horizontally-scalable
processing layer. That is, we can leverage Kafka's consumer group functionality alongside
Heroku's Dynos to achieve scale-in/scale-out use cases for Kafka Connect and KSQL. That way,
worker instances are rebalanced transparently on our behalf as the end user.

### Kafka Connect

This runs on dynos as a distributed system. You can scale dynos up or down depending on how many
instances of Kafka Connect you want. Rebalancing is done behind the scenes. Notice in the `Procfile`
that these instances are started as a `web` dyno. That is, they expose port 80 or 443. This is because
Kafka Connect in distributed mode exposes a REST interface for mangaging Kafka Connectors for end-users
to interact with.

### KSQL

This runs on dynos as a distributed system. You can also scale dynos up or down. Rebalancing is taken 
care of here as well. KSQL also exposes a web port. This is because the `ksql` cli is essentially a
read-eval-print-loop that communicates to the KSQL server(s), which communicate to Kafka brokers on
your behalf. Essentially, you submit a KSQL query via REST and get a response. This response can be
a one-time/ad-hoc response or a streaming response, depending on your query.

## How it works

This repo has an [Aptfile](https://github.com/jeffchao/bottled-water/blob/master/Aptfile).
This file requires that your app uses the [Heroku Apt Buildpack](https://elements.heroku.com/buildpacks/heroku/heroku-buildpack-apt).
This `Aptfile` tells each dyno, that when it boots, to download the Kafka Connect and KSQL dependencies.
These dependencies contain libraries and binaries necessary to run Kafka Connect and KSQL binaries.

These bianries eventually run `java`, so this repo also requires the Heroku JVM Buildpack.

This repo has a [Procfile](https://github.com/jeffchao/bottled-water/blob/master/Procfile) which tells
a dyno what to do when it runs. This `Procfile` has commands on how to start a Kafka Connect cluster
in distributed mode and/or a KSQL server.

This repo has three scripts:

1. `setup_certs`: Takes config vars as input and creates required keystores. This is required because
Apache Kafka on Heroku strictly requires SSL.
2. `start-distributed`: This uses `setup_certs` to create keystores, sets up config properties, downloads any
additional Kafka Connect dependencies, and starts the Kafka Connect binary.
3. `start-ksql`: This uses `setup_certs` to create keystores, sets up config properties, downloads
additional KSQL dependencies, and starts the KSQL binary.

All scripts are run on the dyno in Heroku.

Notice this repo doesn't contain any code. This is because no code is necessary. Instead, all we have
is configuration in the form of [demo-connector.json](https://github.com/jeffchao/bottled-water/blob/master/demo-connector.json) 
which is used for Kafka Connect to specify Debezium and Connect properties. All that's required of us
as the end user is to fill in this config file. More importantly, this file does not even need to be
checked in, but is present in this repo as an example. This config file can be stored anywhere since
it is eventually send over as a request body to Kafka Connect.

## Setup

Create app, add buildpacks, add Kafka add-on, modify configs, deploy to Heroku.

```sh
$ heroku apps:create my-app

$ heroku buildpacks:add --index 1 https://github.com/heroku/heroku-buildpack-apt
$ heroku buildpacks:add heroku/jvm

# Add a credit card to your account so you can create addons.

$ heroku addons:create heroku-kafka:basic-0 --app my-app

$ git push heroku master

# Comment out or un-comment desired `web` dyno in the `Procfile` (or just deploy to separate apps)
# and scale appropriate dynos.
$ heroku ps:scale web=1
```

### Kafka Connect

Interact with Kafka Connect via [REST API](https://docs.confluent.io/current/connect/references/restapi.html).

Example:

```sh
$ curl -X POST -H "Content-Type: application/json" https://my-app.herokuapp.com/connectors
```

To deploy a connector, use the `demo-connetor.json` as the body and `POST` it to your app at the `/connectors` endpoint.

### KSQL

Interact with KSQL via the [ksql cli](https://docs.confluent.io/current/ksql/docs/installation/cli-config.html).

Example:

```sh
$ ksql https://my-app.herokuapp.com
```

## Caveats

1. Kafka Connect and KSQL uses a variety of internal topics consumer groups. Heroku's
multitenant Kafka plans leverage Kafka ACLs. This means that you will have to
pre-create internal topics and consumer groups in order for this to work on multitenant plans.
2. The Heroku Runtime has an ephemeral file system. This means that leveraging RocksDB is yet not possible.
Fortunately, RocksDB is not a hard requirement because log-compacted Kafka topcis are the source of truth.
However, the trade-off is that this would result in longer boot-up times due to worker instances replaying data.
