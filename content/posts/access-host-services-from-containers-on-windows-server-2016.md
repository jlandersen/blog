+++
author = "Jeppe Lund Andersen"
categories = ["Docker", "Windows Server 2016", "Windows", ".NET", "Continuous Integration", "DevOps", "ALM"]
date = 2017-01-31T17:30:31Z
description = ""
draft = false
slug = "access-host-services-from-containers-on-windows-server-2016"
tags = ["Docker", "Windows Server 2016", "Windows", ".NET", "Continuous Integration", "DevOps", "ALM"]
title = "Access Host Services from Containers on Windows Server 2016"

+++

Occasionally it is necessary to access services running on the host machine from a container. You may be inclined to use localhost to reach the host, but this will not work as the container itself is localhost. Here is how to access host services from a container running on Windows Server 2016. 

> This applies to containers running with NAT networking mode. This is the default, so unless you [explicitly use something else](https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/container-networking), this should work for you.

If you read up [Windows Container Networking](https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/container-networking) you will find that with NAT network, by default, each container will get a virtual network adapter assigned to an IP in a subnet (by default `172.16.0.0/12`). The gateway will be the host machine unless you manually create a NAT network and specify it explicitly. With this knowledge we can retrieve the host address in two ways.


####  Method 1) Virtual Network Adapter on Host Machine
Open a new CMD or Powershell session and hit `ipconfig`. You should now see the vNIC acting as the gateway for containers, having an IP address starting with 172. In the picture below it is the *HNS Internal NIC*, with IP address `172.20.128.1`:
![Access docker host IP address](/images/2017/01/screen1.PNG)

####  Method 2) Virtual Network Adapter in Container
Similarly, in a running container, the same IP will appear as the gateway. Here is a screenshot from [accessing a running container](http://nocture.dk/2016/12/18/accessing-a-running-windows-server-2016-container/) and issuing `ipconfig`:
![ipconfig in running container](/images/2017/01/screen2.PNG)

From the running container, an application can use this information to get the address `172.20.128.1` and access services on the host machine.

