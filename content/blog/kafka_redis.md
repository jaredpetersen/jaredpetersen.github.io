---
title: Sinking and Sourcing Redis Data With Kafka Connect Redis
date: 2020-11-11T00:00:00Z
description: "Tutorial for connecting Redis to Kafka with kafka connect redis"
---

[jaredpetersen/kafka-connect-redis](https://github.com/jaredpetersen/kafka-connect-redis) is a new open source, MIT-licensed Kafka Connect plugin that can integrate Redis into your Kafka data pipelines.

Need data from Redis streamed to another application in realtime via Kafka? You’re covered with the source connector, enabling you to create records based on [pub/sub messages](https://redis.io/topics/pubsub) or [keyspace notifications](https://redis.io/topics/notifications).

Need to write some data into Redis via Kafka messages? Great! Just use the sink connector and produce messages that describe the write operation you’d like to perform (`SET`, `SADD`, etc.).

I've put together a nifty demonstration to show off the connector’s capabilities. All you need is minikube so that we can set up the Redis and the various Kafka components without installing them directly on your computer.

Let’s get started!

## Basic Setup
As mentioned earlier, we’re going to use minkube to help us run all of the different components in Kubernetes.

Once you have that installed, set it up:
```sh
minikube start --cpus 2 --memory 10g
```

Nice! Nothing’s running in it so far so let’s fix that. Download the [jaredpetersen/kafka-connect-redis](https://github.com/jaredpetersen/kafka-connect-redis) repository and navigate to docs/demo/docker/ in your terminal. From here, we’re going to build a Docker image that contains our Kafka connector.

First, download the latest version of Kafka Connect Redis. At the moment, that’s version 1.0.1:
```sh
curl -O https://oss.sonatype.org/service/local/repositories/releases/content/io/github/jaredpetersen/kafka-connect-redis/1.0.1/kafka-connect-redis-1.0.1.jar
```

Next, we’ll want to set up our terminal environment so that we can build a Docker image that can be used inside minikube:
```sh
eval $(minikube docker-env)
```

Then build the image:
```sh
docker build -t jaredpetersen/kafka-connect-redis:latest .
```

Now that we have a Docker image for the connector inside our minikube instance, close out this terminal. The previous `eval` command pollutes your terminal environment with minikube-specific configuration and we don’t want that to affect any future commands.

Open up a new terminal and navigate back to docs/demo in the downloaded Kafka Connect Redis repository. Now we can start installing all those awesome Kafka and Redis components.

Run the following command to apply the Kubernetes manifests:
```sh
kubectl apply -k kubernetes
```

Monitor the rollout progress. This can take a few minutes, so be patient!
```sh
kubectl -n kcr-demo get pods
```

Once all the pods are up and ready to go, we need to do one last thing to configure the applications running in Kubernetes; we need to tell Redis to run as a cluster. To accomplish that, we’re just going to create a temporary pod that runs a redis-cli command:
```sh
kubectl -n kcr-demo run -it --rm redis-client --image redis:6 -- redis-cli --pass IEPfIr0eLF7UsfwrIlzy80yUaBG258j9 --cluster create $(kubectl -n kcr-demo get pods -l app=redis-cluster -o jsonpath='{range.items[*]}{.status.podIP}:6379 ') --cluster-yes
```

You’re well on your way to trying out the connector! It’s a bit of setup for sure but it’s well worth the effort.

## Sink Connector
Let’s start sinking data from Kafka into Redis!

Unlike other Redis sink connectors, [jaredpetersen/kafka-connect-redis](https://github.com/jaredpetersen/kafka-connect-redis) supports more than just Redis `SET` commands. Several different commands are supported at this time (`SET`, `SADD`, `GEOADD`, etc.) — check out the repository for the full list.

The first thing we need to do is make a POST request against the Kafka Connect REST API running in minikube to tell it to create a new instance of the Redis connector. In this demonstration, we’re going to be using Connect JSON as the Kafka record serialization format to make it a little easier to use. Avro is supported as well and the steps to use Avro instead are documented in the repository.

```sh
curl --request POST \
    --url "$(minikube -n kcr-demo service kafka-connect --url)/connectors" \
    --header 'content-type: application/json' \
    --data '{
        "name": "demo-redis-sink-connector",
        "config": {
            "connector.class": "io.github.jaredpetersen.kafkaconnectredis.sink.RedisSinkConnector",
            "key.converter": "org.apache.kafka.connect.json.JsonConverter",
            "value.converter": "org.apache.kafka.connect.json.JsonConverter",
            "tasks.max": "1",
            "topics": "redis.commands",
            "redis.uri": "redis://IEPfIr0eLF7UsfwrIlzy80yUaBG258j9@redis-cluster",
            "redis.cluster.enabled": true
        }
    }'
```

Now that our connector is up and running, we can write some records to Kafka and watch the data appear in Redis!

Set up an interactive, ephemeral Kubernetes Pod with the Kafka command line tools already installed:

```sh
kubectl -n kcr-demo run -it --rm kafka-write-records --image confluentinc/cp-kafka:6.0.0 --command /bin/bash
```

Start up the console producer and configure it to write records to the `redis.commands` topic:

```sh
kafka-console-producer \
    --broker-list kafka-broker-0.kafka-broker:9092 \
    --topic redis.commands
```

This will bring up an interactive command-line application that allows you to write records to Kafka. Each newline represents a new record. Since we’re using the Connect JSON format, we need to include our schema and the actual data (called the payload) in one JSON object.

Write the following records:

**Redis SET command**
```
{"payload":{"key":"{user.1}.username","value":"jetpackmelon22"},"schema":{"name":"io.github.jaredpetersen.kafkaconnectredis.RedisSetCommand","type":"struct","fields":[{"field":"key","type":"string","optional":false},{"field":"value","type":"string","optional":false},{"field":"expiration","type":"struct","fields":[{"field":"type","type":"string","optional":false},{"field":"time","type":"int64","optional":true}],"optional":true},{"field":"condition","type":"string","optional":true}]}}
```

**Redis SET command with expiration and an insert condition**
```
{"payload":{"key":"{user.2}.username","value":"anchorgoat74","expiration":{"type":"EX","time":2100},"condition":"NX"},"schema":{"name":"io.github.jaredpetersen.kafkaconnectredis.RedisSetCommand","type":"struct","fields":[{"field":"key","type":"string","optional":false},{"field":"value","type":"string","optional":false},{"field":"expiration","type":"struct","fields":[{"field":"type","type":"string","optional":false},{"field":"time","type":"int64","optional":true}],"optional":true},{"field":"condition","type":"string","optional":true}]}}
```

**Redis SADD command**
```
{"payload":{"key":"{user.1}.interests","values":["reading"]},"schema":{"name":"io.github.jaredpetersen.kafkaconnectredis.RedisSaddCommand","type":"struct","fields":[{"field":"key","type":"string","optional":false},{"field":"values","type":"array","items":{"type":"string"},"optional":false}]}}
```

**Redis SADD command with multiple members**
```
{"payload":{"key":"{user.2}.interests","values":["sailing","woodworking","programming"]},"schema":{"name":"io.github.jaredpetersen.kafkaconnectredis.RedisSaddCommand","type":"struct","fields":[{"field":"key","type":"string","optional":false},{"field":"values","type":"array","items":{"type":"string"},"optional":false}]}}
```

**Redis GEOADD command with multiple members**
```
{"payload":{"key":"Sicily","values":[{"longitude":13.361389,"latitude":13.361389,"member":"Palermo"},{"longitude":15.087269,"latitude":37.502669,"member":"Catania"}]},"schema":{"name":"io.github.jaredpetersen.kafkaconnectredis.RedisGeoaddCommand","type":"struct","fields":[{"field":"key","type":"string","optional":false},{"field":"values","type":"array","items":{"type":"struct","fields":[{"field":"longitude","type":"double","optional":false},{"field":"latitude","type":"double","optional":false},{"field":"member","type":"string","optional":false}]},"optional":false}]}}
```

Okay, let’s get to the big payoff!

In a new terminal, run the following command to create an interactive, ephemeral Kubernetes pod with `redis-client` installed so that we can run commands against Redis and check our work.

```sh
kubectl -n kcr-demo run -it --rm redis-client --image redis:6 -- /bin/bash
```

Use `redis-cli` in the pod to connect to the Redis cluster:

```sh
redis-cli -c -u 'redis://IEPfIr0eLF7UsfwrIlzy80yUaBG258j9@redis-cluster'
```

And run the following Redis commands to confirm that our connector worked:

```
GET {user.1}.username
GET {user.2}.username
SMEMBERS {user.1}.interests
SMEMBERS {user.2}.interests
GEOPOS Sicily Catania
```

Congratulations! You just successfully used [jaredpetersen/kafka-connect-redis](https://github.com/jaredpetersen/kafka-connect-redis) to sink data into Redis via Kafka Connect!

## Source Connector
TODO
