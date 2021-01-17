---
layout: post
title: Running Container directly on Crun
date: 2021-01-17 23:23
category: container
author: 
tags: [container, container-runtime, crun]
summary: Running container directly on a container-runtime without container engine.We will be using crun
---

We are goint to try running container directly on a container-runtime without container engine. Will be using crun for this experiment. We will be using ubuntu 18.04 for setting up the crun. Since we are setting up the crun from source, installation of crun should be independent of distro provided that the required build packages are available. I have given package details for ubuntu / debian. Similar package has to be installed if you are using other distribution. You can try static build, then no build dependency needs to be met.

## Setting up crun in linux machine

1. Install the required build requirement

    ``` bash
    sudo apt install -y autoconf automake autotools-dev libtool pkg-config \
    libcap-dev libseccomp-dev libyajl-dev
    ./configure
    make
    ```

2. You can generate static build as-well

    ``` bash
    nix build -f nix/
    ```

## Creating container using crun

1. Download alpine rootfs and extract it (can be any rootfs)

    ``` bash
    wget https://dl-cdn.alpinelinux.org/alpine/v3.13/releases/x86/alpine-minirootfs-3.13.0-x86.tar.gz
    mkdir rootfs && tar -xvf alpine-minirootfs-3.13.0-x86.tar.gz -C rootfs
    ```

2. Generate the config.json file

    ``` bash
    ./result/bin/crun spec
    ```

3. Run the container

    ``` bash
    ./result/bin/crun run alpine
    ```