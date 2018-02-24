---
layout: post
title:  "Using SeaHorn with Docker"
subtitle: "using Docker images"
date:   2018-02-24 10:53:01
categories: [seahorn, install, docker]
---

[Docker](https://www.docker.com/) is used to deploy applications in an predefined, portable environment which is isolated from the underlying system. Using containers and images, it encapsulates a prebuilt version of an application which can then be downloaded and run with reproducible results.

### Download and run a Docker image containing SeaHorn:

First install Docker (more details for [Ubuntu](https://docs.docker.com/installation/ubuntulinux/), [OS X](https://docs.docker.com/installation/mac/), and [Windows](https://docs.docker.com/installation/windows/)), then pull the latest version from Dockerhub using `docker pull seahorn/seahorn`, and finally, run it in a container by using ``docker run -v `pwd`:/host -it seahorn/seahorn``. This should change your shell prompt to something like: `usea@00e7e5010518:/opt/seahorn`. From here,
 * find the SeaHorn executable in `/opt/seahorn/bin`
 * run the tests using `cd /opt/seahorn/share/seahorn/test; lit simple solve abc dsa`
 * move files between the container and the rest of your machine using `/host` to represent the directory where you executed the `docker run` command (for example, `cp /host/a b` will copy a file named `a` from your host directory to a file named `b` inside the container)
 * exit the container by typing `exit`
 * to persist your changes inside the container, add `--name=your_name_here` to the `docker run` command, and restart the container using `docker start -ai your_name_here`

### Alternatively, build a SeaHorn image using the source code:

 * `docker build --build-arg UBUNTU=xenial --build-arg BUILD_TYPE=Release -t seahorn/seahorn-build:xenial -f docker/seahorn-build.Dockerfile .` to build dependencies (optional, as the next step will pull this image from Dockerhub)
 * `docker build --build-arg UBUNTU=xenial --build-arg BUILD_TYPE=Release --build-arg TRAVIS=true -t seahorn_xenial_rel -f docker/seahorn-full-size-rel.Dockerfile .` to build a full size image
 * ``docker run -v `pwd`:/host -it seahorn_xenial_rel /bin/sh -c "cp build/*.tar.gz /host/"`` outputs a tarball to the current directory
 * `docker build -t seahorn/seahorn -f docker/seahorn.Dockerfile .` creates the image previously described from the tarball

### A few important notes about Docker:

The SeaHorn image is based on Ubuntu 16.04, the default user is "usea" and has root access (**do not** use this in a production environment), and the version of Clang installed is 3.8.