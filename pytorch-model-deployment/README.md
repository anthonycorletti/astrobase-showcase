# PyTorch ML Model Deployment

Deploying pre-built PyTorch models in an API on EKS with Astrobase.

This example is less involved than [this one](../pytorch-ml-lifecycle-full). There is no need for GPUs in this exercise.

1. [Install Docker](#install-docker)
1. [Create an Image Classifier API with PyTorch](create-an-image-classifier-api-with-pytorch)
1. [Create your Dockerfile](create-your-dockerfile)
1. [Build and run the container](build-and-run-the-container)
1. [Deploying to EKS with Astrobase](deploying-to-eks-with-astrobase)

## Install Docker

Visit https://docs.docker.com/get-docker/ to install docker.

Validate that docker is installed and working properly by running the following commands and validating the output

```sh
$ docker version; docker run hello-world
Client: Docker Engine - Community
 Cloud integration: 1.0.9
 Version:           20.10.5
 API version:       1.41
 Go version:        go1.13.15
 Git commit:        55c4c88
 Built:             Tue Mar  2 20:13:00 2021
 OS/Arch:           darwin/amd64
 Context:           default
 Experimental:      true

Server: Docker Engine - Community
 Engine:
  Version:          20.10.5
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.13.15
  Git commit:       363e9a8
  Built:            Tue Mar  2 20:15:47 2021
  OS/Arch:          linux/amd64
  Experimental:     true
 containerd:
  Version:          1.4.3
  GitCommit:        269548fa27e0089a8b8278fc4fc781d7f65a939b
 runc:
  Version:          1.0.0-rc92
  GitCommit:        ff819c7e9184c13b7c2607fe6c30ae19403a7aff
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
b8dfde127a29: Pull complete
Digest: sha256:308866a43596e83578c7dfa15e27a73011bdd402185a84c5cd7f32a88b501a24
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

## Create an Image Classifier API with PyTorch

See [app.py](./app.py)

Download your image classifing index

```sh
$ curl https://raw.githubusercontent.com/raghakot/keras-vis/master/resources/imagenet_class_index.json -o imagenet_class_index.json
```

## Create your Dockerfile

See [Dockerfile](./Dockerfile)

## Build and run the container

```sh
docker build -t mlapi .; docker run -p 5000:5000 -it mlapi
```

Now let's test our model

You should see something like this

```sh
$ curl -s -X POST -d '{"url": "https://i.imgur.com/NZHKADF.jpg"}' -H 'Content-Type: application/json' http://:5000/predict | jq
[
  "golden_retriever",
  66.94181823730469
]
```

## Deploying to EKS with Astrobase

Now let's deploy our application to EKS via astrobase.

First, deploy your container to hub.docker.com.

We're tagging the container as `astrobase/mlapi` because we own astrobase on hub.docker.com – but you should tag it as your own user, e.g. `my-dockerhub-username/mlapi`.

```sh
$ docker build -t astrobase/mlapi .
$ docker push astrobase/mlapi
```

Initialize astrobase, check commands, and make sure you have a profile that references AWS credentials

```sh
$ astrobase init
$ astrobase check commands
$ astrobase profile current
```

Create a cluster

```sh
$ export CLUSTER_ROLE_ARN=$(aws iam list-roles | jq -r '.Roles[] | select(.RoleName == "AstrobaseEKSRole") | .Arn')
$ export NODE_ROLE_ARN=$(aws iam list-roles | jq -r '.Roles[] | select(.RoleName == "AstrobaseEKSNodegroupRole") | .Arn')
$ export SUBNET_ID_0=$(aws ec2 describe-subnets --query 'Subnets[].SubnetId[]' | jq -r '.[0]')
$ export SUBNET_ID_1=$(aws ec2 describe-subnets --query 'Subnets[].SubnetId[]' | jq -r '.[1]')
$ export SECURITY_GROUP=$(aws ec2 describe-security-groups --query 'SecurityGroups[].GroupId' | jq -r '.[0]')
$ astrobase apply -f cluster.yaml -v "CLUSTER_ROLE_ARN=$CLUSTER_ROLE_ARN NODE_ROLE_ARN=$NODE_ROLE_ARN SUBNET_ID_0=$SUBNET_ID_0 SUBNET_ID_1=$SUBNET_ID_1 SECURITY_GROUP=$SECURITY_GROUP"
```

Apply resources

```sh
astrobase apply -f resources.yaml -v "LOCATION=us-east-1
```

It takes a couple of minutes to get the service running, but once you get the loadbalancer, make your request like so

```sh
$ curl -s -X POST -d '{"url": "https://i.imgur.com/NZHKADF.jpg"}' -H 'Content-Type: application/json' http://a57319df8f136416d9306f9c1b0da046-1956442723.us-east-1.elb.amazonaws.com/predict | jq
[
  "golden_retriever",
  66.94183349609375
]
```
