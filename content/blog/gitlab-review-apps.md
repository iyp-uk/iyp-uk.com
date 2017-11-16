---
title: "Gitlab Review Apps"
date: 2017-11-14T16:20:25Z
tags: [ "Continuous deployment", "Gitlab", "Review Apps", "docker" ]
---

Let's have a look at [Gitlab's review Apps](https://about.gitlab.com/features/review-apps/), 
available for available in GitLab.com, GitLab Community Edition, and GitLab Enterprise Edition.

## So what's the plan?

In this particular example, we want to demonstrate a use case of Gitlab's review apps.
We will also demonstrate that in the context of a trivial docker-based python app, which essentially returns an "Hello world".

[Gitlab CI](https://docs.gitlab.com/ee/ci/README.html) allows us to do plenty of things, and in particular we will want to:

1. Run tests (there are no tests in our dummy app but we'll bring a "stage" for it in case you want to fill them with your needs)
1. Build the docker image
1. Deploy the app to heroku

> The source code is available on [gitlab](https://gitlab.com/iyp-uk/gitlab-review-apps)

## Let's get started

First of all, let's have a look at our app: [Here it is](https://gitlab.com/iyp-uk/gitlab-review-apps/tree/app-init).

That's a Flask app, just [greeting you with a nice "Hello world!"](https://gitlab.com/iyp-uk/gitlab-review-apps/blob/app-init/app.py).

As you may see, there's no definition of any CI/CD pipeline. It's just the code.

## Add Continuous integration support

Adding a `.gitlab-ci.yml` will enable this functionality. 
So let's add one with basic stuff so that we can validate it's being picked up.

```yaml
myfirstjob:
  script: echo "It works!"
```

[Commit and push](https://gitlab.com/iyp-uk/gitlab-review-apps/commit/70678236c9dffb3416337fd2f984da010aa2705d).
Gitlab comes with free runners to run this, you may just have to wait for them to become available.
You can also set up your [own runner(s)](https://docs.gitlab.com/runner/) to increase availability.

Either way, the result is the same: 
The [job](https://docs.gitlab.com/ee/ci/yaml/README.html#jobs) has fired and you can [see its results](https://gitlab.com/iyp-uk/gitlab-review-apps/-/jobs/40425183).  

Codebase now looks like [this](https://gitlab.com/iyp-uk/gitlab-review-apps/tree/set-up-ci).

> Read more:
> * [Getting started with CI/CD in Gitlab](https://docs.gitlab.com/ee/ci/quick_start/README.html)
> * [.gitlab-ci.yml reference](https://docs.gitlab.com/ee/ci/yaml/README.html)
> * [Gitlab's CI docs](https://docs.gitlab.com/ee/ci/README.html)

## Configure a more advanced CI

OK, let's now move on to something more useful.

We can set a sequence of [stages](https://docs.gitlab.com/ee/ci/yaml/README.html#stages). 
This is called a [pipeline](https://docs.gitlab.com/ee/ci/pipelines.html)

Let's ask this runner to build our docker image and run tests.
We will then set up 2 stages: `build` and `test` in our `.gitlab-ci.yml`.

```yaml
stages:
  - build
  - test
```

Our docker images can be stored in the [Gitlab container registry](https://gitlab.com/help/user/project/container_registry).
This is the one we will use here, but it could be any registry.

For the first time, you may want to try to push to the registry manually. 
For all other times, it will be done by the pipeline.

So according the Gitlab's registry tab (from your local machine, within your project):

Login:
```console
$ docker login registry.gitlab.com
```
Then build and push your image:
```console
$ docker build -t registry.gitlab.com/iyp-uk/gitlab-review-apps .
$ docker push registry.gitlab.com/iyp-uk/gitlab-review-apps
``` 

This tags your image with `latest` and pushes it to the registry where you [can see it](https://gitlab.com/iyp-uk/gitlab-review-apps/container_registry)

Let's now [add these steps to our build stage](https://gitlab.com/help/ci/docker/using_docker_build.md#using-the-gitlab-container-registry):

```yaml
services:
  - docker:dind

variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_NAME

before_script:
  - docker login -u gitlab-ci-token -p $CI_JOB_TOKEN $CI_REGISTRY

build:
  stage: build
  script:
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
```

Where:

* `$CI_REGISTRY_IMAGE`: Address of the registry tied to the specific project (Would resolve to `registry.gitlab.com/iyp-uk/gitlab-review-apps` here)
* `$CI_COMMIT_REF_NAME`: Branch or tag name for which project is built
* `$CI_JOB_TOKEN`: Token used for authenticating with the GitLab Container Registry
* `$CI_REGISTRY`: Address of GitLab's Container Registry (resolves to `registry.gitlab.com`)

And: 

* `services`: Specifies docker image to use (here it's using Docker in Docker `dind`)
* `variables`: Defines variables for later use in jobs 
* `before_script`: Command which runs before all jobs 

We will keep the previous job's script to run during the `test` stage for now, so that our complete definition looks like:

```yaml
image: docker:latest
services:
  - docker:dind

stages:
  - build
  - test

variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_NAME

before_script:
  - docker login -u gitlab-ci-token -p $CI_JOB_TOKEN $CI_REGISTRY

build:
  stage: build
  script:
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG

test:
  stage: test
  script: echo "It works!"
``` 

Commit and push:

* [Check the pipeline](https://gitlab.com/iyp-uk/gitlab-review-apps/pipelines/13966918)
* [Check the registry](https://gitlab.com/iyp-uk/gitlab-review-apps/container_registry): Notice a new image tagged `master` has been created as we pushed from the `master` branch

Codebase now looks like [this](https://gitlab.com/iyp-uk/gitlab-review-apps/tree/set-up-pipelines).

> * [Read more about these variables](https://docs.gitlab.com/ee/ci/variables/README.html)
> * [Read .gitlab-ci.yml reference](https://docs.gitlab.com/ee/ci/yaml/README.html) for `services`, `variables` and `before_script` definitions

## Deploying the app

So the purpose of this article is to deploy our app for Merge Requests. 
It's now time to deploy!

We will deploy in different ways:

* Deploy to staging when something gets merged into master
* Deploy to production on tags
* Deploy to a temporary environment for Merge Requests

> We will deploy to [Heroku](https://www.heroku.com/) for that, so go ahead and create an account if you don't already have one.

In Heroku, create 2 "apps":

* `{name}-gitlab-review-apps-staging` 
* `{name}-gitlab-review-apps-production`

Again, to start with, we will do the first steps from local.
Navigate to your project's directory and:

1. [Install Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli) if you don't already have it
1. Test it out
    ```console
    $ heroku --version
    heroku-cli/6.14.36 (darwin-x64) node-v8.9.1
    ```

1. Login with 
    ```console
    $ heroku login
    ```

1. Login Heroku's container registry
    ```console
    $ heroku container:login
    ```

1. Tag your image within Heroku's registry
    ```console
    $ docker tag registry.gitlab.com/iyp-uk/gitlab-review-apps:latest registry.heroku.com/gitlab-review-apps-staging/web:latest
    ```

1. Push to Heroku's registry which deploys it automatically
    ```console
    $ docker push registry.heroku.com/gitlab-review-apps-staging/web:latest
    ```

So with our current codebase it doesn't go through as Heroku has some specific requirements, in particular:

* No `EXPOSE` in `Dockerfile`
* Assignment of a random `$PORT` for web server which container has to bind to

So let's make a few changes to test that out.

Tag and push again to Heroku:

```console
$ docker tag registry.gitlab.com/iyp-uk/gitlab-review-apps:latest registry.heroku.com/gitlab-review-apps-staging/web:latest
$ docker push registry.heroku.com/gitlab-review-apps-staging/web:latest  
```

See the app in action: https://gitlab-review-apps-staging.herokuapp.com/

We can then also push it to the production app:

```console
$ docker tag registry.gitlab.com/iyp-uk/gitlab-review-apps:latest registry.heroku.com/gitlab-review-apps-production/web:latest
$ docker push registry.heroku.com/gitlab-review-apps-production/web:latest  
```
And then visit the production app running: https://gitlab-review-apps-production.herokuapp.com/

OK so now that we know how that works locally, let's make it integrated into our pipeline.
Essentially the steps will remain the same, apart from the way we login.

Login for CI tools on the heroku registry:

```console
$ docker login --username=_ --password=$(heroku auth:token) registry.heroku.com
```

The password will be set to a [private variable](https://docs.gitlab.com/ee/ci/variables/#secret-variables) (Notice they aren't as private as you may think)

From your local project root, get that token with:

```console
$ heroku auth:token
replies-with-a-token
```
And set it in Gitlab under Settings > CI/CD > Secret Variables under `HEROKU_AUTHENTICATION_TOKEN`.

Here's what the job looks like:

```yaml

deploy_staging:
  stage: deploy
  script:
    - docker pull $IMAGE_TAG
    - docker login --username=_ --password=$HEROKU_AUTHENTICATION_TOKEN registry.heroku.com
    - docker tag $IMAGE_TAG $HEROKU_IMAGE
    - docker push $HEROKU_IMAGE
  environment:
    name: staging
    url: https://gitlab-review-apps-staging.herokuapp.com/
  only:
    - master
```
And we've also updated the `variables` section with:
```yaml
variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_NAME
  HEROKU_IMAGE: registry.heroku.com/gitlab-review-apps-staging/web:latest
```
> Here's what the pipeline looks like: https://gitlab.com/iyp-uk/gitlab-review-apps/pipelines/13974492

It builds, tests and then deploys to Heroku on the staging environment when things are pushed on `master` branch.

Codebase now looks like [this](https://gitlab.com/iyp-uk/gitlab-review-apps/tree/deploy-to-heroku-staging).

Notice the `staging` environment is now available under [CI/CD > Environments](https://gitlab.com/iyp-uk/gitlab-review-apps/environments) where you have a direct link to it.

> Read more about [Gitlab's environments](https://docs.gitlab.com/ee/ci/environments.html)

## Deploying Review Apps

Here we are finally! Deploying to our temporary environments on Merge Requests.

The idea is to create an heroku app on the fly, then push the container there.

To create apps from Heroku CLI:

```console
$ heroku create <yourappname>
```

Fine, but within your CI pipeline, you'd have to install the Heroku CLI, 
so there's another way of doing it using the [Heroku API](https://devcenter.heroku.com/articles/platform-api-quickstart#calling-the-api).

So here's the principle:

```yaml
deploy_review_app:
  stage: review
  script:
    - docker pull $IMAGE_TAG
    - docker login --username=_ --password=$HEROKU_AUTHENTICATION_TOKEN registry.heroku.com
    - apk add --no-cache curl
    - >-
        curl
        -H "Accept: application/vnd.heroku+json; version=3"
        -H "Authorization: Bearer $HEROKU_AUTHENTICATION_TOKEN"
        -H "Content-Type: application/json"
        -X POST
        -d '{"name":"'"gitlab-review-apps-$CI_PIPELINE_ID"'"}'
        https://api.heroku.com/apps
    - docker tag $IMAGE_TAG $HEROKU_IMAGE_REVIEW
    - docker push $HEROKU_IMAGE_REVIEW
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://gitlab-review-apps-$CI_PIPELINE_ID.herokuapp.com/
  only:
    - branches
  except:
    - master
```

and under `variables`:

```yaml
variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG
  HEROKU_IMAGE_STAGING: registry.heroku.com/gitlab-review-apps-staging/web:latest
  HEROKU_IMAGE_REVIEW: registry.heroku.com/gitlab-review-apps-$CI_PIPELINE_ID/web:latest
```

With `curl` we can create an app on the fly without any dependency as the only thing we need it the Heroku Authentication Token.

This triggers on any `branches` (so no `tags`), except `master`.
The app is build and pushed to Heroku which suffice to get it live at https://gitlab-review-apps-$CI_PIPELINE_ID.herokuapp.com/

And because we still have

```yaml
deploy_staging:
  stage: deploy
  script:
    - docker pull $IMAGE_TAG
    - docker login --username=_ --password=$HEROKU_AUTHENTICATION_TOKEN registry.heroku.com
    - docker tag $IMAGE_TAG $HEROKU_IMAGE_STAGING
    - docker push $HEROKU_IMAGE_STAGING
  environment:
    name: staging
    url: https://gitlab-review-apps-staging.herokuapp.com/
  only:
    - master
```

Whenever the Merge Request gets merged into Master, it will deploy the update to staging automatically

## Deploying tags to production

Whenever we tag and push to gitlab, get the tag deployed to `production` environment automatically.

That means: 

1. fetching the `master` image (we don't need to rebuild it)
1. tag it appropriately (like with the version number)
1. tag it as `latest`
1. Deploy to production

Something like that should do:

```yaml

deploy_production:
  stage: deploy
  script:
    - docker pull $CI_REGISTRY_IMAGE:master
    # Push to Gitlab registry with the tag name as well as 'latest'.
    - docker tag $CI_REGISTRY_IMAGE:master $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_NAME
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_NAME
    - docker tag $CI_REGISTRY_IMAGE:master $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:latest
    # Push to Heroku registry and to production environment.
    - docker login --username=_ --password=$HEROKU_AUTHENTICATION_TOKEN registry.heroku.com
    - docker tag $CI_REGISTRY_IMAGE:latest $HEROKU_IMAGE_PRODUCTION
    - docker push $HEROKU_IMAGE_PRODUCTION
  environment:
    name: production
    url: https://gitlab-review-apps-production.herokuapp.com/
  only:
    - tags
```

## Summary and overview of what's been done

* Full `.gitlab-ci.yml` defintion can be found [on gitlab](https://gitlab.com/iyp-uk/gitlab-review-apps/blob/master/.gitlab-ci.yml) 
* [Example of a review app in the context of a Merge Request](https://gitlab.com/iyp-uk/gitlab-review-apps/merge_requests/3/diffs)
    * Along with its [auto-created environment and running review app](https://gitlab-review-apps-14018076.herokuapp.com/) (there's a link for that on the Merge Request, it may take some time to appear though)
* [Staging app](https://gitlab-review-apps-staging.herokuapp.com/)
* [Production app](https://gitlab-review-apps-production.herokuapp.com/)

## What's next?

* Delete the app when the MR gets merged.
