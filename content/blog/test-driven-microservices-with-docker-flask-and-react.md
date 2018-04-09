---
title: "Test Driven Microservices With Docker Flask And React"
date: 2017-08-30T15:56:58+01:00
draft: true
tags: [ "TDD", "microservices", "docker", "python", "flask", "react" ]
---

## What you will learn here

* Build and secure a user management store, with different permissions levels, using a micro-service and TDD approach.
* Learn about authentication through JWT (JSON Web Tokens) and authorization
* Have a micro service approach with 4 layers:
    * main: An admin / orchestration layer
    * users: A user management layer: It's our main API here, persisting users data in a Postgres database, exposing a secure API
    * client: A front-end client layer in React
    * swagger: Presenting our API through swagger
* Manage different environments with `docker-machine`

## Related articles

You may want to read [this article about setting up docker for local development]({{< relref "docker-compose-python-eve-mongodb.md" >}}) too.

## Let's get started!

Assuming you've got:

* `docker`, `docker-compose` and `docker-machine` installed

```console
$ docker -v
Docker version 17.06.1-ce, build 874a737
$ docker-compose -v
docker-compose version 1.14.0, build c7bdf9e
$ docker-machine -v
docker-machine version 0.12.0, build 45c69ad
```

You may wonder why we require `docker-machine` at this point. We will use it to manage the different environments and ensure consistency.
In another article, we may refactor that to make use of `kubernetes` instead. 
As all these three come built-in with Docker for mac or for Windows, which is likely to be you local dev environment, let's keep things easy for now.

### Add a project directory

