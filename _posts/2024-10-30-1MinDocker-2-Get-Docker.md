---
title:  "1MinDocker #2 - Get Docker"
categories: Beginners
---
As we said in [the last article](https://dev.to/astrabert/1mindocker-1-what-is-docker-3baa), Docker is a **cross-platform** technology, so in this article we will go through all the platform-specific installation processes that will allow us to get Docker faster than we can imagine!🚀
### Docker on Windows
[Docker on Windows](https://docs.docker.com/desktop/install/windows-install/) is only available as Docker Desktop, a Desktop application that manages _Docker engine_ (the technology that actually runs the virtual machines) through either **WSL2** or **Hyper-V**  as backend systems. 

You need Windows 10 or 11 to make Docker work and, if you're using a _Windows Subsystem Linux_ (WSL), make sure to have upgraded it to WSL2. 

For the installation, you should get the binary that best suits your Windows machine from the link at the beginning of this paragraph, and from there simply click on `Docker Desktop Installer.exe` and follow up as you are prompted by the installation interface.🤗
### Docker on MacOS
[Docker on MacOS](https://docs.docker.com/desktop/install/mac-install/), as we said for Windows, can only be downloaded as Docker Desktop.

Docker, for now, supports only the three latest releases of macOS, and requires at least 4GB RAM to run.

You can get the binaries that most suit your platform from the link at the beginning of this paragraph, and then click on the `Docker.dmg` installer so obtained, dragging the Docker icon into the `Applications` folder.  Now you will have a `Docker.app`, which you will be able to initialize and set up following the prompts on the installation interface.

### Docker on Linux

There are [many supported platforms](https://docs.docker.com/engine/install/#supported-platforms) and many ways to install Docker for each one of them.

In this article, we will see the easiest possible way to get Docker CLI on **Ubuntu**.

You can simply run the following command:

```bash
sudo apt update && sudo apt install docker.io
```

And check the installation by running:

```bash
sudo docker --version
```

This method **will not** install `docker-compose` or `docker-buildx`, Docker plugins that we will talk about in the advanced section of this series. 

For the sake of completeness, we will nevertheless report here how to [get also the `compose` and `buildx` plugin](https://docs.docker.com/compose/install/linux/):

1. Start by getting Docker's official GPG Key:
```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
2. Now add Docker and plugins to your `apt` repositories:
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```
3. Now just install `compose` and `buildx`:
```bash
sudo apt-get install docker-compose-plugin docker-buildx-plugin
```

We will stop here for this article, but in the next one we will dive deep into the theory behind Docker, understanding its basic components 🥰