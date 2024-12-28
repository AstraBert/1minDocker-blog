---
title:  "1MinDocker #11 - Advanced compose example"
category: Intermediate 
---

In the [last article](https://dev.to/astrabert/1mindocker-10-the-compose-file-4ihf), we introduced the `compose` file reference, and we went through numerous top-level elements and their attributes. In this article, we will present an advanced example using `docker compose` to deploy a multi-container application on the cloud. 

<!-- more -->

We will refer, for this tutorial, to a slightly modified version of the `compose.yaml` file proposed for ElasticSearch-Logstash-Kibana by the [awesome-compose](https://github.com/docker/awesome-compose) repository by Docker on GitHub. 

## Background

Why would we build a multi-container environment with ElasticSearch, Logstash and Kibana? Well, the idea is that this stack (also known as the **ELK stack**), can help us monitor logs and data sent by third party services (Logstash), store and index them for fast search/retrieval (ElasticSearch) and visualize statistics in real time (Kibana). 

This is optimal when we have real-time data flows (such as with IoT devices, social media or web traffic), we want to track logs from big servers and analyze them and/or we want to search data-heavy applications, such as e-commerce ones.

If you want to know more, please refer to:

- [ElasticSearch docs](https://www.elastic.co/docs)
- [Logstash reference](https://www.elastic.co/guide/en/logstash/current/introduction.html)
- [Kibana guide](https://www.elastic.co/guide/en/kibana/current/index.html)

As you can see, these three services are all provided by [Elastic](https://www.elastic.co/).

## Set-up

### Getting the needed files

First of all, we clone the `awesome-compose` repository:

```bash
git clone https://github.com/docker/awesome-compose.git
```

And we head over to our folder of interest:

```bash
cd awesome-compose/elasticsearch-logstash-kibana/
```

> _**TIP**💡: you can use all the folders in this repository to experiment with different `compose` settings_

And we can take a look at the structure of the repository:

```bash
# You might need to install tree before using it
tree . -L 2
```

We will get out this structure:

```
.
|__ logstash/
|  |__nginx.log
|  |__pipeline/
|__compose.yaml
```

The `logstash` folder contains a `pipeline` subfolder and a `nginx.log` file: their purpose is not important for our scopes, but we have to keep in mind that they are there. 

### Adding some modifications

To showcase more of the `compose` file elements, we introduce a `.env` file, which we can create in this way:

```bash
touch .env
```

And then modify with our favorite text editor (for me it's VSCode):

```bash
code .env
```

In the `.env` file, let's create the following keys and values:

```bash
JAVA_OPTS="-Xms512m -Xmx512m"
DISCOVERY_SEED_HOSTS="logstash"
API_TOKEN="super-secret-token"
```

We will use them in the `compose` file (see below).

## The `compose` file

Let's take a look to the compose file, which is slightly different from the one proposed by `awesome-compose`:

```yaml
services:
  elasticsearch:
    image: elasticsearch:7.16.1
    container_name: es
    environment:
      discovery.type: single-node
      ES_JAVA_OPTS: $JAVA_OPTS # as we mentioned in the last article, compose can access the env variables we set in our .env file
    ports:
      - "9200:9200"
      - "9300:9300"
    healthcheck:
      test: ["CMD-SHELL", "curl --silent --fail localhost:9200/_cluster/health || exit 1"]
      interval: 10s
      timeout: 10s
      retries: 3
    networks:
      - elastic
  logstash:
    image: logstash:7.16.1
    container_name: log
    environment:
      discovery.seed_hosts: $DISCOVERY_SEED_HOSTS
      LS_JAVA_OPTS: $JAVA_OPTS
    secrets:
        - api_token
    volumes:
      - ./logstash/pipeline/logstash-nginx.config:/usr/share/logstash/pipeline/logstash-nginx.config
      - ./logstash/nginx.log:/home/nginx.log
    ports:
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "5044:5044"
      - "9600:9600"
    depends_on:
      - elasticsearch
    networks:
      - elastic
    command: logstash -f /usr/share/logstash/pipeline/logstash-nginx.config
  kibana:
    image: kibana:7.16.1
    container_name: kib
    volumes: 
        - kibana-cache:/etc/logs/cache
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
    networks:
      - elastic
networks:
  elastic:
    driver: bridge
secrets:
   api_token:
      environment: "API_TOKEN"
volumes:
    kibana-cache:
        external: true
        name: kibana_cache_volume
```

We will now break this `compose` file down service by service. 

### ElasticSearch

ElasticSearch is based on the Docker image **elasticsearch:7.16.1**, and the container that is launched within the service is named **es**. 

It accesses two environment variables: **ES_JAVA_OPTS**, that is read from the `.env` file, and **discovery.type**, which is set in-line. 

It is connected to two ports: **9200 and 9300**, accessible to the local device host on the same port addresses.

There is an **healthcheck**, that is performed for a maximum of three times with 10s intervals and a timeout (maximum time of execution of the test command) of 10s. 

The service is bound to the **elastic** network, whose driver is the most common: **bridge** (simply connects all the containers together, _bridging_ among them). 

This container does not depend on anything, so it is **the first to be started**. 

### Logstash

ElasticSearch is based on the Docker image **logstash:7.16.1**, and the container that is launched within the service is named **log**. 

It accesses two environment variables: **LS_JAVA_OPTS**, that is read from the `.env` file, and **discovery.seed_hosts**, which is also read from the .env` file.

It also has access to a secret, **api_token**, which is read from the environment file also as the **API_TOKEN** variable. 

Inside the container, two volumes are mounted from the local file system (the `pipeline` subfolder and the `nginx.log` file) into the container's system. 

It is connected to three ports: **5600, 5044 and 9300**, accessible to the local device host on the same port addresses.

There is **no healthcheck**, but the service depends on ElasticSearch so it is the second one to be started: as soon as the container is started, the command **`logstash -f /usr/share/logstash/pipeline/logstash-nginx.config`** is executed, overriding the eventual `CMD` entries in the original Dockerfile from the `logstash` image. 

The service is bound to the **elastic** network.

### Kibana

Kibana is based on the Docker image **kibana:7.16.1**, and the container that is launched within the service is named **kib**. 

It does not access environment variables or secrets.

Inside the container, there is one volume mounted, **kibana-cache**, the depends on a volume whose life cycle is externally managed, and which is named **kibana_cache_volume**.

It is connected to one ports: **5061**, accessible to the local device host on the same port address.

There is **no healthcheck**, but the service depends on ElasticSearch so it is the second one to be started, along with Logstash.

The service is bound to the **elastic** network.

## Launch and stop the service

Now, we can just launch our multi-container application with:

```bash
docker compose up
```

Or we can use:

```bash
docker compose up --detach
```

If we do not want the services logs to be displayed (and, potentially swamp) on our terminal.

If we want to see what is currently running, we can use:

```bash
docker compose ps
```

And if we want to stop the services, or take down the `compose` app, we can simply run:

```bash
# Stop the services without removing containers
docker compose stop
# Stop the services and remove containers
docker compose down
```

We will stop here for this article, and in the next one we will explore continuous-integration/continuous-deployment solution for Docker images. See you in 2025!🥰