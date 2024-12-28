---
title:  "1MinDocker #4 - Docker CLI essentials"
category: Beginners
---

In [the last article](https://dev.to/astrabert/1mindocker-3-fundamental-concepts-55ph) we explored the fundamental concepts of Docker: with this theory background, we can now start getting our hands dirty with some code!

<!-- more -->

From now on, we will use the command line interface (CLI) instead of Docker Desktop, as this is the easiest way to get to know Docker from the developer side🤗

### 0. Where do we work?

As the name says, a CLI needs a command line to work. A command line is actually part of a bigger interface, a _command prompt_ or _terminal_. Docker's commands, differently from most of other commands, work in the same way on every platform, we just need to find out where we will be working:

- On **Windows** you can use both the [Command Prompt](https://support.kaspersky.com/common/windows/14637#block0) and [Windows PowerShell](https://learn.microsoft.com/en-us/powershell/scripting/windows-powershell/starting-windows-powershell?view=powershell-7.4&viewFallbackFrom=powershell-7)
- On **macOS** you can use [Terminal](https://support.apple.com/guide/terminal/get-started-pht23b129fed/2.14/mac/15.0)
- On **Ubuntu** and on other **Linux** distros, you can use the associated [terminal](https://ubuntu.com/tutorials/command-line-for-beginners#3-opening-a-terminal)

You will find guides on how to activate the mentioned tools following the links.

### 1. Download and run our first image

Let's say we don't have Ubuntu on our machine and we want to have it up and running. Instead of downloading it as a distro for WSL or setting up a whole machine, we can pull the image from Docker Hub and run it on our machine:

```bash
docker pull ubuntu
```

This is the easiest way in which we could perform the pull: we are directly calling the image name, without specifying the author or the version.
To do a complete pull, we should use this convention:

```bash
docker pull author/image:tag
```

> The tag is, in a way, similar to a version (it can be an actual version, a name or a product status)

We can write, for example:

```bash
docker pull astrabert/books-mixer-ai:1.1.0
```

By default, when we don't specify the tag, Docker pulls the `latest` image. Find a complete reference for the `pull`  command [here](https://docs.docker.com/reference/cli/docker/image/pull/).

Now that we have Ubuntu as a Docker image, we can try to run it, by simply doing:

```bash
docker run ubuntu:20.04
```

Or we can add a command, like:

```bash
docker run ubuntu:20.04 echo "Hello world!"
```

If we want to really interact with our container from command line, we should run it in _interactive_ mode:

```bash
docker run -i ubuntu:20.04
```

This will actually start an Ubuntu shell directly in our terminal.

If we want to expose some ports because we will run programs inside of the container that will require that, we can simply map our local machine ports to the container's ones:

```bash
docker run -i -p 3000:3000 -p 4000:8000 ubuntu:20.04
```

In this case, our local port 3000 corresponds to container's port 3000, whereas our local port 4000 corresponds to container's port 8000. 

If we also would like to inject our local file system into our Docker container, we need to mount a volume:

```bash
docker run -p 3000:3000 -p 4000:8000 -i -v /home/user/data/:/opt/volume/ ubuntu:20.04
```

Now all the data under `/home/user/data` on your machine will be accessible under `/opt/volume` in Ubuntu's container.

We can also add environment variables or an environment file: 

```bash
docker run -p 3000:3000 -p 4000:8000 -i -v /home/user/data/:/opt/volume/ -e VARENV=foo --env-file ./.env.local ubuntu:20.04
```

There are lots of other options we can add to the `run` command, and we'll see some of them later on: for now, you can find a complete reference [here](https://docs.docker.com/reference/cli/docker/container/run)

### 2. Let's see available image and containers

If we want to see our current downloaded images, we can run:

```bash
docker images
```

And if we want to find all the containers, we can just run:

```bash
docker ps -a
```

We can also filter our containers: for example, if we started multiple containers and we want to see which ones are running, we can do this by inputting: 

```bash
docker ps --filter "status=running"
```

If we want only the container IDs, we can run:

```bash
docker ps -a -q
```

Find the complete reference for [docker images](https://docs.docker.com/reference/cli/docker/image/ls/) and [docker ps](https://docs.docker.com/reference/cli/docker/container/ls/)  following the two links.

### 3. Stop and eliminate a container, remove an image

To stop a container, we can use:

```bash
docker stop <container-id>
```

We can find the container ID in the list previously generated with `ps`. 

Once a container is stopped, we can remove it by running:

```bash
docker rm <container-id>
```

If we want to stop all the containers and then remove them, we can run:

```bash
docker stop $(docker ps -a)
docker rm $(docker ps -aq)
```

If we want to remove an image from our local registry, we can run:

```bash
docker rmi author/image:tag
```

Always remember that `rm` and `rmi` are **different**!😊

A complete reference for the two commands can be found [here](https://docs.docker.com/reference/cli/docker/container/rm/) and [here](https://docs.docker.com/reference/cli/docker/image/rm/).

We will stop here for this article, but in the next one we will dive deep into how to build our first image ever and to push it to the Docker Hub!
For now, remember to sign up to [Docker portal](https://app.docker.com/signup), as it will be very important for our next steps🥰
