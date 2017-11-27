---
title: "Kong Authentication"
date: 2017-11-27T10:07:25Z
tags: [ "kong", "microservices", "api" ]
---

## Run kong

We will run kong in docker here, just for ease of use.

Run the database:
```console
$ docker run -d --name kong-database \
              -p 9042:9042 \
              cassandra:3
```

Run migrations on Cassandra here (takes some time, let it run, also it can take some time for the `kong-database` container to become available):
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

Or if you want to read it more comfortably:
```console
$ curl  http://localhost:8001/ | python -m json.tool
## Returns a pretty formatted json
```

## Add an API

We will use httpbin here on their `/headers` endpoint to see which headers are returned.

Test it out!
```console
$ curl -i -H 'custom: header' https://httpbin.org/headers
HTTP/1.1 200 OK
Connection: keep-alive
Server: meinheld/0.6.1
Date: Mon, 27 Nov 2017 10:30:08 GMT
Content-Type: application/json
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
X-Powered-By: Flask
X-Processed-Time: 0.00105690956116
Content-Length: 158
Via: 1.1 vegur

{
  "headers": {
    "Accept": "*/*", 
    "Connection": "close", 
    "Custom": "header", 
    "Host": "httpbin.org", 
    "User-Agent": "curl/7.54.0"
  }
}
```

Add the api:
```console
$ curl -i -X POST \
  --url http://localhost:8001/apis/ \
  --data 'name=example-api' \
  --data 'hosts=example.com' \
  --data 'upstream_url=https://httpbin.org'

HTTP/1.1 201 Created
...
```

Test it out!
```console
$ curl -i -X GET \
  --url http://localhost:8000/headers \
  --header 'Host: example.com'

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 173
Connection: keep-alive
Server: meinheld/0.6.1
Date: Mon, 27 Nov 2017 10:33:11 GMT
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
X-Powered-By: Flask
X-Processed-Time: 0.0010461807251
Via: kong/0.11.1
X-Kong-Upstream-Latency: 360
X-Kong-Proxy-Latency: 0

{
  "headers": {
    "Accept": "*/*", 
    "Connection": "close", 
    "Host": "httpbin.org", 
    "User-Agent": "curl/7.54.0", 
    "X-Forwarded-Host": "example.com"
  }
}
```

## Enable the authentication plugin

Kong [comes with an authentication plugin](https://getkong.org/plugins/key-authentication/) that you can easily enable.

```console
$ curl -i -X POST \
  --url http://localhost:8001/apis/example-api/plugins/ \
  --data 'name=key-auth'

HTTP/1.1 201 Created
...
```

Test it out! Querying the same endpoint as before now returns a `401 Unauthorized`. 
```console
$ curl -i -X GET \
  --url http://localhost:8000/headers \
  --header 'Host: example.com'

HTTP/1.1 401 Unauthorized
```

We now need to add consumers which will be granted an API key.

## Add consumers

```
$ curl -i -X POST \
  --url http://localhost:8001/consumers/ \
  --data "username=Jason"

HTTP/1.1 201 Created
```

Add a key for this consumer:
```console
$ curl -i -X POST \
  --url http://localhost:8001/consumers/Jason/key-auth/ \
  --data 'key=SECRET'

HTTP/1.1 201 Created
```

Now query again the same endpoint, but now passing the API key for this user (passing the API key in URL):
```console
$ curl -i -X GET \
  --url http://localhost:8000/headers\?apikey\=SECRET \
  --header "Host: example.com"  

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 272
Connection: keep-alive
Server: meinheld/0.6.1
Date: Mon, 27 Nov 2017 10:43:30 GMT
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
X-Powered-By: Flask
X-Processed-Time: 0.00125885009766
Via: kong/0.11.1
X-Kong-Upstream-Latency: 352
X-Kong-Proxy-Latency: 153

{
  "headers": {
    "Accept": "*/*", 
    "Connection": "close", 
    "Host": "httpbin.org", 
    "User-Agent": "curl/7.54.0", 
    "X-Consumer-Id": "e4e774ce-8b8e-412c-ae2f-c8ac131d9aa0", 
    "X-Consumer-Username": "Jason", 
    "X-Forwarded-Host": "example.com"
  }
}
```

Take note of the interesting forwarded headers: `X-Consumer-Id`, `X-Consumer-Username` which can help you at the app level to know which consumer is actually calling.

You probably want to pass this key in the headers instead:

```console
$ curl -i -X GET \
  --url http://localhost:8000/headers \        
  --header "Host: example.com" \
--header "apikey: SECRET"
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 297
Connection: keep-alive
Server: meinheld/0.6.1
Date: Mon, 27 Nov 2017 10:48:08 GMT
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
X-Powered-By: Flask
X-Processed-Time: 0.000652074813843
Via: kong/0.11.1
X-Kong-Upstream-Latency: 553
X-Kong-Proxy-Latency: 0

{
  "headers": {
    "Accept": "*/*", 
    "Apikey": "SECRET", 
    "Connection": "close", 
    "Host": "httpbin.org", 
    "User-Agent": "curl/7.54.0", 
    "X-Consumer-Id": "e4e774ce-8b8e-412c-ae2f-c8ac131d9aa0", 
    "X-Consumer-Username": "Jason", 
    "X-Forwarded-Host": "example.com"
  }
}
```



