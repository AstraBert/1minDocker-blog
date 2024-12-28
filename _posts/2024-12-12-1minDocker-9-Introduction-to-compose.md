---
title:  "1MinDocker #9 - Introduction to compose"
category: Intermediate 
---


In [the last article](https://dev.to/astrabert/1mindocker-8-advanced-concepts-for-buildx-2olc) we finished our deep dive on `docker buildx`, a popular plugin that is aimed at easing and automating the building process.

From this article on, we will talk about another plugin, `docker compose`, which presents numerous fields of application and a high potential for deployment and development automation.

<!-- more -->

## What is `compose`?

`compose` is a technology that allow users to run one or more container in a easy and reproducible way. 

Through `compose` you can control the entire tech stack and environment needed for your application, using simple but elegant YAML code in a input file.

`compose` provides also a very simple and intuitive CLI that, with few commands, lets your run, inspect, interact with and stop the containers you defined as services inside your YAML input file.

## Why `compose`?

Choosing `compose` might come for a variety of reasons:

- It's **simple**: it only needs few key words to work correctly, it leverages intuitive CLI commands and does not need the complex configurations that are set when running directly with `docker run` 
- It's **compact**: everything you need (images, volumes, networks...) is in one file
- It's **the easiest way to set up a working environment**: imaging managing multiple databases, switching among various stacks for backend and frontend and manage several different API services: this would be very difficult to implement natively but, with `compose`, you can easily combine several different Docker images and just run them all together as a perfectly harmonic orchestra 
- It's easily **sharable**: you don't have to transfer entire codebases, deal with conflicts and with local machine versioning problems when giving your compose YAML file to other people from your team or from other team. This enhances **reproducibility** and fosters **collaboration**

## Getting started

Getting started with `compose` is simple and easy. We just need to have it installed (see [the second article of this series](https://dev.to/astrabert/1mindocker-2-get-docker-kh)) and we can then proceed with creating our first `compose` YAML file, which we will call `compose.yaml` (suggested by Docker docs over `compose.yml` and `docker-compose.yaml`, which can still be used, though). 

Let's say we want to build a React.js application and we want it to be interfaced with a Postgres database, whose status we also want to monitor through Adminer. We can exploit the `node:18-alpine` image to build an environment where we can install and run our local application mounted as a volume, the `postgres` image to get a PostgreSQL DB instance up and running on port 5432 and the `adminer` image to start Adminer on port 8080.

Let's see how the compose file will look like:

```yaml
services:
  db:
    image: postgres
    restart: always
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: $PG_DB
      POSTGRES_USER: $PG_USER
      POSTGRES_PASSWORD: $PG_PASSWORD
    volumes:
      - pgdata:/var/lib/postgresql/data 

  app:
    image: node:18-alpine
    restart: always
    ports:
      - "3000:3000"
    volumes:
      - "appsrc:/app/src"
      - "apppublic:/app/public"
      - "./package.json:/app/"
      - "./.env:/app/"
    entrypoint: "cd /app && npm install && npm start"

  adminer:
    image: adminer
    restart: always
    ports:
      - "8080:8080"

volumes:
  pgdata:
  appsrc: "./src/"
  apppublic: "./public/"
```

> *Notice that we use PG_DB, PG_USER and PG_PASSWORD as environmental variable: this means that you should have set them in a `.env` file*

In this case, we have all our three services available at once: the `app` (exposed on port 3000), that is injected from the local file system into the container and built on the fly every time the service is started, the `db` (exposed on port 5432), that is accessible through user, password and database name on `adminer` (exposed on port 8080).

To start everything, we just need to go to the directory in which our `compose` file is stored and run:

```bash
docker compose up
```

And, if we want to stop them, we can simply run:

```bash
docker compose down
```

We will stop here for this article, but in the next one we will dive into the `compose` files and how to build the best out of them!🥰