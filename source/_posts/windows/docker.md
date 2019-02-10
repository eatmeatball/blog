---
title: docker
date: 2019-01-01 16:25:59
tags: linux
categories: linux
---

```shell
$docker-machine create --driver virtualbox --virtualbox-boot2docker-url=/Users/hzj/.docker/machine/cache/boot2docker.iso  default
```

```
Running pre-create checks...
(default) Boot2Docker URL was explicitly set to "/Users/hzj/.docker/machine/cache/boot2docker.iso" at create time, so Docker Machine cannot upgrade this machine to the latest version.
Creating machine...
(default) Boot2Docker URL was explicitly set to "/Users/hzj/.docker/machine/cache/boot2docker.iso" at create time, so Docker Machine cannot upgrade this machine to the latest version.
(default) Downloading C:\Users\hzj\.docker\machine\cache\boot2docker.iso from /Users/hzj/.docker/machine/cache/boot2docker.iso...
(default) Creating VirtualBox VM...
(default) Creating SSH key...
(default) Starting the VM...
(default) Check network to re-create if needed...
(default) Windows might ask for the permission to configure a dhcp server. Sometimes, such confirmation window is minimized in the taskbar.
(default) Waiting for an IP...
Waiting for machine to be running, this may take a few minutes...
Detecting operating system of created instance...
Waiting for SSH to be available...
Detecting the provisioner...
Provisioning with boot2docker...
Copying certs to the local machine directory...
Copying certs to the remote machine...
Setting Docker configuration on the remote daemon...

This machine has been allocated an IP address, but Docker Machine could not
reach it successfully.

SSH for the machine should still work, but connecting to exposed ports, such as
the Docker daemon port (usually <ip>:2376), may not work properly.

You may need to add the route manually, or use another related workaround.

This could be due to a VPN, proxy, or host file configuration issue.

You also might want to clear any VirtualBox host only interfaces you are not using.
Checking connection to Docker...
Docker is up and running!
To see how to connect your Docker Client to the Docker Engine running on this virtual machine, run: D:\Software\Docker Toolbox\docker-machine.exe env default
```