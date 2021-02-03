---
layout: post
title: Running Podman in Embedded devices
date: 2021-02-03 14:33
category: container
author: 
tags: [container, container-runtime, crun, podman]
summary: Running podman with crun container run-time in an embedded devices
---

## Installing podman with crun in Yocto qemuarm machine

[Podman](https://podman.io/) is a daemonless container engine for developing, managing, and running OCI Containers on your Linux System. This is light weight compared to other docker engines (docker, rkt, etc). This is a drop in replacement for the container engine [Docker](https://www.docker.com/).

Based on the [OCI](https://opencontainers.org/) standard any OCI complaint container engine will only provide the top level user management (image, network, mount points, etc). The actual container management will be handled by the container runtime. There are several container runtimes (CRT) out of which most widely used one is [runc](https://github.com/opencontainers/runc). Podman by default used runc as its CRT. But runc is little bulky and since we are targetting for embedded devices we want it to be lean. So there is a light weight version of runc available called [crun](https://github.com/containers/crun). This consumes 10 to 50 times less footprint compared to runc and basically provides almost similar funtionality compared to runc. The difference between these two comes from the fact that runc is written in Go and crun is written in C and was written from scratch with light weight in mind. For better understand the difference, have a look at this - [An intrduction to crun](https://www.redhat.com/sysadmin/introduction-crun)

In this topic we will integrate podman with crun using [Yocto](https://www.yoctoproject.org/) and try to use it in default Yocto machine (qemuarm). Lets go through the steps.

## Steps to install podman in yocto based system

1. Clone the required repositories

```bash
git clone -b dunfell git://git.openembedded.org/meta-openembedded && \
git clone -b dunfell git://git.yoctoproject.org/meta-virtualization && \
git clone -b deunfell git://git.yoctoproject.org/meta-security
```

2. Add the cloned repository to the bblayer.conf

```bash
bitbake-layers add-layer ../meta-openembedded/meta-oe/ \
	../meta-openembedded/meta-filesystems/ \
  	../meta-openembedded/meta-python/ \
	../meta-openembedded/meta-networking/ \
	../meta-openembedded/meta-perl/ \
	../meta-virtualization/ \
	../meta-security/
```

3. Patch the ../meta-virtualization/recipes-containers/podman/podman_git.bb so that,

```bash
SRCREV = "eec482cae3289ecaad45c602629657da7062ce9c"
SRC_URI = "git://github.com/containers/libpod.git;branch=v2.0"
PV = "2.0.0+git${SRCREV}"
do_compile() {
    .
    oe_runmake pkg/varlink/iopodman.go GO=go
    .
}
```

4. Enable `CONFIG_NSENTER` in busybox
5. Enable `CONFIG_SECCOMP` in linux-yocto
6. Add required packages to rootfs

```bash
echo -e "
# podman container system related packages
RPROVIDES_crun += \" virtual/runc\"
PREFERRED_PROVIDER_virtual/runc = \"crun\"
DISTRO_FEATURES_append = \" virtualization\"
IMAGE_INSTALL_append = \" podman cgroup-lite procps ca-certificates kernel-modules\"
IMAGE_ROOTFS_EXTRA_SPACE = \"2097152\"" >> conf/local.conf
```
7. Build the final image

```bash
bitbake core-image-minimal
```

8. Run qemu with sudo permisson

```bash
sudo runqemu nographic
```

## Reference
1. https://davidjenei.com/post/yocto-podman/
2. https://www.techrepublic.com/article/how-to-deploy-a-pod-with-podman/
