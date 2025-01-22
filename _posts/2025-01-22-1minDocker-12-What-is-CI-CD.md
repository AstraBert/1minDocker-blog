---
title: "1MinDocker #12 - What is CI/CD?"
category: Intermediate 
---

Hello world and Happy 2025!🐳✨

It's time we resume our 1minDocker series, starting our last learning block, consisting of 4 articles, that will introduce us to **CI/CD practices with Docker**.

In order to get started with this last learning block, tho, we need to understand what CI/CD is: let's dive in!

## What does CI/CD mean?

CI/CD is short for **Continuous Integration/Continuous Delivery** or, sometimes, **Continuous Deployment**. Let's break it down:

- Continuous *Integration*: the CI piece of CI/CD means that everything, even small modification, geta integrated in the main code, constantly. This is a key feature when you need to fix small bugs and/or technicalities that would otherwise require you to package a new, standalone, patch release, that will have to be manually integrated user-side through updates. With continuous integrations, small and big changes/fixes are immediately available. 
- Continuous *Delivery*: the CD part is the consequence of continuous integration. Every time we integrate a new modification of our source code, we prompt a certain number of steps that will then, in the end, bring to a delivery/deployment of our code into production. This happens constantly, and allows companies with wise source control and orchestration to ship new features into production within **minutes** from their request. Obviously, this might be an edge case applicable to those huge tech companies with big developer teams working 24/7 on their products, but still the continuous delivery ensures high speed also to other smaller companies, that can deploy new features within hours instead of days or weeks.

## What are the main steps of CI/CD?

As this image shows:

![CI/CD](https://www.blackduck.com/glossary/what-is-cicd/_jcr_content/root/synopsyscontainer/column_1946395452_co/colRight/image_copy.coreimg.svg/1727199377195/cicd.svg)

_Image from [**Black duck**: What is CI/CD](https://www.blackduck.com/glossary/what-is-cicd.html)_

The CI/CD pipeline contains several really important steps, that are inserted in a "infinite" loop (that's the main idea behind the _continuous_ thing):

- We start with the **code**: developers all around the world write their code in their comfortable IDE on their computers, and then, once they are done, push the changes they made into a version control system. The most widespread is [Git](https://git-scm.com/), and the most used services on this end are [GitHub](htts://github.com) and [GitLab](https://about.gitlab.com/) 
- The code, once in a code management system, **needs to be built**: builds are generally automatized and test if there is some error in this phase, that would result in breakings and/or other errors further down the road.
- After the build is complete and the code is deemed clean on this end, we can proceed with **testing its actual capabilities**. There are several possible tests, most of them depend on the use case that your code is tackling. A widespread strategy is **integration testing**, which looks into the possibility that the code is compatible with the requirements and the standard of its use case.
- If all the tests are passed, we can finally **release** our code out in the wild
- Following the release there's often the **deployment**: the code is pushed into production and is finally ready to be used
- **Operating** and **Monitoring** are then the last two phases before starting to modify the code again: we see how users interact with it, how well our products perform and we collect feedback. With this information, we can start fixing bugs, creating new features and making our products shine :)

## Why CI/CD and Docker?

As you can see, CI/CD requires several environments: a build environment, a test environment and a deployment environment. Docker is a perfect solution, as we can manage everything through different containers, just using simple commands like `docker build` and `docker run`. Docker is also perfect for deployment, as it does not require complex environment set-up from scratch: you can build a portable image with all the dependencies and simply deploy it building from `Dockerfile` and running it. 

Docker handles secrets, takes care of data transfer from the local context to the container and manages networks.

With Docker, you can even manage all of this with **one command**: you simply need to create a `compose.yaml` file and define your services, their sources (if pre-existing images or on-the-fly builds) and their specs (secrets, volumes, networks). After that, you simply need `docker compose up`. It's really simple, isn't it?🐳

## Sources

- [**Akamai Developer**: CI/CD Explained - How DevOps Use Pipelines for Automation](https://youtu.be/M4CXOocovZ4?feature=shared)
- [**Be a Better Dev**: The IDEAL and Practical CI/CD Pipeline - Concepts Overview](https://youtu.be/OPwU3UWCxhw?feature=shared)