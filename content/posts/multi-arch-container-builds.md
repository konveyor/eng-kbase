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
  
Create a `.travis.yml` as well as image build and manifest push scripts in place of the GitHub Action.  
  
Update the `IMAGE`, `DEFAULT_BRANCH`, and `DOCKERFILE` env vars as required in `.travis.yml`.  
  
.travis.yml:
```
language: bash
os: linux
services: docker
sudo: required
dist: bionic

env:
  global:
    IMAGE: quay.io/jmontleon/travis-multiarch-test
    DEFAULT_BRANCH: main
    DOCKERFILE: Dockerfile
jobs:
  include:
   - stage: build image
     arch: ppc64le
     script: ./.travis-build-image.sh
   - arch: amd64
     script: ./.travis-build-image.sh
   - arch: s390x
     script: ./.travis-build-image.sh
   - arch: arm64
     script: ./.travis-build-image.sh
   - stage: push manifest
     arch: x86_64
     script: ./.travis-push-manifest.sh
```

.travis-build-image.sh
```
#!/bin/bash

if [ "${TRAVIS_BRANCH}" == "${DEFAULT_BRANCH}" ]; then
  export TAG=latest
else
  export TAG=${TRAVIS_BRANCH}
fi

export ARCH=$(uname -m)

docker build -t ${IMAGE}:${TAG}-${ARCH} -f ${DOCKERFILE} .
docker login quay.io -u "${QUAY_ROBOT}" -p ${QUAY_TOKEN}
docker push ${IMAGE}:${TAG}-${ARCH}
```

.travis-push-manifest.sh
```
#!/bin/bash

if [ "${TRAVIS_BRANCH}" == "${DEFAULT_BRANCH}" ]; then
  export TAG=latest
else
  export TAG=${TRAVIS_BRANCH}
fi

export DOCKER_CLI_EXPERIMENTAL=enabled

#Without this docker manifest create fails
#https://github.com/docker/for-linux/issues/396
sudo chmod o+x /etc/docker

docker manifest create \
  ${IMAGE}:${TAG} \
  ${IMAGE}:${TAG}-x86_64 \
  ${IMAGE}:${TAG}-ppc64le \
  ${IMAGE}:${TAG}-s390x \
  ${IMAGE}:${TAG}-aarch64
echo $?

docker manifest inspect ${IMAGE}:${TAG}
echo $?

docker login quay.io -u "${QUAY_ROBOT}" -p ${QUAY_TOKEN}

docker manifest push ${IMAGE}:${TAG}
```

## References
[Konveyor GitHub Action Example](https://github.com/konveyor/builder/blob/main/.github/workflows/multi_arch_image_build.yml)  
[Konveyor Travis Example](https://github.com/konveyor/travis-multiarch-test)  
[GitHub Action Blog Post by Rafael Sene](https://rpsene.wordpress.com/2021/09/09/cicd-building-multi-arch-container-images-with-github-actions/)  
[Travis Example by Rafael Sene](https://github.com/rpsene/multi-arch-travis)
