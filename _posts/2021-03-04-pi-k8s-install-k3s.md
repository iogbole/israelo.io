---
layout: post
title:  "Raspberry Pi Cluster: Setting up K3s Kubernetes using Ansible - Part#3"
author: israel
categories: [ 'Cloud Native', 'Pi' ]
tags: [containers, raspberry,  pi, cloud-native, kubernetes, IoT, edge ]
image: https://user-images.githubusercontent.com/2548160/110161490-54e1d080-7de5-11eb-8b73-676ce5b0a9c7.jpg
date:   2021-03-06 15:01:35 +0300
description: "This tutorial will be a brief walk through the process of getting K3s Kubernetes up and running on Raspberry Pi - Using Ansible..."
#permalink: /draft/pi3
---

This tutorial will be a brief walk through the process of getting K3s up and running on Raspberry Pi.

 <a href="https://k3s.io/"  target="_blank"> K3s (by RancherLab) </a> and <a href="https://microk8s.io/" target="_blank"> MicroK8s (by Canonical) </a> are the two most popular lightweight Kubernetes for IoT an Edge computing in the industry today. I have used both and I found K3s easier to setup with more advanced configurations for High Availability via an Infrastructure-as-Code automation.

### What you'll learn

- Deploying K3s Kubernetes on Raspberry Pi using Ansible
- Expose the Kubernetes Service to the public internet

### What you'll need

- Components: Please refer to the <a href="https://www.israelo.io/blog/pi-k8s-overview/" target="_blank"> first blog post </a> in this series
- Prepared RPis : Please refer to the <a href="https://www.israelo.io/blog/pi-k8s-prepare/" target="_blank"> second blog post </a> in this series

## Why Ansible?

Ansible is an open-source software provisioning, configuration management, and application-deployment tool enabling infrastructure as code. We will use Ansible to install K3s on the master and worker nodes. Ansible will enable you to reset the entire configuration should you need to, or add more nodes to the cluster.  

Ansible installation is simple. You can either install it via `pip`, like this:

`python3 -m pip install`<br>
 or <br>
`brew install ansible.` 

If you're on the Mac. Although the pip option is the recommended method.

## Installing K3s on Raspberry Pi

Rancher has an Ansible playbook that builds a Kubernetes cluster with K3s. Let's set it up! 

1. Download the Ansible playbook to your Ansible control node - <a href="https://github.com/k3s-io/k3s-ansible/archive/master.zip" target="_blank">https://github.com/k3s-io/k3s-ansible/archive/master.zip</a>
2. Duplicate the `inventory/sample` directory to create an inventory for your RPis, that is : 

   `cp -r  inventory/sample inventory/pi`

3. Change the working directory to `inventory/pi` and edit the `hosts.ini` file, as shown below:  
    
   ```cpp

      [master]
      192.168.master.ip.address

      [node]
      192.168.worker.node.1.ip.address
      192.168.worker.node.2.ip.address
      192.168.worker.node.n.ip.address

      [master:vars]
      ansible_ssh_private_key_file=~/pi
      [node:vars]
      ansible_ssh_private_key_file=~/pi

      [k3s_cluster:children]
      master
      node
   ```

4. Edit the `inventory/group_vars/all.yml` file and change the `ansible_user` to `pi`. You may move the `ansible_ssh_private_key_file` from hosts.ini into the group vars too if you prefer - assuming you're using the same key for all.

5. Still in `inventory/group_vars/all.yml`, edit  `k3s_version` with the latest version:   To get the latest version:
    - Navigate to:  <a href="https://github.com/k3s-io/k3s/tags">https://github.com/k3s-io/k3s/tags </a>
    - Click the latest release, then copy the last URL segment, for example: `v1.20.4%2Bk3s1` is the last URL segment of
        `https://github.com/k3s-io/k3s/releases/tag/v1.20.4%2Bk3s1`

    In summary, your `inventory/group_vars/all.yml` should look like this: 

    ```yaml
      ---
      k3s_version: v1.20.4%2Bk3s1 # update this 
      ansible_user: pi # update this too 
      systemd_dir: /etc/systemd/system
      master_ip: "{{ hostvars[groups['master'][0]]['ansible_host'] | default(groups['master'][0]) }}"
      extra_server_args: ""
      extra_agent_args: ""

    ```

6. Run it! 

   `ansible-playbook site.yml -i inventory/pi/hosts.ini`

## Connect to the Cluster

Once it's built, you need to get the kubectl configuration from the master to connect to the cluster: 

`scp -i ~/pi pi@pimaster:~/.kube/config ~/.kube/pi-config`

<i> Note: The purpose of using `~/.kube/pi-config` is to preserve your previous configurations, if any </i>


Next, edit `bash_rc` or `bash_profile`  by adding this line:

```cpp
#K8S
export KUBECONFIG=$KUBECONFIG:~/.kube/config:~/.kube/pi-config
```

Then reload  `source ~/.bash_profile`

## Change context

Next, rename the context from `default`, since you may have a default context already.

`kubectl --kubeconfig=~/.kube/pi-config config rename-context default pi`

Alternatively, edit `~/.kube/pi-config` and change the cluster and context names from `default` to `pi` respectively.

Then switch context:

```cpp
kubectl config get-contexts #list contexts
kubectl config use-context pi  #switch contexts

```
## Well done! 

```cpp 

~ $ k get nodes
NAME          STATUS   ROLES                  AGE   VERSION
pinasworker   Ready    <none>                 24d   v1.20.2-rc1+k3s1
pimaster      Ready    control-plane,master   24d   v1.20.2-rc1+k3s1
piworker1     Ready    <none>                 24d   v1.20.2-rc1+k3s1

```
## Exposing your service

To expose your Service to the world, you may consider using a combination of <a href="https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types" target="_blank"> NodePort </a> or <a href="https://opensource.com/article/20/7/homelab-metallb" target="_blank"> MetalB </a> and Port forwarding configuration in your router.<br>
Or <br>
<a href="https://github.com/inlets/inlets-operator" target="_blank"> The Inlets operator </a>

### SSL

You will need an SSL certificate if you intend to expose your Kubernetes service to the public. You may generate free SSL certificates using <a href ="https://letsencrypt.org" target="_blank">https://letsencrypt.org </a>

# Troubleshooting

Running `kubectl` commands from pi master gave me this error.

```cpp 
pi@pimaster:~ $ kubectl get po
WARN[2021-01-14T22:33:39.735790976Z] Unable to read /etc/rancher/k3s/k3s.yaml, please start server with --write-kubeconfig-mode to modify kube config permissions
error: error loading config file "/etc/rancher/k3s/k3s.yaml": open /etc/rancher/k3s/k3s.yaml: permission denied
```

The solution is to change `k3s.yaml`'s permission.

`$ sudo chmod 644 /etc/rancher/k3s/k3s.yaml`

## Refs

 - https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#from-pip
 

<a href="https://www.israelo.io/blog/pi-k8s-prepare/" target="_blank"> << Prepare Raspberry Pi to run K3s Kubernetes Cluster </a> 