> Alternatively, you can clone [this repository](https://github.com/iyp-uk/docker-flask-react).

```console
$ mkdir docker-flask-react
$ cd docker-flask-react
```
### Add `requirements.txt` 

Run a docker container with Python 3 in it:

```console
$ docker run --name python -v $(pwd):/usr/src/app -d -it python:3 bash
bfb9c1f8c8e70a06f95782a5502208211c79988c032e5b7109e923ba17db573c
```

> We're now running python in a docker container where the current directory is shared with `/usr/src/app` in the container.

Check that your container is running:

```console
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
4eef8ee357ad        python:3            "bash"              3 minutes ago       Up 3 seconds                            python
```

Attach to it:

```console
$ docker attach python
root@4eef8ee357ad:/# 
```

We can now install our dependencies with pip:

```console
root@4eef8ee357ad:/# pip install flask
Collecting flask
  Downloading Flask-0.12.2-py2.py3-none-any.whl (83kB)
    100% |████████████████████████████████| 92kB 1.1MB/s 
Collecting Jinja2>=2.4 (from flask)
  Downloading Jinja2-2.9.6-py2.py3-none-any.whl (340kB)
    100% |████████████████████████████████| 348kB 2.7MB/s 
Collecting Werkzeug>=0.7 (from flask)
  Downloading Werkzeug-0.12.2-py2.py3-none-any.whl (312kB)
    100% |████████████████████████████████| 317kB 2.4MB/s 
Collecting click>=2.0 (from flask)
  Downloading click-6.7-py2.py3-none-any.whl (71kB)
    100% |████████████████████████████████| 71kB 6.5MB/s 
Collecting itsdangerous>=0.21 (from flask)
  Downloading itsdangerous-0.24.tar.gz (46kB)
    100% |████████████████████████████████| 51kB 8.3MB/s 
Collecting MarkupSafe>=0.23 (from Jinja2>=2.4->flask)
  Downloading MarkupSafe-1.0.tar.gz
Building wheels for collected packages: itsdangerous, MarkupSafe
  Running setup.py bdist_wheel for itsdangerous ... done
  Stored in directory: /root/.cache/pip/wheels/fc/a8/66/24d655233c757e178d45dea2de22a04c6d92766abfb741129a
  Running setup.py bdist_wheel for MarkupSafe ... done
  Stored in directory: /root/.cache/pip/wheels/88/a7/30/e39a54a87bcbe25308fa3ca64e8ddc75d9b3e5afa21ee32d57
Successfully built itsdangerous MarkupSafe
Installing collected packages: MarkupSafe, Jinja2, Werkzeug, click, itsdangerous, flask
Successfully installed Jinja2-2.9.6 MarkupSafe-1.0 Werkzeug-0.12.2 click-6.7 flask-0.12.2 itsdangerous-0.24
root@4eef8ee357ad:/# pip install flask-script
Collecting flask-script
  Downloading Flask-Script-2.0.5.tar.gz (42kB)
    100% |████████████████████████████████| 51kB 738kB/s 
Requirement already satisfied: Flask in /usr/local/lib/python3.6/site-packages (from flask-script)
Requirement already satisfied: click>=2.0 in /usr/local/lib/python3.6/site-packages (from Flask->flask-script)
Requirement already satisfied: Werkzeug>=0.7 in /usr/local/lib/python3.6/site-packages (from Flask->flask-script)
Requirement already satisfied: itsdangerous>=0.21 in /usr/local/lib/python3.6/site-packages (from Flask->flask-script)
Requirement already satisfied: Jinja2>=2.4 in /usr/local/lib/python3.6/site-packages (from Flask->flask-script)
Requirement already satisfied: MarkupSafe>=0.23 in /usr/local/lib/python3.6/site-packages (from Jinja2>=2.4->Flask->flask-script)
Building wheels for collected packages: flask-script
  Running setup.py bdist_wheel for flask-script ... done
  Stored in directory: /root/.cache/pip/wheels/e2/ea/d8/8d114e46cef819f7d9879504a7f9cb2a88a479af2858223d9f
Successfully built flask-script
Installing collected packages: flask-script
Successfully installed flask-script-2.0.5
```

And finally we can freeze requirements:

```console
root@4eef8ee357ad:/# pip freeze > /usr/src/app/requirements.txt
```

This should create a file `requirements.txt` in the current working directory (if you're following it's the root of your project).

Cool, we can now dispose of that container.

```console
root@4eef8ee357ad:/# exit
exit
$ docker rm python
python
```

Let's have a look at this `requirements.txt`:

```console
$ cat requirements.txt
click==6.7
Flask==0.12.2
Flask-Script==2.0.5
itsdangerous==0.24
Jinja2==2.9.6
MarkupSafe==1.0
Werkzeug==0.12.2
```

What we *really* need here, is just `flask` and `flask-script`, so we could delete the rest as they are dependencies.

### Add the skeleton for flask and flask-script

We're now adding 3 files:

* `app/__init__.py`
* `app/config.py`
* `manage.py`

Your IDE will probably scream because you don't have the python dependencies installed.
Don't worry, we cover that part in the next paragraph.

```python
# app/__init__.py
from flask import Flask, jsonify

app = Flask(__name__)

# set config
app_settings = os.getenv('APP_SETTINGS')
app.config.from_object(app_settings)


@app.route('/ping', methods=['GET'])
def ping_pong():
    return jsonify({
        'status': 'success',
        'message': 'pong!'
    })
```

```python
# app/config.py
class BaseConfig:
    """Base configuration"""
    DEBUG = False
    TESTING = False


class DevelopmentConfig(BaseConfig):
    """Development configuration"""
    DEBUG = True


class TestingConfig(BaseConfig):
    """Testing configuration"""
    DEBUG = True
    TESTING = True


class ProductionConfig(BaseConfig):
    """Production configuration"""
    DEBUG = False
```

```python
# manage.py
from flask_script import Manager
from app import app

manager = Manager(app)

if __name__ == '__main__':
    manager.run()
```

### Set up Docker:

Let's add a `Dockerfile` to describe our app container.

```Dockerfile
FROM python:3

EXPOSE 5000

WORKDIR /usr/src/app

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD [ "python", "manage.py", "runserver", "--host=0.0.0.0" ]
```

Notice how it makes use of the `requirements.txt` we just created.

Now add a `docker-compose.yml` file to the mix at the root of your project.

```yaml
version: '2'
services:
  app:
    build: .
    ports:
      - "5000:5000"
    volumes:
      - .:/usr/src/app
    environment:
      - APP_SETTINGS=app.config.DevelopmentConfig
```

Let's try it now:

```console
$ docker-compose up
Starting dockerflaskreact_app_1 ... 
Starting dockerflaskreact_app_1 ... done
Attaching to dockerflaskreact_app_1
app_1  |  * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
app_1  |  * Restarting with stat
app_1  |  * Debugger is active!
app_1  |  * Debugger PIN: 333-640-417
```

```console
$ curl -i localhost:5000/ping
HTTP/1.0 200 OK
Content-Type: application/json
Content-Length: 49
Server: Werkzeug/0.12.2 Python/3.6.2
Date: Wed, 30 Aug 2017 17:23:01 GMT

{
  "message": "pong!", 
  "status": "success"
}
```

Yay! The ping pongs, our app works!

At this stage your code should look like [that](https://github.com/iyp-uk/docker-flask-react/tree/initial-set-up).

> You can also use a browser to see it if you don't have `curl` (no comment on that): http://localhost:5000/ping

Just go ahead and make some changes in the response from the `/ping`, you should see them live without the need to reload anything manually as we're using flask in debug mode. 

### (Optional) Set up python interpreter in your IDE

Example below with PyCharm:

{{< figure src="/img/project-interpreter-pycharm.png"  >}}

### Add a database

To persist information, we will need a database, introducing [Postgres](https://www.postgresql.org/).

So Postgres will run [in its own container here](https://hub.docker.com/_/postgres/), however, the flask container needs to be able to communicate with it. 
Introducing [SQLAlchemy](https://www.sqlalchemy.org/), an ORM. It will sit on the flask container thanks to the [`flask-sqlalchemy` package](http://flask-sqlalchemy.pocoo.org/2.2/).

Let's add `flask-sqlalchemy` (currently version 2.2) to the flask's container requirements.

```console
$  echo "Flask-SQLAlchemy==2.2" >> requirements.txt 
$  echo "psycopg2==2.7.3" >> requirements.txt 
```

and update our `docker-compose.yml` so that it includes this service:

```yaml
version: '2'
services:
  app:
    build: .
    ports:
      - "5000:5000"
    volumes:
      - .:/usr/src/app
    environment:
      - APP_SETTINGS=app.config.DevelopmentConfig
      - DATABASE_URL=postgres://postgres:postgres@postgres:5432/app
    links:
      - postgres
  postgres:
    image: "postgres:alpine"
    environment:
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=app
```
Default username for postgres is simply `postgres` (see [docker postgres documentation](https://hub.docker.com/_/postgres/)).

Let's now add some configuration and our Users schema:

In `app/config.py`, `BaseConfig` now has:

```python
SQLALCHEMY_TRACK_MODIFICATIONS = False
SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL')
```

In `app/__init__.py`, new imports:

```python
import datetime
from flask_sqlalchemy import SQLAlchemy
```

and DB instantiation and set up:

```python
# instantiate the db
db = SQLAlchemy(app)

# model
class User(db.Model):
    __tablename__ = "users"
    id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    username = db.Column(db.String(128), nullable=False)
    email = db.Column(db.String(128), nullable=False)
    active = db.Column(db.Boolean(), default=False, nullable=False)
    created_at = db.Column(db.DateTime, nullable=False)

    def __init__(self, username, email):
        self.username = username
        self.email = email
        self.created_at = datetime.datetime.utcnow()
```

Let's relaunch all that:

```console
$ docker-compose up --build
```

Cool, let's now add some functionality to our manager:

```python
# manage.py
from flask_script import Manager
from app import app, db

manager = Manager(app)


@manager.command
def recreate_db():
    """Recreates a database."""
    db.drop_all()
    db.create_all()
    db.session.commit()


if __name__ == '__main__':
    manager.run()
```

The `create_all` method here will call the model we've defined in `app/__init__.py`.

Let's see:

```console
$ docker-compose ps
           Name                          Command               State           Ports          
---------------------------------------------------------------------------------------------
dockerflaskreact_app_1        python manage.py runserver ...   Up      0.0.0.0:5000->5000/tcp 
dockerflaskreact_postgres_1   docker-entrypoint.sh postgres    Up      5432/tcp       
```
Our 2 containers are running. We can connect to postgres with:
```console
$ docker exec -ti $(docker ps -aqf "name=dockerflaskreact_postgres_1") psql -U postgres
psql (9.6.5)
Type "help" for help.

postgres=# 
```
> Check out the [available commands here](https://gist.github.com/Kartones/dd3ff5ec5ea238d4c546).
 
The name of our database is `app`:
```psql
postgres=# \c app 
You are now connected to database "app" as user "postgres".
```
But nothing is in there yet:
```psql
app=# \dt
No relations found.
```
In another terminal, let's run our `recreate_db` script:
```console
$ docker-compose run app python manage.py recreate_db                                  
Starting dockerflaskreact_postgres_1 ... done
```
Back to our postgres console:
```psql
app=# \dt
         List of relations
 Schema | Name  | Type  |  Owner   
--------+-------+-------+----------
 public | users | table | postgres
(1 row)

app=# \q
```
At this point, [here's what your codebase should look like](https://github.com/iyp-uk/docker-flask-react/tree/postgres-set-up).
And [that's the summary of changes](https://github.com/iyp-uk/docker-flask-react/compare/initial-set-up...postgres-set-up)

### Add tests

We said this would be TDD, so it's time to set up tests now.

Add a new dependency, [flask-testing](https://flask-testing.readthedocs.io/en/latest/):
```console
$ echo "Flask-Testing==0.6.2" >> requirements.txt
```
And rebuild the stack:
```console
$ docker-compose up -d --build
```
We can now write our tests. Let's add a new directory:
```console
$ mkdir tests
```
Add the different tests we will use:
```console
$ cd tests && touch __init__.py base.py test_config.py test_users.py
```
A bit of explanation here:

* `__init__.py` define a package
* `base.py` will define the base class for our tests
* `test_config.py` will ensure the configuration is how we expect it to be
* `test_users.py` will hold tests against the users service

Just add the tests [as you find them in the repository](https://github.com/iyp-uk/docker-flask-react/tree/tests-set-up/tests), and launch them with:

```console
$ docker-compose run app python manage.py test
Starting dockerflaskreact_postgres_1 ... done
test_app_is_development (test_config.TestDevelopmentConfig) ... FAIL
test_app_is_production (test_config.TestProductionConfig) ... FAIL
test_app_is_testing (test_config.TestTestingConfig) ... FAIL
test_users (test_users.TestUserService)
Ensure the /ping route behaves correctly. ... ok

======================================================================
FAIL: test_app_is_development (test_config.TestDevelopmentConfig)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/src/app/tests/test_config.py", line 13, in test_app_is_development
    self.assertTrue(app.config['SECRET_KEY'] == 'my_precious')
AssertionError: False is not true

======================================================================
FAIL: test_app_is_production (test_config.TestProductionConfig)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/src/app/tests/test_config.py", line 40, in test_app_is_production
    self.assertTrue(app.config['SECRET_KEY'] == 'my_precious')
AssertionError: False is not true

======================================================================
FAIL: test_app_is_testing (test_config.TestTestingConfig)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/src/app/tests/test_config.py", line 26, in test_app_is_testing
    self.assertTrue(app.config['SECRET_KEY'] == 'my_precious')
AssertionError: False is not true

----------------------------------------------------------------------
Ran 4 tests in 0.156s

FAILED (failures=3)
``` 
Yes, that's intended, now add the `SECRET_KEY` to `app/config.py` in the `BaseConfig` class:
```python
class BaseConfig:
    """Base configuration"""
    DEBUG = False
    TESTING = False
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL')
    SECRET_KEY = 'my_precious'
```
Run the tests again:
```console
$ docker-compose run app python manage.py test
Starting dockerflaskreact_postgres_1 ... done
test_app_is_development (test_config.TestDevelopmentConfig) ... ok
test_app_is_production (test_config.TestProductionConfig) ... ok
test_app_is_testing (test_config.TestTestingConfig) ... ok
test_users (test_users.TestUserService)
Ensure the /ping route behaves correctly. ... ok

----------------------------------------------------------------------
Ran 4 tests in 0.118s

OK
```
Boom! Success!

[See the summary of changes introducing tests](https://github.com/iyp-uk/docker-flask-react/compare/postgres-set-up...tests-set-up).

### Refactoring: Blueprints

> Read about Blueprints on [flask documentation](http://flask.pocoo.org/docs/0.12/blueprints/)

We will add an `api` blueprint here, under our `app` folder. Let's create a new directory called `api` along with some files:

```console
$ mkdir app/api && touch app/api/__init__.py app/api/views.py app/api/models.py  
```
> The class definitions for models are together in `models.py`, the route definitions are in `views.py`. 

If you [take a look at how our app is currently initialised](https://github.com/iyp-uk/docker-flask-react/blob/tests-set-up/app/__init__.py), we can move some things to where they belong:

`views.py` holds routes: 
```python
# app/api/views.py
from flask import Blueprint, jsonify

users_blueprint = Blueprint('users', __name__)


@users_blueprint.route('/ping', methods=['GET'])
def ping_pong():
    return jsonify({
        'status': 'success',
        'message': 'pong!'
    })
``` 
and `models.py` holds models:
```python
# app/api/models.py
import datetime
from app import db

class User(db.Model):
    __tablename__ = "users"
    id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    username = db.Column(db.String(128), nullable=False)
    email = db.Column(db.String(128), nullable=False)
    active = db.Column(db.Boolean(), default=False, nullable=False)
    created_at = db.Column(db.DateTime, nullable=False)

    def __init__(self, username, email):
        self.username = username
        self.email = email
        self.created_at = datetime.datetime.utcnow()
```
So we could remove these bits from `app/__init__.py` and rewrite it a bit (you'll se why later):
```python
# app/__init__.py
import os
from flask import Flask
from flask_sqlalchemy import SQLAlchemy

# instantiate the db
db = SQLAlchemy()


def create_app():
    app = Flask(__name__)

    # set config
    app_settings = os.getenv('APP_SETTINGS')
    app.config.from_object(app_settings)

    # set up extensions
    db.init_app(app)

    # register blueprints
    from app.api.views import users_blueprint
    app.register_blueprint(users_blueprint)

    return app
```
There are several files where we imported the `app`, so let's update these files to have that instead:
```python
from app import create_app

app = create_app()
```
Make sure all's still fine by running the tests!
```console
$ docker-compose run app python manage.py test
## ...
Ran 4 tests in 0.083s

OK
```
Well done! 
[See changes summary](https://github.com/iyp-uk/docker-flask-react/compare/tests-set-up...blueprints-set-up?diff=split&name=blueprints-set-up)

### RESTful Routes

We've gone through a lot, but nothing currently helps us adding users or even listing them, which is what our app is supposed to do!
Well, it's now time to do it!

| Endpoint | HTTP Method | CRUD Method | Result |
|---|---|---|---|
| `/users` | `GET` | READ | get all users |
| `/users/:id` | `GET` | READ | get single user |
| `/users` | `POST` | CREATE | add a user |

As we're TDD, we will:

1. Add a test of each of these
1. Run the test and watch it fail
1. Write just enough code to make it pass
1. Refactor eventually

#### POST

So let's update our `TestUserService` class with:
```python
def test_add_user(self):
    """Ensure we can create a new user in the database"""
    with self.client:
        response = self.client.post(
            '/users',
            data=json.dumps(dict(
                username='michael',
                email='michael@realpython.com'
            )),
            content_type='application/json',
        )
        data = json.loads(response.data.decode())
        self.assertEqual(response.status_code, 201)
        self.assertIn('user@example.com was added!', data['message'])
        self.assertIn('success', data['status'])
```
Run the test and watch it fail:
```console
$ docker-compose run app python manage.py test       
Starting dockerflaskreact_postgres_1 ... done
test_app_is_development (test_config.TestDevelopmentConfig) ... ok
test_app_is_production (test_config.TestProductionConfig) ... ok
test_app_is_testing (test_config.TestTestingConfig) ... ok
test_add_user (test_users.TestUserService)
Ensure we can create a new user in the database ... ERROR
test_users (test_users.TestUserService)
Ensure the /ping route behaves correctly. ... ok
## ...
```
So let's add a route which will allow "POSTing" users to the `/users` endpoint.
In `app/api/views.py`:

```python
@users_blueprint.route('/users', methods=['POST'])
def add_user():
    post_data = request.get_json()
    username = post_data.get('username')
    email = post_data.get('email')
    db.session.add(User(username=username, email=email))
    db.session.commit()
    response_object = {
        'status': 'success',
        'message': f'{email} was added!'
    }
    return jsonify(response_object), 201
```
Run tests again, they pass:
```console
$ docker-compose run app python manage.py test
Starting dockerflaskreact_postgres_1 ... done
test_app_is_development (test_config.TestDevelopmentConfig) ... ok
test_app_is_production (test_config.TestProductionConfig) ... ok
test_app_is_testing (test_config.TestTestingConfig) ... ok
test_add_user (test_users.TestUserService)
Ensure we can create a new user in the database ... ok
test_users (test_users.TestUserService)
Ensure the /ping route behaves correctly. ... ok

----------------------------------------------------------------------
Ran 5 tests in 0.225s

OK
```
And again, they *still* pass... Why? 
Well, have a look at the `BaseTestCase` class. Methods `setUp` and `tearDown` are called automatically when the tests are run.
This means users are not persisted, and 2 consequent tests run would be fine.
We probably want to avoid registering the same user/email twice, so we will add a test for it, along with some other common things we want to avoid, such as:

* Payload is empty
* Payload is wrongly formatted

Write the tests first, then the routes. Do the same for the other endpoints.

[See these tests, the corresponding route definition and other endpoints' tests and routes on the repository](https://github.com/iyp-uk/docker-flask-react/tree/restful-routes-set-up)

Phew, it's been quite something, [let's see where we are now](https://github.com/iyp-uk/docker-flask-react/compare/blueprints-set-up...restful-routes-set-up)

## Ready to go live!

At this point, we're somehow ready to go live. After all, we've got our API set up, it could work.
However, we probably want to:

* Serve on port `80`, not `5000`
* Serve with a proper web server, not flask
* Provide an interface for humans, something more readable than raw JSON
* Do some other things around security

### AWS

So, our testing and production environments will be in AWS.
If you're not registered yet, go ahead and register, it's likely that the steps below will be free of charge as you may be eligible for the free tier.

Make sure you're also setting up the CLI along with a user, which will result in a `~/.aws/credentials` file.
Check it now if you're unsure:
```console
$ ls ~/.aws
config      credentials
```
If you don't see these files, then [follow the CLI configuration on AWS](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html).

### Adding a machine in AWS

```console
$ docker-machine create --driver amazonec2 aws
Running pre-create checks...
Creating machine...
(aws) Launching instance...
Waiting for machine to be running, this may take a few minutes...
Detecting operating system of created instance...
Waiting for SSH to be available...
Detecting the provisioner...
Provisioning with ubuntu(systemd)...
Installing Docker...
Copying certs to the local machine directory...
Copying certs to the remote machine...
Setting Docker configuration on the remote daemon...
Checking connection to Docker...
Docker is up and running!
To see how to connect your Docker Client to the Docker Engine running on this virtual machine, run: docker-machine env aws
```
You can specify the region too if you don't want to use the default AWS region.
```console
$ docker-machine create --driver amazonec2 --amazonec2-region eu-west-2 aws
```
Let's see what we have now:
```console
$ docker-machine ls
NAME      ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
aws       *        amazonec2    Running   tcp://35.177.228.232:2376           v17.05.0-ce   
```
You can also go ahead and log in to your AWS console, you'll see this EC2 instance running.

### Multiple environments set up

We will refactor our `docker-compose.yml` file so that we can use the [composition mechanisms](https://docs.docker.com/compose/extends/).

We will end up having:

* `docker-compose.yml`: Our base definition of the services in use, including as little information as possible
* `docker-compose.override.yml`: This one is picked up automatically when present on a `docker-compose up` command. It it where we will have our local development overrides.
* `docker-compose.prod.yml`: The description of our services in production (use of gunicorn for instance instead of flask built-in server)

```yaml
# docker-compose.yml
version: '2'
services:
  app:
    build: .
    ports:
      - "5000:5000"
    environment:
      - DATABASE_URL=postgres://postgres:postgres@postgres:5432/app
    links:
      - postgres
  postgres:
    image: "postgres:alpine"
    environment:
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=app
```

```yaml
# docker-compose.override.yml
version: '2'
services:
  app:
    volumes:
      - .:/usr/src/app
    environment:
      - APP_SETTINGS=app.config.DevelopmentConfig
      - DATABASE_URL=postgres://postgres:postgres@postgres:5432/app
```

```yaml
# docker-compose.prod.yml
version: '2'
services:
  app:
    environment:
      - APP_SETTINGS=app.config.ProductionConfig
      - DATABASE_URL=postgres://postgres:postgres@postgres:5432/app
```
As you can see, the `docker-compose.yml` defines the service, the other files just override whatever is required at their level. 
You don't need to repeat everything.

Let's see:
```console
$ docker-compose up -d --build
$ docker-compose run app python manage.py test
```
All tests pass! OK, let's try them on production now.

Start by instructing your local docker to connect with the aws machine:
```console
$ eval $(docker-machine env aws)
```

> From now on, all commands will be against the aws machine, not your local docker!

```console
$ docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
$ docker-compose -f docker-compose.yml -f docker-compose.prod.yml run app python manage.py test
```

Yes, commands are a bit long... but they are the same as previously, just we specify the docker-compose files to use.
And that's just for production, so it will be in a CI tool of some sort, you don't need to worry too much about it.
Instead, just realised you've created a machine in AWS and ran some containers on it in a few commands.

Now let's open port `5000` so that we can access it too: 

1. Login to your AWS console
1. Navigate to your EC2 Dashboard
1. Under "security groups" find the group for "docker-machine" (or create your own)
1. In the inbound rules, add a custom TCP with port `5000` on custom source `0.0.0.0/0` 

> [Read this for more](https://stackoverflow.com/questions/26338301/ec2-how-to-add-port-8080-in-security-group).

Try it with a simple `curl`:
```console
$ curl -i $(docker-machine ip aws):5000/ping
HTTP/1.0 200 OK
Content-Type: application/json
Content-Length: 49
Server: Werkzeug/0.12.2 Python/3.6.2
Date: Sun, 03 Sep 2017 19:21:32 GMT

{
  "message": "pong!", 
  "status": "success"
}
```

Cool, but we want to access it over port 80, not 5000.
Let's add `gunicorn` and `nginx` to the mix.

[Gunicorn](http://gunicorn.org/) is a Python WSGI HTTP Server. It can handle requests on a production environment, unlike the flask built-in server, optimised for development.
[Nginx](https://nginx.org/en/) is an HTTP and reverse proxy server, a mail proxy server, and a generic TCP/UDP proxy server.
We will use it as a proxy, pass requests on port 80 to gunicorn, listening on 5000.

> We don't need gunicorn for the development, so it will just be part of the `docker-compose.prod.yml`.

#### Add gunicorn

Add gunicorn to our `requirements.txt`:
```console
$ echo "gunicorn==19.7.1" >> requirements.txt 
```
And to our `docker-compose.prod.yml`
```yaml
command: gunicorn -b 0.0.0.0:5000 manage:app
```
This command, in the `app` service, will override what is defined in the `Dockerfile` for this service.
As a reminder, it's currently `CMD [ "python", "manage.py", "runserver", "--host=0.0.0.0" ]`.

Let's try it out:
```console
$ docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
```
And verify the server is now *gunicorn* (Was previously *Werkzeug/0.12.2 Python/3.6.2*, Flask's built-in server):
```console
$ curl -i $(docker-machine ip aws):5000/ping                                   
HTTP/1.1 200 OK
Server: gunicorn/19.7.1
Date: Sun, 03 Sep 2017 20:47:41 GMT
Connection: close
Content-Type: application/json
Content-Length: 49

{
  "message": "pong!", 
  "status": "success"
}
```
Cool, but we're still on port 5000, let's set up `nginx` now.

#### Add Nginx

Let's add a new directory called `nginx` at the root of our project, with the following files:

```console
$ mkdir nginx
$ touch nginx/Dockerfile nginx/app.conf
```

and edit them:
```Dockerfile
# nginx/Dockerfile
FROM nginx:alpine

RUN rm /etc/nginx/conf.d/default.conf
COPY app.conf /etc/nginx/conf.d
```
```nginx
# nginx/app.conf
server {

    listen 80;

    location / {
        proxy_pass http://app:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

}
```
We will keep our `nginx` across all environments, so let's update our `docker-compose.yml`.

Let's try:
```console
$ docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
```
And:
```console
$ curl -i $(docker-machine ip aws):5000/ping  
## Same response as before
```
Now add 80 to the security group in AWS just like you did for port 5000.
And finally:
```console
$ curl -i $(docker-machine ip aws)/ping  
## We're now querying straight to port 80
HTTP/1.1 200 OK
Server: nginx/1.13.3
Date: Sun, 03 Sep 2017 21:03:15 GMT
Content-Type: application/json
Content-Length: 49
Connection: keep-alive

{
  "message": "pong!", 
  "status": "success"
}
```
Notice the `Server: nginx/1.13.3` header.

We can now remove port 5000 from the security group.

Let's tear down production as we won't use it for a while.

```console
$ docker-machine stop aws
Stopping "aws"...
Machine "aws" was stopped.
```
And stop using `docker-machine` by resetting the environment variables. Optional, you can switch to your 'dev' machine if you're using it.
```console
$ eval $(docker-machine env -u) 
```
Control:
```console
$ docker-machine active
No active host found
```
Rebuild your local containers:
```console
$ docker-compose up -d --build
```
Do you have an error because your port 80 is already in use?
If so you have several options:

* Do a port mapping on the nginx service on something else than 80 (like `"8080:80"` in your `docker-compose.override.yml` and access it from `localhost:8080`)
* Add a "dev" machine for local usage
* Stop whatever runs on your port 80...

[See what the code looks like now](https://github.com/iyp-uk/docker-flask-react/tree/gunicorn-and-nginx-set-up) and [the diff with the previous checkpoint](https://github.com/iyp-uk/docker-flask-react/compare/restful-routes-set-up...gunicorn-and-nginx-set-up).

### Add some templates

Our `/users` endpoint currently just returns some JSON. 
But when using a browser, it would be more convenient to view some HTML.
Introducing templates!

```console
$ mkdir app/api/templates
$ touch app/api/templates/users.html
```
Now, we will provide JSON to clients by default as it's an API, and deliver HTML when asked.
```python
# app/api/views.py
def client_accepts_json():
    """Checks if client accepts JSON as a response format"""
    best = request.accept_mimetypes.best_match(['application/json', 'text/html'])
    # if application/json is ranked higher, then return true
    return best == 'application/json' and request.accept_mimetypes[best] > request.accept_mimetypes['text/html']
```
Now update the GET `/users`:


Test it out:
```console
## client says he accepts json (this header is optional, you'd get the same without it as it's our default)
$ curl -Haccept:application/json localhost/users                    
{
  "data": {
    "users": []
  }, 
  "status": "success"
}

## client says he accepts html 
$ curl -Haccept:text/html localhost/users
<!DOCTYPE html>
<html>
  <head>
    ....
```
Add some tests and update the routes in `app/api/views.py` accordingly.

[See what the code should look like now](https://github.com/iyp-uk/docker-flask-react/tree/html-set-up) and [measure progress from previous checkpoint](https://github.com/iyp-uk/docker-flask-react/compare/gunicorn-and-nginx-set-up...html-set-up).

## Refactor into services

At this point in time we will now refactor this project into several parts, not quite to our goal yet, but close.

* main: An admin / orchestration layer
* users: A user management layer: It's our main API here, persisting users data in a Postgres database, exposing a secure API
* client: A front-end client layer in React
* ~~swagger: Presenting our API through swagger~~ *not yet :)*

### Create new projects

The current project you're working on, `docker-flask-react` will be our "main" project. So let's create 2 others:
```console
$ cd ..
$ mkdir docker-flask-react-users
$ mkdir docker-flask-react-client
```
These projects can be found at [docker-flask-react-users](https://github.com/iyp-uk/docker-flask-react-users) and [docker-flask-react-client](https://github.com/iyp-uk/docker-flask-react-client) on [Github](https://github.com/iyp-uk).

At this point in time, we will want to keep in the main project, just what is administrative and brings the services together. That is
```console
$ mv docker-flask-react/app docker-flask-react-users/ 
$ mv docker-flask-react/tests docker-flask-react-users/ 
$ mv docker-flask-react/Dockerfile docker-flask-react-users/ 
$ mv docker-flask-react/manage.py docker-flask-react-users/ 
$ mv docker-flask-react/requirements.txt docker-flask-react-users/ 
$ cp docker-flask-react/.gitignore docker-flask-react-users/ 
```
So your main project now contains:
```console
$ ls docker-flask-react/ 
docker-compose.override.yml 
docker-compose.prod.yml
docker-compose.yml
nginx
```
Let's update the references in our `docker-compose` files.
We will replace the `app` service to use the new users project instead. 
Now you can reference the files in the other directory, or a repository. 
As we want as much separation of concerns as possible, we will choose to reference the repository.
```yaml
# docker-compose.yml
version: '2'
services:
  nginx:
    build: nginx/.
    restart: always
    ports:
      - "80:80"
    links:
      - app
  app:
    build: https://github.com/iyp-uk/docker-flask-react-users.git
    ports:
      - "5000:5000"
    environment:
      - DATABASE_URL=postgres://postgres:postgres@postgres:5432/app
    links:
      - postgres
  postgres:
    image: "postgres:alpine"
    environment:
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=app
```
Due to this, we can't share volumes anymore (it's ok, when developing this service, we can still mount a shared volume there), so we need to update the corresponding compose file.
```yaml
# docker-compose.override.yml
version: '2'
services:
  app:
    environment:
      - APP_SETTINGS=app.config.DevelopmentConfig
      - DATABASE_URL=postgres://postgres:postgres@postgres:5432/app
```
That's it!
Just give it a try:
```console
$ docker-compose down
$ docker-compose up -d --build
$ docker-compose run app python manage.py recreate_db
$ docker-compose run app python manage.py test
...
Ran 15 tests in 1.105s

OK
```
Cool, our tests still run with our project split across 2 repositories!

[Here's how our project looks like now](https://github.com/iyp-uk/docker-flask-react/tree/project-refactor), pretty neat eh? 
And [that's the difference with the previous check point](https://github.com/iyp-uk/docker-flask-react/compare/html-set-up...project-refactor?diff=unified&name=project-refactor).

But hey, hang on, how much is actually covered by our tests? Introducing [coverage.py](http://coverage.readthedocs.io/en/latest/)!
> Coverage.py is a tool for measuring code coverage of Python programs. 
It monitors your program, noting which parts of the code have been executed, then analyzes the source to identify code that could have been executed but was not.
Coverage measurement is typically used to gauge the effectiveness of tests. 
It can show which parts of your code are being exercised by tests, and which are not.

### Code coverage

**Switch to your `docker-flask-react-users` project**

(Optional) If you're using an IDE such as PyCharm, you'd have to reconfigure it for this "new" project.

In particular you'd probably want to:

* Set the environment variables
* Set the volumes bindings
* Set the ports bindings
* Set the interpreter to be the one in the docker container

Run your container, and test out the `/ping` endpoint. Works? Good.
```console
$ docker build -t docker-flask-react-users-service .
$ docker run -p 5000:5000 --name users docker-flask-react-users-service 
/usr/local/lib/python3.6/site-packages/flask_sqlalchemy/__init__.py:819: UserWarning: SQLALCHEMY_DATABASE_URI not set. Defaulting to "sqlite:///:memory:".
  'SQLALCHEMY_DATABASE_URI not set. Defaulting to '
/usr/local/lib/python3.6/site-packages/flask_sqlalchemy/__init__.py:839: FSADeprecationWarning: SQLALCHEMY_TRACK_MODIFICATIONS adds significant overhead and will be disabled by default in the future.  Set it to True or False to suppress this warning.
  'SQLALCHEMY_TRACK_MODIFICATIONS adds significant overhead and '
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
172.17.0.1 - - [05/Sep/2017 14:45:13] "GET /ping HTTP/1.1" 200 -
```
Yep, it misses the `SQLALCHEMY_DATABASE_URI` and `SQLALCHEMY_TRACK_MODIFICATIONS` environment variables... 

So you may wonder why we don't have another `docker-compose.yml` in the users service. 
After all, a service is supposed to come along with its data store. 
Yes and No, we probably want to support more data stores than just postgres at some point.
And the services composition [must be done explicitly](https://docs.docker.com/compose/extends/) in `docker-compose`.
Have a look at some famous projects and how they are distributed in the docker hub. 

Now, we could recreate a `docker-compose.yml` in our users service, that would be convenient.
We can also just run a postgres instance and attach our built container to it for tests.
You'll shortly see how we set up [travis-ci](https://travis-ci.org/) and postgres comes as a service too.
So let's choose the second approach, having another postgres container and attach our app to it.

```console
## Start a postgres instance
$ docker run --name postgres -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=app -d postgres:alpine
9c5c00ce39d9fe7adf324ad744ef7b0774b404482be73b55c9cd4d7443deb555

## Remove our previous users container
$ docker rm users
users

## Start our app and link it to the running postgres
$ docker run -p 5000:5000 --name users --link postgres -e APP_SETTINGS=app.config.DevelopmentConfig -e DATABASE_URL=postgres://postgres:postgres@postgres:5432/app -d docker-flask-react-users-service 
f8cf97609dc97d4e8197b4b273c8ab5576afd3016d85d95ea6287351f875930e

## Run tests
$ docker exec -it users python manage.py recreate_db
$ docker exec -it users python manage.py test
...
Ran 15 tests in 1.200s

OK
```
Cool! The command is not super friendly but we've got it.
Note that if you're using an IDE, you can configure all these environment variables etc to make it more convenient.

Add `coverage.py` to the requirements:

```console
echo "coverage==4.4.1" >> requirements.txt
```

Add a new command to the manager, which wraps the existing `test()` function and provides coverage reports.

Get rid of the hardcoded `SQLALCHEMY_DATABASE_URI`.

Run the test coverage:
```console
$ docker exec -it users python manage.py cov
test_app_is_development (test_config.TestDevelopmentConfig) ... ok
test_app_is_production (test_config.TestProductionConfig) ... ok
test_app_is_testing (test_config.TestTestingConfig) ... ok
test_add_user (test_users.TestUserService)
Ensure we can create a new user in the database ... ok
test_add_user_duplicate_user (test_users.TestUserService)
Ensure error is thrown when user already exists ... ok
test_add_user_empty_payload (test_users.TestUserService)
Ensure error is thrown when payload is empty ... ok
test_add_user_invalid_json (test_users.TestUserService)
Ensure error is thrown when payload is invalid ... ok
test_get_user (test_users.TestUserService)
Ensure we can get a single user based on its ID ... ok
test_get_user_invalid_id (test_users.TestUserService)
Ensures an error is thrown when the user id is not found ... ok
test_get_user_no_id (test_users.TestUserService)
Ensure an error is thrown when no integer id is specified ... ok
test_get_users (test_users.TestUserService)
Ensure we can retrieve all users ... ok
test_get_users_html (test_users.TestUserService)
Ensure we can retrieve all users in HTML ... ok
test_get_users_html_no_users (test_users.TestUserService)
Ensure we get correct response when no users in HTML ... ok
test_main_add_user (test_users.TestUserService)
Ensure a new user can be added through the HTML form ... ok
test_users (test_users.TestUserService)
Ensure the /ping route behaves correctly. ... ok

----------------------------------------------------------------------
Ran 15 tests in 1.033s

OK
Coverage Summary:
Name                  Stmts   Miss Branch BrPart  Cover
-------------------------------------------------------
app/__init__.py          12      5      0      0    58%
app/api/__init__.py       0      0      0      0   100%
app/api/models.py        13      0      0      0   100%
app/api/views.py         59      0     12      0   100%
app/config.py            14      0      0      0   100%
-------------------------------------------------------
TOTAL                    98      5     12      0    95%
```

> Also notice the `htmlcov` directory at the root of your project. You can navigate through your code thanks to HTML files :)

[This is what your codebase should now look like](https://github.com/iyp-uk/docker-flask-react-users/tree/set-up-code-coverage).
And [what we've done since the last checkpoint](https://github.com/iyp-uk/docker-flask-react-users/compare/initial-set-up...set-up-code-coverage)

> Now go back to your "main" project and notice the `cov` command is available now, without any change at this level.
**Note:** *If you have chosen to reference your service using a repository you need to push first...*

```console
## In main project
$ docker-compose up --build -d
...
$ docker-compose run app python manage.py cov
...
Ran 15 tests in 1.130s

OK
Coverage Summary:
Name                  Stmts   Miss Branch BrPart  Cover
-------------------------------------------------------
app/__init__.py          12      5      0      0    58%
app/api/__init__.py       0      0      0      0   100%
app/api/models.py        13      0      0      0   100%
app/api/views.py         59      0     12      0   100%
app/config.py            14      0      0      0   100%
-------------------------------------------------------
TOTAL                    98      5     12      0    95%
```

## Continuous Integration

How about these tests run automatically when a change is made and before it's checked in?
Introducing [travis-ci](https://travis-ci.org/)! One way to handle CI. 
There are many ways to do it, but we've chosen to present travis here as it's free for open source projects and very well integrated with github.

[Get started with travis as per their instructions](https://docs.travis-ci.com/user/getting-started/) up to the creation of a `.travis.yml` file.

Ready? 

### CI on users service

We're now in the [docker-flask-react-users](https://github.com/iyp-uk/docker-flask-react-users) project. Our users service.

Create a new branch, say `test-travis` and add a `.travis.yml` file at the project's root.

```yaml
language: python

python:
  - "3.6"

service:
  - postgresql

install:
  - pip install -r requirements.txt

before_script:
  - export APP_SETTINGS="app.config.TestingConfig"
  - export DATABASE_URL=postgresql://postgres:@localhost/users
  - psql -c 'create database users;' -U postgres

script:
  - python manage.py test
```

This will test our user service with the tests we've defined using a postgres instance in travis.

> Read more about [database services in travis](https://docs.travis-ci.com/user/database-setup/).

Push your branch to github and [see the build happening and passing](https://travis-ci.org/iyp-uk/docker-flask-react-users/jobs/273639184).
Open a pull request and notice the status of the build(s) is reported on the PR so that whoever reviews the PR is aware that tests have passed or failed.

(optional) Create another branch with a failed test and open a PR to see the difference [or view an example here](https://github.com/iyp-uk/docker-flask-react-users/pull/1)

[Current codebase](https://github.com/iyp-uk/docker-flask-react-users/tree/set-up-travis) - [Diff from previous checkpoint](https://github.com/iyp-uk/docker-flask-react-users/compare/set-up-code-coverage...set-up-travis)

### CI on main 

Switch to the [docker-flask-react](https://github.com/iyp-uk/docker-flask-react) "main" project.

[Travis can do many things with docker](https://docs.travis-ci.com/user/docker/), in particular it can run [docker compose](https://docs.travis-ci.com/user/docker/#Using-Docker-Compose) so that it builds the entire stack and runs tests.

Switch to a new branch, yes `travis-test` for instance and add a `.travis.yml` file at the root of your project:

```yaml
sudo: required

services:
  - docker

env:
  - DOCKER_COMPOSE_VERSION=1.14.0

before_install:
  - sudo rm /usr/local/bin/docker-compose
  - curl -L https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/docker-compose-`uname -s`-`uname -m` > docker-compose
  - chmod +x docker-compose
  - sudo mv docker-compose /usr/local/bin

before_script:
  - docker-compose up --build -d

script:
  - docker-compose run app python manage.py test

after_script:
  - docker-compose down
```
Commit, push, check the build. **Fails**.

Eh, why? Let's have a [closer look](https://travis-ci.org/iyp-uk/docker-flask-react/builds/273645871).

Our app can't connect to postgres. Could it be that our postgres service is not running yet when we launch the tests?
Introducing [`healthcheck`](https://docs.docker.com/compose/compose-file/compose-file-v2/#healthcheck) and [conditional `depends_on`](https://docs.docker.com/compose/compose-file/compose-file-v2/#depends_on)!

For that, we will need our `docker-compose.*.yml` files to [update to version 2.1 file format](https://docs.docker.com/compose/compose-file/compose-versioning/#version-21).

While we're at making changes, which environment are we testing against here? Which compose files will be picked up? 
`docker-compose.yml` and `docker-compose.override.yml`! So we're testing for local devs. 
We will use [environment variables](https://docs.travis-ci.com/user/environment-variables) to test all environments variations.

Let's add a `docker-compose.test.yml` for our testing environment.

This is how our `.travis.yml` looks like now:
```yaml
sudo: required

services:
  - docker

env:
  global:
    - DOCKER_COMPOSE_VERSION=1.14.0
  matrix:
    - COMPOSE_FILES=""
    - COMPOSE_FILES="-f docker-compose.yml -f docker-compose.test.yml"
    - COMPOSE_FILES="-f docker-compose.yml -f docker-compose.prod.yml"

before_install:
  - sudo rm /usr/local/bin/docker-compose
  - curl -L https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/docker-compose-`uname -s`-`uname -m` > docker-compose
  - chmod +x docker-compose
  - sudo mv docker-compose /usr/local/bin

before_script:
  - docker-compose ${COMPOSE_FILES} up --build -d

script:
  - docker-compose ${COMPOSE_FILES} run app python manage.py test

after_script:
  - docker-compose ${COMPOSE_FILES} down
``` 
Notice the `env` section, defining environment variables, where `global` defines "static" variables, which do not vary across builds, and `matrix` which will trigger one build each.

[Here's what we've changed compared to previous checkpoint](https://github.com/iyp-uk/docker-flask-react/compare/project-refactor...set-up-ci) and [our current codebase](https://github.com/iyp-uk/docker-flask-react/compare/project-refactor...set-up-ci).

### Decoupled front end

Let's introduce now [ReactJS](https://facebook.github.io/react/) to replace our server-side flask templates.

Switch now to your [`docker-flask-react-client`](https://github.com/iyp-uk/docker-flask-react-client) project.

This frontend will be available as a service so that we can pick it up in our main project with docker compose. 
It will then run in Docker too!

We will use the [create-react-app](https://github.com/facebookincubator/create-react-app) cli tool to generate our project's skeleton. 
It is actually optional as anyone who picks up the project wouldn't require it on its machine since it will use docker for developing.

#### Creating your app's skeleton (optional)

> You need to have node and [create-react-app](https://github.com/facebookincubator/create-react-app) on your machine for this step.

```console
$ create-react-app app
...
``` 
The above command creates a new directory, `app`, where your react application will live.
There's also an interesting README file under `app/README.md`. Have a look!

If you've completed that step, you can probably just do
```console
$ cd app && npm start
```
Which will launch your browser at http://localhost:3000/ and you can admire your react app running.
Stop it with Ctrl+C.

#### Run react in Docker

Let's add a `Dockerfile`:

```Dockerfile
FROM node:boron

# Create app directory
WORKDIR /usr/src/app

# Install app dependencies
COPY app/package.json .
COPY app/package-lock.json .

RUN npm install --silent

# Bundle app source
COPY ./app .

EXPOSE 3000
CMD [ "npm", "start" ]
``` 
You probably also want to have a `.dockerignore` file to exclude at least the `node_modules` from being synced.
```dockerignore
app/node_modules
```
Now let's test it up:
```console
$ docker build -t docker-flask-react-client .
$ docker run -p 3000:3000 -v $(pwd)/app:/usr/src/app --name react-client docker-flask-react-client  
```
Open your browser to http://localhost:3000/ and you'll see the same app, this time running in the container.
> As always, you can configure your IDE so that you don't have to type these commands all the time.
Don't forget to do your volume binding to enable live reload.

[Have a look at the current codebase](https://github.com/iyp-uk/docker-flask-react-client/tree/initial-set-up) so that we start from the same state.

Now, let's have a little look around inside our `app` directory:

* `public` is where you'll probably do the less. It's not meant to be changed really.
* `src` is where you'll work daily.

In `app/public/index.html`, notice the `div` element with id `root`, this is where our app will be attached to.
While you're in this file, we will add the bootstrap CSS library:
```html
<!-- In app/public/index.html inside the <head> section, add: -->
<link href="//maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-BVYiiSIFeK1dGmJRAkycuHAHRg32OmUcww7on3RYdg4Va+PmSTsz/K68vbdEjh4u" crossorigin="anonymous">
<!-- Update title to: -->
<title>Docker flask react client</title>
```
You can optionally update the `app/public/manifest.json` to your needs.
Under `app/src/index.css` and `app/src/App.css` we will keep it empty for now (you can also remove the files as long as you remove the references to it), just keep files content to: 
```css
/*empty*/
```
Finally, the cool part. In `app/src/App.js`. We will use the same markup as in our `docker-flask-react-users` project:
```JSX
import React, { Component } from 'react';
import './App.css';

class App extends Component {
  render() {
    return (
      <div className="App">
        <div className="container">
          <div className="row">
            <div className="col-md-4">
              <br/>
              <h1>All Users</h1>
              <hr/><br/>
            </div>
          </div>
        </div>
      </div>
    );
  }
}

export default App;
```
Right, that looks a bit empty. Let's connect to our `docker-flask-react-users` service!
```console
## in docker-flask-react-users

## Launch your postgres instance
$ docker run --name postgres -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=app -d postgres:alpine

## Launch your users service
$ docker run -p 5000:5000 --name users --link postgres -e APP_SETTINGS=app.config.DevelopmentConfig -e DATABASE_URL=postgres://postgres:postgres@postgres:5432/app -d docker-flask-react-users-service 

## Recreate db
$ docker exec -it users python manage.py recreate_db


## Running?
$ curl -i localhost:5000/users 
HTTP/1.0 200 OK
Content-Type: application/json
Content-Length: 60
Server: Werkzeug/0.12.2 Python/3.6.2
Date: Sun, 10 Sep 2017 15:10:02 GMT

{
  "data": {
    "users": []
  }, 
  "status": "success"
}
```
So, at this point in time, you have 2 containers running:

* http://localhost:3000/ where your react is
* http://localhost:5000/ where your flask is

#### Add HTTP client in React

There are plenty of clients available for React, each with its strengths and weaknesses. 
We've chosen [axios](https://github.com/mzabriskie/axios) as it supports [javascript promises](https://spring.io/understanding/javascript-promises).

Back in our `docker-flask-react-client`:

```console
$ cd app
$ npm install axios --save
```
We can now use it in our `App.js`, add these methods:
```JSX
constructor() {
  super()
  this.getUsers()
}
getUsers() {
  axios.get(`${process.env.REACT_APP_USERS_SERVICE_URL}/users`)
  .then((res) => { console.log(res); })
  .catch((err) => { console.log(err); })
}
```
The `super()` is required when you have a constructor, which is not always the case.
Also notice the `REACT_APP_USERS_SERVICE_URL` environment variable.
It needs to start with `REACT_APP_` otherwise it [will be ignored](https://github.com/facebookincubator/create-react-app/blob/master/packages/react-scripts/template/README.md#adding-custom-environment-variables).
Rebuild and relaunch with the environment variable set to `http://localhost:5000/users`. Navigate to http://127.0.0.1:3000/ with a console opened. You should see:
```console
XMLHttpRequest cannot load http://localhost:5000/users. No 'Access-Control-Allow-Origin' header is present on the requested resource. Origin 'http://127.0.0.1:3000' is therefore not allowed access.
```
We're denied access as our 2 parts are considered different origins and AJAX requests are not allowed in this context.

Introducing [CORS](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing)!

Back to our `docker-flask-react-users` service again, we will enable CORS support in our API so that the frontend can access it.

```console
$ echo "flask-cors==3.0.3" >> requirements.txt
```
Enable CORS in `create_app()`:
```python
app = Flask(__name__)

# enable CORS
CORS(app)

#...
```
Then check presence of header:
```console
$ curl -I localhost:5000/users 
HTTP/1.0 200 OK
Content-Type: application/json
Access-Control-Allow-Origin: *
Content-Length: 60
Server: Werkzeug/0.12.2 Python/3.6.2
Date: Sun, 10 Sep 2017 15:51:19 GMT
```
Notice header `Access-Control-Allow-Origin: *`. Also make sure the `OPTION` method returns `200` as some browsers do a [pre-flight request](https://developer.mozilla.org/en-US/docs/Glossary/Preflight_request).
```console
$ curl -I -X OPTIONS localhost:5000/users 
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Allow: HEAD, GET, POST, OPTIONS
Access-Control-Allow-Origin: *
Content-Length: 0
Server: Werkzeug/0.12.2 Python/3.6.2
Date: Sun, 10 Sep 2017 16:02:15 GMT
``` 
We've got our `200` and our `Access-Control-Allow-Origin: *` header. We can now move on.

Have another look at your react app browser console at http://127.0.0.1:3000/ and you should now see the response from our `GET` request sent by `axios`.

#### State and lifecycle

> Review [react's official documentation](https://facebook.github.io/react/docs/state-and-lifecycle.html) on state and lifecycle.

We will ensure to place our HTTP calls at the right moment in the component's lifecycle and update our state accordingly.

First, let's define our state in the constructor:
```JSX
  constructor() {
    super()
    this.state = {
      users: []
    }
  }
```
And update our `getUsers` method:
```JSX
  getUsers() {
    axios.get(`${process.env.REACT_APP_USERS_SERVICE_URL}/users`)
    .then((res) => { this.setState({users: res.data.data.users}) })
    .catch((err) => { console.log(err); })
  }
```
Notice how we [set the state](https://facebook.github.io/react/docs/state-and-lifecycle.html#using-state-correctly) here.
Finally, as it's not in our constructor anymore let's add the call to `getUsers` where it belongs, in `componentDidMount`: 
```JSX
  componentDidMount() {
    this.getUsers();
  }
```
Why in `componentDidMount`? 
We want to ensure the component is ready before calling any external API and avoid introducing a [race condition](https://en.wikipedia.org/wiki/Race_condition).

Finally, we can update our `render` method to display the users from the state (yes, it will be in its own component later):
```JSX
// ...
  <h1>All Users</h1>
  <hr/><br/>
  {
    this.state.users.map((user) => {
      return <h4 key={user.id} className="well"><strong>{ user.username }</strong> - <em>{user.created_at}</em></h4>
    })
  }
// ...
``` 
Eventually add some users in http://localhost:5000/users (the flask version) and see them showing up in http://localhost:3000 :)

Notice the date format is different between our flask template and the react one.

#### Creating the user list component

> Make sure you read about [the distinction between Presentational and Container components (aka dumb vs smart or pure vs impure)](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0) here or [here](http://lucybain.com/blog/2016/react-state-vs-pros/).
There's also another interesting read about [setting up your react project structure](https://daveceddia.com/react-project-structure/).

Add a new `components` directory under `app/src` along with a `UsersList.js` file:

```console
$ mkdir app/src/components
$ touch app/src/components/UsersList.js
``` 
Yes, it is a dumb component, as it will simply display a list of users, not more!

```JSX
import React from 'react';

const UsersList = (props) => {
  return (
    <div>
      {
        props.users.map((user) => {
          return <h4 key={user.id} className="well"><strong>{user.username }</strong> - <em>{user.created_at}</em></h4>
        })
      }
    </div>
  )
}

export default UsersList;
```
Update the `App.js` to include this new component:
```JSX
<h1>All Users</h1>
<hr/><br/>
<UsersList users={this.state.users} />
```
Don't forget to include it:
```JSX
import UsersList from './components/UsersList'
```
Phew, that was quite something! Let's summarise where we are now:

* `docker-flask-react`: our main project hasn't changed. [repo](https://github.com/iyp-uk/docker-flask-react/tree/set-up-ci) - diff
* `docker-flask-react-users`: we've added CORS. [repo](https://github.com/iyp-uk/docker-flask-react-users/tree/set-up-cors) - [diff](https://github.com/iyp-uk/docker-flask-react-users/compare/set-up-travis...set-up-cors)
* `docker-flask-react-client`: Shiny new react app is now running in docker! [repo](https://github.com/iyp-uk/docker-flask-react-client/tree/set-up-state) - [diff](https://github.com/iyp-uk/docker-flask-react-client/compare/initial-set-up...set-up-state)

### Adding form in React

In our `docker-flask-react-client` project, We will add a new dumb component, `AddUser.js`. 
It will just display the form and listen to the `App.js` to get his `props`.

We've had to slightly update the `docker-flask-react-users` project to handle the `POST` better and include some documentation.

So, here's where we are now:

* `docker-flask-react`: our main project hasn't changed. [repo](https://github.com/iyp-uk/docker-flask-react/tree/set-up-ci) - diff
* `docker-flask-react-users`: we've updated the POST handling and added some documentation. [repo](https://github.com/iyp-uk/docker-flask-react-users/tree/post-from-react) - [diff](https://github.com/iyp-uk/docker-flask-react-users/compare/set-up-cors...post-from-react)
* `docker-flask-react-client`: Shiny new react app is now running in docker! [repo](https://github.com/iyp-uk/docker-flask-react-client/tree/post-from-react) - [diff](https://github.com/iyp-uk/docker-flask-react-client/compare/set-up-state...post-from-react)

### Order users by descending date 

In our `docker-flask-react-users` project, let's update `get_all_users()` so that it returns a list of users, ordered by descending creation date.

```python
users = User.query.order_by(User.created_at.desc()).all()
```
Run the tests, `test_get_users` fails.
That's right, the position of the users is not quite right.
`testuser2` is created *after* `testuser` so it will be returned *before*.

Let's add the possibility to add users at a chosen date, so that we can import users in bulk if we wanted to.

1. Update our model (`User` class)
1. Update our helper functions in tests (`add_user`)
1. Update our test to ensure ordering (`test_get_users`)

[Review the change](https://github.com/iyp-uk/docker-flask-react-users/pull/3/files) and the build status.

Here's where we are now:

* `docker-flask-react`: no change
* `docker-flask-react-users`: Sort users by creation date descending. [repo](https://github.com/iyp-uk/docker-flask-react-users/tree/sort-users-date-desc) - [diff](https://github.com/iyp-uk/docker-flask-react-users/compare/post-from-react...sort-users-date-desc)
* `docker-flask-react-client`: no change

## Enforcing security

In this chapter, we will:

* Add hashed password for users
* Update our database schema and use migrations
* Use [JWTs (JSON Web Tokens)](https://jwt.io/introduction/) for authentication 
* Implement user authorisation (lock down access to some resources)
* Add routes in React

Here's how our API will look like at the end of this phase:

| Endpoint | HTTP Method | Result | Authenticated? |
|---|---|---|---|
| `/auth/register` | `POST` | Register user | No |
| `/auth/login` | `POST` | Login user | No |
| `/auth/logout` | `GET` | Logout user | Yes |
| `/auth/status` | `GET` | Check user status | Yes |
| `/users` | `GET` | get all users | No | 
| `/users/:id` | `GET` |  get single user | No |
| `/users` | `POST` | add a user | Yes (admin) |
| `/ping` | `GET` | Sanity check | No |

### Set up the new user model

* We want to add passwords
* Know if the user is active (default `True`)
* Unique `email` and `username`

Let's start by writing the appropriate tests in the `docker-flask-react-users` project.

Add a new file `tests/test_users_model.py` which will validate our database schema and add this class along with the appropriate imports:

```python
class TestUserModel(BaseTestCase):
    """Validate our User model."""

    def test_add_user(self):
        user = User(
            username='test',
            email='test@mail.com'
        )
        db.session.add(user)
        db.session.commit()
        self.assertTrue(user.id)
        self.assertEqual(user.username, 'test')
        self.assertEqual(user.email, 'test@mail.com')
        self.assertTrue(user.active)
        self.assertTrue(user.created_at)
```
```python
from app import db
from app.api.models import User
from tests.base import BaseTestCase
```
Run the tests:
```console
docker exec -it users python manage.py test
...
======================================================================
FAIL: test_add_user (test_users_model.TestUserModel)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/src/app/tests/test_users_model.py", line 19, in test_add_user
    self.assertTrue(user.active)
AssertionError: False is not true

----------------------------------------------------------------------
Ran 16 tests in 1.149s

FAILED (failures=1)
```
The active property is set to `False` by default in the model. 

Because we will make changes to our schema, let's use [Flask migrate](https://flask-migrate.readthedocs.io/en/latest/)

add `flask-migrate==2.1.1` to our `requirements.txt` and rebuild your container.

Eventually add a few users to our database, to see the beauty of the migration.

Add support for migrate in our app initialisation.

```python
# app/__init__.py
import os
from flask import Flask
from flask_cors import CORS
from flask_migrate import Migrate
from flask_sqlalchemy import SQLAlchemy

# instantiate the db
db = SQLAlchemy()

# instantiate flask migrate
migrate = Migrate()


def create_app():
    app = Flask(__name__)

    # enable CORS
    CORS(app)

    # set config
    app_settings = os.getenv('APP_SETTINGS')
    app.config.from_object(app_settings)

    # set up extensions
    db.init_app(app)
    migrate.init_app(app, db)

    # register blueprints
    from app.api.views import users_blueprint
    app.register_blueprint(users_blueprint)

    return app
```
As well as new commands for our manager:
```python
# manage.py
from flask_migrate import MigrateCommand
from flask_script import Manager
import unittest
import coverage

from app import create_app, db

COV = coverage.coverage(
    branch=True,
    include='app/*',
    omit=[
        'tests/*'
    ]
)
COV.start()

app = create_app()
manager = Manager(app)

manager.add_command('db', MigrateCommand)


@manager.command
def recreate_db():
# ... shortened...


if __name__ == '__main__':
    manager.run()
```
Now let's prepare the migration:

Create the migration repository:
```console
$ docker exec -it users python manage.py db init
Creating directory /usr/src/app/migrations ... done
Creating directory /usr/src/app/migrations/versions ... done
Generating /usr/src/app/migrations/env.py ... done
Generating /usr/src/app/migrations/README ... done
Generating /usr/src/app/migrations/script.py.mako ... done
Generating /usr/src/app/migrations/alembic.ini ... done
Please edit configuration/connection/logging settings in '/usr/src/app/migrations/alembic.ini' before proceeding.
```
Generate an initial migration
```console
$ docker exec -it users python manage.py db migrate
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.ddl.postgresql] Detected sequence named 'users_id_seq' as owned by integer column 'users(id)', assuming SERIAL and omitting
INFO  [alembic.env] No changes in schema detected.
```

Apply migration to the database (at this point there should be no change):
```console
$ docker exec -it users python manage.py db upgrade
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
```

OK, we can now make changes to our model.

Set `username` and `email` to be unique and `active` to `True` by default:
```python
import datetime
from app import db


class User(db.Model):
    __tablename__ = "users"
    id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    username = db.Column(db.String(128), unique=True, nullable=False)
    email = db.Column(db.String(128), unique=True, nullable=False)
    active = db.Column(db.Boolean(), default=True, nullable=False)
    created_at = db.Column(db.DateTime, nullable=False)

    def __init__(self, username, email, created_at=datetime.datetime.utcnow()):
        self.username = username
        self.email = email
        self.created_at = created_at
```
Now run the migration commands again (apart from `init`... hehe):
```console
$ docker exec -it users python manage.py db migrate
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.ddl.postgresql] Detected sequence named 'users_id_seq' as owned by integer column 'users(id)', assuming SERIAL and omitting
INFO  [alembic.autogenerate.compare] Detected added unique constraint 'None' on '['email']'
INFO  [alembic.autogenerate.compare] Detected added unique constraint 'None' on '['username']'
  Generating /usr/src/app/migrations/versions/8a00ce0aebfb_.py ... done
```

```console
$ docker exec -it users python manage.py db upgrade
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 8a00ce0aebfb, empty message
```

You can inspect the tables in postgres, you'll see that the new schema has been applied successfully. 
Note that the `migrations` folder has to be kept under version control.

Run the tests, they pass!

We can add more specific tests to cover duplicate email and username.

```python
def test_add_duplicate_username(self):
    user = User(
        username='test',
        email='test@mail.com'
    )
    db.session.add(user)
    db.session.commit()
    duplicate_user = User(
        username='test',
        email='test@mail2.com'
    )
    db.session.add(duplicate_user)
    self.assertRaises(IntegrityError, db.session.commit)

# Add a similar one for email, test_add_duplicate_email
```

Let's do some refactor so that we can reuse the `add_user` function from the `test_users.py` file. 
Move it to a `tests/utils.py` file and update imports accordingly.

> Have you noticed every time we run tests our database is wiped out?
This is because the tests recreate and remove database which is the same as the current environment every time they run.

Let's fix that by setting another database just for tests.

In our `app/config.py` we will specify another database for these tests to run against.

```python
SQLALCHEMY_DATABASE_URI = os.environ.get('TEST_DATABASE_URL')
```
So we need to define it. Where will this run? Let's add another postgres service for this purpose so that we keep things separated.

We also now need to update the main project to cover this new service.

### Add password field

... and encrypt it! Introducing [flask-bcrypt](https://flask-bcrypt.readthedocs.io/en/latest/)

Add `flask-bcrypt==0.7.1` to the `requirements.txt` and update app initialisation:

```python
# instantiate the extensions
db = SQLAlchemy()
migrate = Migrate()
bcrypt = Bcrypt()
```
And
```python
# set up extensions
db.init_app(app)
bcrypt.init_app(app)
migrate.init_app(app, db)
```
Then add a test in `tests/test_users_model.py` to ensure passwords are random:
```python
def test_passwords_are_random(self):
    user_one = add_user('test', 'test@test.com', 'test')
    user_two = add_user('test2', 'test@test2.com', 'test')
    self.assertNotEqual(user_one.password, user_two.password)
``` 
Update all occurrences to the `add_user` test helper function to pass a password.
Update the `test_add_user` function to validate password (`self.assertTrue(user.password)`)  

Run the migrations as we will shortly update our db model:
```console
$ docker exec -it users python manage.py db migrate
$ docker exec -it users python manage.py db upgrade
```
Note that if you've restarted or changed your postgres service, you may want to indicate the last status of the migrations is assumed ok before running the commands above: 
```console
$ docker exec -it users python manage.py db stamp head  
``` 
Now apply your schema changes to our model:
```python
password = db.Column(db.String(255), nullable=False)
```
and
```python
self.password = bcrypt.generate_password_hash(
    password, current_app.config.get('BCRYPT_LOG_ROUNDS')
).decode()
```
with updated inclusions:
```python
from flask import current_app
from app import db, bcrypt
```
Now you can run the tests and change your code until they pass.

Have you noticed how slow they now run? 
[Hashing the password with bcrypt](https://en.wikipedia.org/wiki/Bcrypt) takes time...

Let's have a closer look:
`flask-bcrypt` sets the number of rounds to `12` by default: `self._log_rounds = app.config.get('BCRYPT_LOG_ROUNDS', 12)`

Let's reduce that for our `Development` and `Testing` environments, and increase it to `13` elsewhere.

In your `app/config.py`
```python
class BaseConfig:
    """Base configuration"""
    # Your existing config here.
    BCRYPT_LOG_ROUNDS = 13
    
class DevelopmentConfig(BaseConfig):
    """Development configuration"""
    # Your existing config here.
    BCRYPT_LOG_ROUNDS = 4
    
class TestingConfig(BaseConfig):
    """Testing configuration"""
    # Your existing config here.
    BCRYPT_LOG_ROUNDS = 4
```

Measure the difference:

Before:
```console
$ time docker exec -it users python manage.py test
.... your tests passing
docker exec -it users python manage.py test  0.02s user 0.01s system 0% cpu 11.632 total
```
After:
```console
$ time docker exec -it users python manage.py test
.... your tests passing
docker exec -it users python manage.py test  0.02s user 0.01s system 0% cpu 3.485 total
```
Almost 4 times faster!

Here's where we are now:

* `docker-flask-react`: no change
* `docker-flask-react-users`: Add passwords to users. [repo](https://github.com/iyp-uk/docker-flask-react-users/tree/set-up-users-passwords) - [diff](https://github.com/iyp-uk/docker-flask-react-users/compare/sort-users-date-desc...set-up-users-passwords)
* `docker-flask-react-client`: no change

### Setting up JWT

It's now time to set up our [JSON Web Tokens](https://jwt.io/introduction/), using [PyJWT](http://pyjwt.readthedocs.io/en/latest/index.html)

We're now still in the `docker-flask-react-users` project.

Add the package `pyjwt==1.5.3` to your `requirements.txt` and redeploy your container.

Add a test to ensure we issue properly encoded tokens (`encode_auth_token` doesn't exist yet...):
```python
def test_encode_auth_token(self):
    """Ensures the auth token is encoded"""
    user = add_user('test', 'test@test.com', 'test')
    auth_token = user.encode_auth_token(user.id)
    self.assertIsInstance(auth_token, bytes)
```
So let's add our `encode_auth_token` method!
```python
@staticmethod
def encode_auth_token(user_id):
    """Encodes the JWT based on user id
    :rtype: bytes|string
    :param user_id: the user id
    :return: encoded JWT
    """
    try:
        payload = {
            # Expiration Time Claim (exp)
            'exp': datetime.datetime.utcnow() + datetime.timedelta(seconds=1),
            # Issued At Claim (iat)
            'iat': datetime.datetime.utcnow(),
            # Subject (sub)
            'sub': user_id
        }
        return jwt.encode(payload, current_app.config.get('SECRET_KEY'), algorithm='HS256')
    except Exception:
        return 'Could not encode token.'
```
Notice the use of [Registed claim names](https://tools.ietf.org/html/rfc7519#section-4.1)

Run the tests again and they should now pass.

Great, let's now decrypt that. Again, first we will add a test for it.
```python
def test_decode_auth_token(self):
    """Ensures the auth token can be decoded"""
    user = add_user('test', 'test@test.com', 'test')
    auth_token = user.encode_auth_token(user.id)
    self.assertTrue(user.decode_auth_token(auth_token), user.id)
```
add the method to the `User` class:
```python
@staticmethod
def decode_auth_token(auth_token):
    """Decodes auth token
    """
    try:
        payload = jwt.decode(auth_token, current_app.config.get('SECRET_KEY'), algorithms='HS256')
        return payload['sub']
    except jwt.ExpiredSignatureError:
        return 'Expired token, please login again.'
    except jwt.InvalidTokenError:
        return 'Invalid token'
    except Exception:
        return 'Could not decode token.'
```
OK, our token expires after 1 second, it's great for tests but totally unusable in real conditions.
Let's make it more flexible, so that we allow it to be valid for 1h on all environments but testing.
Add a config parameter, and add it to the `test_config.py` to ensure it's equal to `1` in testing, and `3600` elsewhere.

Here's how the code looks like now:

* `docker-flask-react`: no change
* `docker-flask-react-users`: Add JWT support. [repo](https://github.com/iyp-uk/docker-flask-react-users/tree/set-up-jwt) - [diff](https://github.com/iyp-uk/docker-flask-react-users/compare/set-up-users-passwords...set-up-jwt)
* `docker-flask-react-client`: no change

> Interesting notes about JWT:
>* [why you must *not* use jwt for sessions](http://cryto.net/~joepie91/blog/2016/06/13/stop-using-jwt-for-sessions/)
>* [why you must *not* use jwt for sessions (continued)](http://cryto.net/~joepie91/blog/2016/06/19/stop-using-jwt-for-sessions-part-2-why-your-solution-doesnt-work/)  

### Set up authentication routes

Time to concentrate on this table below now that we've got all prerequisites satisfied:

| Endpoint | HTTP Method | Result | Authenticated? |
|---|---|---|---|
| `/auth/register` | `POST` | Register user | No |
| `/auth/login` | `POST` | Login user | No |
| `/auth/logout` | `GET` | Logout user | Yes |
| `/auth/status` | `GET` | Check user status | Yes |

We will create a new blueprint for 'auth' routes, and we then need to reorganise our project a little:

1. Create a new `views` directory at the same level as `templates`
    ```console
    $ mkdir app/api/views
    ```
1. Move the current `views.py` into `app/api/views/users.py`
1. Create empty `app/api/views/__init__.py`
1. Update references:
    * In `app/__init__.py`, `from app.api.views import users_blueprint` becomes `from app.api.views.users import users_blueprint`
    * In the moved `users.py`, the `template_folder` is now one level above, relatively, so `users_blueprint = Blueprint('users', __name__,  template_folder='../templates')`
1. Run your tests and ensure everything passes

All good? Great! Let's now add our new `auth` blueprint:

```console
$ touch app/api/views/auth.py
```
Let's now edit this file with basic blueprint definition:
```python
from flask import Blueprint

auth_blueprint = Blueprint('auth', __name__)
```
And register it in our `app/__init__.py`:
```python
from app.api.views.auth import auth_blueprint
app.register_blueprint(auth_blueprint)
```
Add a new test file:
```console
$ touch tests/test_auth.py
```
Add now a basic silly test in `tests/test_auth.py`:
```python
from tests.base import BaseTestCase


class TestAuth(BaseTestCase):
    def test_user_registration(self):
        self.assertTrue(True)
```
Run tests. They shall pass, including the one we just created.

Now let's write the actual test for the user registration.
We need to ensure:

* Response is a success
* A message indicating the user is registered
* Token is retrieved 
* `application/json` response content-type
* HTTP Status code indicates creation:
    * `2xx` type as it's a success
    * but as we'll create one `201` is more appropriate
    * [more on this](https://httpstatuses.com/))

Here's how it looks like:
```python
class TestAuth(BaseTestCase):
    def test_user_registration(self):
        self.assertTrue(True)
        with self.client:
            response = self.client.post(
                '/auth/register',
                data=json.dumps(USER_BASIC),
                content_type='application/json'
            )
            data = json.loads(response.data.decode())
            self.assertTrue(data['status'] == 'success')
            self.assertTrue(data['message'] == 'Successfully registered.')
            self.assertTrue(data['auth_token'])
            self.assertTrue(response.content_type == 'application/json')
            self.assertEqual(response.status_code, 201)
```
So what's this new constant `USER_BASIC`?
Because we're often reusing the same sets of "user definitions", we've wrapped them in `tests/constants.py` as follow:
```python
USER_BASIC = dict(username='test', email='test@test.com', password='test')
USER_INVALID_STRUCTURE = dict(email='test@test.com')
USER_NO_USERNAME = dict(username='', email='test@test.com', password='test')
USER_NO_EMAIL = dict(username='test', email='', password='test')
USER_NO_PASSWORD = dict(username='test', email='test@test.com', password='')
```
It makes the test code more readable in some ways, but obscures the real parameters being used.

Let's now add more tests cases, what if the user has:

* no username
* no password
* no email
* a malformed request (missing json keys) 
* is a duplicate? 

We can write all these tests!

This part is a little long, there are many tests attached, so we'll just point the most significant steps:

* [Basis](https://github.com/iyp-uk/docker-flask-react-users/commit/b45eeb5771acef43eaba551083be6da455105834)
* [Refactor and add registration](https://github.com/iyp-uk/docker-flask-react-users/commit/cad4f0f48a8a8b6d97d7c77e78406bd3f1553992)
* [Add login](https://github.com/iyp-uk/docker-flask-react-users/commit/4d5180ac6b126ee39e094905e6136259eaac56a0)
* [Add logout](https://github.com/iyp-uk/docker-flask-react-users/commit/b053a6ec2e1c39436fe1dfbe41b541e31b928db7)
* [Add status](https://github.com/iyp-uk/docker-flask-react-users/commit/d66a776992154c1cc9b833e076ab16184d4f8ed0)

Here's how the code looks like now:

* `docker-flask-react`: no change
* `docker-flask-react-users`: Add Authentication routes. [repo](https://github.com/iyp-uk/docker-flask-react-users/tree/set-up-auth-routes) - [diff](https://github.com/iyp-uk/docker-flask-react-users/compare/set-up-jwt...set-up-auth-routes)
* `docker-flask-react-client`: no change

Just a look on our test coverage:

```console
$ docker exec -it users python manage.py cov
... test details here...
an 43 tests in 11.380s

OK
Coverage Summary:
Name                        Stmts   Miss Branch BrPart  Cover
-------------------------------------------------------------
app/__init__.py                22     10      0      0    55%
app/api/__init__.py             0      0      0      0   100%
app/api/models.py              72      6     12      3    89%
app/api/views/__init__.py       0      0      0      0   100%
app/api/views/auth.py          44      0      8      0   100%
app/api/views/users.py         38      0      8      0   100%
app/config.py                  20      0      0      0   100%
-------------------------------------------------------------
TOTAL                         196     16     28      3    92%
```
92%, it's not too bad. Notice how our `auth` and `users` views are fully covered. 
Some improvements could be done on the `models` though. 

Well, who's gonna consume all these routes? Let's update our `docker-flask-react-client` project!

## React routing

> Before we dive into this, you're invited to read a brief history of [routing on the client side](http://krasimirtsonev.com/blog/article/deep-dive-into-client-side-routing-navigo-pushstate-hash)
> You're also invited to [have a look at some routing examples](https://reacttraining.com/react-router/web/example/basic) to get familiar with routing in React.
> Or even [take the quickstart guide](https://reacttraining.com/react-router/web/guides/quick-start) if you're really new to it.

So, let's now switch to our `docker-flask-react-client` project.

We will create an `/about` route to start with.

> You need to have the `docker-flask-react-users` service running to continue.
Indeed, this is one client, but without a backend, there's only so much it can do, so make sure your project `docker-flask-react-users` is also running.
If you're reading through, it should be the case as we just implemented the authentication routes on it.

Install router package:

```console
$ cd app && npm install --save react-router-dom
```

This will update our `package.json` and `package-lock.json` inside the `app` directory and give us access to some key components:

* [`Route`](https://reacttraining.com/react-router/web/api/Route)
    * Renders some UI when location matches the route's path
* [`Router`](https://reacttraining.com/react-router/web/api/Router)
    * Keeps track of routes and keeps locations and UI in sync. We will use the [`BrowserRouter`](https://reacttraining.com/react-router/web/api/BrowserRouter) here as it makes use of [HTML5's history API](https://html5demos.com/history/) to manipulate browser's history without actually reloading the page.
* [`Switch`](https://reacttraining.com/react-router/web/api/Switch)
    * Renders the first of the routes within it to match the location
* [`Link`](https://reacttraining.com/react-router/web/api/Link)
    * Links to navigate using the routing system
    * See its friend [`NavLink`](https://reacttraining.com/react-router/web/api/NavLink) for styled active links (used for menus)

Example:
```JSX
<Router>
  <div>
    <Route exact path="/" component={Home}/>
    <Route path="/news" component={NewsFeed}/>
  </div>
</Router>
<Link to="/about">About</Link>
```

Build and start (or if you're in webstorm, deploy) your container:
```console
$ docker build -t docker-flask-react-client .
$ docker run -p 3000:3000 -v $(pwd)/app:/usr/src/app --name react-client -e REACT_APP_USERS_SERVICE_URL=http://localhost:5000 docker-flask-react-client  
```

### Wrap the whole app in router

First, let's add our `Router`, which will take care of our **entire** `App`.
So in `app/src/index.js` change
```JSX
ReactDOM.render(<App />, document.getElementById('root'));
```
to:
```JSX
ReactDOM.render((
  <Router>
    <App/>
  </Router>
), document.getElementById('root'));
registerServiceWorker();
```
Add also the appropriate import (and watch for the "renaming"):
```JSX
import { BrowserRouter as Router } from 'react-router-dom';
```

### Add the about component

Now let's create our new `About.js` component:
```console
$ touch app/src/components/About.js
```
And fill in:
```JSX
import React from 'react';

const About = () => {
  return (
    <div>
      <h1>About</h1>
      <hr/>
      <p>Add something relevant here.</p>
    </div>
  )
}

export default About;
```
Now call it from the `app/src/App.js`, right after the list of users:
```JSX
...
<UsersList users={this.state.users} />
<About/>
...
```
And add the `import` accordingly:
```JSX
import About from './components/About'
```

Refresh the page (should have been done automatically anyway as we're in dev mode) and verify that you can see:

* The form to add users
* A list of users, if you added any
* The about component we just created

> Why does everything appear on the same page?
We're building an SPA (Single Page Application), and we haven't defined any routes, so every component is on the same page!
Now you can see `Route` coming into play!

### Add routes

So we will add:

* The about route so that the about information is visible just there
* List all users on the homepage (`/`) and on this page only
* Get rid of the rendering of the form in the `App.js` component as it's outdated (no password field). We will come back to it later.

The innermost part of the `App.js`'s `render` method now looks like
```jsx
<Route path={`/about`} component={About}/>
<Route exact path={`/`} render={() => <UsersList users={this.state.users} />}/>
<Link to={`/about`}>About</Link>
```
And `UsersList.js` gets a "title" now (which was in `App.js` before):
```jsx
const UsersList = (props) => {
  return (
    <div>
      <h1>All Users</h1>
      <hr/>
      {
        props.users.map((user) => {
          return <h4 key={user.id} className="well"><strong>{user.username }</strong> - <em>{user.created_at}</em></h4>
        })
      }
    </div>
  )
}
```
Refresh, click on the "about" link and the back button of your browser and verify no network activity is visible in your browser's console.

### React bootstrap

Let's add some bootstrap fanciness to the mix, with React support.
From the `app` directory:
```console
$ npm install -S react-bootstrap react-router-bootstrap
+ react-router-bootstrap@0.24.4
+ react-bootstrap@0.31.3
```
This will allow us to have a nice Navbar, bootstrap style.

So let's add it to our `components` directory:
```console
$ touch app/src/components/NavBar.js
```
```jsx
import React from 'react';
import { Navbar, Nav, NavItem } from 'react-bootstrap';
import { LinkContainer } from 'react-router-bootstrap';

const NavBar = (props) => (
  <Navbar inverse collapseOnSelect>
    <Navbar.Header>
      <Navbar.Brand>
        <span>{props.title}</span>
      </Navbar.Brand>
      <Navbar.Toggle />
    </Navbar.Header>
    <Navbar.Collapse>
      <Nav>
        <LinkContainer exact to="/">
          <NavItem eventKey={1}>Home</NavItem>
        </LinkContainer>
        <LinkContainer to="/about">
          <NavItem eventKey={2}>About</NavItem>
        </LinkContainer>
        <LinkContainer to="/status">
          <NavItem eventKey={3}>User Status</NavItem>
        </LinkContainer>
      </Nav>
      <Nav pullRight>
        <LinkContainer to="/register">
          <NavItem eventKey={1}>Register</NavItem>
        </LinkContainer>
        <LinkContainer to="/login">
          <NavItem eventKey={2}>Log In</NavItem>
        </LinkContainer>
        <LinkContainer to="/logout">
          <NavItem eventKey={3}>Log Out</NavItem>
        </LinkContainer>
      </Nav>
    </Navbar.Collapse>
  </Navbar>
)

export default NavBar
```
And include it in our `App.js` and remove the `Link` to `About` we set previously.
```jsx
<div className="App">
    <NavBar title={this.state.title}/>
    ...
```
Notice all routes we defined in `docker-flask-react-users` are there, we will just display the relevant ones in a moment.

But before we get there, let's add a login / register form component.
```console
touch app/src/components/LoginRegister.js
```
```jsx
import React from 'react';

const LoginRegister = (props) => {
  return (
    <div>
      <h1>{props.formType}</h1>
      <hr/><br/>
      <form onSubmit={(event) => props.handleUserFormSubmit(event)}>
        {props.formType === 'Register' &&
        <div className="form-group">
          <input
            name="username"
            className="form-control input-lg"
            type="text"
            placeholder="Enter a username"
            required
            value={props.formData.username}
            onChange={props.handleFormChange}
          />
        </div>
        }
        <div className="form-group">
          <input
            name="email"
            className="form-control input-lg"
            type="email"
            placeholder="Enter an email address"
            required
            value={props.formData.email}
            onChange={props.handleFormChange}
          />
        </div>
        <div className="form-group">
          <input
            name="password"
            className="form-control input-lg"
            type="password"
            placeholder="Enter a password"
            required
            value={props.formData.password}
            onChange={props.handleFormChange}
          />
        </div>
        <input
          type="submit"
          className="btn btn-primary btn-lg btn-block"
          value="Submit"
        />
      </form>
    </div>
  )
}

export default LoginRegister
```
Add it to the `App.js`, along with a `Switch`:
```jsx
<Switch>
  <Route exact path={`/`} render={() => <UsersList users={this.state.users} />}/>
  <Route exact path={`/login`} render={() => (
    <LoginRegister
      formType={'Login'}
      formData={this.state.formData}
    />
  )} />
  <Route exact path={`/register`} render={() => (
    <LoginRegister
      formType={'Register'}
      formData={this.state.formData}
    />
  )} />
  <Route path={`/about`} component={About}/>
</Switch>
```
Add properties to the state:
```jsx
this.state = {
  users: [],
  username: '',
  email: '',
  title: 'iyp-uk/docker-flask-react-client',
  formData: {
    username: '',
    email: '',
    password: ''
  }
}
```
Import component
```jsx
import LoginRegister from './components/LoginRegister'
```
And you're good to go! Test it out, so that you see the appropriate fields in each case.

## React tests

### Jest

[Running tests in React](https://github.com/facebookincubator/create-react-app/blob/master/packages/react-scripts/template/README.md#running-tests) 
uses [Jest](https://facebook.github.io/jest/) as its test runner. 

It provides useful utilities such as:

* [Expect](https://facebook.github.io/jest/docs/en/expect.html#content)
* [Mock functions](https://facebook.github.io/jest/docs/en/mock-function-api.html#content) also known as "spies"

### Enzyme

[Enzyme](https://github.com/airbnb/enzyme) is a library provided by [Airbnb](https://github.com/airbnb) 
and provides the ability to test components in isolation.

It provides useful utilities such as:

* [Shallow rendering](https://github.com/airbnb/enzyme/blob/master/docs/api/shallow.md): `shallow` for when you want to test a component in isolation of potential child components
* [Full DOM rendering](https://github.com/airbnb/enzyme/blob/master/docs/api/mount.md): `mount` for when you need the full lifecycle (i.e `componentDidMount`)
* [Static Rendering](https://github.com/airbnb/enzyme/blob/master/docs/api/render.md): `render` for when you want to analyse the resulting HTML of your component

> Official documentation for [enzyme](http://airbnb.io/enzyme/)

## End to end testing

[Comparison of Selenium web driver and TestCafe](http://onpathtesting.com/automated-testing-tool-testcafe-or-selenium-webdriver/)
