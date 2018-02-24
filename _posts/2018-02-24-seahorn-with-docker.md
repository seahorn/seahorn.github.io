---
layout: post
title:  "Using SeaHorn with Docker"
subtitle: "using Docker images"
date:   2018-02-24 10:53:01
categories: [seahorn, install, docker]
---

[Docker](https://www.docker.com/) is used to deploy applications in an
predefined, portable environment which is isolated from the underlying
system. Using containers and images, it encapsulates a pre-built
version of an application which can then be downloaded and run with
reproducible results.

### Using SeaHorn within a Docker container:

First install Docker (more details for
[Ubuntu](https://docs.docker.com/installation/ubuntulinux/), [OS
X](https://docs.docker.com/installation/mac/), and
[Windows](https://docs.docker.com/installation/windows/)), then pull
the latest version from Dockerhub using
```$ docker pull seahorn/seahorn```
Run SeaHorn in a container  using
```docker run -v $(pwd):/host -it seahorn/seahorn```
This should change your shell prompt to something
like: `usea@00e7e5010518:/opt/seahorn`. At this point, you can
    * use SeaHorn executable found in `/opt/seahorn/bin/sea`
    * run SeaHorn tests using the command `cd /opt/seahorn/share/seahorn/test; lit simple solve abc dsa`
    * move files between the container and the rest of your machine using `/host` to represent the directory where you executed the `docker run` command (for example, `cp /host/a b` will copy a file named `a` from your host directory to a file named `b` inside the container)
    * exit the container by typing `exit`
    * to persist your changes inside the container, add `--name=your_name_here` to the `docker run` command, and restart the container using `docker start -ai your_name_here`

### Building a Docker image from SeaHorn sources

It is also possible to build a local Docker container based on SeaHorn
source code by following the instructions below:
    * Optionally build thrid-party dependencies. These are also available as a `seahorn/seahorn-build` container on DockerHub.
    ```docker build --build-arg UBUNTU=xenial --build-arg BUILD_TYPE=Release -t seahorn/seahorn-build:xenial -f docker/seahorn-build.Dockerfile ```
    * Compile SeaHorn binary in a docker cotnainer
    ```docker build --build-arg UBUNTU=xenial --build-arg BUILD_TYPE=Release --build-arg TRAVIS=true -t seahorn_xenial_rel -f docker/seahorn-full-size-rel.Dockerfile ```
    * Extracted compiled binary package from the container to the host system
    ```docker run -v $(pwd):/host -it seahorn_xenial_rel /bin/sh -c "cp build/*.tar.gz /host/"```
    * Create a docker image with SeaHorn based on the binary package created in the previous step:
    ```docker build -t seahorn/seahorn -f docker/seahorn.Dockerfile ```

### Warning about default user in the container

The SeaHorn image is based on Ubuntu 16.04, the default user is "usea"
and has root access (**do not** use this in a production environment),
