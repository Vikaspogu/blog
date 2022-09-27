+++ 
date = 2020-12-05
title = "Setting up docker buildx on Linux"
description = "Docker buildx setup on linux"
slug = "" 
tags = ["docker","buildx","linux"]
categories = []
externalLink = ""
series = []
socialShare=true
+++

## Docker buildx on Linux

Installation instructions for [Ubuntu](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)

To execute docker commands without `sudo`. We need to add the username to the `docker` group.

- Find the current username by typing `who`
- Add username to `docker` group
- Reboot system

```bash
$ who
vikaspogu :0           2020-12-05 10:10 (:0)
$ sudo usermod -aG docker vikaspogu
$ sudo reboot
```

### Enable buildx

Create a new `config.json` file under the `~.docker` folder and enable the experimental feature

```bash
$ mkdir ~/.docker
$ vim ~/.docker/config.json
{
 "experimental": "enabled"
}
```

Now you should be able to do multi-arch builds

```bash
docker buildx build -t vikaspogu/test:v0.0.1 . --push --platform linux/arm64
```

If you encounter the below error when trying to build the image using `buildx`. It would help if you created new build instance

```bash
failed to solve: rpc error: code = Unknown desc = failed to solve with frontend dockerfile.v0: failed to load LLB: runtime execution on platform linux/arm64 not supported
```

### Create build instance

Follow the below steps to create a new build instance for arm64.

1. List existing buildx platforms
2. Create a new build instance named `arm64`
3. Inspect build instance to verify `arm64` platform exists
4. Set builder instance
5. Build multi-platform image

```bash
$ docker buildx ls
NAME/NODE DRIVER/ENDPOINT             STATUS   PLATFORMS
default   docker                               
  default default                     running  linux/amd64, linux/386

# create the builder
$ docker buildx create  --name amr64 --platform linux/amd64,linux/arm64

# start the buildx
$ docker buildx inspect arm64 --bootstrap
Name:   arm64
Driver: docker-container

Nodes:
Name:      arm640
Endpoint:  unix:///var/run/docker.sock
Status:    running
Platforms: linux/amd64*, linux/arm64*, linux/riscv64, linux/ppc64le, linux/386, linux/arm/v7, linux/arm/v6, linux/s390x

# set current builder instance
$ docker buildx use amr64

# build the multi images
$ docker buildx build -t vikaspogu/test:v0.0.1 . --push --platform linux/arm64
```

> If you encounter `/bin/sh: Invalid ELF image for this architecture` error. Run the following docker image and then build

```bash
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```
