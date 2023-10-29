---
layout: post
title:  "Prepare Raspberry Pi to run K3s Kubernetes Cluster - Part#2"
author: israel
categories: [ 'Cloud Native', 'Pi' ]
tags: [ containers, raspberry,  pi, cloud-native, kubernetes, IoT, edge, k3s ]
image: https://user-images.githubusercontent.com/2548160/110127567-b130fa00-7dbd-11eb-9caa-724d99640c9b.jpg
date:   2021-03-05 15:01:35 +0300
description: "This tutorial will be a brief walk through the process of preparing your Raspberry Pi to run Kubernetes Cluster. This setup can be entirely headless..."

---

This tutorial will be a brief walk through the process of preparing your Raspberry Pi to run Kubernetes Cluster.

This setup can be entirely headless or using an HDMI screen and USB keyboard to control your cluster's nodes.

## What you'll need

Please refer to the <a href="https://www.israelo.io/blog/pi-k8s-overview/" target="_blank"> first blog post </a> in this series for detailed information on all the required components.

<p class="aligncenter">
<img alt ="stack" class="lazyimg" src="https://user-images.githubusercontent.com/2548160/107226410-00c81400-6a12-11eb-9dbc-d35b0d69dd17.jpg"/> 
<br>
</p>

## Prepare the RPis

We will need to install the OS, enable (password-less) SSH, change the default password, enable Wifi, configure DCHP etc., before we can install Kubernetes.

Let's begin!

### Download Raspberry Pi Imager

Raspberry Pi Imager is the quick and easy way to install Raspberry Pi OS and other operating systems to a Hard drive or microSD card.
Download it from <a href="https://www.raspberrypi.org/software/" target="_blank"> here </a>, insert your SD card into your computer and follow the instructions to create the OS image.

<p class="aligncenter">
<img alt ="imager" class="lazyimg" src="https://user-images.githubusercontent.com/2548160/110124373-e89da780-7db9-11eb-874f-febabe87b3af.jpg"/> 
<br>
</p>

#### Which OS should I choose?

The Raspberry Pi board uses an ARM processor. Also, the Raspberry Pi OS is a 32-bit operating system; although there's a 64-bit OS in beta (as at the time of writing this blog post). So, suppose you intend to use existing container images in your Kubernetes cluster, in that case, you should select the headless Ubuntu 64-bit OS because AArch64/ARM64 architecture systems can run 32-bit ARM images. In contrast, 32-bit ARM OS such as the currently available Raspberry Pi OS cannot run 64-bit container images.

In summary, first, check if Raspberry Pi 64-bit OS is GA. If not, go with Ubuntu OS if you intend to run 64-bit container images.

In my case, I  have full control of the container images so I used docker's <a href="https://www.docker.com/blog/multi-arch-images/" target="_blank"> buildx </a> to build multi-architecture images. I am using the Raspberry Pi OS and will upgrade to the 64-bit version as soon as it in GA.

```cpp
docker buildx build --platform linux/amd64, \
                      linux/arm64, \
                      linux/arm/v7 \
                      -t io/blogimage:latest
```

Next, insert the SD card (with the OS) into your RPi and power it up!
### Change Boot Order Configuration

Skip this step if you do not intend boot from USB/Hard drive.

If you intend to boot from USB, flash your USB/hard drive using the Imager. Then, launch the terminal and change the boot order using the following steps :

1. ` sudo raspi-config `
2. Select the Advanced option
3. Select Boot Order
4. Select USB Boot
5. Shutdown the RPi  `sudo shutdown -h now`

Next, remove the SD card from the RPi, insert the USB drive into a USB 3.0 port (the blue port). Then power it on again.

### Enable SSH

Open a terminal on the RPi and enter the following commands to enable ssh: 

```cpp
sudo systemctl enable ssh
sudo systemctl start ssh
```

If you're using RPi over headless, insert the SD card or USB/hard drive into your computer, then create a new empty file named `ssh`, without any extension, inside the boot directory. That's it!

### Change the default password

Changing the default password is important if your Raspberry Pi is connected to the Internet or uses SSH. 
The default user is `pi`, and the password is `raspberry`.

Open terminal and type `passwd`, then follow the instruction.

### Enable Wifi 

Enabling Wifi is easy via the UI if you have a keyboard and mouse. It's one of the first options you will see when it starts up.  

If you're using RPi over headless - no keyboard or mouse (like me), do this:

Insert the SD Card or USB drive into your computer, then create a file named <a href="https://www.raspberrypi.org/documentation/configuration/wireless/headless.md" target="_blank"> `wpa_supplicant.conf` </a> inside the boot directory of the drive.

```json

ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=<Insert 2 letter ISO 3166-1 country code here>

network={
 ssid="<Name of your wireless LAN>"
 psk="<Password for your wireless LAN>"
}

```

Replace `country` code, `ssid` and `psk`, then insert it back into the Pi and power it up. *Note down the IP address*

### Enable password-less SSH 

Enable passwordless SSH to avoid passing the password around. 
From the computer where Ansible is installed (yes, we will use Ansible to manage the Kubernetes cluster), or from any PC where you will need to regularly connect to the RPis from, do the following: 

1. Generate the key

   `ssh-keygen -t rsa -f ~/pi`

   *Note: I am outputting the keys to the `home` path. You may choose to put yours in the default .ssh directory*

2. Copy the key from your computer to RPi over `SSH`

    `cat ~/pi.pub | ssh pi@192.168.0.60 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >>  ~/.ssh/authorized_keys"`

    Replace `192.168.0.60` with your RPi's IP address.

3. Test it!

   `ssh -i ~/pi pi@192.168.0.60`

### DHCP - Mac Address Configuration

Your RPi's IP address needs to be sticky for it to work in Kubernetes. 
Login to your router and reserve the IP address by associating your RPi's Mac address to the IP address. 
<p class="aligncenter">
<img alt ="dcp" class="lazyimg" src="https://user-images.githubusercontent.com/2548160/110120573-21874d80-7db5-11eb-80b7-ce5186745919.jpg"/> 
<br>
</p>

### Change Pi Hostname

Since you will have multiple RPi nodes, it can quickly become challenging to know which Pi you want to target over SSH.
I'd recommend you change the server hostnames by SSHing to the RPi and issue these commands: 

```cpp
# Update the system 
sudo apt update 
sudo apt full-upgrade

# Install vim 
sudo apt install vim

# Change the host name 
 sudo vi /etc/hostname
```

Suggested naming conventions is -  master, worker1, worker2, etc. 
Similarly, edit the `/etc/hosts` on your computer to follow suit, for example: 

```cpp
192.168.0.60 nas-worker1
192.168.0.61 master
192.168.0.62 worker2
```

That's it!


<a href="https://www.israelo.io/blog/pi-k8s-overview/" target="_blank"> << Raspberry Pi Kubernetes Cluster: Getting Started </a>  || 
<a href="https://www.israelo.io/blog/pi-k8s-install-k3s/" target="_blank">  Setting up K3s Kubernetes using Ansible >> </a>