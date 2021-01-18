# Uses case example

Back to [README.md](../README.md)

- [Description](#speech_balloon-description)
- [Deploy](#rocket-deploy)
  - [1. Kafka connect plugins](#1-kafka-connect-plugins)
  - [2. Kafka and database](#2-kafka-and-database)
  - [3. Create sink connector](#3-create-sink-connector)
  - [4. Produce data](#4-produce-data)
  - [5. Check the database](#5-check-the-database)
    - [PgAdmin container](#pgadmin-container)
    - [DBeaver](#dbeaver)
- [Troublecases](#bomb-troublecases)
  - [Sink database down](#sink-database-down)
  - [Kafka connect down](#kafka-connect-down)
- [Usefuls links](#link-usefuls-links)
  - [Resources](#resources)
  - [Others](#others)

## :speech_balloon: Description

Let's see an uses case example using environment previously deployed. We have a dedicated topic to sending sms. The goal is storing in a `PostgreSQL` database all the messages written in this kafka topic. On the advanced steps, we simulate issues from different elements.

## :rocket: Deploy

### 1. Kafka connect plugins

We need to use the plugins [`Kafka Connect JDBC`](https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc) and [`Kafka Connect JSON Schema Transformations`](https://www.confluent.io/hub/jcustenborder/kafka-connect-json-schema), make sure to have this .jar in plugins folder.

### 2. Kafka and database

The [based docker environment](../README.md) can be used. But in this example, we doesn't need a source database, so you can use the `uses-case.yml` instead.

### 3. Create sink connector

Create the [postgresql-sms-sink.json](../connectors/postgresql-sms-sink.json) connector

```bash
curl -X POST http://localhost:8083/connectors \
    -H "Content-Type: application/json" \
    --data @connectors/postgresql-sms-sink.json
```

### 4. Produce data

Run a `confluentinc/cp-schema-registry:5.5.0` container in [interactive mode](https://docs.docker.com/engine/reference/commandline/run/#assign-name-and-allocate-pseudo-tty---name--it)

```bash
docker run -it --net=host --rm --name kafka-json-producer confluentinc/cp-schema-registry:5.5.0 bash
```

Create a producer with a structured schema

```bash
kafka-json-schema-console-producer \
    --broker-list localhost:9092 \
    --topic demo.json.sms \
    --property schema.registry.url=http://localhost:8081 \
    --property value.schema='{"type":"object","properties":{"phoneNumberEmitter":{"type":"string"},"phoneNumberReceiver":{"type":"string"},"message":{"type":"string"}},"additionalProperties":false}'
```

Copy/paste the line and press `enter` to send the message

```bash
{"phoneNumberEmitter":"112233","phoneNumberReceiver":"445566","message":"Don't worry, be happy"}
```

It is easier to copy paste an entire message in a `kafka-json-schema-console-producer`. See another data examples below :

```bash
{"phoneNumberEmitter":"332211","phoneNumberReceiver":"665544","message":"The last but not least"}
{"phoneNumberEmitter":"445566","phoneNumberReceiver":"778899","message":"You shall not pass"}
{"phoneNumberEmitter":"778899","phoneNumberReceiver":"665544","message":"It's dangerous to go alone"}
```

### 5. Check the database

To explore the postgresql database, you can use a *free multi-platform database tool* like [`DBeaver`](https://dbeaver.io/download/) or continue with docker philosophy and run a [`pgadmin`](https://www.pgadmin.org/docs/pgadmin4/development/container_deployment.html) container.

#### PgAdmin container

```bash
docker run -p 8085:80 \
    --net=kafka-connect-pgsql_default \
    --name pgadmin4 \
    -e PGADMIN_DEFAULT_EMAIL=user@domain.com \
    -e PGADMIN_DEFAULT_PASSWORD=SuperSecret \
    -d \
    dpage/pgadmin4
```

- On address <http://localhost:8085/>
- Fill the login information with :
  - Email Address / Username : `user@domain.com`
  - Password : SuperSecret
- Add a new server _(Dashboard/Add New Server)_ :
  - **General tab**
    - Name : mySinkDatabase
  - **Connection tab**
    - Hostname/address : postgres-sink
    - Port : 5432
    - Maintenance database : db
    - Username : user
    - Password : password

> See also the [pgAdmin documentation](https://www.pgadmin.org/docs/pgadmin4/latest/server_dialog.html)

#### Dbeaver

In a `DBeaver` installed on host machine, the connection informations are a bit different :

- :warning: Hostname/address : localhost
- :warning: Port : 5433
- Maintenance database : db
- Username : user
- Password : password

> See also the [DBeaver documentation](https://github.com/dbeaver/dbeaver/wiki/Database-Navigator)

## :bomb: Troublecases

### Sink database down

In the case where the sink database is not operational but the messsages continue to be produced on the topic, let's see the data's behaviour on recovery.

#### 1. Stop the sink database

```bash
docker stop postgres-sink
```

#### 2. Produce new messages

> commands in [4. Produce data](#4-produce-data) step

```bash
{"phoneNumberEmitter":"123546","phoneNumberReceiver":"321654","message":"Testing is easier than debugging"}
{"phoneNumberEmitter":"654987","phoneNumberReceiver":"789456","message":"A clever person solves a problem. A wise person avoids it"}
{"phoneNumberEmitter":"112233","phoneNumberReceiver":"445566","message":"Real programmers don't comment their code. If it was hard to write, it should be hard to understand"}
{"phoneNumberEmitter":"665544","phoneNumberReceiver":"998877","message":"Talk is cheap. Show me the code"}
{"phoneNumberEmitter":"789987","phoneNumberReceiver":"456654","message":"The computer was born to solve problems that did not exist before"}
```

At this state, we can read `demo.json.sms` topic to be sure if the messages are in it.

```bash
docker run --net=host --rm \
confluentinc/cp-schema-registry:5.5.0 kafka-json-schema-console-consumer \
    --bootstrap-server localhost:9092 \
    --from-beginning \
    --topic demo.json.sms \
    --property schema.registry.url=http://localhost:8081
```

#### 3. Restart the sink database

```bash
docker start postgres-sink
```

#### 4. Restart kafka-connect task

We can see the tasks status of the connector `postgresql-sms-sink-connector`

```console
$ curl -X GET http://localhost:8083/connectors/postgresql-sms-sink-connector/status | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  3208  100  3208    0     0   3208      0  0:00:01 --:--:--  0:00:01 41128
{
  "name": "postgresql-sms-sink-connector",
  "connector": {
    "state": "RUNNING",
    "worker_id": "localhost:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "FAILED",
      "worker_id": "localhost:8083",
      "trace": "org.apache.kafka.connect.errors.ConnectException: Exiting WorkerSinkTask due to unrecoverable exception.[...]
    }
  ],
  "type": "sink"
}
```

The task `0` is on a [`FAILED`](https://docs.confluent.io/home/connect/monitoring.html#connector-and-task-status) state, we have to restart it to send all message stayed in the topic

```bash
curl -X POST http://localhost:8083/connectors/postgresql-sms-sink-connector/tasks/0/restart
```

```console
$ curl -X GET http://localhost:8083/connectors/postgresql-sms-sink-connector/status | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   181  100   181    0     0    181      0  0:00:01 --:--:--  0:00:01  2873
{
  "name": "postgresql-sms-sink-connector",
  "connector": {
    "state": "RUNNING",
    "worker_id": "localhost:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "localhost:8083"
    }
  ],
  "type": "sink"
}
```

Let's check it out the database to ensure all messages are there.

### kafka-connect down

Same kind of issue but with Kafka connect.

#### 1. Stop the `confluent-connect`

```bash
docker stop confluent-connect
```

#### 2. Produce newest messages

> commands in [4. Produce data](#4-produce-data) step

```bash
{"phoneNumberEmitter":"123546","phoneNumberReceiver":"321654","message":"Il est bon ou quoi?"}
{"phoneNumberEmitter":"654987","phoneNumberReceiver":"789456","message":"Net"}
{"phoneNumberEmitter":"112233","phoneNumberReceiver":"445566","message":"Sakébon"}
{"phoneNumberEmitter":"665544","phoneNumberReceiver":"998877","message":"Kalolo alors"}
{"phoneNumberEmitter":"789987","phoneNumberReceiver":"456654","message":"Ok tal"}
```

#### 3. Restart the `confluent-connect`

```bash
docker start confluent-connect
```

Let's check it out the database to ensure all messages are in the sink database.

## :link: Usefuls links

### Resources

- [JSON Schema Serializer and Deserializer](https://docs.confluent.io/5.5.0/schema-registry/serdes-develop/serdes-json.html)
- [Monitoring Kafka Connect and Connectors](https://docs.confluent.io/platform/current/connect/monitoring.html)

### Others

- [pgAdmin - PostgreSQL Tools](https://www.pgadmin.org/)
  - [dpage/pgadmin4](https://hub.docker.com/r/dpage/pgadmin4)
  - [pgAdmin 4 (Container)](https://www.pgadmin.org/download/pgadmin-4-container/)

Back to [README.md](../README.md)
