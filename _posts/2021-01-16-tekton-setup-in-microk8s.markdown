---
layout: post
title: Tekton Setup using Microk8s in LXD container
date: 2021-01-16 17:51
category: kubernetes
author: 
tags: [microk8s, kubernetes, tekton]
summary: Create kubernetes single node setup using microk8s in LXD and setting up tekton in it
---

We are going to use [Microk8s](https://microk8s.io/) instead of [Kubernetes](https://kubernetes.io/) directly, since it is easy to setup in [LXD](https://linuxcontainers.org/lxd/introduction/) and this is light weight. microk8s is fully compatible with the kubernetes implementation so, any configuration written for kubernetes can be directly used on microk8s. [Containerd](https://containerd.io/) is the container runtime used by MicroK8s to manage images and execute containers.

[Tekton](https://tekton.dev/) is a CI/CD framework which we will use. Note that we are going to setup single node Kubernetes setup but the nodes can be added as on when multiple nodes are created.

This section contains following tasks to be performed in order to install the tekton properly in LXD.

* [LXD VM installation](#lxd-vm-installation)
* [Microk8s Installation](#microk8s-installation)
* [Tekton Installation](#tekton-installation)
* [Port configuration for external access](#port-configuration-for-external-access)
* [Ingress based service expose for external access](#ingress-based-service-expose-for-external-access)

### LXD VM Installation

From release 3.10 (native support from 4.0), LXD supports running VM. We will be using this feature for installing and setting up CI/CD.

1. Install ubuntu VM and do basic configuration.

    ``` bash
    lxc init ubuntu:20.04 cicd-vm --vm
    (
    cat << EOF
    #cloud-config
    apt_mirror: http://us.archive.ubuntu.com/ubuntu/
    ssh_pwauth: yes
    users:
    - name: ubuntu
        passwd: "\$6\$s.wXDkoGmU5md\$d.vxMQSvtcs1I7wUG4SLgUhmarY7BR.5lusJq1D9U9EnHK2LJx18x90ipsg0g3Jcomfp0EoGAZYfgvT22qGFl/"
        lock_passwd: false
        groups: lxd
        shell: /bin/bash
        sudo: ALL=(ALL) NOPASSWD:ALL
    EOF
    ) | lxc config set cicd-vm user.user-data -
    lxc config device add cicd-vm config disk source=cloud-init:config
    ```

2. Set the memory and cpu limits as per requirement.

    ``` bash
    lxc config set cicd-vm limits.cpu 24
    lxc config set cicd-vm limits.memory 64GB
    ```

3. Start the VM

    ``` bash
    lxc start cicd-vm
    ```

### Microk8s Installation

> All the commands are run in the LXD VM shell. Open the VM shell by `lxc console cicd-vm` or `ssh <user>@<vm-ip>`

1. Install and configure latest microk8s from snap store.

    ``` bash
    sudo snap install microk8s --classic
    sudo usermod -a -G microk8s $USER
    sudo chown -f -R $USER ~/.kube
    ```

2. Relogin.
3. Microk8s will take sometime to setup. Wait for it to complete its setup.

    ``` bash
    microk8s status --wait-ready
    ```

4. Now kubectl can be accessed via `microk8s kubectl <sub-commands>`
5. Create a alias for the kubectl command.

    ``` bash
    echo "alias kubectl='microk8s kubectl'" >> ~/.bash_aliases
    ```

6. Now Microk8s is installed and ready to run the required pods. By default it will create a single node with name as hostname.

    ``` bash
    kubectl get nodes
    ```

### Tekton Installation

1. Install Tekton core component and check pods are in running state.

    ``` bash
    kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
    kubectl get pods --namespace tekton-pipelines
    ```

2. Setup the Tekton CLI

    ``` bash
    sudo apt update;sudo apt install -y gnupg
    sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 3EFE0E0A2F2F60AA
    echo "deb http://ppa.launchpad.net/tektoncd/cli/ubuntu eoan main"|sudo tee /etc/apt/sources.list.d/tektoncd-ubuntu-cli.list
    sudo apt update && sudo apt install -y tektoncd-cli
    tkn version
    ```

3. Install Tekton Dashboard

    ``` bash
    kubectl apply --filename https://storage.googleapis.com/tekton-releases/dashboard/latest/tekton-dashboard-release.yaml
    ```

4. Wait for the dashboard to be up in Running state. `kubectl get pods --namespace tekton-pipelines`

### Port configuration for external access

1. Now we have to make port forwarding from outside to LXD. LXD VM supports this but only nat based forwarding is supported. To make it work, get the interface IP which is connected to outside network and note it down. Then execute following command.

    > This has to be executed in host terminal, not in LXD VM terminal.

    > Replace `<interface-ipv4-ip>` and `<interface-ipv6-ip>` with the IP that is noted down.

    ``` bash
    IPV4=`lxc ls | grep cicd-vm | awk -F\| '{print $4}' | xargs | awk '{print $1}' | xargs`
    IPV6=`lxc ls | grep cicd-vm | awk -F\| '{print $5}' | xargs | awk '{print $1}' | xargs`
    IFACE=`lxc ls | grep cicd-vm | awk -F\| '{print $4}' | xargs | awk '{print $2}' | xargs | sed 's/[()]//g'`
    lxc stop cicd-vm
    lxc network set lxdbr0 ipv6.dhcp.stateful true
    lxc config device override cicd-vm eth0 ipv4.address=${IPV4} ipv6.address=${IPV6}
    lxc config device add cicd-vm proxyv4 proxy nat=true listen=tcp:<interface-ipv4-ip>:9097 connect=tcp:0.0.0.0:9097
    lxc config device add cicd-vm proxyv4-alternate proxy nat=true listen=tcp:<interface-ipv4-ip>:31475 connect=tcp:0.0.0.0:31475
    lxc config device add cicd-vm proxyv6 proxy nat=true listen=tcp:[<interface-ipv6-ip>]:9097 connect=tcp:[::]:9097
    lxc config device add cicd-vm proxyv6-alternate proxy nat=true listen=tcp:[<interface-ipv6-ip>]:31475 connect=tcp:[::]:31475
    lxc start cicd-vm
    ```

2. By default Kubernetes doesn't expose the pods. Inorder to make Tekton Dashboard accessible from LXD, we have to enable port forwarding.

    ``` bash
    kubectl --namespace tekton-pipelines port-forward --address 0.0.0.0 svc/tekton-dashboard 9097:9097
    ```

3. From any other machine (in the same network), Access the Tekton Dashboard using IP: `http://<host-server-ip>:9097/`. If below image page is seen then Tekton is configured properly.

### Ingress based service expose for external access

[Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) is a API object which will act as a load-balancer or a proxy server that connects a service in a node to external. In order to use ingress object in kubernetes we need to install [Ingress Controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/). Lets set this up in microk8s.

Microk8s provides an [Ingress Add-on](https://microk8s.io/docs/addon-ingress) which installs [Nginx based Ingress Controller](https://github.com/kubernetes/ingress-nginx) for microk8s nodes. This can be installed by executing following command.

    microk8s enable ingress

But Ingress does not expose arbitrary ports or protocols. But tekton runs on 9097 or 31475 port. This can be done through [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport).

1. Edit the service file of the tekton-dashboard

    ``` bash
    kubectl edit svc tekton-dashboard -n tekton-pipelines
    ```

2. Find the `spec.type` field and change it from `clusterIP` to `NodePort`. Save and exit.
3. Check the available port in the node. `kubectl get svc -A`
4. Tekton can be accessed from `http://<host-server-ip>:31475/`


