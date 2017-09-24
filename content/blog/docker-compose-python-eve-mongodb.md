---
title: "Docker Compose Python Eve Mongodb"
date: 2017-08-18T14:47:35+01:00
draft: true
tags: [ "docker", "python", "eve", "MongoDB", "REST", "PyCharm" ]
---

> This article is inspired by [a blog post from JetBrains](https://blog.jetbrains.com/pycharm/2017/03/docker-compose-getting-flask-up-and-running/).

## PyCharm

As a Python developer you are probably familiar with [PyCharm](https://www.jetbrains.com/pycharm/), an amazing IDE from [JetBrains](https://www.jetbrains.com).
If you don't know anything about it, have a look, you'll probably not regret it :) 
Some of the steps in this article will make mention of PyCharm, however, it's not required and you can still follow if you're using another IDE.
If you're not using any IDE, well, god save your soul.

## What we will do here

We will go through [Eve's quickstart guide](http://python-eve.org/quickstart.html) adding the docker context. 

## Requirements

Check you have the required software on your machine.

```
$ docker -v
Docker version 17.06.0-ce, build 02c1d87
```
```
$ docker-compose -v
docker-compose version 1.14.0, build c7bdf9e
```
If any of these commands are not found, check the [docker](https://www.docker.com/) documentation for install procedure.

## Let's get started!

### Eve's requirements

[Eve's quickstart guide](http://python-eve.org/quickstart.html) requires MongoDB installed and running.

If we translate that into docker, we will have:

* a Python container, running our Eve app
* a MongoDB container, linked to the first one

Ok, that's the role of [docker compose](https://docs.docker.com/compose/) when you can instruct docker about multiple containers and how they interact.

### Python app

The Python app is our Eve app, so let's get it started:

Create a new file, *app.py* with
```python
from eve import Eve
app = Eve()

if __name__ == '__main__':
    app.run(host='0.0.0.0')
```
and another one, *settings.py* with 
```python
MONGO_HOST = "mongo"
DOMAIN = {'people': {}}
```
We need to specify here the host on which mongo runs, otherwise Eve assumes it's local.

> Note the host is just called "mongo", which is the name we will give it in the `docker-compose.yml` file below.

This is our Eve app! Let's instruct docker about it now.

add a new file, called *Dockerfile*
```Dockerfile
FROM python:3

EXPOSE 5000

WORKDIR /usr/src/app

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD [ "python", "./run.py" ]
```
This will use a base `python` image and run our app in it, by:

* Exposing port `5000` so that it can be accessed outside the docker container
* Installing the *requirements.txt* with pip
* Embedding our code into `/usr/src/app`
* Running the command `python run.py` 
 
Ok, where are the *requirements.txt*? Let's add them

```
eve == 0.7.4
```
We just need the `eve` Python package here.  

### MongoDB

For MongoDB, we will just use their [official container](https://hub.docker.com/_/mongo/).

### Let's wrap it all together

Add a new file, *docker-compose.yml*

```yaml
version: '2'
services:
  web:
    build: .
    ports:
      - "5000:5000"
    volumes:
      - .:/usr/src/app
    links:
      - mongo
  mongo:
    image: "mongo"
```

This defines our stack, according to [version 3 of the docker-compose reference](https://docs.docker.com/compose/compose-file/):

* One container called "web", where our Eve app is
  * which port `5000` is bound to host's port `5000` 
  * where the project directory being shared with `/usr/src/app` in the container (this allows us to make changes to the code without having to rebuild the container each time)
  * which links to another container called "mongo"
* One container called "mongo"

### Let's run it now!

```shell
$ docker-compose up
```
After putting all the layers together, this command should end with:
```
web_1    |  * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
```
In a terminal, just do as Eve's quickstart suggests:
```
$ curl -i http://127.0.0.1:5000

HTTP/1.0 200 OK
Content-Type: application/json
Content-Length: 62
Server: Eve/0.7.4 Werkzeug/0.11.15 Python/3.6.2
Date: Fri, 18 Aug 2017 15:31:52 GMT

{"_links": {"child": [{"href": "people", "title": "people"}]}}%   
```
And then try:
```
$ curl http://127.0.0.1:5000/people

{
    "_items": [],
    "_links": {
        "parent": {
            "title": "home",
            "href": "/"
        },
        "self": {
            "title": "people",
            "href": "people"
        }
    },
    "_meta": {
        "page": 1,
        "max_results": 25,
        "total": 0
    }
}
```
This command is useful to test the connection with mongo. If it's not properly configured, it will fail.

Also try the `DELETE` command and verify it's not authorised as suggested by the Quickstart from Eve.

### Awww, Stop it!

Ok we can now stop the containers (CTRL+C) and configure PyCharm now.

## PyCharm setup

Wouldn't it be cool to be able to debug your app while it runs in the container?

That's what we will do now.

> PyCharm [doesn't currently support](https://youtrack.jetbrains.com/issue/PY-22674) docker-compose reference version 3.

### Configure docker

{{< figure src="/img/Preferences.png"  >}} 

### Configure docker-compose

Under project interpreter, click "Add remote", then docker-compose and make sure you select "web".

{{< figure src="/img/Interpreter.png"  >}}

Add path mapping:
{{< figure src="/img/Mapping.png"  >}}

You can now run your debugger which will automatically launch the docker compose command.
{{< figure src="/img/Mapping.png"  >}}



## Source code

Source code is available at https://github.com/iyp-uk/docker-compose-python-eve
