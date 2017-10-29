---
title: "TDD Django Docker"
date: 2017-10-29T10:06:26Z
---

## What you will learn here

We will go through the [current Django tutorial](https://docs.djangoproject.com/en/1.11/intro/tutorial01/) with a particular focus on [TDD]({{< relref "clean-code.md#tdd" >}}) and using [Docker](https://www.docker.com/). 

> This tutorial presents a polls application.

## Prerequisites

Check you have the required software on your machine.

```
$ docker -v
Docker version 17.09.0-ce, build afdb6d4
```
```
$ docker-compose -v
docker-compose version 1.16.1, build 6d1ac21
```

## Getting started

We will be running [Django](https://www.djangoproject.com/) along with a [PostgreSQL](https://www.postgresql.org/) database.

This means we will have 2 services:

* A container running Django, based on Python
* A container running Postgres

hence our `docker-compose` requirement.

> Django supports and uses [SQLite](https://www.sqlite.org/) in its tutorial.

### Project foundations

Create the project's directory:
```console
$ mkdir tdd-django-docker
$ cd tdd-django-docker
$ git init
```
Add relevant files:
```console
$ touch Dockerfile requirements.txt docker-compose.yml
```
Some details here:

* `Dockerfile` describes our Django app
* `requirements.txt` lists requirements for this Django app
* `docker-compose.yml` describes the combination of services to run the app

We need `django` and `psycopg2` to communicate with postgres.
In `requirements.txt`, add:
```
Django==1.11
psycopg2==2.7.3
```
Now, the container will have to pick these requirements, so that the django part can be run.

In `Dockerfile`:

```Dockerfile
FROM python:3

EXPOSE 8000

WORKDIR /usr/src/app

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
```
Finally, in `docker-compose.yml`:


You're now able to create your django project:
```console
$ docker-compose run web django-admin startproject pollapp .
```
> Notice the trailing dot which will generate the app in the directory defined in the `Dockerfile`, avoiding Django's additional directory level. 

This will generate the files for your django `pollapp` project:
```console
$ tree
.
├── Dockerfile
├── README.md
├── docker-compose.yml
├── manage.py
├── pollapp
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
└── requirements.txt

1 directory, 9 files
```
At this point, you will want to instruct your app about the postgres database, so go ahead and edit the `DATABASES` setting from `pollapp/settings.py`:

```python
# Database
# https://docs.djangoproject.com/en/1.11/ref/settings/#databases

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'postgres',
        'USER': 'postgres',
        'HOST': 'db',
        'PORT': 5432,
    }
}
```
the `NAME` and `USER` are provided by the default [`postgres` docker image](https://hub.docker.com/_/postgres/)

> We're simplifying quite a bit here, on a normal project you'd make these environment variables. 

You're now ready to start building this app. Let's run it:

```console
$ docker-compose up
```
You should now be able to navigate to http://127.0.0.1:8000/ as per the Django tutorial and see a “Welcome to Django” page, in pleasant, light-blue pastel. It worked!

> Check the [codebase at this stage](https://github.com/iyp-uk/tdd-django-docker/tree/app-set-up)

## Creating the polls app

> We're following the Django's tutorial here.

Run the following command:
```console
$ docker-compose run web python manage.py startapp polls
```
Notice without the docker context it would have been:
```console
$ python manage.py startapp polls
```
> See how similar they are? Just instruct the `web` service from `docker-compose` to `run` the command you supply.
Couldn't be easier :)

That creates a new `polls` directory which in context is like:

```console
$ tree
.
├── Dockerfile
├── LICENSE
├── README.md
├── docker-compose.yml
├── manage.py
├── pollapp
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── polls
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── migrations
│   │   └── __init__.py
│   ├── models.py
│   ├── tests.py
│   └── views.py
└── requirements.txt
```

### Baby steps

We want to add a `/polls` route.

But we're TDD here, remember? `RED > GREEN > REFACTOR`

So we will add a test to check that this route exists, make sure it fails (RED) and write just enough to make it pass (GREEN).

#### Defining tests

First let's confirm you can run tests as it will be our first one.

```console
$ docker-compose run web python manage.py test
Starting tdddjangodocker_db_1 ... done
Creating test database for alias 'default'...
System check identified no issues (0 silenced).

----------------------------------------------------------------------
Ran 0 tests in 0.000s

OK
Destroying test database for alias 'default'...
```
OK, the tests can run, there are none for now but not for long!

You may have noticed a `polls/tests.py`, that's where we'll add them!

So let's write the **minimum amount of code** to have this test verify our requirements and fail (RED phase).

```python
from django.test import TestCase


class PollsView(TestCase):

    def test_route_to_polls(self):
        response = self.client.get('/polls/')
        self.assertEqual(response.status_code, 200)
```
Now run the tests:
```console
docker-compose run web python manage.py test
Starting tdddjangodocker_db_1 ... done
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
F
======================================================================
FAIL: test_route_to_polls (polls.tests.PollsView)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/src/app/polls/tests.py", line 7, in test_route_to_polls
    self.assertEqual(response.status_code, 200)
AssertionError: 404 != 200

----------------------------------------------------------------------
Ran 1 test in 0.023s

FAILED (failures=1)
Destroying test database for alias 'default'...
```
Fails! fantastic, we can now move on to the next phase: writing **just enough** production code to get the test to pass (GREEN phase).

We will then add:
 
* a [view](https://docs.djangoproject.com/en/1.11/topics/http/views/) to deal with the logic
* an [url dispatcher](https://docs.djangoproject.com/en/1.11/topics/http/urls/) to route requests to the view

Follow the tutorial and:

* Update `polls/views.py`
* Add `polls/urls.py`
* Update `pollapp/urls.py`

Running tests again, they pass:
```console
$ docker-compose run web python manage.py test
Starting tdddjangodocker_db_1 ... done
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
.
----------------------------------------------------------------------
Ran 1 test in 0.021s

OK
Destroying test database for alias 'default'...
```
So we're now in the REFACTORING phase. 
Anything to refactor? Not quite, let's move on.

#### Run the tests in CI

So we've got our first test. That's cool but we want to enforce it so that any change / PR runs them too and not rely on developers' good will to run them.
Introducing [travis](https://travis-ci.org/)! There are plenty of them, this is just an example here.

Add a `.travis.yml` at the root of the project:

```console
$ touch .travis.yml
```
And fill it in with:
```yaml
sudo: required

services:
  - docker

env:
  global:
    - DOCKER_COMPOSE_VERSION=1.14.0

before_install:
  - sudo rm /usr/local/bin/docker-compose
  - curl -L https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/docker-compose-`uname -s`-`uname -m` > docker-compose
  - chmod +x docker-compose
  - sudo mv docker-compose /usr/local/bin

before_script:
  - docker-compose up --build -d

script:
  - docker-compose run web python manage.py test

after_script:
  - docker-compose down
```
[Read the travis documentation](https://docs.travis-ci.com/user/getting-started/) for further instructions.

You can even [add a fancy badge to your project's readme file](https://docs.travis-ci.com/user/status-images/).

> Check the [codebase at this stage](https://github.com/iyp-uk/tdd-django-docker/tree/add-polls-app) and the [diff with previous checkpoint](https://github.com/iyp-uk/tdd-django-docker/compare/app-set-up...add-polls-app).

## Database set up

Our database connection is already set, but we will now:

* Update Internationalisation settings
* Prepare `migrate` (see below)

```console
$ docker-compose run web python manage.py migrate
Starting tdddjangodocker_db_1 ... done
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying sessions.0001_initial... OK
```

### Adding models

Update `polls/models.py` and `pollapp/settings.py` as per tutorial.

Finally create new migrations based on the schema changes we've added.

```console
$ docker-compose run web python manage.py makemigrations polls
Starting tdddjangodocker_db_1 ... done
Migrations for 'polls':
  polls/migrations/0001_initial.py
    - Create model Choice
    - Create model Question
    - Add field question to choice
```
Notice it's now created `polls/migrations/0001_initial.py` which defines your schema update.

You can also read this in a SQL version:
```console
$ docker-compose run web python manage.py sqlmigrate polls 0001
Starting tdddjangodocker_db_1 ... done
BEGIN;
--
-- Create model Choice
--
CREATE TABLE "polls_choice" ("id" serial NOT NULL PRIMARY KEY, "choice_text" varchar(200) NOT NULL, "votes" integer NOT NULL);
--
-- Create model Question
--
CREATE TABLE "polls_question" ("id" serial NOT NULL PRIMARY KEY, "question_text" varchar(200) NOT NULL, "pub_date" timestamp with time zone NOT NULL);
--
-- Add field question to choice
--
ALTER TABLE "polls_choice" ADD COLUMN "question_id" integer NOT NULL;
CREATE INDEX "polls_choice_question_id_c5b4b260" ON "polls_choice" ("question_id");
ALTER TABLE "polls_choice" ADD CONSTRAINT "polls_choice_question_id_c5b4b260_fk_polls_question_id" FOREIGN KEY ("question_id") REFERENCES "polls_question" ("id") DEFERRABLE INITIALLY DEFERRED;
COMMIT;
```

> By running `makemigrations`, you’re telling Django that you’ve made some changes to your models (in this case, you’ve made new ones) and that you’d like the changes to be stored as a migration.

> Read more about [migrations](https://docs.djangoproject.com/en/1.11/topics/migrations/).

Reading these is a way to crosscheck that the changes we want to apply are correct.

We can now apply these changes to the database:

```console
$ docker-compose run web python manage.py migrate              
Starting tdddjangodocker_db_1 ... done
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, polls, sessions
Running migrations:
  Applying polls.0001_initial... OK
```

> [Read more about `manage.py` commands](https://docs.djangoproject.com/en/1.11/ref/django-admin/).

### Using the API

Yes, even with docker you can run the python command line.
```console
$ docker-compose run web python manage.py shell 
Starting tdddjangodocker_db_1 ... done
Python 3.6.2 (default, Jul 24 2017, 19:47:39) 
[GCC 4.9.2] on linux
Type "help", "copyright", "credits" or "license" for more information.
(InteractiveConsole)
>>> 
```
Go through the tutorial's steps.

### Make the polls app modifiable in admin

Still follow the tutorial...

> Check the [codebase at this stage](https://github.com/iyp-uk/tdd-django-docker/tree/set-up-database) and the [diff with previous checkpoint](https://github.com/iyp-uk/tdd-django-docker/compare/add-polls-app...set-up-database).

## Adding more views

We now want to add more routes, such as:

* questions details
* questions results
* votes 

Let's start by adding more tests for all these routes.

Add the following tests under `polls/tests.py` after the one we already have:

```python
    def test_question_details(self):
        response = self.client.get('/polls/5/')
        self.assertEqual(response.status_code, 200)

    def test_question_results_details(self):
        response = self.client.get('/polls/5/results/')
        self.assertEqual(response.status_code, 200)

    def test_question_vote_details(self):
        response = self.client.get('/polls/5/vote/')
        self.assertEqual(response.status_code, 200)
```
Run the tests:
```console
docker-compose run web python manage.py test
Starting tdddjangodocker_db_1 ... done
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
FFF.
======================================================================
FAIL: test_question_details (polls.tests.PollsView)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/src/app/polls/tests.py", line 12, in test_question_details
    self.assertEqual(response.status_code, 200)
AssertionError: 404 != 200

======================================================================
FAIL: test_question_results_details (polls.tests.PollsView)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/src/app/polls/tests.py", line 16, in test_question_results_details
    self.assertEqual(response.status_code, 200)
AssertionError: 404 != 200

======================================================================
FAIL: test_question_vote_details (polls.tests.PollsView)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/src/app/polls/tests.py", line 20, in test_question_vote_details
    self.assertEqual(response.status_code, 200)
AssertionError: 404 != 200

----------------------------------------------------------------------
Ran 4 tests in 0.051s

FAILED (failures=3)
Destroying test database for alias 'default'...
```
OK, we can now add some *basic* stuff to these routes (follow as per tutorial).

Run the tests again, they pass.

Well that's ok to test the routes, but there's not much logic in there.

### Write views that actually do something

#### The index (`/polls`)

We want the `index` to return a list of the 5 more recent questions.

So we can define for instance a test case like:

```python
def test_polls_lists_5_most_recent(self):
    questions = [
        "How long is a piece of string?",
        "How are you today?",
        "What have you done yesterday?",
        "What's your mother's maiden name?",
        "When is your birthday?",
        "Are you happy?",
        "What is the meaning of life?"
    ]
    for question in questions:
        add_question(question)

    response = self.client.get('/polls/')
    for question in questions[2:]:
        self.assertContains(response, question)

    for question in questions[:2]:
        self.assertNotContains(response, question)
```
This creates 7 questions, checks that the 5 most recent appear in the list and that 2 other which are less recent don't appear.

The corresponding production code is in the tutorial.

We can even assert more when it comes down to the template level and check for markup.

Likewise, we can also test when there are no posts:
```python
def test_polls_list_returns_message_when_no_questions(self):
    response = self.client.get('/polls/')
    self.assertContains(response, escape("No polls are available."), html=True)
```
> Notice here the HTML version, and it's better to escape what you pass in just in case it contains single quotes (apostrophe)

#### A questions details page

So here, we could start by checking that an existing question returns `200` and a `404` exception is raised, resulting in a `404` page when no question is found at details page:

```python
def test_question_details(self):
    q = add_question()
    response = self.client.get(f'/polls/{q.id}/')
    self.assertEqual(response.status_code, 200)

def test_question_details_does_not_exist(self):
    q = add_question()
    response = self.client.get(f'/polls/{q.id + 1}/')
    self.assertEqual(response.status_code, 404)
```

But also be more specific, like ensuring a given template is used or check for related entities (here relation with `choice`):
```python
def test_question_details(self):
    q = add_question()
    q.choice_set.create(choice_text='Not much', votes=0)
    q.choice_set.create(choice_text='The sky', votes=0)
    response = self.client.get(f'/polls/{q.id}/')
    title = '<h1>%s</h1>' % escape(q.question_text)
    self.assertContains(response, title, html=True)
    self.assertContains(response, "Not much", html=True)
    self.assertContains(response, "The sky", html=True)
    self.assertTemplateUsed(response, 'polls/detail.html')
```

> Check the [codebase at this stage](https://github.com/iyp-uk/tdd-django-docker/tree/add-routes) and the [diff with previous checkpoint](https://github.com/iyp-uk/tdd-django-docker/compare/set-up-database...add-routes).

This can go on and on. 
