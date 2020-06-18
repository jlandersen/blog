---
title: ".NET Core Diagnostics Tools and Containers"
date: 2020-06-17T09:46:40+02:00
slug: "dotnetcore-diagnostics-tools-and-containers"
draft: false
tags: [".NET Core", "ASP.NET Core", "DevOps", "Observability", "Diagnostics"]
---


The story around diagnostics tools is still evolving, but now have a good set of [tools available](https://github.com/dotnet/diagnostics). In my previous post, we tested the counters tool a bit. The most problematic part, currently, is to use these tools where the application is running as containers and extracting the dumped files for anaysis. In this post I will show examples of using these tools to troubleshoot .NET Core containers. I will focus on the `dump`, `trace` and `counters` tools, but anything should go for others as well.

There are two issues being addressed:
 - Accessing the running dotnet process with the diagnostics tools
 - Extracting the collected data

 As a brief background, from .NET Core 3.0, all applications opens a pipe through which the tools can communicate. At the time of writing, the protocol is still in progress and little documentation is available (Github issues is currently the best source - the protocol does have a [markdown](https://github.com/dotnet/diagnostics/blob/master/documentation/design-docs/ipc-protocol.md)). On Linux and macOS, this is done with a Unix Domain Socket in the `/tmp` folder. Here is the `/tmp` folder for one of my running applications:

 ```bash
user@ca47cea32705:/tmp# ls -l /tmp
total 4
prwx------ 1 root root    0 Jun 17 09:04 clr-debug-pipe-1-213386-in
prwx------ 1 root root    0 Jun 17 09:04 clr-debug-pipe-1-213386-out
srw------- 1 root root    0 Jun 17 09:04 dotnet-diagnostic-1-213386-socket
drwxr-xr-x 2 root root 4096 Jun 17 09:12 system-commandline-sentinel-files
 ```

Maybe obvious from the file name, `dotnet-diagnostic-1-213386-socket` is the socket (otherwise, the *s* in the mode `srw` indicates it).

With `socat` we can open the socket and say hello:

```
user@ca47cea32705:/tmp# socat - UNIX-CLIENT:/tmp/dotnet-diagnostic-1-213386-socket
.
DOTNET_IPC_V1����
```

This is enough knowledge to connect the pieces. From Github issues, it appears there will be more options coming in the future for connecting and configuring this. For now, this is always where we will find this socket.

# Bundle Tools in the Container Image
This is what I would consider the simplest approach, that could work well for you. The idea is to bundle up the necessary tools inside your images.
A Dockerfile for building such an image could look like this:

```dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS build

# Install the tools
RUN dotnet tool install --tool-path /tools dotnet-trace
RUN dotnet tool install --tool-path /tools dotnet-dump
RUN dotnet tool install --tool-path /tools dotnet-counters

# Build the app
WORKDIR /app
COPY ./*.csproj ./
RUN dotnet restore /app/DiagnosticsApp.csproj

COPY ./ ./
RUN dotnet publish -c Release -o out

FROM mcr.microsoft.com/dotnet/core/aspnet:3.1 AS runtime
WORKDIR /app
COPY --from=build /app/out ./
COPY --from=build /tools /tools
ENTRYPOINT ["dotnet", "DiagnosticsApp.dll"]
```

Let's run the application and then do a trace after:
```bash
# Start the application, with a bind mount on /tmp/data on the host to /data inside the container
docker run --name diagnosticsapp -p 8000:80 -v "/tmp/data:/data" -d diagnosticsapp

# Collect a trace and write it to the volume
docker exec -it diagnosticsapp /tools/dotnet-trace collect -o /data/trace.nettrace -p 1
```

When you are done, the trace is now available in `/tmp/data` on the host. I am assuming the PID of your app is 1 here, which it will be in most cases if you use the Microsoft base images. Often, you won't have a host to access, so as an alternative, you could mount a volume using an NFS or S3 driver and get your diagnostics data out.

# Tools in a Separate Container
Knowing the communication happens through the socket in `/tmp`, we can use that to run the tools outside the application container. Aside from general separation of concerns, this is also nice to avoid using additional resources inside application container (from experience, the time you want these data, the application is often struggling in the first place...).

Let's build a simple "tools image":

```dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:3.1 as tools
RUN dotnet tool install --tool-path /tools dotnet-trace
RUN dotnet tool install --tool-path /tools dotnet-dump
RUN dotnet tool install --tool-path /tools dotnet-counters

FROM mcr.microsoft.com/dotnet/core/runtime:3.1 AS runtime
COPY --from=tools /tools /tools
```


To use it, first run your application image and then use the new image to do traces or dumps:
```bash
# Run an application
docker run --name app -p 8000:80 -v "/tmp/data:/data" -v "diagnostics:/tmp" --cap-add=SYS_PTRACE -d app

# Collect dump from the app container process
docker run --rm --pid container:app -v "diagnostics:/tmp" -v "/tmp/data:/data" -it tools /tools/dotnet-dump collect -p 1 -o /data/dump
```

Some of these arguments require a bit of explanation:
 - `--pid container:app` we make the tool container join the PID namespace of the application. Technically this is not needed, because the socket is used for communicating, but it will make a `ps` command show the actual dotnet application.
 - `-v "diagnostics:/tmp"` use a volume `diagnostics` shared between the containers mounted at `/tmp` in both. This is what connects the two.
 - `-v "/tmp/data:/data"` we still need a way to extract the collected data. You might wonder why do the bind mount on both containers. The reason is, some of the tools collect and write the data, while others trigger the application process to write out the data (`dump` and `gcdump` does this).

# Diagnostics as a Sidecar
Building on the previous example, we can deploy a tools container along the application container, to have it ready in case a trace or dump is needed. This is my favourite approach. We don't have to make any changes to the images for this - here is an example `docker-compose.yml` that powers it:

```yaml
version: "3.8"
services:
    web:
        image: webapp:latest
        ports:
            - "80:80"
        volumes:
            - diagnostics:/tmp
            - data:/data
    diagnostics:
        image: tools:latest
        volumes:
            - diagnostics:/tmp
            - data:/data
        tty: true  # Keep the container alive

volumes:
    diagnostics:
    data:
```

I am not doing any bind volumes here - to extract the collected files i'd bundle up the necessary scripts in the tools image to ship the files from /data to storage (unless you use a suitable volume driver as previously mentioned).

# Conclusion
Cross platform diagnostics is getting better for .NET Core. For now we still have to go through some hoops to get a smooth ride when trouble shooting applications, but it has reached a point where it has helped me several times. I am excited to see what is coming with, and after, .NET 5. For now, I hope this post gave some pointers to how you might go about getting more insights into your containerized .NET Core applications.


