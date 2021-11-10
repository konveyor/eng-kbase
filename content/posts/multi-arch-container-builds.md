---
title: "Multi Arch Container Builds"
date: 2021-10-30T13:20:40-04:00
draft: false
authors:
- Jason Montleon
tags:
- arch
- Multi-Arch
- multi-arch
- MultiArch
- multiarch
- docker
- podman
- buildx
- ppc64le
- s390x
- aarch64
- arm64
- amd64
- x86_64
---

## Introduction

OADP is a backup solution for OpenShift and Kubernetes. As its popularity grows the desire to use it for backups on platforms other than x86_64 has grown. Hardware for s390x, aarch64/arm64, and ppc64le is not readily available so we had to find an alternative solution for performing builds. We used podman locally to test builds, and then implemented automated builds for a golang 1.16 builder in one org where GitHub Actions are preferred, and then built the application images in another org where Travis is required.

## Local Builds
podman can use qemu-user-static for performing builds locally. While the process is very similar to building for your native architecture there are minor differences.  
  
- Install the required software.
```
sudo dnf -y install podman qemu-user-static
```
- Run a build.
```
DOCKER_BUILDKIT=1 podman build -f Dockerfile \
  --manifest quay.io/jmontleon/travis-multiarch-test:latest \
  --platform linux/s390x,linux/amd64,linux/arm64,linux/ppc64le .
```
- Push the manifest.
```
podman manifest push \
  quay.io/jmontleon/oadp-operator:latest \
  docker://quay.io/jmontleon/oadp-operator:latest
```
  
Although this is very similar to building and pushing an image for your native architecture, note that there are some differences.
- The build uses `--manifest` instead of `--tag`.
- We use `manifest push` instead of `push`.
- It is also necessary to provide the destination repository for a manifest push.

## GitHub Actions
To automate builds with a GitHub action we used the following workflow that makes use of docker buildx.
- Create the action in the `.github/workflows/` directory in your GitHub repo , with a `.yml` extension.
- Change the `tags` parameter to the project and tag you want to push to.
- Adjust the `platforms` and `file` parameters if desired.
- Add `QUAY_PUBLISH_ROBOT` and `QUAY_PUBLISH_TOKEN` secrets to your organization or repository in order to push to Quay.

```
name: 'multi-arch images build'

on:
  workflow_dispatch:
  push:
jobs:
  multiarch-build:
    runs-on: ubuntu-latest
    steps:
      - name: add checkout action...
        uses: actions/checkout@v2

      - name: configure QEMU action...
        uses: docker/setup-qemu-action@master
        with:
          platforms: all

      - name: configure Docker Buildx...
        id: docker_buildx
        uses: docker/setup-buildx-action@master

      - name: login to Quay.io...
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v1
        with:
          registry: quay.io
          username: ${{ secrets.QUAY_PUBLISH_ROBOT }}
          password: ${{ secrets.QUAY_PUBLISH_TOKEN }}

      - name: build Multi-arch images...
        uses: docker/build-push-action@v2
        with:
          builder: ${{ steps.docker_buildx.outputs.name }}
          context: .
          file: ./Dockerfile
          platforms: linux/amd64,linux/ppc64le,linux/arm64,linux/s390x
          push: true
          tags: quay.io/konveyor/builder:latest
```

## Travis
Travis is also capable of performing builds for multiple architectures.  
  
Create a `.travis.yml` and update the `IMAGE`, `DEFAULT_BRANCH`, and `DOCKERFILE` env vars as required in `.travis.yml`. You will also want to add `QUAY_ROBOT` and `QUAY_TOKEN` secrets in your Travis settings for the repo and set appropriate credential values for pushing the images.   
  
One thing to be aware of is that Travis times out a job if it does not detect output for 10 minutes. At the same time alternative architectures are mostly limited to using LXD, which buffers output. If enough output is not generated to cause the buffer to be written to the console and the build takes more than 10 minutes it will result in the job being canceled. For this reason we are adding `-v` to go commands which adds a lot of verbosity and keeps the console receiving data in short intervals to keep the job running.  
  
In the example below we are using the `before_install` section to make modifications to the `Dockerfile` to do dep downloads from outside the container and copy them in since we have also seen intermittent problems within docker containers in Travis on alternate architectures .  
  
.travis.yml:
```
os: linux
services: docker
dist: focal
language: go
go: stable

env:
  global:
  - IMAGE: quay.io/jmontleon/travis-multiarch-test
  - DEFAULT_BRANCH: main
  - DOCKERFILE: Dockerfile
  - DOCKER_CLI_EXPERIMENTAL: enabled
  - GOPROXY: https://goproxy.io,direct

before_install:
- |
  if [ "${TRAVIS_BRANCH}" == "${DEFAULT_BRANCH}" ]; then
    export TAG=latest
  else
    export TAG="${TRAVIS_BRANCH}"
  fi

# Builds routinely fail due to download failures inside alternate arch docker containers
# Here we are downloading outside the docker container and copying the deps in
# Also use -v for downloads/builds to stop no output failures from lxd env buffering.
before_script:
- go mod vendor -v
- sed -i 's|^RUN go mod download$|COPY vendor/ vendor/|g' ${DOCKERFILE}
- sed -i 's|-mod=mod|-mod=vendor|g' ${DOCKERFILE}
- sed -i 's|go build|go build -v|g' ${DOCKERFILE}

script:
- docker build -t ${IMAGE}:${TAG}-${TRAVIS_ARCH} -f ${DOCKERFILE} .
- if [ -n "${QUAY_ROBOT}" ]; then docker login quay.io -u "${QUAY_ROBOT}" -p ${QUAY_TOKEN}; fi
- if [ -n "${QUAY_ROBOT}" ]; then docker push ${IMAGE}:${TAG}-${TRAVIS_ARCH}; fi

jobs:
  include:
  - stage: build images
    arch: ppc64le
  - arch: s390x
  - arch: arm64-graviton2
    virt: lxd
    group: edge
  - arch: amd64
  - stage: push manifest
    language: shell
    arch: amd64
    before_script: []
    script:
    - |
      if [ -n "${QUAY_ROBOT}" ]; then
        docker login quay.io -u "${QUAY_ROBOT}" -p ${QUAY_TOKEN}
        docker manifest create \
          ${IMAGE}:${TAG} \
          ${IMAGE}:${TAG}-amd64 \
          ${IMAGE}:${TAG}-ppc64le \
          ${IMAGE}:${TAG}-s390x \
          ${IMAGE}:${TAG}-aarch64
        docker manifest push ${IMAGE}:${TAG}
      fi
```

## References
[Konveyor GitHub Action Example](https://github.com/konveyor/builder/blob/main/.github/workflows/multi_arch_image_build.yml)  
[Konveyor Travis Example](https://github.com/konveyor/travis-multiarch-test)  
[GitHub Action Blog Post by Rafael Sene](https://rpsene.wordpress.com/2021/09/09/cicd-building-multi-arch-container-images-with-github-actions/)  
[Travis Example by Rafael Sene](https://github.com/rpsene/multi-arch-travis)
