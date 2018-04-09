---
title: "Streaming Applications"
date: 2017-11-17T09:46:54Z
tags: [ "Streaming", "Kafka", "Docker", "MQTT" ]
---

## What's the plan?

The plan is to define a distributed architecture to get streams and make some sense out of it.

We will create:

| Component | Role |
|---|---|
| Producer | Produce streams of (potentially heterogeneous) data |
| API Gateway | Is the entry point to the collection of services |
| Stream Processor | Manages the incoming stream, transforms and allow services to consume |
| Data warehouse | OLAP for streams |
| Web app | A web app, exposing some of the data to a portal |

Ok let's start Easy Peasy as it's a vast program here: Producer.

### Producer

Its role is to produce messages, which are potentially of an heterogeneous nature.

There are multiple ways of producing messages that we could be interested in:

* Publishing them to a REST API
* Publishing them to an MQTT broker

### API gateway

This is the entry point for communications with the "outer world".
It's an interface to the devices (stream producers as seen above), but also to any client (e.g. web portals or mobile apps).

### Stream processor

This is the core of the system, ingesting a high amount of messages, eventually transforming them and make them available for consumers to use.
It is the source of truth. All records would eventually end up here. 

### Data warehouse

The data warehouse should be filled continuously rather than by [ETL](https://en.wikipedia.org/wiki/Extract,_transform,_load) but still accept it when needed.
It would then feed analytics-centric applications such as [Tableau](https://www.tableau.com/).

### Web apps

Web applications which consume ingested data. They could be anything. 

Example: A dashboard presenting few information about devices status on a web page or mobile app.

## Producer

Simple producer could be as simple as:

```console
$ while true; do task; sleep 2; done >> allout.txt 2>&1 &
```
As it runs in the background, we can have as many as we want, example:

```console
$ while true; do; echo "google"; sleep 1; done >> allout.txt 2>&1 &
[1] 23113
$ while true; do; echo "facebook"; sleep 2; done >> allout.txt 2>&1 &
[2] 23131
$ tail -f allout.txt 
google
google
facebook
google
google
facebook
google
google
facebook
google
^C
```

## API gateway

We will make use of [Kong](https://konghq.com/) here and run it as [docker containers](https://getkong.org/install/docker/) along with Cassandra.

> Notice there's also a [`docker-compose` template](https://github.com/Mashape/docker-kong/tree/master/compose).

For now, just follow the instructions here: https://getkong.org/install/docker/

Run the database:
```console
$ docker run -d --name kong-database \
              -p 9042:9042 \
              cassandra:3
```

Run migrations on Cassandra here (takes some time, let it run):
```console
$ docker run --rm \
    --link kong-database:kong-database \
    -e "KONG_DATABASE=cassandra" \
    -e "KONG_PG_HOST=kong-database" \
    -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
    kong:latest kong migrations up
```

Finally run kong 
```console
$ docker run -d --name kong \
    --link kong-database:kong-database \
    -e "KONG_DATABASE=cassandra" \
    -e "KONG_PG_HOST=kong-database" \
    -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
    -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
    -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
    -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
    -p 8000:8000 \
    -p 8443:8443 \
    -p 8001:8001 \
    -p 8444:8444 \
    kong:latest
```

And test it out:
```console
$ curl -i http://localhost:8001/
HTTP/1.1 200 OK
Date: Sat, 18 Nov 2017 14:06:02 GMT
Content-Type: application/json; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/0.11.1
...
```
Cool, but that's just a gateway.

Let's add a component, which has one endpoint `/ping` and:

* Responds with `pong` on `GET` requests
* Responds with whatever you've sent on `POST` requests

> Get this component on github at https://github.com/iyp-uk/ping-api

> If your data is not sensitive you can also use https://httpbin.org/ instead.

We will then add it to Kong.

> As kong runs in containers, and our API runs on localhost too (or in its container), [localhost within Kong's context doesn't make sense](https://github.com/Kong/kong/issues/2149).
We will then pass an IP instead. If running in docker, inspect the container and get its `IPAddress`.

```console
$ PING_API_HOST=172.17.0.5
$ curl -i -X POST \
  --url http://localhost:8001/apis/ \
  --data 'name=ping-api' \
  --data 'hosts=ping-api.com' \
  --data 'upstream_url=http://'"$PING_API_HOST"':3000'
```
Test it out!
```console
$ curl -i http://localhost:8000/ping -H 'Host: ping-api.com'
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 4
Connection: keep-alive
X-Powered-By: Express
ETag: W/"4-DlFKBmK8tp3IY5U9HOJuPUDoGoc"
Date: Sat, 18 Nov 2017 16:47:43 GMT
X-Kong-Upstream-Latency: 5
X-Kong-Proxy-Latency: 11
Via: kong/0.11.1

pong
```
Yes, it goes through Kong and returns `pong`, which is what we expect.

Let's POST some stuff now:
```console
$ curl -i -X POST \
  --data 'name=ping-api' \
  --url http://localhost:8000/ping \
  --header 'Host: ping-api.com'
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 19
Connection: keep-alive
X-Powered-By: Express
ETag: W/"13-M5ymCJDs5nnA1Q5gUa3RN1s9/uc"
Date: Sat, 18 Nov 2017 16:48:44 GMT
X-Kong-Upstream-Latency: 3
X-Kong-Proxy-Latency: 0
Via: kong/0.11.1

{"name":"ping-api"}
```

### See how that scales

Let's ask the producer to send many requests to this gateway.

```console
$ facebook () { curl -i -X POST \
    --data 'date='"$(date)"'' \
    --data 'name=facebook' \
    --url http://localhost:8000/ping \
    --header 'Host: ping-api.com' \
    }
$ google () { curl -i -X POST \
    --data 'date='"$(date)"'' \
    --data 'name=google' \
    --url http://localhost:8000/ping \
    --header 'Host: ping-api.com' \
    }
$ while true; do; google; sleep 1; done >> allout.txt 2>&1 &
$ while true; do; facebook; sleep 2; done >> allout.txt 2>&1 &
$ tail -f allout.txt
^C
$ kill %1 %2
```
OK, it can handle it, we still have out little sequence of facebooks and googles.

Faster? Let's ask to have 100.000 messages sent per second.

> This might not be super relevant. 
Even though they are in containers, it is still on the same machine, so not sure about results.

We will then use [apache bench](https://httpd.apache.org/docs/2.4/programs/ab.html) for that.

Ask it to send 500.000 requests within 5 seconds. No concurrency.

Let's start with querying the API straight:
```console
$ echo '{"some":"data"}' > payload.txt
$ ab -p payload.txt -T application/json -n 500000 -t 5 http://localhost:3000/ping
This is ApacheBench, Version 2.3 <$Revision: 1796539 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)
Finished 1720 requests


Server Software:        
Server Hostname:        localhost
Server Port:            3000

Document Path:          /ping
Document Length:        15 bytes

Concurrency Level:      1
Time taken for tests:   5.002 seconds
Complete requests:      1720
Failed requests:        0
Total transferred:      380120 bytes
Total body sent:        266600
HTML transferred:       25800 bytes
Requests per second:    343.84 [#/sec] (mean)
Time per request:       2.908 [ms] (mean)
Time per request:       2.908 [ms] (mean, across all concurrent requests)
Transfer rate:          74.21 [Kbytes/sec] received
                        52.05 kb/s sent
                        126.25 kb/s total

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.1      0       3
Processing:     1    3   1.3      2      20
Waiting:        1    2   1.2      2      19
Total:          2    3   1.3      3      20

Percentage of the requests served within a certain time (ms)
  50%      3
  66%      3
  75%      3
  80%      3
  90%      3
  95%      4
  98%      7
  99%     10
 100%     20 (longest request)
```
99% of requests are served within 10ms. But we're far from our objective of 100.000 req/sec.

Now let's query through Kong. 
Don't forget to pass the `Host` header.

```console
$ ab -p payload.txt -T application/json -H 'Host: ping-api.com' -n 500000 -t 5 http://localhost:8000/ping
This is ApacheBench, Version 2.3 <$Revision: 1796539 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)
Finished 1442 requests


Server Software:        
Server Hostname:        localhost
Server Port:            8000

Document Path:          /ping
Document Length:        15 bytes

Concurrency Level:      1
Time taken for tests:   5.001 seconds
Complete requests:      1442
Failed requests:        0
Total transferred:      421071 bytes
Total body sent:        220626
HTML transferred:       21630 bytes
Requests per second:    288.35 [#/sec] (mean)
Time per request:       3.468 [ms] (mean)
Time per request:       3.468 [ms] (mean, across all concurrent requests)
Transfer rate:          82.23 [Kbytes/sec] received
                        43.08 kb/s sent
                        125.31 kb/s total

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.1      0       2
Processing:     2    3   2.4      2      37
Waiting:        2    3   2.4      2      37
Total:          2    3   2.4      3      37

Percentage of the requests served within a certain time (ms)
  50%      3
  66%      3
  75%      4
  80%      4
  90%      5
  95%      8
  98%     10
  99%     13
 100%     37 (longest request)
```
OK, it's slower but still acceptable (no failed request).

## Steam processor

Let's now add a message broker to the mix, so that applications upsteam can make the most of this data being sent.

### Kafka

Let's [get started with Kafka](http://kafka.apache.org/quickstart) and set up a topic.

> You may need different console tabs for that, or make them run in the background.

Start the server along with zookeeper:
```console
$ bin/zookeeper-server-start.sh config/zookeeper.properties 
$ bin/kafka-server-start.sh config/server.properties
```

Add a `topic` (we will call it `ping`):
```console
$ bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic ping
Created topic "ping".
```

Send and read some messages with the `kafka-console-producer` and `kafka-console-consumer`, just to see the topic in action:
```console
$ bin/kafka-console-producer.sh --broker-list localhost:9092 --topic ping
>blah
>sushi
>more
>
```
In another console:
```console
$ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic ping --from-beginning
blah
sushi
more
```

Ok that works. However, this is not quite what we need as data is coming to us through rest, no supplied in the console.

The are multiple ways of supplying data into Kafka:

* [Kafka clients](https://cwiki.apache.org/confluence/display/KAFKA/Clients)
    * For when you have access to the application supplying data
    * Allows Kafka read / write operations from your application
* [Kafka connect](http://kafka.apache.org/documentation/#connect)
    * For when you want to connect to existing datastores, filled by other applications on which you have no control
    * To connect to other datastores (see the [example with the filesystem in the quickstart guide](http://kafka.apache.org/quickstart#quickstart_kafkaconnect))
    * See [some available connectors](https://www.confluent.io/product/connectors/)

In this case, 

* there is no other data store
* we have access to the application

We will then use a Kafka client, [`node-rdkafka`](https://github.com/Blizzard/node-rdkafka) to be more exact.

Let's redo our `ab` test here:

```console
$ ab -p payload.txt -T application/json  -n 500000 -t 5 -c 2 http://localhost:3000/ping
This is ApacheBench, Version 2.3 <$Revision: 1796539 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)
Finished 995 requests


Server Software:        
Server Hostname:        localhost
Server Port:            3000

Document Path:          /ping
Document Length:        15 bytes

Concurrency Level:      2
Time taken for tests:   5.000 seconds
Complete requests:      995
Failed requests:        0
Total transferred:      219895 bytes
Total body sent:        154380
HTML transferred:       14925 bytes
Requests per second:    199.00 [#/sec] (mean)
Time per request:       10.050 [ms] (mean)
Time per request:       5.025 [ms] (mean, across all concurrent requests)
Transfer rate:          42.95 [Kbytes/sec] received
                        30.15 kb/s sent
                        73.10 kb/s total

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.2      0       3
Processing:     2   10   4.7      9      40
Waiting:        2    9   4.3      8      35
Total:          3   10   4.7      9      40

Percentage of the requests served within a certain time (ms)
  50%      9
  66%     11
  75%     12
  80%     13
  90%     16
  95%     20
  98%     23
  99%     29
 100%     40 (longest request)
```
We're now down to 200 req/sec with a single producer and standalone kafka. It is not representative of real-file set ups. 

To be continued...