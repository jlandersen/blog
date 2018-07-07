+++
author = "Jeppe Lund Andersen"
categories = ["Docker", "Containers", "DevOps", "Windows Server 2016", "ALM", "Windows"]
date = 2016-12-18T12:16:05Z
description = ""
draft = false
slug = "accessing-a-running-windows-server-2016-container"
tags = ["Docker", "Containers", "DevOps", "Windows Server 2016", "ALM", "Windows"]
title = "Accessing a Running Windows Server 2016 Container"

+++

Occasionally it can be useful to access a running container. On Linux it is possible to open a bash session with a new instance of the container's shell with `docker exec -it [containername] bash`.

Because the Docker interface works the same way on Windows it is surprisingly easy to open a command prompt for a running container. Simply execute `docker exec -it [containername] cmd`. When you want to exit again, just use the `exit` command.

You can also open a Powershell session instead, just change the command to be `docker exec -it [containername] powershell`.

Happy container administration!