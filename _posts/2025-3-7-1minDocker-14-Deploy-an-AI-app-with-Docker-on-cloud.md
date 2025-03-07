---
title:  "1MinDocker #14 - Deploy an AI app on the cloud with Docker"
category: Advanced
---

In the [last article](https://dev.to/astrabert/1mindocker-13-push-build-and-dockerize-with-github-actions-52gb) we dove into the world of continuous integration with GitHub Actions. Now it's time to take a step forward and to talk about deploying a Docker application and make it available to everyone. To do this, we could exploit a local server, but local servers are usually costly to set up, initialize and maintain (on the long run): cloud solutions are, on the other hand, simpler and faster to boot and set up, especially the one we are going to use for this tutorial, [**Linode**](https://www.linode.com/)

## Step 1: your Linode instance

Setting up a Linode instance couldn't be easier: you just need to sign up or log in to [Linode](https://www.linode.com/).

Once you land into your dashboard, you just need to click on `Create` (green button on the top left corner) -> `Linode` . Then you will be prompted to select the settings of your instance (operating system, region, name, root password and, eventually, an SSH key). The set up is extremely intuitive and, for our application, I'd suggest:

- Choose Ubuntu 22.04 as OS
- Choose a 2GB RAM - 1 vCPU hardware
- Choose the region that is closer to you
- Choose a strong password for your root user

Once you are set up and your instance is booted and running, you can connect to it from your terminal. Regardless that you are on Windows, macOS or Linux (although I prefer the last one), you can simply use the SSH protocol and authenticate with the root password. To do so, you need to get the public IP address of your Linode (which will be useful also later) - you can comfortably find it in you dashboard.

```bash
ssh root@<PUBLIC-IP-ADDRESS>
```

You'll be prompted to input the password and, after that, you'll be finally inside your Linode's terminal!
## Step 2: preparing your Linode for the application

Since we want to deploy our application with Docker, we need to install it within our Linode virtual machine.

If you followed my advice and you created an Ubuntu 22.04 machine, you can simply try these commands, that you can also find on the official [Docker installation page](https://docs.docker.com/engine/install/ubuntu/):

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

After this, run:

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Test the successful installation with:

```bash
sudo docker run hello-world
```

And BAM! You installed Docker🐋

## Step 3: Get the application

For this tutorial, I already prepared an application for you: it's called **SciNewsBot** and it's a BlueSky bot which publishes daily science news from trusted publishers. 

SciNewsBot exploits [Mistral AI](https://mistral.ai/) to summarizes into an effective and catchy headlines the titles and content of news from Google News publishers that are labelled as trustworthy by Media Bias/Fact Check. These news spans four domains (Science, Environment, Energy and Technology), and are scraped and published 4 times a day, with a pause of 3 hours in between and with a pause of 12 hours from the last news report of one day to the first news report of the following day. You can see the bot working in [this page](https://bsky.app/profile/sci-news-bot.bsky.social)

So, from within your Linode instance (which you connected to via SSH in previous steps) clone the application from GitHub:

```bash
git clone https://github.com/AstraBert/SciNewsBot.git
cd SciNewsBot/
```

Now the only things you need to do are:

1. Get a [Mistral AI API key](https://console.mistral.ai/api-keys) (you can create one for free)
2. Create a BlueSky user for your bot, and you can do it [here](https://bsky.app/)
3. Modify your `.env.example` file with reporting the Mistral API key, the BlueSky username and password 
4. Rename the `.env.example` file to `.env` with the following command:

```bash
mv .env.example .env
```

## Step 4: Deploy!

Now we're just one step away from deployment, and that step is launching our application through Docker. Let's take a look to the `compose.yaml` file that we have in the SciNewsBot folder

```yaml
name: news-sci-bot

services:
  bot:
    build: 
      context: ./docker/
      dockerfile: Dockerfile
    secrets:
      - mistral_key
      - bsky_usr
      - bsky_psw
    networks:
      - mynet

secrets:
  mistral_key:
    environment: mistral_api_key
  bsky_usr:
    environment: bsky_username
  bsky_psw:
    environment: bsky_password
  
networks:
  mynet:
    driver: bridge
```

This file creates a container from the `docker` subfolder we have, mounting within it three environment-derived secrets, i.e the Mistral AI API key, the BlueSky username and the password. It then attaches the container to a network named `mynet`.

Each of this secrets is accessible through the path `/run/secrets/<secret_name>`, and that's how we do that in our python scripts (we read these files).

Now, to deploy we just need to run:

```bash
docker compose up -d
```

> _Don't forget to put the `-d` option, otherwise you will kill the container execution one you exit from the Linode terminal: the `-d` option detaches the container execution from the main terminal, allowing you to close it without stopping the container._

Congrats! You just deployed your first Docker application on cloud!🎉

We will stop here for this article, but in the next (and last article) we will see more complex deployment use cases and will wrap up the 1minDocker journey: stay tuned and have fun!🥰
