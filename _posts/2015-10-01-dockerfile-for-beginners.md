---
layout: post
title: "Dockerfile Tutorial for Beginners"
date: 2015-09-31
author: Virendra Singh Bhalothia
tags: containers docker dockerfile microservices best-practices
category: tutorial
excerpt: "This is a Dockerfile tutorial for beginners. Build, run, ship containers in less than 30 minutes."
---


## Introduction

`Docker` can build images automatically by reading the instructions from a `Dockerfile`. A `Dockerfile` is a text document that contains all the commands a user could call on the command line to assemble an image. Using `docker` build users can create an automated build that executes several command-line instructions in succession.


![containers][15]


There are two ways to create a `docker` image:

- Create a container and alter its state by running commands in it; create an image with `docker commit`
- Create a `Dockerfile` and create an image with `docker build`

Most image authors will find that using a `Dockerfile` is a much easier way to repeatably create an image. A `Dockerfile` is made up of instructions, several of which will be discussed in this guide.

You can find the complete `Dockerfile` instruction reference [here][7] or [here][13] or [best practices][9] from `docker` itself.


![docker_logo][16]


We will use `Dockerfile` approach in this tutorial and dockerize [RunDeck][6] in this tutorial, all because of my love for this tool.  

We will demonstrate the best practices and methods to make most of `docker` and containers via `Dockerfiles`. We will use a [base image][8] and build the RunDeck image step by step.


## Docker: What the fuss is all about?

`Docker` is an open source project for creating lightweight, portable, self-sufficient application containers. `Docker` containers wrap up a piece of software in a complete filesystem that contains everything it needs to run: code, runtime, system tools, system libraries – anything you can install on a server. This guarantees that it will always run the same, regardless of the environment it is running in.

For deep diving into `docker` and it's components, please refere to [this article][10].


## Dockerfile Syntax and Commands

These are some commands which `dockerfile` can contain to have `docker` build an image.



| ADD         | CMD     | ENTRYPOINT     | ENV      | EXPOSE     | FROM        |
|-------------|---------|----------------|----------|------------|-------------|
| **PUBLISH** | **RUN** | **MAINTAINER** | **USER** | **VOLUME** | **WORKDIR** |



- ADD - `Syntax`: `ADD [source directory or URL] [destination directory]`
- CMD - `Syntax`: `CMD application "argument", "argument", ..`
- ENTRYPOINT - `Syntax`: `ENTRYPOINT application "argument", "argument", ..`
- ENV - `Syntax`: `ENV key value`
- EXPOSE - `Syntax`: `EXPOSE [port]`
- FROM - `Syntax`: `FROM [image name]`
- RUN - `Syntax`: `RUN [command]`
- MAINTAINER - `Syntax`: `MAINTAINER [name]`
- USER - `Syntax`: `USER [UID]`
- VOLUME - `Syntax`: `VOLUME ["/dir_1", "/dir_2" ..]`
- WORKDIR - `Syntax`: `WORKDIR /path`
- PUBLISH - `To expose ports to the host, at runtime, use the -p flag or the -P flag.`

## Sample [Dockerfile][11] : Creating a [docker image][12] to Install Configure [RunDeck][6]

> Here starts the `Dockerfile`, with source information.


```
# Dockerfile for docker image of RunDeck
# https://github.com/bhalothia/docker-rundeck
# RunDeck plugins from https://github.com/rundeck-plugins
```

> Setting the base image to use

`FROM debian:wheezy`

> Defining the maintainer of this project

```MAINTAINER Virendra Singh Bhalothia <bhalothia@theremotelab.com>```

> Setting environment variables

```
ENV DEBIAN_FRONTEND noninteractive
ENV SERVER_URL http://localhost:4440
ENV RDECK_BASE /var/lib/rundeck
```
> Installing packages

```
# Updating and installing packages
RUN apt-get -qq update && apt-get -qqy upgrade && apt-get -qqy install --no-install-recommends bash supervisor procps sudo ca-certificates openjdk-7-jre-headless openssh-client mysql-server mysql-client pwgen curl git && apt-get clean
```
> Copying or downloading data inside the container

```
# Downloading RunDeck package
ADD http://dl.bintray.com/rundeck/rundeck-deb/rundeck-2.5.3-1-GA.deb /tmp/rundeck.deb
# Copying the prequisities content
ADD prerequisites/ /
```
> Running a lot of commands to install/configure RunDeck and MySql

