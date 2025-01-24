---
title:  "1MinDocker #13 - Push, build and dockerize with GitHub Actions"
category: Intermediate 
---

In the [last article](https://dev.to/astrabert/1mindocker-12-what-is-cicd-2ap6) we talked about CI/CD: but how do we put in practice a CI/CD pipeline in a hassle-free and very simple framework?

Well, say no more! **GitHub Actions** have all that you need to set up a perfect environment to commit -> build -> dockerize your app. 

Let's dive in!

## Setting up - GitHub

We do not take anything for granted, so let's assume you don't have a GitHub account and you just wanna start from scratch:

- Head over to [GitHub Signup](https://github.com/signup) and register there with your email and password. You will be asked also to create a username
- Activating 2 factors authentication ([2FA](https://docs.github.com/en/authentication/securing-your-account-with-two-factor-authentication-2fa/configuring-two-factor-authentication)) is optional, but recommended
- Once you are signed in and set up, it's time to create your first repository!

To create a repository, you generally have to click on a `New` or a `Create new repository` green button: you will be prompted to choose the visibility (if the repo is public or private) and the name of the repository. I suggest that, for this tutorial, you create a **public repository called `hello-world-github-docker`**. 

## Setting up - Application

Now let's build an application: we'll be using Python, a versatile programming language that you can use for lots of things, from building app to data analysis, from creating websites to machine learning. 

Let's first of all clone the GitHub repository (i.e. make a local copy of it) with `git` (see how to install it [here](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) if you don't have it)

```bash
git clone https://github.com/username/hello-world-docker-github
# Remember to change 'username' with your actual username!
cd hello-world-docker-github/
```

And now, let's create and start editing our `app.py` file (`.py` is the extension that python scripts have):

```bash
touch app.py
code app.py
```

The `touch` command creates the file, whereas the `code` command opens the file within Visual Studio Code (a IDE, _Integrated Development Environment_) to modify it. You can obviously use different IDEs: no IDE is better than another one, as long as it works for you.

Generally, a "hello world" application prints "Hello world!" on the terminal, like this:

```python
print("Hello world!")
```

But we want to do something more: we don't like vanilla white text on our terminal, we want some 🌈color🌈.

To do this, we simply need to install the [`termcolor`](https://pypi.org/project/termcolor/) package:

```bash
pip install termcolor
```

Now we just import and use it into our script:

```python
from termcolor import cprint
# cprint stands for "colorer print", and allows us to print colored text

cprint("Hello world!", color="red")
```

Now the printed text will be red, but let's add even some more spice, and let the program choose a random color to use in printing the string to terminal:

```python
from termcolor import cprint
import random
# random is the library that allows us to extract random items from a list
colors = ["red", "green", "blue", "magenta", "yellow"]

color = random.choice(colors)

cprint("Hello world!", color=color)
```

Now our script will randomly choose one of the colors and print the "Hello world!" in that color :)

> _**NOTE**: `random` is a built-in library in python, which means it comes within the language: that's why we did not need to install it._

## Setting up - Docker

Ok, now we have our application: how can we dockerize it?

Our first option could be to write down a Dockerfile, build an image from that and push it to Docker Hub. That's a viable solution, but we would need to re-build the image on our local machine and push it to Docker Hub every time we make a change to the app. While our "Hello world" app is relatively small and that would not be a burden for our computer, we for sure would want to avoid this with bigger applications. 

Another solution could be, as we said, writing the Dockerfile, and then uploading our application to GitHub and let the platform take care of building and pushing the image to `ghcr`, i.e. _GitHub Container Registry_, a registry where the Docker images built on GitHub (also known as _packages_) are stored. 

Since our goal is to exploit GitHub Actions, let's do that!

The first thing we have to do is to create a `requirements.txt` file that will install all the necessary dependencies inside our Docker image. In our case, we only need `termcolor`, so we can just do it like this:

- Create and open the file for editing:

```bash
touch requirements.txt
code requirements.txt
```

- Write the `termcolor` package in the file: 

```
termcolor
```

Now let's build our `Dockerfile`. We want our image to contain `python`, and we want it also to install the needed dependencies, so let's define it like this:

```dockerfile
# Define your python version
ARG PY_VERSION="3.11.9-slim-bookworm"

# Base image
FROM python:${PY_VERSION}

# Define your working directory
WORKDIR /app/
# Copy your local file system into the working directory
COPY ./ /app/

# Install the necessary dependencies
RUN pip cache purge
RUN pip install --no-cache-dir -r requirements.txt

# Run the application as an entrypoint
ENTRYPOINT python3 app.py
```
Now our local folder structure will look like this:

```
.
|__ app.py
|__ requirements.txt
|__ Dockerfile
```

The only thing that we need is to configure a workflow file that will trigger GitHub Actions and tell them to build and push the image. For this, we can use the pre-built template offered by GitHub.

We need to first of all create the file:

```bash
# The file is placed in a special directory, .github/workflows
mkdir -p .github/workflows
touch .github/workflows/docker-publish.yml
code .github/workflows/docker-publish.yml
```
As you can see, the workflow file is in YAML format, a powerful markup language that allows you to specify all the steps you want in the build. Copy and paste the below text in the file you are now editing:

```yaml
name: Docker

# This workflow uses actions that are not certified by GitHub.
# They are provided by a third-party and are governed by
# separate terms of service, privacy policy, and support
# documentation.

on:
  schedule:
    - cron: '40 8 * * *'
  push:
    branches: [ "main" ]
    # Publish semver tags as releases.
    tags: [ 'v*.*.*' ]
  pull_request:
    branches: [ "main" ]

env:
  # Use docker.io for Docker Hub if empty
  REGISTRY: ghcr.io
  # github.repository as <account>/<repo>
  IMAGE_NAME: ${{ github.repository }}


jobs:
  build:

    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      # This is used to complete the identity challenge
      # with sigstore/fulcio when running outside of PRs.
      id-token: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      # Install the cosign tool except on PR
      # https://github.com/sigstore/cosign-installer
      - name: Install cosign
        if: github.event_name != 'pull_request'
        uses: sigstore/cosign-installer@59acb6260d9c0ba8f4a2f9d9b48431a222b68e20 #v3.5.0
        with:
          cosign-release: 'v2.2.4'
      # Set up BuildKit Docker container builder to be able to build
      # multi-platform images and export cache
      # https://github.com/docker/setup-buildx-action
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@f95db51fddba0c2d1ec667646a06c2ce06100226 # v3.0.0

      # Login against a Docker registry except on PR
      # https://github.com/docker/login-action
      - name: Log into registry ${{ env.REGISTRY }}
        if: github.event_name != 'pull_request'
        uses: docker/login-action@343f7c4344506bcbf9b4de18042ae17996df046d # v3.0.0
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # Extract metadata (tags, labels) for Docker
      # https://github.com/docker/metadata-action
      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@96383f45573cb7f253c731d3b3ab81c87ef81934 # v5.0.0
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}

      # Build and push Docker image with Buildx (don't push on PR)
      # https://github.com/docker/build-push-action
      - name: Build and push Docker image
        id: build-and-push
        uses: docker/build-push-action@0565240e2d4ab88bba5387d719585280857ece09 # v5.0.0
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      # Sign the resulting Docker image digest except on PRs.
      # This will only write to the public Rekor transparency log when the Docker
      # repository is public to avoid leaking data.  If you would like to publish
      # transparency data even for private images, pass --force to cosign below.
      # https://github.com/sigstore/cosign
      - name: Sign the published Docker image
        if: ${{ github.event_name != 'pull_request' }}
        env:
          # https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#using-an-intermediate-environment-variable
          TAGS: ${{ steps.meta.outputs.tags }}
          DIGEST: ${{ steps.build-and-push.outputs.digest }}
        # This step uses the identity token to provision an ephemeral certificate
        # against the sigstore community Fulcio instance.
        run: echo "${TAGS}" | xargs -I {} cosign sign --yes {}@${DIGEST}
```
It's now time to test this workflow!

## Testing - First push

Let's first of all add, commit and push all the local changes to the online repository:

```bash
git add .
git commit -m "first commit"
git branch -M main
git push -u origin main
```

You will be prompted to insert your GitHub username and password. As password, you should use a GitHub access token, that you can create [here](https://github.com/settings/tokens). 

Once the change is pushed, you will see, on your online GitHub repo, a brown dot: it means that the workflow we triggered with our push is now working and will build and push the image. If the workflow run is successful, you will see a green tick at the end of it, whereas if it fails, you will see a red cross. 

Make sure that, once the package is created, it is **public**:  if not, change its visibility to public, otherwise you won't be able to download it. 
## Testing - Trying our app

As we said, the Docker image will be loaded as a package under `ghcr.io`; we can then pull and run the image like this:

```bash
docker pull ghcr.io/username/hello-world-github-docker:main
docker run -t ghcr.io/username/hello-world-github-docker:main
```

If Docker says that it can't pull the image because you are not logged in to the GitHub Container Registry, you can simply run:

```
docker login ghcr.io -u username -p GITHUB-ACCESS-TOKEN
```

And this should do the magic!

When we run our application, we should see a colored "Hello world!" printed on the terminal.

## Modifying our application

Now, let's say that we want to let the user choose the color they want "Hello world" to be printed with. We can modify our `app.py` like this:

```python
from termcolor import cprint

colors = ["red", "green", "blue", "magenta", "yellow"]

# Tell the user the instructions
print(f"Hello user! What color would you like 'Hello world' to be printed with?\nChoose among: {', '.join(colors)}")

# Take the input from the user
color = input("-->")

# Check if the input is in the available colors: if not, tell the user that it is not available
if color.lower() in colors:
    cprint("Hello world!", color=color)
else:
    print("ERROR! The color you chose is not among the available colors :(")   
```

To modify the Docker image, it is now sufficient to add, commit and push the local changes to the online repo:

```bash
git add .
git commit -m "user defined color"
git push origin main
```

If the workflow run is successful again, we should be able to pull and run the new image with updated features:

```bash
docker pull ghcr.io/username/hello-world-github-docker:main
docker run -it ghcr.io/username/hello-world-github-docker:main
```

We will stop here for this article, but in the next one we will see, hands-on, how to use Docker for on-cloud deployments: stay tuned and have fun!🥰
