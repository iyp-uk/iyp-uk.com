---
title: "Kong Custom Plugin"
date: 2017-12-04T12:27:18Z
tags: [ "kong", "microservices", "api" ]
---

## Purpose

So you're using [Kong](https://getkong.org/) and have fun with it.
But the provided plugins are not enough for your needs? You're demanding!

We will show here how to write your own and distribute it.

> In this article we will build a custom plugin named `aws-lambda-status-code`.
You can find its [source code on Github](https://github.com/iyp-uk/aws-lambda-status-code) or [get it from Luarocks](https://luarocks.org/modules/miaoulafrite/kong-plugin-aws-lambda-status-code).

## Development

### Setting up

> Also read the [official documentation](https://getkong.org/docs/latest/plugin-development/).

The easiest way to get it up and running quickly is to use the [`kong-vagrant`](https://github.com/Kong/kong-vagrant) repository.
There is a section dedicated to custom plugins.

It could be interesting to set up another directory, like mapping `./custom-plugins` into the VM's `/custom-plugins` where you'd have all your custom plugins.

You could then have for instance something like:

```console
$ vagrant@vagrant-ubuntu-trusty-64:/custom-plugins$ tree  .
.
└── aws-lambda-status-code
    ├── kong
    │   └── plugins
    │       └── aws-lambda-status-code
    │           ├── handler.lua
    │           ├── schema.lua
    │           └── v4.lua
    ├── kong-plugin-aws-lambda-status-code-0.1.0-1.rockspec
    └── README.md

4 directories, 5 files
```
> Just add other plugins as you see fit in this directory.

### Installing your plugin

Just `cd` in the directory where it lives

```console
$ vagrant@vagrant-ubuntu-trusty-64:/custom-plugins/aws-lambda-status-code$ luarocks make
kong-plugin-aws-lambda-status-code 0.1.0-1 is now installed in /usr/local (license: MIT)
``` 

### Enabling your plugin

It's useful to just use the `KONG_CUSTOM_PLUGINS` environment variable to get your plugin enabled.

```console
$ export KONG_CUSTOM_PLUGINS=aws-lambda-status-code
```

Start Kong (or restart)
```console
$ kong start --run-migrations
```

### Add your plugin to an API

Add an API:
```console
$ curl -i -X POST \
  --url http://localhost:8001/apis/ \
  --data 'name=myapi' \
  --data 'upstream_url=http://mockbin.org/request' \
  --data 'uris=/myapi'
```

Add your plugin to this API:
```console
$ curl -i -X POST \
  --url http://localhost:8001/apis/myapi/plugins/ \
  --data 'name=aws-lambda-status-code' \
  --data-urlencode "config.aws_key=AWS_KEY" \
  --data-urlencode "config.aws_secret=AWS_SECRET" \
  --data "config.aws_region=AWS_REGION" \
  --data "config.function_name=LAMBDA_NAME" \
  --data "config.forward_request_body=true" \
  --data "config.forward_request_uri=true" \
  --data "config.forward_request_headers=true" \
  --data "config.forward_request_method=true" \
  --data "config.config.unhandled_status=500"
```

### Working on your plugin

You can edit the code directly from the `/custom-plugins/aws-lambda-status-code` directory, 
and changes are being picked up after you restart kong,
so you'll probably do a lot of 

```console
$ kong restart; curl -i localhost:8000/myapi/foo/bar # plus whatever parameters you need in your API
```

## Share it

Happy with your plugin? Share it on [Luarkocks](https://luarocks.org)!

### Generate an API key on Luarocks

1. Login or register or luarocks
1. In your account settings, go to [API keys](https://luarocks.org/settings/api-keys)
1. Click "Generate new key"
1. Copy that key, you'll need it later

### Deploying your code to github

If your plugin is not version controlled yet, just initialise a git repo in it.

In your `.rockspec` file you probably specify a git repository like so

```lua
source = {
  -- these are initially not required to make it work
  url = "git://github.com/iyp-uk/aws-lambda-status-code",
  tag = "0.1.0"
}
```
So you'd have to tag your code and push it to the specified url.

> As the comment mentioned in the default `.rockspec` template suggests, it's not initially required to make it work.
But trying to push didn't work without it...

### Pushing to Luarocks

```console
$ luarocks upload --api-key YOURKEY kong-plugin-aws-lambda-status-code-0.1.0-1.rockspec
Sending kong-plugin-aws-lambda-status-code-0.1.0-1.rockspec ...

Error: A JSON library is required for this command. Failed loading 'cjson', 'dkjson', and 'json'. Use 'luarocks search <partial-name>' to search for a library and 'luarocks install <name>' to install one.
```

Alright, let's add one then:
```console
$ luarocks install luajson
...
```

And again:
```console
$ luarocks upload --api-key YOURKEY kong-plugin-aws-lambda-status-code-0.1.0-1.rockspec
Sending kong-plugin-aws-lambda-status-code-0.1.0-1.rockspec ...
Will create new module (kong-plugin-aws-lambda-status-code)
Packing kong-plugin-aws-lambda-status-code
Cloning into 'aws-lambda-status-code'...
remote: Counting objects: 13, done.
remote: Compressing objects: 100% (11/11), done.
remote: Total 13 (delta 0), reused 12 (delta 0), pack-reused 0
Receiving objects: 100% (13/13), 7.54 KiB | 7.54 MiB/s, done.
Note: checking out '5782750c076eefbed2751fe4474222d47a251ddb'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by performing another checkout.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -b with the checkout command again. Example:

  git checkout -b <new-branch-name>

  adding: kong-plugin-aws-lambda-status-code-0.1.0-1.rockspec (deflated 57%)
  adding: aws-lambda-status-code/ (stored 0%)
  adding: aws-lambda-status-code/LICENSE (deflated 41%)
  adding: aws-lambda-status-code/kong/ (stored 0%)
  adding: aws-lambda-status-code/kong/plugins/ (stored 0%)
  adding: aws-lambda-status-code/kong/plugins/aws-lambda-status-code/ (stored 0%)
  adding: aws-lambda-status-code/kong/plugins/aws-lambda-status-code/handler.lua (deflated 67%)
  adding: aws-lambda-status-code/kong/plugins/aws-lambda-status-code/schema.lua (deflated 76%)
  adding: aws-lambda-status-code/kong/plugins/aws-lambda-status-code/v4.lua (deflated 67%)
  adding: aws-lambda-status-code/README.md (deflated 51%)
  adding: aws-lambda-status-code/kong-plugin-aws-lambda-status-code-0.1.0-1.rockspec (deflated 57%)
Sending /path/to/kong-custom-plugins/aws-lambda-status-code/kong-plugin-aws-lambda-status-code-0.1.0-1.src.rock ...

Done: http://luarocks.org/modules/miaoulafrite/kong-plugin-aws-lambda-status-code
```

So this creates a source rock file (`.src.rock`) and sends it to your account.

> See [Upload](https://github.com/luarocks/luarocks/wiki/upload) for further details.

## Install it somewhere

Once you've published it to the luarocks plugin repository, you can now from anywhere do

```console
$ luarocks install kong-plugin-aws-lambda-status-code
```

And add it to your custom plugins.
This time, instead of the environment variable, you probably want to add it to your kong configuration file

```console
$ grep custom_plugins /etc/kong/kong.conf.default 
#custom_plugins =                # Comma-separated list of additional plugins
```

So copy this `kong.conf.defaults` to `kong.conf`, add your plugin in there and where you start kong, just specify this configuration file with

```console
$ kong start -c /etc/kong/kong.conf 
```

## Resources

* Source code: https://github.com/iyp-uk/aws-lambda-status-code
* Luarocks plugin page: https://luarocks.org/modules/miaoulafrite/kong-plugin-aws-lambda-status-code
