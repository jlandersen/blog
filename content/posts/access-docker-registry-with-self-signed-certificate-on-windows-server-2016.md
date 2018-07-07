+++
author = "Jeppe Lund Andersen"
categories = ["Docker", "Windows", "Windows Server 2016", "Containers", "ALM", "DevOps"]
date = 2016-10-26T05:13:09Z
description = ""
draft = false
slug = "access-docker-registry-with-self-signed-certificate-on-windows-server-2016"
tags = ["Docker", "Windows", "Windows Server 2016", "Containers", "ALM", "DevOps"]
title = "Access Docker Registry with Self-Signed Certificate on Windows Server 2016"

+++

With Windows Server 2016 available it's time to find out all the little details when using Docker to administrate containers. A common scenario is accessing a private/internal hosted Docker Registry protected with a self-signed certificate ([Details](https://docs.docker.com/registry/insecure/#/using-self-signed-certificates)). 

Here is how you enable the Docker daemon and CLI on Windows Server 2016 to use your certificate when talking to the registry.

* Have your certificate (.crt) file available on the server
* Copy it to `C:\ProgramData\docker\certs.d\[hostname][port]\ca.crt` 

Example with internal registry on DNS and port `myinternalregistry:5000`:

* `C:\ProgramData\docker\certs.d\myinternalregistry5000\ca.crt` 

Then restart the Docker daemon in a PowerShell session:

* `Restart-Service docker`

You can now docker run/pull/push to your private registry.

Note that the `certs.d` folder may not exist - in this case just create it. If you ever used Docker on Linux you would have a `certs.d` as well, with the registry folder in the format `[hostname]:[port]`. On Windows you cannot have `:` in folder names. If you dive into the [source code](https://github.com/docker/docker/blob/12c67f42d85103c57c14f27579ff46fccdc3ee07/registry/config_windows.go#L10) you can see Docker simply strips the colon from the path on Windows


