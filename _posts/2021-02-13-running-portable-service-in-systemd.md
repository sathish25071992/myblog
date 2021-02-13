---
layout: post
title: Running portable service in systemd
date: 2021-02-13 13:38
category: systemd
author: 
tags: [systemd, portable-service, portable, service]
summary: Running portable service in systemd which will run service in isolated environment
---

Portable service is New and Preview feature added in systemd which isolates a group of services. This will just attach a subfodler or raw image that contains the `OS-Tree` to the host systemd. This will not provide all kind of isolation as containers does but will provide strict access control for the running service. For detailed explanation on the internals please check [systemd portable service](https://systemd.io/PORTABLE_SERVICES/).

In this Blog will see how to create a image subfolder, populating basic hello-world application service then running that service as portable service.

## Pre-requisite

1. Ubuntu 20.04 or higher
2. Portable systemd enabled. In ubuntu this can be done by installing `systemd-container` package.

    ```bash
    sudo apt install systemd-container
    ```

3. gcc installed. In ubuntu it can be done by following command,

    ```bash
    sudo apt install gcc
    ```

## Creating the image sub-folder and hello world service

Portable service attach a subfolder or the raw os image to the running systemd. But the requirement is this subfolder or raw image should contain some basic folder structure (os-tree) in place. In this section we will create this structure and create a hello world service

1. Create the basic folder structure.

    ```bash
    mkdir ./portabletest
    cd ./portabletest
    mkdir -p ./usr/bin/ ./usr/lib/systemd/system/ /usr/lib/ ./etc/ ./proc/ ./sys/ ./dev/ ./run/ ./tmp/ ./var/tmp/
    cd -
    ```

2. Create required os files.

    ```bash
    touch portabletest/etc/resolv.conf
    touch portabletest/etc/machine-id
    cp /usr/lib/os-release portabletest/usr/lib/
    ```

3. Create a hello world application service.

    ```bash
    echo -e "#include <stdio.h>
    int main(void)
    {
        printf(\"hello-world\n\");
        return 0;
    }
    " > main.c && gcc -static main.c -o ./portabletest/usr/bin/helloworld

    echo -e "[Unit]
    Description=hello world

    [Service]
    Type=oneshot
    ExecStart=/usr/bin/helloworld
    RemainAfterExit=true
    StandardOutput=journal

    [Install]
    WantedBy=multi-user.target" > ./portabletest/usr/lib/systemd/system/portabletest-helloworld.service
    ```

## Attaching the portable service and start

1. Attach the portable service.

    ```bash
    portablectl attach ./portabletest
    ```

2. Start the service and verify it is running.

    ```bash
    systemctl start portabletest-helloworld.service
    systemctl status portabletest-helloworld.service
    ```

Now the Portable service can be treated as any other systemd service.

## Detach or remove the portable service

To detach the portable service, execute the following command,

```bash
portablectl --now detach ./portabletest
```

> `--now` flag is required if this portable service contains services which is already running and you want to stop that and detach.

## Reference

1. [https://systemd.io/PORTABLE_SERVICES/](https://systemd.io/PORTABLE_SERVICES/)
2. [https://gist.github.com/drmalex07/d006f12914b21198ee43](https://gist.github.com/drmalex07/d006f12914b21198ee43)