```
# Installing RunDeck package
RUN dpkg -i /tmp/rundeck.deb && rm /tmp/rundeck.deb
# Owning the temp directory
RUN chown rundeck:rundeck /tmp/rundeck
# Making the run file executable
RUN chmod u+x /opt/run
# Creating a directory for ssh keys
RUN mkdir -p /var/lib/rundeck/.ssh
# Chowing that ssh directory
RUN chown rundeck:rundeck $RDECK_BASE/.ssh

# Creating directories for Supervisor
RUN mkdir -p /var/log/supervisor && mkdir -p /opt/supervisor
# Making the rundeck and mysql files executable
RUN chmod u+x /opt/supervisor/rundeck && chmod u+x /opt/supervisor/mysql_supervisor
```
> Exposing the ports

`EXPOSE 4440 4443`

> Mounting volumes

`VOLUME  ["/etc/rundeck", "/var/rundeck", "/var/lib/rundeck", "/var/lib/mysql", "/var/log/rundeck"]`

> Setting default container command

`ENTRYPOINT ["/opt/run"]`



## Build Process

`docker pull bhalothia/docker-rundeck:v1`

> Above step will pull the image version 1 - which is the latest as well.


__Usage__
Start a new container and bind to host's port 4440


`sudo docker run -p 4440:4440 -e SERVER_URL=http://MY.HOSTNAME.COM:4440 -t bhalothia/docker-rundeck:v1`


**Note**: If you are using docker-machine, then you need to do find out the docker-machine ip and pass it as the SERVER_URL


## Environment variables

```
SERVER_URL - Full URL in the form http://MY.HOSTNAME.COM:4440, http//123.456.789.012:4440, etc

DATABASE_URL - For use with (container) external database

RUNDECK_PASSWORD - MySQL 'rundeck' user password

DEBIAN_SYS_MAINT_PASSWORD
```

## Volumes

```
/etc/rundeck
/var/rundeck
/var/lib/rundeck - Not recommended to use as a volume as it contains webapp.  For SSH key you can use the this volume: /var/lib/rundeck/.ssh
/var/lib/mysql
/var/log/rundeck
```

Here's the [README][14] for the source repo of this `docker` image.


**Hope this helps! Keep forking.**

## The Remote Lab DevOps Offerings:
<iframe src="//www.slideshare.net/slideshow/embed_code/key/h9h9GNjX5Gncpi" width="595" height="485" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC; border-width:1px; margin-bottom:5px; max-width: 100%;" allowfullscreen> </iframe> <div style="margin-bottom:5px"> <strong> <a href="//www.slideshare.net/bhalothia/the-remote-lab-devops-offerings" title="The Remote Lab DevOps Offerings" target="_blank">The Remote Lab DevOps Offerings</a> </strong> from <strong><a href="//www.slideshare.net/bhalothia" target="_blank">Virendra Bhalothia</a></strong> </div>

Please leave your comments below if you have any doubts or questions.

**Need DevOps help? - Get in touch with** [The Remote Lab][1]
[LinkedIn][2] [Facebook][3] [Github][4] [Twitter][5]


  [1]: http://theremotelab.com
  [2]: https://www.linkedin.com/company/the-remote-lab
  [3]: https://www.facebook.com/TheRemoteLab
  [4]: https://github.com/TheRemoteLab
  [5]: https://twitter.com/TheRemoteLab
  [6]: https://rundeck.org
  [7]: http://www.projectatomic.io/docs/docker-building-images/
  [8]: https://hub.docker.com/r/library/debian/
  [9]: https://docs.docker.com/articles/dockerfile_best-practices/
  [10]: https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-getting-started
  [11]: https://github.com/TheRemoteLab/docker-rundeck/blob/master/Dockerfile
  [12]: https://hub.docker.com/r/bhalothia/docker-rundeck/
  [13]: http://crosbymichael.com/dockerfile-best-practices.html
  [14]: https://github.com/TheRemoteLab/docker-rundeck/blob/master/README.md
  [15]: http://i.giphy.com/OP7kIfBat5sGY.gif
  [16]: https://d21ii91i3y6o6h.cloudfront.net/gallery_images/from_proof/1026/large/1396373089/docker.png
