---
title:  "1MinDocker #7 - Superpower your builds with buildx"
category: Intermediate 
---

In the [last article](https://dev.to/astrabert/1mindocker-6-building-further-39al) we talked about the possibility of expanding our build capacity with multi-staged builds and if-else statements: in this article, we'll see how to superpower our builds with `buildx`, a popular Docker plugin that is intended at replacing the legacy `docker build` command. 

### Getting `buildx`

If you correctly installed Docker Desktop for Windows or macOS (see our [second article](https://dev.to/astrabert/1mindocker-2-get-docker-kh)), `buildx` should be already included. 

If you are on Linux and running `docker buildx --version` returns an error because the plugin wasn't installed, you should follow the instructions you can find in [1minDocker #2](https://dev.to/astrabert/1mindocker-2-get-docker-kh) and/or on [Docker official documentation](https://docs.docker.com/engine/install/)

Once you have `buildx`, you can set it as a default builder by running:

```bash
docker buildx install
```

This will dismiss the legacy builder (`docker build`) and default to that of the plugin (`docker buildx build`).

### What `buildx` can that `build` can't

`buildx` has multiple features that the legacy builder does not provide

#### 1. Drivers

You can choose the environment where to run the build: this environment is called _driver_ and by default is set to the same as the normal builder (the `docker` driver), but it can also exploit [`docker-container`](https://docs.docker.com/build/builders/drivers/docker-container/) (a containerized environment for the build), [`kubernetes`](https://docs.docker.com/build/builders/drivers/kubernetes/) (that connects local environments to Kubernetes clusters) or [`remote`](https://docs.docker.com/build/builders/drivers/remote/) (allows access to an externally managed building environment)

#### 2. Isolated builder instances

You can create multiple isolated builder instances assigning them to different nodes through `buildx create` (and there are a handful of commands to manage those instances). There is also the possibility to give your builder instances a default template with the `buildx context` command.

#### 3. Multi-platform builds

You can specify the platform for which you're building through the `--platform` flag (available: `linux/amd64`, `linux/arm64`, `darwin/amd64`). When you're backed by `docker-container` or `kubernetes`, you can actually do a multi-platform build at once, using different strategies:

- Specifying stages in the Dockerfile that can cross-compile through different platforms
- Using different builder instances that compile for different architectures
- Using kernel emulation through QEMU (easiest solution)

For what concerns kernel emulation, if this solution is enabled in your node, it just automatically recognizes secondary available architectures and builds also for them. QEMU can be installed with Docker as simple as this command:

```bash
docker run --privileged --rm tonistiigi/binfmt --install all
```

And the builder instances will be able to use it. 

You can also encounter more complicated cases where QEMU is not sufficient. In those cases you can either build on multiple nodes, like in this example:

```bash
docker buildx create --use --name mybuild node-amd64
docker buildx create --append --name mybuild node-arm64
docker buildx build --platform linux/amd64,linux/arm64 .
```

Or you can use multi-platform builds in your Dockerfile:

```dockerfile
# syntax=docker/dockerfile:1
FROM --platform=$BUILDPLATFORM golang:alpine AS build
ARG TARGETPLATFORM
ARG BUILDPLATFORM
RUN echo "I am running on $BUILDPLATFORM, building for $TARGETPLATFORM" > /log
FROM alpine
COPY --from=build /log /log
```


We will stop here for this article, but in the next one we will go through common `buildx` commands and how they work🥰.

> _The content for this article is mainly based on [`docker/buildx`](https://github.com/docker/buildx) GitHub repo: make sure to visit them and give them a star!⭐_
