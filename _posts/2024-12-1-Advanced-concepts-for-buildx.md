---
title:  "1MinDocker #8 - Advanced concepts for buildx"
category: Intermediate 
---

In the [last article](https://dev.to/astrabert/1mindocker-7-superpower-your-builds-with-buildx-123m), we started using `buildx` to add more building capacity to our Docker core.

<!-- more -->

In this article, we will dive deep into `buildx`'s subcommands.



### `docker buildx bake`

#### Starting example
`bake` is a high-level command for `buildx`. 
It is able to automate the build for multiple images at once, taking as reference a JSON, compose or HCL (HashiCorp configuration Language) file.

On a smaller scale, `bake` does not make any difference from build. If we consider having only one image to build, there is no performance gap, and:

```
docker build . -t user/name:tag
```

Is the same as building the following HCL file:

```hcl
target "image" {
    dockerfile = "Dockerfile"
    tag = ["user/image:tag"]
}
```

And then run:

```
docker buildx bake image
```

Things change when we have multiple images to build together.

#### Bake file

Let's nevertheless take a step back and ask ourselves: how do we build a bake file? We will explore the HCL format, because it is the easiest and the most intuitive to use.
The file structure resembles the one of a JSON, and has the following three main keywords:

- `target`: objects that are specified under this key are images that should be built. Target objects generally contain information on the context on which we are building the Docker image and on the tags to assign.
- `group`: a list of targets are put under this keyword, so that everytime we want to build all the images together we can just bake the group name instead of calling the targets one by one
- `variable`: works as an ARG or an ENV in a Dockerfile. It sets a variable that can be used downstream in the HCL file

Let's look at an example:

```hcl
group "all" {
    targets = ["app", "db"]
}

variable PYTHON_TAG {
    default = "3.11.9-slim-bookworm"
}

target "backend" {
    dockerfile = "Dockerfile.backend"
    tag = ["user/python-backend:prod", "user/python-backend:latest"]
    args = {
        PYTHON_VERSION = ${PYTHON_TAG}
    }
}

target "db" {
    dockerfile = "Dockerfile.postgres"
    tag = ["user/postgres-db:prod", "user/postgres-db:latest"]
    no-cache = true
    platforms = ["linux/amd64", "linux/arm64"]
}
```

Now, we could run:

```bash
docker build . \
    -f Dockerfile.backend \
    -t user/python-backend:prod \
    -t user/python-backend:latest

docker build . \
    -f Dockerfile.postgres \
    -t user/postgres-db:prod \
    -t user/postgres-db:latest
```

Or we could also run:

```bash
docker buildx bake backend db
```

But the easiest way to do this is to leverage the group of targets that we specified:

```bash
docker buildx bake all
```

Bake will take care of the two builds at the same time.

### `docker buildx create`

The `create` subcommand will create a new build environment instance. You can append some context to it as a node:

```bash
docker buildx node-0
```

This will produce an environment with a name which will be returned on your terminal (let's say `happy_euclid`).

You can use this name to append a new node to the environment.

```bash
docker buildx --name happy_euclid --append node-1
```

You can use the `--node` flag with the node name to modify or create a node.

`create` should be provided with a daemon configuration file through the `--buildkitd-config` flag (if not, it defaults to the `buildkitd.default.toml` file contained in the config directory of `buildx`). You can find an example of a complete configuration file in [`buildkit` official documentation](https://github.com/moby/buildkit/blob/master/docs/buildkitd.toml.md) on GitHub.

If you nevertheless want to specify some BuildKit configuration flags for your builder instances overwriting the ones of the config file, you can do it by adding the `--buildkitd-flags` option:

```bash
docker buildx create node-0 --buildkitd-config ./buildkitdconfig.local.toml --buildkitd-flags '--debug --debugaddr 0.0.0.0:6666'
```

You should also specify the driver (see [last article](https://dev.to/astrabert/1mindocker-7-superpower-your-builds-with-buildx-123m)) for your builder instances with the `--driver` option: the default one is `docker` (your local Docker), but you can also choose `docker-container` (runs locally but based on a Docker image), `kubernetes` (a Kubernetes pod) and `remote` (a remote environment to which you're connected).

If you want to specify the platform(s) for which a builder is intended, you can do that passing the `--platform` option (like `--platform linux/amd64` or `--platform darwin/amd64,linux/arm64`).

Deleting a node is also very simple: you just add the `--leave` flag followed by the name of the node you want to eliminate (specifying the name of the builder and the name of the node):

```bash
docker buildx create --name kitty_builds --node kitty0 --leave
```

### `docker buildx build` 

The `build` subcommand, as one might expect, has lots of options. Let's focus on the most important ones:

#### `--build-arg`

This option passes arguments for the build as in the following example:

```bash
docker buildx build --build-arg HTTP_PROXY=http://10.20.30.2:1234 --build-arg FTP_PROXY=http://40.50.60.5:4567 .
```

Arguments here are passed only at build-time (so not exposed while running the image) and can only modify non-persistent arguments in a Dockerfile set with the `ARG` keyword.

#### `--build-context`

This option sets additional building context for our build operation. For example you can specify an additional Docker image or stage that can be accessed through the Dockerfile using the `FROM` keyword or the `--from` flag:

```bash
docker buildx build --build-context myimage=docker-image://myimage@sha256:0123456789 .
```

The argument can be also a local or remote directory:

```bash
docker buildx build --build-context project=path/to/project/source .

docker buildx build --build-context gitproject=https://github.com/myuser/project.git .
```

You can access all this values in the Dockerfile:

```Dockerfile
FROM myimage

COPY --from=project node_modules/* /app/node_modules/
COPY --from=gitproject src/* /app/sec/
```

####  `--cache-from`

With `--cache-from`, you can import a previously existent cache for your build from a local folder, a GitHub repo, a Docker registry cache or a S3 bucket.

Here are some examples on how to use this option command:
 
```bash
## IMAGE REGISTRY - 1
docker buildx build --cache-from=user/image:cache .

## IMAGE REGISTRY - 2
docker buildx build --cache-from=user/image .

## IMAGE REGISTRY - 3
docker buildx build --cache-from=type=registry,ref=ghcr.io/user/image .

## LOCAL
docker buildx build --cache-from=type=local,src=path/to/cache .

## GITHUB REPO
docker buildx build --cache-from=type=gha .

## S3 BUCKET
docker buildx build --cache-from=type=s3,region=eu-west-1,bucket=mybucket .
```

####  `-f,--file`

Specify the Dockerfile for your build.

#### `--load`

Load the image resulting from the build into a local Docker image. This flags is the same as setting `--output=type=docker` 

#### `--push`

Push directly the image resulting from the build to a registry. This flag is the same as setting: `--output=type=registry`

#### `--platform`

Specify the target platform for which you are building the image.

The platform specification should follow the `os/arch` or `os/arch/variant` syntax and can be also a list of comma-separated platforms, but only if you are not using `docker` as a driver.

You can also configure the platform as `local`, which makes `buildx` picking the local platform on which BuildKit is configured for the build.

#### `--secret`

You can expose a secret during a build, mounting it inside your Dockerfile and on your command line from a file (`type=file`) or from an environmental variable (`type=env`). 

If you're using a file-based secret, you should specify the file origin:

```bash
docker buildx build --secret type=file,id=hf_token,src=$HOME/.gitcredentials/HF_TOKEN .
```

Abd you can import it inside your Dockerfile like this:

```Dockerfile
FROM python:3.11.9-slim-bookworm
RUN pip install huggingface-cli
RUN --mount=type=secret,id=hf_token,target=/root/.gitcredentials/HF_TOKEN \
  huggingface-cli login --token $hf_token
```

Using `type=env` instead loads the secret from an environmental variable. 

You can set it like this:

```bash
export SECRET_TOKEN=token 
docker buildx build --secret id=SECRET_TOKEN .
```

As long as the ID matches with the name of the environmental variable, you don't have to specify `type=env`.

You can import it in your Dockerfile with:

```Dockerfile
# syntax=docker/dockerfile:1
FROM node:alpine
RUN --mount=type=bind,target=. \
  --mount=type=secret,id=SECRET_TOKEN,env=SECRET_TOKEN \
  yarn run test
```

You can also use the `src`/`source` flag but you need to specify `type=env` otherwise `buildx` will look for a file named as the name you reported for `src`/`source`:

```bash
export API_KEY=sk-your-supersecret-key-api
docker buildx build --secret type=env,id=api,src=API_KEY .
```

This might be useful when you don't want your secret's ID to match with the name of the environmental variable.

#### `-t, --tag` 

Used to specify the name and tag of an image for the build.

### Minor commands

- `docker buildx imagetools`: it helps with managing registry-based images creating new ones from a list of manifests and/or inspecting already-existent manifests, for instance to check for multi-platform attestation.
- `docker buildx use`: Changes the current builder instances to the specified one.
- `docker buildx rm`: Removes the specified builder instance(s).
- `docker buildx prune`: Removes data from a builder cache, giving you precise control over data elimination.
- `docker buildx stop`: Stops the current specified builder instance but allows restarting, and is driver-dependent

We will stop here for this article, but in the next one we will dive into `compose`, another popular Docker plugin🥰


> _The content for this article is mainly based on [`docker buildx` command documentation](https://docs.docker.com/reference/cli/docker/buildx/) : make sure to visit them to get to know more!_