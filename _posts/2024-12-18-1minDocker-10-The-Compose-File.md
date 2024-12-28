---
title:  "1MinDocker #10 - The Compose File"
category: Intermediate 
---

In the [last article](https://dev.to/astrabert/1mindocker-9-introduction-to-compose-11di) we introduced `compose`, a popular Docker plugin to build multi-container applications and to manage complex environments in an easy and sharable way.

In this post we will focus on the `compose` file, i.e. the YAML file that contains the instructions that `docker compose` reads and runs when it is launched.

As we saw for the Dockerfile, also the compose file has keywords: these keywords are named *elements*, and the most important of them are known as *top level* elements. We will learn about them in the following paragraphs.

## `name` and `version`

The `version` top element is obsolete, and it is used only for backward compatibility with older version of Compose, where the program actually validated the YAML file structured according to a precise schema known as *Specification*. Newer versions of `compose` no longer parse their input file for validation and, if they encounter a wrongly compiled/unknown field, `compose` would simply throw an error.

The `name` top level element is set to give a name to the project you are launching with `compose`, and overrides the default ones. 

For example:

```yaml
name: new_app

services:
    app:
        image: foo/bar
        command: echo "I'm running ${COMPOSE_PROJECT_NAME}"
```

## `services`

The `services` elements defines the various containers your `compose` project will run, with several potential configurations and specifications.

There are numerous elements linked to the `services` one, we will go through the most used (excluding the ones referenced in the next sections):
### image

Specifies the image that the container is running on: if the image has not already been pulled locally, it is pulled from the hub on the fly when the service is started.

```yaml
services:
   app:
      image: node:18-alpine
      ...
   db:
      image: Postgres
      ...

```

### build

If you want your container to run on a custom image you configured through a Dockerfile, you can use the `build` element, which will build the image on the fly based on the context provided (you can also specify the Dockerfile name):

```yaml
services:
   app:
      build: .
      dockerfile: "Dockerfile.node"
      ...
   db:
      image: postgres
      ...
```

### env_file and environment

`compose` by default can read the environment variables you set in a `.env` file is that is placed in the same directory in which the `compose.yml` file is situated. Nevertheless, you can specify your environment file through the `env_file` element:

```yaml
env_file: "./envs_config/.raw.env"
```

You can also specify the format of the `env_file` (more on Docker docs [here](https://docs.docker.com/reference/compose-file/services/#format)) and if it is required or not (more on Docker docs [here](https://docs.docker.com/reference/compose-file/services/#required)).

For example, you could write:

```yaml
env_file:
  - path: ./default.env
    required: true # default
    format: raw
  - path: ./override.env
    required: false
```

The `environment` element works as an env file, but from inside the compose file:

```yaml
environment:
    - PG_USR: user
    - PG_PSW: password
    - PG_DATABASE: postgres
```

If the `environment` element is used in the `compose` file, it has priority over the same variables used in the env file.

### depends_on

The `depends_on` element is useful when it comes to set the order in which several services are started.

For example:

```yaml
services:
  app:
    build: .
    depends_on:
      - db
      - redis
  redis:
    image: redis
  db:
    image: postgres
```

In this case, the `app` container is built only after the `db` and `redis` one are ready. 

This can be accompanied by `conditions` on how to actually control the starting of a container:

```yaml
services:
  app:
    build: .
    depends_on:
      db:
        condition: service_healthy
        restart: true
      redis:
        condition: service_started
  redis:
    image: redis
  db:
    image: postgres
```

In this case, the `app` container is started only if the `db` container passes its health check (see below) and when the `redis` container is started (no need for health check).

### command and entrypoint

`command` element specifies a command that overrides the execution of a `CMD`-dependent command from the Docker image.

For example:

```yaml
command: bundle exec thin -p 3000
```

`entrypoint`, on the other hand, overrides the `ENTRYPOINT` set for the service's Docker image:

```yaml
entrypoint: bash /app/post_create_command.sh
```
### ports

Ports associated with the service and exposed from the container: 

```yaml
services:
    semantic_db:
        image: qdrant/qdrant:latest
        volumes: 
            - "./qdrant_storage:/qdrant/storage"
        ports:
            - "6333:6333"
            - "6334:6334"
```

In this case, the `semantic_db` container will have associated and exposed the ports 6333 and 6334, which will be accessible to the user on their `localhost` under the same port number.

### restart

It can happen that a container fails to start or terminates abruptly/prematurely its execution. `restart` takes care of this, defining what is the policy when termination happens:

```yaml
restart: "no" # no restarting whatsoever 
restart: always # always restart upon termination
restart: on-failure # restart only if the container produced an error
restart: on-failure:3 # restart max 3 times on failure
restart: unless-stopped # restart only if the container wasn't stopped or removed externally
```

### healthcheck

`healthcheck` element is used to test the correct functioning of the service to which it is associated. It generally uses this syntax:

```yaml
healthcheck:
    - test: ["CMD", "curl","-f", "http://localhost"] 
    - interval: 1m10s
    - timeout: 30s
    - retries: 5
    - start_period: 30s
    - start_interval: 5s
```

- **test** is the command to be executed during the check. CMD or CMD-SHELL specify that the command is executed in the default shell for the container (`/bin/sh` for Linux, generally).
- **interval** is the time that occurs between retries.
- **timeout** is the maximum duration for a health check before it is considered failed
- **retries** sets the maximum number of failures before the container is considered unhealthy 
- **start_period** is the "protected time" in which health checks occur during the start of a container that needs bootstrap. If these health checks fail, they do not count toward the maximum number of retries, whereas if they pass the container is considered started
- **start_interval** works as interval but for health checks during the start time 

### container_name

Set the name of the container, in order to make it easier to detect it when you run `docker ps -a`:

```yaml
services:
    app:
        image: node:alpine-18
        container_name: "reactjs_app"
        ...        
```

## `volumes`

`volumes` is a top-level element that ensures that data from the local file system are injected into the container. `volumes` is both a top level element and an attribute for `services` elements, and to ensure that a service has access to the volume you need to explicitly specify it inside the service specification itself.

```yaml
services:
    app:
        image: python:3.11.9-slim-bookworm
        volumes: 
            - app-data:/app/data/

volumes:
    app-data:
```

If you don't need to mount anything inside the `/app/data` path, you can simply leave the `app-data` field under `volumes` blank. Otherwise, you simply have to specify the path to your data in the local file system.

A volume, like a network (see below), can have a `driver` (whose options are specified through `driver_opts`) and can be managed outside the container (`external: true` is specified).


Let's see a complex example:

```yaml
services:
    app:
        image: python:3.11.9-slim-bookworm
        volumes: 
            - app-data:/app/data/
            - app-cache:/etc/logs/cache

volumes:
    app-data:
        driver: local
        driver_opts:
              type: none
              device: /data/db_data
              o: bind
    app-cache:
        external: true
        name: appcache_vol
```

## `networks`

`networks` are a very important element for a `compose` file, because they allow the different services to communicate with each other, instead of isolating them. `compose` by default sets a single network for your app, but this is not always optimal: we may want some networks to be accessible only to specific services, and that's why we should specify the networks attached to each of our services.

For example:

```yaml
services:
    frontend:
        image: user/webapp
        networks:
            - foo
            - bar

networks:
    foo:
    bar:
```

Networks can obviously be configured, so let's see two important elements in their configuration:

- **driver**: this attribute provides information about the driver used to build the network and provide its core functionalities: `bridge` is the default one (ensure communication among containers in your app), but you can also use `host` (exploit directly the host networking, removing network isolation between the container and the Docker host), `overlay` (allow connectivity across different Docker daemons, networking across nodes for Swarm services), `ipvlan` (gives user control over IPv4 or IPv6 addressing and may be used for underlay network integration), `macvlan` (assigns your MAC address to a container, making it a visible device in your network) or `none` (completely isolate the container's network). Drivers can be configured through **driver_opts**
- **internal**/**external**: specified if the network is managed outside or inside the application. By default, every network is internal and, if set external, `compose` throws an error if it is not able to connect to it.

Let's see a complete example:

```yaml
services:
  proxy:
    build: ./proxy
    networks:
      - frontend
      - outside
  app:
    build: ./app
    networks:
      - frontend
      - backend
  db:
    image: postgres
    networks:
      - backend

networks:
  frontend:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.host_binding_ipv4: "127.0.0.1"
  backend:
    driver: bridge
  outside:
    external: true
```
## `configs`

`configs` are specific configurations that can be accessed by services (if explicitly declared under the `configs` attribute) and that modify a Docker image without having to build it from scratch.

Configs are by default owned by the user who is running the services and generally have world-readable permissions (that can be overridden by the services if they are configured to do so).

Configs have the following attributes:

- **file**: the configuration file for the container (provided as a path referring to the local file system)
- **environment**: the configuration is set as an environment variable
- **content**: configuration is passed in-line inside the `compose` file
- **external**: the config was already created and its lifecycle is externally managed
- **name**: the name of the configuration (by default is `<project_name>_config_key`)

Let's see an example:

```yaml
services:
    app:
      image: foo/bar
      configs:
         - app_config
    http_server:
      build: ./http_server/
      configs:
         - http_config
    db:
       image: postgres
       configs:
         - db_config

configs:
  http_config:
    file: ./httpd.conf
  app_config:
    content: |
      debug=${DEBUG}
      spring.application.admin.enabled=${DEBUG}
      spring.application.name=${COMPOSE_PROJECT_NAME}
  db_config:
    external: true 
```

## `secrets`

A secret can be specified as a file or an environment variable, and to be accessed by a service, it has to specified as a `service` attribute. Here is an example:

```yaml
services:
  frontend:
    image: example/webapp
    secrets:
      - server-certificate
  db:
    image: postgres
    secrets:
       - postgres_psw
       - postgres_user
       - postgres_db
secrets:
  server-certificate:
    file: ./server.cert
  postgres_psw:
    environment: "POSTGRES_PSW"
  postgres_user:
    environment: "POSTGRES_USER"
  postgres_db:
    environment: "POSTGRES_DB"
```

We will stop here for this article, but in the next one we will explore an advanced `compose` example, in which we will see the nuances of this powerful plugin🥰


> _The content for this article is mainly based on [`docker compose` file reference documentation](https://docs.docker.com/reference/compose-file/) : make sure to visit them to get to know more!_
