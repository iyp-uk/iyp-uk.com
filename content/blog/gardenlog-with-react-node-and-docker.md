---
title: "Gardenlog With React, Node and Docker"
date: 2017-07-31T12:01:35+01:00
tags: [ "nodejs", "reactjs", "docker", "aws", "Elasticsearch" ]
---

This will show the rewriting of a project using different technologies.

## What is Gardenlog?

[Gardenlog](https://gardenlog.webmee.fr) is a project which: 

 * monitors garden (even if it could be anything), gathering all sorts of meteorological information
 * allows users to interact with it
   
IYP UK has built it using several technologies:

 * Backend: Python and Elasticsearch
 * Frontend: Angular 4 with Material
 * Hosting: All hosted on AWS, apart from the physical part, running on a raspberry pi
   
## Scope of this project

In this project, we will rewrite a good part of the Gardenlog project to demonstrate use of the following:
 
 * Backend: NodeJS and Elasticsearch
 * Frontend: ReactJS with Redux
 * Docker and Kubernetes
 * Hosting: Similar to original, but using different AWS services
 
## Part 1: Backend

A service currently generates logs every hour, or on demand thanks to the API. 
Logs are stored in AWS Elasticsearch and the API sits on the Raspberry Pi so that it can use device capabilities.

Now this is fine for demonstration purposes but it wouldn't scale nicely.
We would like to deport the API into the Cloud, using docker containers. 
Only the interface with the device will remain on the Pi, using MQTT, lowering the work done by it.

See detailed article at [Nodejs Docker And Aws]({{<ref "nodejs-docker-and-aws.md">}}).

## Part 2: Frontend
 
Frontend is currently in Angular 4, which is fine. 
But Read with Redux provides a convenient store for state which we would like to use.

## Part 3: Hosting

Adjust AWS services accordingly.