# presentation-kafka-connect

## :speech_balloon: Description

Kafka Connect Demo

The purpose of this repository is to :

- [ ] provide a slideshow that explains how Kafka Connect can be used, its principles and guidelines to set it up to connect Kafka and Psql
- [ ] A set of script based on Docker images and shell scripts to let people run the demo by themselves
- [ ] Show some use cases

## :books: Prerequisites

- Download [git](https://git-scm.com/downloads)
- Get started with [docker](https://www.docker.com/get-started)
- Install [docker-compose](https://docs.docker.com/compose/install/)
- `curl` command available

## :rocket: How to use

### Confluent Kafka-Connect

> See also [Kafka Connect Tutorial on Docker](https://docs.confluent.io/5.0.0/installation/docker/docs/installation/connect-avro-jdbc.html)

In this way to use `kafka-connect`, we will deploy the entire stack on `docker` environnement. Here we use the `Confluent` elements :

- [`schema-registry`](https://docs.confluent.io/platform/current/schema-registry/index.html) - *"It provides a RESTful interface for storing and retrieving your Avro, JSON Schema, and Protobuf schemas."*
- [`kafka-connect`](https://docs.confluent.io/platform/current/connect/index.html) - *"Kafka Connect is a tool for scalably and reliably streaming data between Apache Kafka and other data systems."*

#### Kafka Connect plugins

`kafka-connect` need plugins to interact with databases, download them and extract the content in the `plugins` folder :

- [Kafka Connect JDBC](https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc) - *"JDBC source and sink connectors"*

- [Debezium PostgreSQL CDC Connector](https://www.confluent.io/hub/debezium/debezium-connector-postgresql) - *"Debezium’s PostgreSQL Connector can monitor and record the row-level changes in the schemas of a PostgreSQL database"*

#### Deployment

Now, we can launch the environment :

```console
$ docker-compose --project-name kafka-connect-pgsql -f kafka-connect.yml up -d
Creating postgres-sink   ... done
Creating postgres-source ... done
Creating zookeeper       ... done
Creating kafka           ... done
Creating confluent-registry ... done
Creating confluent-connect  ... done
```

Check if `confluent-connect` container is up and ready :

```console
$ docker logs confluent-connect | grep started
...
[2021-01-08 00:32:41,837] INFO REST resources initialized; server is started and ready to handle requests (org.apache.kafka.connect.runtime.rest.RestServer)
[2021-01-08 00:32:41,837] INFO Kafka Connect started (org.apache.kafka.connect.runtime.Connect)
```

#### Connectors creations

For create connectors to extract/inject datas between Kafka and external databases, we utilize curl commands trough the api `kafka-connect` endpoints :

[`source`](https://docs.confluent.io/kafka-connect-jdbc/current/source-connector/index.html) - *Database to Kafka*

```bash
curl -X POST -H "Content-Type: application/json" --data @connectors/postgresql-source.json http://localhost:8083/connectors
```

[`sink`](https://docs.confluent.io/kafka-connect-jdbc/current/sink-connector/index.html) - *Kafka to Database*

```bash
curl -X POST -H "Content-Type: application/json" --data @connectors/postgresql-sink.json http://localhost:8083/connectors
```

> See more endpoints on [Connect REST Interface](https://docs.confluent.io/platform/current/connect/references/restapi.html) documentation

See [Explanation](#explanations) section for more details

## :gear: Others Commands

### Docker

#### Teardown & uninstall

```console
$ docker-compose --project-name kafka-connect-pgsql -f kafka-connect.yml stop
Stopping confluent-connect  ... done
Stopping confluent-registry ... done
Stopping kafka              ... done
Stopping postgres-source    ... done
Stopping zookeeper          ... done
Stopping postgres-sink      ... done
```

```console
$ docker-compose --project-name kafka-connect-pgsql -f kafka-connect.yml rm
Going to remove postgres, connect, kafka, zookeeper
Are you sure? [yN] y
Removing confluent-connect  ... done
Removing confluent-registry ... done
Removing kafka              ... done
Removing postgres-source    ... done
Removing zookeeper          ... done
Removing postgres-sink      ... done
```

### Kafka

#### List topics

```bash
docker run --net=host --rm
    wurstmeister/kafka:2.11-2.0.0 sh opt/kafka_2.11-2.0.0/bin/kafka-topics.sh \
        --zookeeper localhost:2181 \
        --list
```

#### Read topic

```bash
docker run --net=host --rm
    wurstmeister/kafka:2.11-2.0.0 sh opt/kafka_2.11-2.0.0/bin/kafka-console-consumer.sh \
        --bootstrap-server localhost:9092 \
        --topic hrdata.public.employees \
        --timeout-ms 3000 \
        --from-beginning
```

### Read topic (Avro)

```bash
docker run --net=host --rm \
    confluentinc/cp-schema-registry:5.0.0 kafka-avro-console-consumer \
        --bootstrap-server localhost:9092 \
        --topic hrdata.public.employees \
        --timeout-ms 3000 \
        --from-beginning \
```

## :information_source: Explanations

With this above instructions, we tranfer datas based on events : inserts and updates between two databases via kafka topics asynchronously. Let's take a deep look on differents steps.

:construction: *Work in progress* :construction:

## :link: Usefuls links

### Resources

#### Confluent

- [Kafka Connect Tutorial on Docker](https://docs.confluent.io/5.0.0/installation/docker/docs/installation/connect-avro-jdbc.html)
- [Schema Management Overview](https://docs.confluent.io/platform/current/schema-registry/index.html)
- [Kafka Connect](https://docs.confluent.io/platform/current/connect/index.html)
- [JDBC Source Connector for Confluent Platform](https://docs.confluent.io/kafka-connect-jdbc/current/source-connector/index.html)
- [JDBC Sink Connector for Confluent Platform](https://docs.confluent.io/kafka-connect-jdbc/current/sink-connector/index.html)
- [Discover Kafka connectors and more](https://www.confluent.io/hub)
  - [Kafka Connect JDBC](https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc)
  - [Debezium PostgreSQL CDC Connector](https://www.confluent.io/hub/debezium/debezium-connector-postgresql)
- [Connect REST Interface](https://docs.confluent.io/platform/current/connect/references/restapi.html)

#### Debezium

- [Debezium - Stream changes from your database](https://debezium.io/)
- [Tutorial](https://debezium.io/documentation/reference/1.3/tutorial.html)
- [Avro Serialization](https://debezium.io/documentation/reference/1.3/configuration/avro.html)
- [Debezium connector for PostgreSQL](https://debezium.io/documentation/reference/1.3/connectors/postgresql.html#postgresql-connector-properties)

### Others

- [Error: Value (STRUCT) type doesn't have a mapping to the SQL database column type](https://github.com/confluentinc/kafka-connect-jdbc/issues/939)
- [How to change postgres docker image wal level on setup?](https://stackoverflow.com/questions/59416301/how-to-change-postgres-docker-image-wal-level-on-setup)