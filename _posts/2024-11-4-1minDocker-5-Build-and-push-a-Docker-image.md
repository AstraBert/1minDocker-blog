---
title:  "1MinDocker #5 - Build and push a Docker image"
category: Intermediate 
---

In the [last article](https://dev.to/astrabert/1mindocker-4-docker-cli-essentials-33pl) we saw how we can pull an image, run it inside a container, list images and containers and remove them: now it's time to build, so we'll create our first simple Docker image.

<!-- more -->

### The Dockerfile

As we already said in our [conceptual introduction to Docker](https://dev.to/astrabert/1mindocker-3-fundamental-concepts-55ph), a Dockerfile is a sort of recipe: it contains all the instructions to collect the ingredients (the *image*) that will make the cake (the _container_). 

But what exactly can a Dockerfile contain? We will see, in our example (that you can find [here](https://github.com/AstraBert/1minDocker/tree/master/code_snippets/build_an_image_1)), the following base key words:

- `FROM`: this key word is fundamental. It specifies the base image from which we mount our environment
- `RUN`: with this key you can specify a command (like `RUN python3 -m pip install --no-cache-dir requirements.txt`) that will be executed during _build time_ (only once) and will be stored in an image layer
- `WORKDIR`: you can specify the working directory that will be the base for your Docker image (for example `WORKDIR /app/`)
- `COPY` or `ADD`: These two key words are very similar. `COPY` allows you to copy specific local folders into a folder inside the image (like `COPY src/ /app/`) whereas `ADD` adds the whole local specified path to the destination directory inside the Docker image (`ADD . /app/`)
- `EXPOSE`: it specifies the port that is exposed inside the container to the outside (`EXPOSE 3000`)
- `ENTRYPOINT`: this key word specifies the default executable that should be run when the image is launched in a container (`ENTRYPOINT ["npm", "start"]`). It must be specified at the end of your Docker file and only once (otherwise the last `ENTRYPOINT` instance will override the other ones). Although the `ENTRYPOINT` executable cannot be overridden by other commands provided through CLI when we run the container, it's arguments can be changed from CLI upon container start.
- `CMD`: Similar to `ENTRYPOINT`, this key word specifies a command that runs every time the image is started inside a container. Differently from `ENTRYPOINT`, though, it can be completely overridden and generally is used as a set of extra arguments for `ENTRYPOINT`, like here:
```Dockerfile
ENTRYPOINT [ "streamlit", "run" ]
CMD [ "scripts/app.py" ]
```
In this case, every time we start the container we will run a Streamlit app, but we can choose the path of the app by providing it to the container from the `docker run` command line.
- `ARG`: this key word is used to set build arguments, which are local variable that can be overridden by other specified at build-time with the `docker build` CLI. They're especially useful if you use a value more than once in your Dockerfile and don't want to repeat it:
```Dockerfile
ARG NODE_VERSION="20"
ARG ALPINE_VERSION="3.20"

FROM node:${NODE_VERSION}-alpine${ALPINE_VERSION}
```

This can be easily overridden by:
```
docker build . --build-args NODE_VERSION="18" 
```
- `ENV`: this key word, as the name suggest, sets an _environment_ variable. Environment variables are fixed and cannot be changed at build-time, and they can be useful when we want a variable to be accessible to all image build stages.

### Let's build a Dockerfile

To build a Dockerfile, we need to know what application we are going to ship through the image we're about to set up.

In this tutorial, we will build a very simple python application with [Gradio](https://gradio.app), a popular framework to build elegant and beautiful frontend for AI/ML python apps.

Our folder will look like this:

```
build_an_image_1/
|__ app.py
|__ Dockerfile
```

To fill up `app.py`, we will use a template that [Hugging Face](https://huggingface.com) itself provides for Gradio ChatBot Spaces:


```python
import gradio as gr


def respond(
    message,
    history):
    message_back = f"Your message is: {message}"
    response = ""
    for m in message_back:
        response += m
        yield response

demo = gr.ChatInterface(
    respond,
    title="Echo Bot",
)

if __name__ == "__main__":
    demo.launch(server_name="0.0.0.0", server_port=7860)
```

This is a simple bot that echoes every message we send. 
We will just copy this code into our main script, `app.py`.

Now we're ready to build our Docker image, starting with modifying our Dockerfile.

#### 1. The base image

For our environment we need Python 3, so we will need to find a suitable base image for that.

Luckily, Python itself provides us with Alpine-based (a Linux distro) python images, so we will just use `python:3.11.9`.

We just then need to specify:

```Dockerfile
ARG PYTHON_VERSION="3.11.9"
FROM python:${PYTHON_VERSION}
```

At the very beginning of our Dockerfile.

As we said, if we want a different python version, we just need to run:

```bash
docker build . --build-args PYTHON_VERSION="3.10.14"
```

#### 2. Get the needed dependencies

Our app depends exclusively on `gradio`, so we can do a quick `pip install` for that!

We also set the version (5.4.0) as an ARG and ENV:

```Dockerfile
ARG GRADIO_V="5.4.0"
ENV GRADIO_VERSION=${GRADIO_V}

RUN python3 -m pip cache purge
RUN python3 -m pip install gradio==${GRADIO_VERSION}
```

You cannot change `GRADIO_VERSION` directly, but you can pass `GRADIO_V` as a build argument and modify also the `ENV` value!

```bash
docker build . --build-args GRADIO_V="5.1.0"
```

#### 3. Start the application

We need to start the application, something that we would normally do as `python3 app.py`.

But our `app.py` file is locally stored, not available to the Docker image, so we need to copy it into our Docker working directory:

```Dockerfile
WORKDIR /app/
COPY ./app.py /app/
```

Since our application runs on http://0.0.0.0:7860, we need to expose port 7860:

```Dockerfile
EXPOSE 7860
```

Now we can make our application run:

```Dockerfile
ENTRYPOINT ["python3"]
CMD ["/app/app.py"]
```

We will not be able to change the base executable (`python3`) but we will be able to override the `CMD` instance specifying another path at runtime (for example if we mount a volume while running the container).

#### 4. Full Dockerfile

Our full Dockerfile will look like this:

```Dockerfile
ARG PYTHON_VERSION="3.11.9"
FROM python:${PYTHON_VERSION}

WORKDIR /app/
COPY ./app.py /app/

ARG GRADIO_V="5.4.0"
ENV GRADIO_VERSION=${GRADIO_V}

RUN python3 -m pip cache purge
RUN python3 -m pip install gradio==${GRADIO_VERSION}

EXPOSE 7860

ENTRYPOINT ["python3"]
CMD ["/app/app.py"]
```

Now we just need to build the image!

### Build and push the image

When we build the image, we need to specify the _context_, meaning the directory in which our Dockerfile is. For starters, we will also use the `-t` flag, which specifies the _name_ and *tag* of our image:

```bash
docker build . -t YOUR-USERNAME/gradio-echo-bot:0.0.0 -t YOUR-USERNAME/gradio-echo-bot:latest
```

As you can see, you can specify multiple tags.

This build, once launched, will take some minutes to complete, and then you will have your images locally!

If you want to make this images available to everyone, you need to login to your Docker account:

```bash
docker login -u YOUR-USERNAME --password-stdin
```

You will be prompted to input the password from your console. 

You won't put your Docker password, but an [access token](https://docs.docker.com/security/for-developers/access-tokens/#create-an-access-token) (follow the link for a guide on how to obtain it). 

Now let's push our image to the Docker Hub registry:

```bash
docker push YOUR-USERNAME/gradio-echo-bot:0.0.0
docker push YOUR-USERNAME/gradio-echo-bot:latest
```

The push generally takes some time, but after that our image will be live on Docker Hub: we published our first Docker image!🎉

We will stop here for this article, but in the next one we will dive into more advanced build concepts🥰 