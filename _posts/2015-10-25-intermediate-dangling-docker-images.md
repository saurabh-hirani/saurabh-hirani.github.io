---
layout: post
title: "Intermediate and Dangling Docker Images"
date: 2015-10-25
author: Virendra Singh Bhalothia
tags: docker containers filesystem images intermediate best-practices
category: tutorial
excerpt: "In-depth tutorial about Docker Image File Systems. Don't be afraid of Docker &lt;none&gt;:&lt;none&gt; images. Not all that is black is charcoal."
---

### Hello world!

So, we figured out how to build `Docker` images using `Dockerfile` in the [last article][6]. I hope everyone now knows how to build a `Docker` image from scratch.

Before we start this article, let's get familiar with `Docker` image filesystem. How many of you know about the `Docker` image filesystem?

`Docker` images are read-only templates, which could contain an operating system and an application on top of it. It is then used for creating `Docker` containers.

## But most importantly, we should know how does a Docker image work?

`Docker` image file system is layered. It uses [Union file systems][7] to combine these layers to form a single image.

One of the reasons Docker is so lightweight is because of these layers. When you change a Docker image—for example, update an application to a new version— a new layer gets built. Thus, rather than replacing the whole image or entirely rebuilding, as you may do with a virtual machine, only that layer is added or updated. Now you don’t need to distribute a whole new image, just the update, making distributing Docker images faster and simpler.

Source: [Official Docker Documentation][8]

## Deep Diving to understand Docker Image File System


Let's do some more digging to understand it practically. I'm using [Docker-Machine][9] on mac and I have started the default environment:  

![default_env][10]

...and I've removed everything so it should not show me any containers, images or intermediate images.

![nothing_is_running][11]

You can see that, my docker host is clean and no signs of an image or a container.


**Did you ever think where are Docker images stored?**

The first place to look is in `/var/lib/docker/`. We are using `Machine`, so I'm going to ssh into my `Docker-Machine` and show you:

![docker-machine-ssh][12]

Right, now nothing is present as we have not pulled or build any images. Let me pull the `Docker` RunDeck image from my [DockerHub repository][13] and see what happens:

`docker pull bhalothia/docker-rundeck:v1.1`

![docker-pull-rundeck][14]


Alright, now the RunDeck image is downloaded and we are good to create containers out of it. We can check what all is updated inside `/var/lib/docker` folder on the `Docker-Machine`.

Before that, let's do `docker images` and `docker images -a`:


![layered-image][15]

Now, at this point of time you might be wondering what is the difference between `docker images` and `docker images -a`?

`Docker` file system layers are by default stored at `/var/lib/docker/graph`, which is called as the graph database. And these layers corresponds to the `IMAGE ID`

![docker-graph][16]


When I did `docker pull bhalothia/docker-rundeck:v1.1`, the image was downloaded one layer at a time. Layer `ba249489d0b6` was downloaded as `<none>:<none>` image and so on. The final image was `bhalothia\docker-rundeck:v1` and all other layers were downloaded as intermediate images. `Docker` allows you to see all intermediate images using the flat `-a`.

Every layer, one after the other, starting from `ba249489d0b6` to the final image `241f5ca9955c` have the parent-child hierarchical relationship.

To verify this, you can run this command: `cd /var/lib/docker/graph && more layer_id/json`

![parent-child][17]

**So, if you are worried about the `<none>:<none>` images, which are also called as intermediate images then you can relax.**

## But, wait...what is Dangling Image?

Hey, you need to know about the evil `<none>:<none>` images as well.. those which aren't intermediate and which shows up when you do `docker images`.

Docker keeps all of the images in its cache that you have used in the disk, even if those are not actively running.

Check all [open issues][18] on `Docker` repository.


Here's how you can clean up dangling images:

`docker rmi $(docker images --quiet --filter "dangling=true")`

It will give you an error if there's no dangling image present. If you don't want that, then this might work for you:

`docker images -qf dangling=true | xargs docker rmi`

This command will do the manual garbage collection.


  [1]: http://theremotelab.com
  [2]: https://www.linkedin.com/company/the-remote-lab
  [3]: https://www.facebook.com/TheRemoteLab
  [4]: https://github.com/TheRemoteLab
  [5]: https://twitter.com/TheRemoteLab
  [6]: http://theremotelab.com/blog/dockerfile-for-beginners/
  [7]: https://en.wikipedia.org/wiki/UnionFS
  [8]: http://docs.docker.com/introduction/understanding-docker/#how-does-a-docker-image-work
  [9]: https://docs.docker.com/machine/
  [10]: https://s3-ap-southeast-1.amazonaws.com/trl-blog/docker_intermediate_2.png
  [11]: https://s3-ap-southeast-1.amazonaws.com/trl-blog/docker_intermediate_1.png
  [12]: https://s3-ap-southeast-1.amazonaws.com/trl-blog/docker_intermediate_3.png
  [13]: https://hub.docker.com/r/bhalothia/docker-rundeck/
  [14]: https://s3-ap-southeast-1.amazonaws.com/trl-blog/docker_intermediate_4.png
  [15]: https://s3-ap-southeast-1.amazonaws.com/trl-blog/docker_intermediate_5.png
  [16]: https://s3-ap-southeast-1.amazonaws.com/trl-blog/docker_intermediate_6.png
  [17]: https://s3-ap-southeast-1.amazonaws.com/trl-blog/docker_intermediate_7.png
  [18]: https://github.com/docker/docker/issues?utf8=%E2%9C%93&q=is%3Aissue+is%3Aopen+dangling
