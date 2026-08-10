# Bonus Episode: Building and deploying a Docker container to GitHub Packages

:::{admonition} Overview
:class: note
**Teaching:** 40 min

**Questions**
- How to build a Docker container for python packages?
- How to share Docker images?

**Objectives**
- To be able to build a Docker container and share it via GitHub Packages
:::

:::{admonition} Prerequisites
:class: caution
For this lesson you should already be familiar with Docker images.
Head over to [our training on Docker](https://hsf-training.github.io/hsf-training-docker/) if you aren't already!
:::

## Docker Container for python packages

Python packages can be installed using a Docker image. The following example illustrates how to write a Dockerfile for building an image containing python packages.

```dockerfile
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
  && apt-get install -y wget dpkg-dev cmake g++ gcc binutils libx11-dev \
    libxpm-dev libxft-dev libxext-dev libssl-dev libgsl-dev libtiff-dev \
    python3 python3-pip python3-venv \
  && rm -rf /var/lib/apt/lists/*

RUN python3 -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

RUN pip install numpy awkward uproot particle hepunits matplotlib \
  mplhep vector fastjet iminuit
```

As we see, several packages are installed.

:::{admonition} Why the virtual environment?
:class: tip
On Ubuntu 24.04 the system Python is *externally managed* ([PEP 668](https://peps.python.org/pep-0668/)), so `pip` refuses to install packages into it. The idiomatic fix is to create a virtual environment inside the image and put its `bin/` first on `PATH`, as done above; a quick-and-dirty alternative is `pip install --break-system-packages`.
:::

## Publish Docker images with GitHub Packages and share them!

It is possible to publish Docker images with [GitHub Packages](https://github.com/features/packages), via its container registry `ghcr.io`.
To do so, one needs to use GitHub CI/CD. A step-by-step guide is presented here.

* **Step 1**: Create a GitHub repository and clone it locally.
* **Step 2**: In the empty repository, make a folder called `.github/workflows`. In this folder we will store the file containing the YAML script for a GitHub workflow, named `Docker-build-deploy.yml` (the name doesn't really matter).
* **Step 3**: In the top directory of your GitHub repository, create a file named `Dockerfile`.
* **Step 4**: Copy-paste the content above and add it to the Dockerfile. (In principle it is possible to build this image locally, but we will not do that here, as we wish to build it with GitHub CI/CD).
* **Step 5**: In the `Docker-build-deploy.yml` file, add the content below.
* **Step 6**: Add LICENSE and README as recommended in the [SW Carpentry Git-Novice Lesson](https://swcarpentry.github.io/git-novice/), and then the repository is good to go.

```yaml
name: Create and publish a Docker image

on:
  pull_request:
  push:
    branches: main

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push-image:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v6

      - name: Log in to the Container registry
        uses: docker/login-action@v4
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Docker Metadata
        id: meta
        uses: docker/metadata-action@v6
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v7
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

Note the line `push: ${{ github.event_name != 'pull_request' }}`: on pull requests we only *build* the image as a test; only pushes to `main` actually publish it to the registry.

:::{admonition} Key Points
:class: note
- Python packages can be installed in Docker images along with ubuntu packages.
- It is possible to publish and share Docker images via GitHub Packages.
:::
