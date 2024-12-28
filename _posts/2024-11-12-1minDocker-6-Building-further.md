---
title:  "1MinDocker #6 - Building further"
category: Intermediate 
---

In [the last article](https://dev.to/astrabert/1mindocker-5-build-and-push-a-docker-image-1kpm) we saw how to build an image from scratch and we introduced several keywords to work with Dockerfiles. 

We will now try to understand how to take our building capacity to the next level, adding more complexity and more layers to our images.

### Case study

Imagine that we want to build an image to run our data analysis pipelines written in python and R.

To manage python and R dependencies separately we can wrap them inside [conda](https://docs.conda.io/projects/conda/en/latest/index.html) environments.

Conda is a great tool for environment management, but is often outpaced by [mamba](https://mamba.readthedocs.io/en/latest/installation/mamba-installation.html) in some operations such as environment creation and installation.

We will then use conda to organize and run the environments, while mamba will create them and install what's needed.

Let's say we need the following packages for python data analysis:

+ [pandas](https://pandas.pydata.org/)
+ [polars](https://pola.rs/)
+ [numpy](https://numpy.org/)
+ [scikit-learn](https://scikit-learn.org/stable/)
+ [scipy](https://scipy.org)
+ [matplotlib](https://matplotlib.org/)
+ [seaborn](https://seaborn.pydata.org/)
+ [plotly](https://plotly.com/python/)

And we need the following for our R data analysis:

- [dplyr](https://dplyr.tidyverse.org/)
- [ggplot2](https://ggplot2.tidyverse.org/)
- [tidyr](https://tidyr.tidyverse.org/)
- [caret](https://topepo.github.io/caret/)
- [purrr](https://purrr.tidyverse.org/)
- [lubridate](https://lubridate.tidyverse.org/)

We store the environment creation and the installation of everything in this file called `conda_deps_1.sh` (find all the code for this article [here](https://github.com/AstraBert/1minDocker/tree/master/code_snippets/build_an_image_2)):

```bash
eval "$(conda shell.bash hook)"

micromamba create \
    python_deps \
    -y \
    -c conda-forge \
    -c bioconda \
    python=3.10

conda activate python_deps

micromamba install \
    -y \
    -c bioconda \
    -c conda-forge \
    -c anaconda \
    -c plotly \
    pandas polars numpy scikit-learn scipy matplotlib seaborn plotly

conda deactivate

micromamba create \
    R \
    -y \
    -c conda-forge \
    r-base

conda activate R

micromamba install \
    -y \
    -c conda-forge \
    -c r \
    r-dplyr r-lubridate r-tidyr r-purrr r-ggplot2 r-caret

conda deactivate  
```
From these premises, we will build our data science Docker image. 

### Building on top of the building

We are very lucky with mamba and conda, because they both provide a docker image for their smaller and lightweight versions, [micromamba](https://hub.docker.com/r/mambaorg/micromamba) and [miniconda](https://hub.docker.com/r/conda/miniconda3/) . 

We want then to combine micromamba with miniconda, but how? We can exploit a feature in Docker builds, which is basically the same as "building on top of a building": we start with an image as base, we copy the most important things from there to our actual image and then we continue building on top of it. 

The syntax may be as follows: 

```Dockerfile
FROM author/image1:tag as base
FROM author/image2:tag

COPY --from=base /usr/local/bin/* /usr/local/bin/
```

Which means that, from `image1` as `base` we take only the files stored under `/usr/local/bin` and place them in `image2`. 

In our case, it would be:

```Dockerfile
ARG CONDA_VER=latest
ARG MAMBA_VER=latest

FROM mambaorg/micromamba:${MAMBA_VER} as mambabase

FROM conda/miniconda3:${CONDA_VER} 

COPY --from=mambabase /usr/bin/micromamba /usr/bin/
```

We copied `micromamba` from it's original location into our image.

### Install environments

We can now take the `conda_deps_1.sh`, copy and execute it into our build:

```Dockerfile
WORKDIR /data_science/

RUN mkdir -p /data_science/installations/

COPY ./conda_deps_1.sh /data_science/installations/

RUN bash /data_science/installations/conda_deps_1.sh
```

But let's say we also want to provide our image with an environment for AI development, that we only want to add to our build if the user specifies it at build time.

In this case, we can use `if...else` conditional statements in our Dockerfile!

We will create another file, `conda_deps_2.sh` with a python environment for AI development in which we will put some base packages such as:

- [transformers](https://huggingface.co/docs/transformers/en/index)
- [pytorch](https://pytorch.org/)
- [tensorflow](https://www.tensorflow.org/learn)
- [langchain](https://www.langchain.com/), langchain-community, langchain-core
- [gradio](https://gradio.app)

```bash
eval "$(conda shell.bash hook)"

micromamba create \
    python_ai \
    -y \
    -c conda-forge \
    -c bioconda \
    python=3.11

conda activate python_ai

micromamba install \
    -y \
    -c conda-forge \
    -c pytorch \
    transformers pytorch tensorflow langchain langchain-core langchain-community gradio

conda deactivate
```

Now we just add a condition to our Dockerfile:

```Dockerfile
ARG BUILD_AI="False"

RUN if [ "$BUILD_AI" = "True" ]; bash /data_science/installations/conda_deps_2.sh; \
    elif [ "$BUILD_AI" = "False" ]; then echo "No AI environment will be built"; \
    else echo "BUILD_AI should be either True or False: you passed an invalid value, thus no AI environment will be built"; fi
```

### Building and its options

Now let's take a look at the complete Dockerfile:

```Dockerfile
ARG CONDA_VER=latest
ARG MAMBA_VER=latest

FROM mambaorg/micromamba:${MAMBA_VER} as mambabase

FROM conda/miniconda3:${CONDA_VER} 

COPY --from=mambabase /usr/bin/micromamba /usr/bin/

WORKDIR /data_science/

RUN mkdir -p /data_science/installations/

COPY ./conda_deps_?.sh /data_science/installations/

RUN bash /data_science/installations/conda_deps_1.sh

ARG BUILD_AI="False"

RUN if [ "$BUILD_AI" = "True" ]; bash /data_science/installations/conda_deps_2.sh; \
    elif [ "$BUILD_AI" = "False" ]; then echo "No AI environment will be built"; \
    else echo "BUILD_AI should be either True or False: you passed an invalid value, thus no AI environment will be built"; fi

CMD ["/bin/bash"]
```

We can build our image tweaking and twisting the `build-args` as we please:

```bash
# BUILD THE IMAGE AS-IS
docker build . \
    -t YOUR-USERNAME/data-science:latest-noai

# BUILD THE IMAGE WITH AI ENV
docker build . \
    --build-args BUILD_AI="True" \
    -t YOUR-USERNAME/data-science:latest-ai

# BUILD THE IMAGE WITH A DIFFERENT VERSION OF MICROMAMBA

docker build . \
    --build-args MAMBA_VER="cuda12.1.1-ubuntu22.04" \
    -t YOUR-USERNAME/data-science:mamba-versioned
```

Then you can proceed and push the image to Docker Hub or to another registry as we saw in the last article.

You can now run your image interactively, loading also your pipelines as a volume, and activate all the environments as you please:

```bash
docker run \
    -i \
    -t \
    -v /home/user/datascience/pipelines/:/app/pipelines/ \
    YOUR-USERNAME/data-science:latest-noai \
    "/bin/bash"

# execute the following commands inside the container
source activate python_deps
conda deactivate 
source activate R
conda deactivate
```

We will stop here for this article, but in the next one we will dive into how to use the `buildx` plugin!🥰 
