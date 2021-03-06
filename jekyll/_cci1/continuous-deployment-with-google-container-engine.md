---
layout: classic-docs
title: Continuous Deployment with Google Container Engine
short-title: Continuous Deployment with GKE
categories: [how-to]
description: "How to continuously deploy your application using Google Container Engine, Google Container Registry, and CircleCI."
---

## Introduction
Google's Cloud Platform (GCP) is a huge, scalable hosting platform that can be 
great for deploying an application for today's needs and tomorrow's. CircleCI 
Docs has a brief [GCP overview]( {{ site.baseurl }}/1.0/google-cloud-platform/) doc 
however in this guide, we'll go over continuously deploying an example Rails 
app to Google Container Engine (GKE). General information on continuous 
deployment with CircleCI can be found 
[here]( {{ site.baseurl }}/1.0/introduction-to-continuous-deployment/).

### Tools & Concepts Overview

#### Google Container Engine (GKE)
[GKE](https://cloud.google.com/container-engine/) is Google's service to run 
[Docker](https://www.docker.com/) containers, all powered by Google's 
open-source software, [Kubernetes](http://kubernetes.io/) (K8s for short). This 
guide takes advantage of newer K8s features (was tested with Kubernetes v1.2).

* [Container Cluster](https://cloud.google.com/container-engine/docs/clusters/) - GKE 
offers clusters that simply a regular K8s cluster for you. It automatically 
runs a master endpoint and any nodes you need across Google Compute Engine 
(GCE) VMs.
* [Deployments](http://kubernetes.io/docs/user-guide/deployments/) - We'll be 
using K8s deployments in this guide. This is a newer concept that replaces or 
wraps pods and replication controllers/replica sets. Deployments make it easy 
to manage which pods are running, how many, and upgrading containers smartly.
* [gcloud](https://cloud.google.com/sdk/gcloud/) - command-line tool for GCP. 
This is pre-installed for us on CircleCI.
* [kubectl](http://kubernetes.io/docs/user-guide/kubectl-overview/) - command-line 
tool for K8s. We install this via `gcloud`.

#### Google Container Registry (GCR)
Docker images need to be hosted somewhere for easy deployment. Most people will 
be familiar with the free registry provided by Docker itself, 
[Docker Hub](https://hub.docker.com/). Among several alternatives, Google 
provides their own registry to store your images with direct support for gcloud, 
GKE and K8s.

## Setup

### Prerequisites

This guide makes the following assumptions:

1. Your project source code is hosted on a CircleCI compatible repository.
1. You already have a GCP project registered. Keep the project name handy.
1. There is an already running cluster on GKE. Keep the cluster name handy.
1. You're familiar with Docker.

### Project Settings on CircleCI
This guide will be using the example project 
[docker-hello-google](https://github.com/circleci/docker-hello-google) on GitHub.

#### Environment Variables
This project will use several environment variables (envar). Most will be set 
in `circle.yml` which we go over below. There is one that we will set via 
[Project Settings]( {{ site.baseurl }}/1.0/environment-variables/#setting-environment-variables-for-all-commands-without-adding-them-to-git) 
due to the need to keep it a secret. This is the 
[GCP Service Account](https://cloud.google.com/storage/docs/authentication#service_accounts). 
This key is generated by Google as a JSON file. Follow the 
[Authentication with Google Cloud Platform]( {{ site.baseurl }}/1.0/google-auth/) 
guide to learn how to generate and base64 encode the key. Create a new envar 
with the name "GCLOUD_SERVICE_KEY" and use the base64 encoded JSON key from 
Google as the value.

#### circle.yml
We'll cover the relevant snippets of the `circle.yml` file here. You can find 
the complete file [here](https://github.com/circleci/docker-hello-google/blob/master/circle.yml).

```
machine:
  environment:
    PROJECT_NAME: circle-ctl-test
    CLUSTER_NAME: docker-hello-google-cluster
    CLOUDSDK_COMPUTE_ZONE: us-central1-f
```

In the `machine` section we set some envars for use to use with gcloud later. 
You should replace the values of these three variables for your specific project.

```
dependencies:
  pre:
    - sudo /opt/google-cloud-sdk/bin/gcloud --quiet components update --version 120.0.0
    - sudo /opt/google-cloud-sdk/bin/gcloud --quiet components update --version 120.0.0 kubectl
    - echo $GCLOUD_SERVICE_KEY | base64 --decode -i > ${HOME}/gcloud-service-key.json
    - sudo /opt/google-cloud-sdk/bin/gcloud auth activate-service-account --key-file ${HOME}/gcloud-service-key.json
    - sudo /opt/google-cloud-sdk/bin/gcloud config set project $PROJECT_NAME
    - sudo /opt/google-cloud-sdk/bin/gcloud --quiet config set container/cluster $CLUSTER_NAME
    - sudo /opt/google-cloud-sdk/bin/gcloud config set compute/zone ${CLOUDSDK_COMPUTE_ZONE}
    - sudo /opt/google-cloud-sdk/bin/gcloud --quiet container clusters get-credentials $CLUSTER_NAME
    #...
```

`kubectl` is installed via `gcloud` so that we can interact with the cluster 
directly. The JSON key is used to authenticate with GCP, settings are set, and 
the last line uses `gcloud` to download creds for the cluster so that `kubectl` 
can use them during the deployment phase.

```
dependencies:
  pre:
    #...
    - docker build -t us.gcr.io/${PROJECT_NAME}/hello:$CIRCLE_SHA1 .
    - docker tag us.gcr.io/${PROJECT_NAME}/hello:$CIRCLE_SHA1 us.gcr.io/${PROJECT_NAME}/hello:latest
```

The Docker image for this example Rails app is build using the Git commit hash 
as a tag. A common practice. The second line also applies the 'latest' tag to 
the image for convenience.

```
deployment:
  prod:
    branch: master
    commands:
      - ./deploy.sh
```

Finally, if the build passes and the current branch was the master branch, we 
run deploy.sh to do the actual deployment work.

#### deploy.sh
Our deployment script for this repository consist of pushing the newly created 
Docker image out to the registry, then updating the K8s deployment to use the 
new image. The following snippets walk through the process. The full 
`deploy.sh` file can be found on 
[GitHub](https://github.com/circleci/docker-hello-google/blob/master/deploy.sh).

```
sudo /opt/google-cloud-sdk/bin/gcloud docker push us.gcr.io/${PROJECT_NAME}/hello
```

Typically when you push a Docker image to a registry, you use the `docker push` 
command. While we can still do that here, this `gcloud` command provides a 
convenient wrapper that will handle authentication and push the image all at 
once. Our new image is now available in GCR for all our GCP infrastructure to 
access.

```
sudo chown -R ubuntu:ubuntu /home/ubuntu/.kube
```

Since we used `gcloud` as root (via sudo), the newly installed `kubectl` binary 
doesn't have the right permissions. So we fix it. We're looking to simplify 
the use of `gcloud` and `kubectl` in the future.

```
kubectl patch deployment docker-hello-google -p '{"spec":{"template":{"spec":{"containers":[{"name":"docker-hello-google","image":"us.gcr.io/circle-ctl-test/hello:'"$CIRCLE_SHA1"'"}]}}}}'

```

This command utilizes the patch subcommand of `kubectl` to make a change to our 
running deployment on the fly. This is extremely useful in a CI environment. 
Normally you might use `kubectl edit deployment` which will open your 
deployment spec file in your default text editor.

This command finds the line that specifies the image to use for our container, 
and replaces it with the image tag of the image just built. The K8s deployment 
then intelligently upgrades the cluster by shutting down old containers and 
starting up-to-date ones.

## Summary & Notes
We've seen how we can use CircleCI to deploy an app, in this examples, a Rails 
app, to Google Container Engine while using Google Container Registry to store 
the images. An example of a green (passing) build for the example project can 
be found [here](https://circleci.com/gh/circleci/docker-hello-google/43).

### The Example Project Doesn't Run on GKE
If you were following along with the 
[example project on GitHub](https://github.com/circleci/docker-hello-google), 
you may have noticed that the Rails app doesn't run on GKE. This is because the 
environment variable 'SECRET_KEY_BASE' is needed. In `circle.yml` this is set 
for the test to pass. This also needs to be set in your GKE deployment. There 
is more than one way to do this. For this project, the deployment spec was 
manually edited to set that variable using the command 
`kubectl edit deployment`.

### How To Setup The Project and Cluster.
This is best left to GCP and Kubernetes docs:

* Getting Started - <https://cloud.google.com/container-engine/docs/before-you-begin>
* Container Cluster Operations - <https://cloud.google.com/container-engine/docs/clusters/operations>
* Your First Deployment (WordPress example) - <https://cloud.google.com/container-engine/docs/tutorials/hello-wordpress>
