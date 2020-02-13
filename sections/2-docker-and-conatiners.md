[Index](index) > Docker and containers
======================================
_In this section, we'll learn a bit about Docker, the most famous example of containerization software, and we'll dockerize the speedtest system._

What is Docker?
---------------
Docker is a is a system that allows us to pack and run computer programs in a virtual environment (containers), using operating system -level virtualization. This description is a bit abstruse, so let's look at some of the benefits this gives us.

### Programs packaged as virtual machines
By packaging the software we're building, along with the virtual machine used to run them, we can more closely control that our software will run in a similar environment locally, in test and production. It's also easier to support different technologies in all environments, since you only have to be able to run docker containers, while the containers themselves can run a lot of different technologies.

### Containers, a kind of lightweight virtual machine
Most of the things that Docker do, could be done with traditional virtualization technology before. The main difference is how easy it's to build, distribute and deploy. A lot of this is caused by the tooling being geared towards software development, and that OS-level virtualization can create virtual machines that's a lot more lightweight than before.

### Images, a snapshot of a container that can be shared and extended
One of the particularly useful parts of Docker, is that snapshots of the container "virtual machine" can be shared and extended as images. As we'll see later in this section, this enables us to start with empty .NET Core images, created by Microsoft, and use them to build our own images with a relatively small amount of configuration.

### Let's play with an image
You already ran an image when you verified your Docker installation by running `docker run hello-world`. Let's try it again, but this time using the `wernight/funbox` image:

```shell
&> docker run -it wernight/funbox:latest sl
```

* `docker run -it` tells docker that we want to run an image in "interactive mode". _Check [this page](https://docs.docker.com/engine/reference/run/) if you want the nitty gritty details of the `-it` flag_
* `wernight/funbox:latest` is a reference to a specific image. `wernight` is the publisher of this image called `funbox`, and we want to use the `latest` version of this image.
* The `funbox` image accepts some arguments of it's own, and we pass it the argument `sl`, that displays a train running across the terminal.

_You might want to run `clear` to clean up your terminal at this point._

The dockerization of speedtest-logger
-------------------------------------
Dockerizing an application, usually involves creating one or more Dockerfiles, that specify how your image should be built by docker. We'll now have a look at how we can create Dockerfiles for the different applications in the speedtest system, but first we'll take a detour to the humble terminal.

### Before you can dockerize, you must first understand how you publish your application
Diving into dockerizing your own applications can be very confusing, if you don't have a good grasp of how your program is compiled and packaged for production from the terminal. The reason for this, is that a Dockerfile mainly consists of us telling Docker to run a specific sequence of terminal commands inside the container/"virtual machine". It's easy to get lost if you cannot clearly identify the commands used for building your application from the commands used by Docker. For this reason, we'll start with a quick refresher on how to build and pack speedtest-logger for production.

### .NET Core applications is published with dotnet publish
Since .NET Core version 2.1, you can build an application with just `dotnet publish` from a folder containing a csproj-file for the application you want to publish. Previously, you would have to run `dotnet restore` first, in order to install dependencies like third-party libraries, but since 2.1, `dotnet publish` will restore and build your code if needed. Navigate into `/speedtest-logger/SpeedTestLogger` and try it out!

```shell
$ speedtest-logger/SpeedTestLogger> dotnet publish --output ./PublishedApp --configuration Release
Microsoft (R) Build Engine version 16.0.450+ga8dc7f1d34 for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.

  Restore completed in 59,06 ms for .../speedtest-logger/SpeedTestLogger/SpeedTestLogger.csproj.
  SpeedTestLogger -> .../speedtest-logger/SpeedTestLogger/bin/Release/netcoreapp2.1/SpeedTestLogger.dll
  SpeedTestLogger -> .../speedtest-logger/SpeedTestLogger/PublishedApp/
```

* `--output ./PublishedApp` tells `dotnet publish` to put your published files in a folder named `/PublishedApp` under the current folder.
* `--configuration Release` specifies that we want to build speedtest-logger for production.

Let's have a quick look in `/PublishedApp`. Here you'll find all the .dll -files from your code and imported packages. If you wanted, you could move the `/PublishedApp`-folder anywhere on your system, and it would still work, since all your dependencies are contained in that folder.

We can run the published speedtest-logger by invoking `SpeedTestLogger.dll` directly:

```shell
$ speedtest-logger/SpeedTestLogger> dotnet ./PublishedApp/SpeedTestLogger.dll
Starting SpeedTestLogger
Finding best test servers
...
```

_You might not notice it, since this project is so small, but the speedtest-logger starts and runs quite a bit faster this way._

### Out game-plan
Let's recap what we've learned before we start creating a Dockerfile for speedtest-logger:
1. A container is a bit like a virtual machine, and an image is a snapshot of a virtual machine that we can build upon.
1. It we're going to compile and run .NET Core code, we'll need to start with an image with the .NET Core SDK installed.
2. Then we need to move our code onto this "virtual machine"
3. We then have to publish our app with `dotnet publish`
4. Finally we'll have to somehow tell Docker to run `dotnet SpeedTestLogger.dll` when starting the container, so speedtest-logger will run.


Writing the Dockerfile
----------------------
Open `/SpeedTestLogger/Dockerfile` in your favorite editor, and quickly delete all the contents from the file.

### What's a Dockerfile?
A Dockerfile is a set of instructions that tells Docker how to build an image. In short, a Dockerfile + `docker build` will create an image that you can run just like the other images we have played with so far. This image can then be shared/published, downloaded and ran on other machines running Docker.

### Finding an image to start with
Microsoft publishes official images with the .NET Core SDK or runtime installed to [docker hub](https://hub.docker.com/_/microsoft-dotnet-core). Docker hub is a container registry, in the same way that [NuGet](https://www.nuget.org/) is a registry for .NET libraries and [npm](https://www.npmjs.com/) is a registry for a lot of things. You can find pre built images of different operating systems with tooling for programming environments like [openjdk](https://hub.docker.com/_/openjdk), [node](https://hub.docker.com/_/node) and [golang](https://hub.docker.com/_/golang). You can also find images with [bare linux distributions](https://hub.docker.com/_/alpine), if you're working with something really left-field.

We're targeting .NET Core 3.1, so we'll be using the image `mcr.microsoft.com/dotnet/core/sdk:3.1`. To state this in our Dockerfile use the `FROM ... AS ...` statement.

```Dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS build-stage
```

By starting a section of our dockerfile with this statement, we're declaring that we'll base our container on the `dotnet/core/sdk` image from `mcr.microsoft.com` with version `3.1`.

### Copying our code into the container
Before copying any code into our container, we should declare in what directory we're working inside the container. This is done with the `WORKDIR`-command.

```Dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS build-stage
WORKDIR /SpeedTestLogger
```

This simply declares that we'll be working in a folder called `/SpeedTestLogger` inside the container, and creates this folder if it doesn't exist. Now we have a folder inside the container to move code into.

_**A quick work about contexts:** Next we need to understand how Docker moved files into a container during `docker build`. When building the container, we give `docker build` a context from which it can copy files. The context is nothing more special than a folder on your computer, the important thing is that Docker cannot copy any files that's not part of the context into the image. We'll revisit the context later, when we're ready to build the image, but for now it's enough to know that the context we'll use is the same as the speedtest-logger repository folder, i.e. `/speedtest-logger`._

With the context out of the way, we can start writing out first `COPY`-statement.

```Dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS build-stage
WORKDIR /SpeedTestLogger

COPY /SpeedTestLogger ./
```

This simply copies all files from the folder `/SpeedTestLogger` in the build context (the folder `/speedtest-logger` on our machine) into our current working folder in the image (conveniently set to `/SpeedTestLogger` by the `WORKDIR` statement).

_If you're wondering what `./` is referencing, it's a path to the same folder you're currently inside. Another way of referencing the same location is by just using `.`._

The following diagram can be a useful mental model to have when working out how contexts and `COPY` moves files into the image:
```
  //////////////////      ////////////////////////////////      /////////////////
 /Folder with code/  =>  /Context passed to docker build/  =>  /COPY into image/
//////////////////      ////////////////////////////////      /////////////////
```

_Keeping track of what's in the context, and how the different relative paths from the context and into the container, is usually something that can be a little tricky when writing your own Dockerfiles. Just keep at it, and you'll eventually get there!_

### Building speedtest-logger inside the container
Now we're ready to build speedtest-logger inside the image. `RUN` is a command that can be used to execute arbitrary commands inside the image, and we'll use it to run `dotnet publish`.

```Dockerfile
...
COPY /SpeedTestLogger ./
RUN dotnet publish --output ./PublishedApp --configuration Release
```

When working with long commands with a lot of different parameters, it can be useful to break them up into several lines using `\`.
```Dockerfile
...
COPY /SpeedTestLogger ./
RUN dotnet publish \
    --output ./PublishedApp \
    --configuration Release
```

This will make the version control diff nicer when somebody decides to change what parameters to use.

### Making the build multi-staged
Up until now, we've been using the `mcr.microsoft.com/dotnet/core/sdk:3.1` image. When running in production, we probably don't want to drag the entire .NET Core SDK around everywhere. Fortunately, Microsoft also publishes the runtime version of .NET Core as a ready-to-use image called `mcr.microsoft.com/dotnet/core/aspnet:3.1`.

Using several images as part of the same Dockerfile, is called creating a multi-stage build. We can use a lot of different images to build our code, but only the last image will be kept as our final built image.

To start writing the next stage in our Dockerfile, we simply use another `FROM` ... statement, this time leaving out `AS`, since we won't be needing to reference this stage any other place in our build.
```Dockerfile
...
RUN dotnet publish \
    --output ./PublishedApp \
    --configuration Release

FROM mcr.microsoft.com/dotnet/core/aspnet:3.1
LABEL repository="github.com/k8s-101/speedtest-logger"
WORKDIR /SpeedTestLogger
```

As for the build-stage, we use `WORKDIR` to declare what folder to use inside the new image. This time we've also included a `LABEL` statement. Images can be labeled with different "tags". They can be nice to organize images, and can tell a story about what build and commit created a given image, but they're totally optional.

### Copying files from build-stage to the next stage
We now need to move the published files from our build-stage to the new stage. This is also done with `COPY`, this time using `--from` to specify the stage we want to copy from.

```Dockerfile
...
LABEL repository="github.com/k8s-101/speedtest-logger"
WORKDIR /SpeedTestLogger

COPY --from=build-stage /SpeedTestLogger/PublishedApp ./
```

### Setting the entrypoint of the container
Now we're one step away from a finished Dockerfile! The final step would be to run `dotnet SpeedTestLogger.dll` to start speedtest-logger. `ENTRYPOINT` is the command we'll be using to declare this. An `ENTRYPOINT` statement takes an array of a single executable `dotnet` in our case, and zero or more arguments. This statement is special, because it tells Docker that this is the command we want to run when a container is started based on our image.

```Dockerfile
...
COPY --from=build-stage /SpeedTestLogger/PublishedApp .
ENTRYPOINT ["dotnet", "SpeedTestLogger.dll"]
```

### Lets test the speedtest-logger Dockerfile!
By now your `Dockerfile` should look something like this:

```Dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS build-stage
WORKDIR /SpeedTestLogger

COPY /SpeedTestLogger ./
RUN dotnet publish \
    --output ./PublishedApp \
    --configuration Release

FROM mcr.microsoft.com/dotnet/core/aspnet:3.1
LABEL repository="github.com/k8s-101/speedtest-logger"
WORKDIR /SpeedTestLogger

COPY --from=build-stage /SpeedTestLogger/PublishedApp ./
ENTRYPOINT ["dotnet", "SpeedTestLogger.dll"]
```

Navigate to the `speedtest-logger` folder and run `docker build` with the following arguments:
```shell
$ speedtest-logger> docker build -f Dockerfile -t speed-test-logger:0.0.1 ./
Sending build context to Docker daemon  328.2kB
Step 1/9 : FROM microsoft/dotnet:2.1-sdk AS build-stage
 ---> d81b18feaa95
Step 2/9 : WORKDIR /SpeedTestLogger
 ---> Using cache
 ---> 7f15c3c21a43
Step 3/9 : COPY /SpeedTestLogger ./
 ---> 40014f055358
Step 4/9 : RUN dotnet publish     --output ./PublishedApp     --configuration Release
 ---> Running in 89e153e5ea01
Microsoft (R) Build Engine version 16.0.450+ga8dc7f1d34 for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.

  Restore completed in 14.73 sec for /SpeedTestLogger/SpeedTestLogger.csproj.
  SpeedTestLogger -> /SpeedTestLogger/bin/Release/netcoreapp2.1/SpeedTestLogger.dll
  SpeedTestLogger -> /SpeedTestLogger/PublishedApp/
Removing intermediate container 89e153e5ea01
 ---> 6f697a45bf63
Step 5/9 : FROM microsoft/dotnet:2.1-aspnetcore-runtime
 ---> 35cfaa4a836c
Step 6/9 : LABEL repository="github.com/k8s-101/speedtest-logger"
 ---> Using cache
 ---> c8071823ac49
Step 7/9 : WORKDIR /SpeedTestLogger
 ---> Using cache
 ---> ef80a9002afa
Step 8/9 : COPY --from=build-stage /SpeedTestLogger/PublishedApp ./
 ---> Using cache
 ---> 0c739916b693
Step 9/9 : ENTRYPOINT ["dotnet", "SpeedTestLogger.dll"]
 ---> Using cache
 ---> b7eab9df8b67
Successfully built b7eab9df8b67
Successfully tagged speed-test-logger:0.0.1
```

* `-f Dockerfile` tells Docker to the Dockerfile in this folder.
* `-t speed-test-logger:0.0.1` is the name and version of the image we'll be building.
* `./` is the build context, i.e. the folder we're currently running the command from aka. `speedtest-logger`.

Let's test our new image with `docker run`:
```shell
$ speedtest-logger> docker run -it speed-test-logger:0.0.1
Starting SpeedTestLogger
Finding best test servers
Testing download speed
Testing upload speed
Got download: 19.67 Mbps and upload: 4.99 Mbps
...
```

Optimizing the Dockerfile
-------------------------
As you probably noted, running `docker build` can take some time, but we can do several things to speed it up.

### Using layers to cache restored packages more efficiently
Try running `docker build -f Dockerfile -t speed-test-logger:0.0.1 ./` again. Note how quick it was the second time? What's going on?

Docker will cache as many steps as possible in order to only re-build the relevant parts of the image. Since we didn't change any files in the build-context, nothing needed to change in the image, and the build was super quick. Let's add a newline or something else somewhere in our code and run `docker build` again.

Notice how we had to wait several seconds while `dotnet publish` restored all dependencies, even though we didn't change any of them? We can improve this!
```shell
...
Step 4/9 : RUN dotnet publish     --output ./PublishedApp     --configuration Release
 ---> Running in b72b2ba71b72
Microsoft (R) Build Engine version 16.0.450+ga8dc7f1d34 for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.

  Restore completed in 14.95 sec for /SpeedTestLogger/SpeedTestLogger.csproj.
...
```

We can use another `COPY` and `RUN` statement to ensure that Docker only restores packages when we change `SpeedTestLogger.csproj`, since this file is the only place we're declaring which dependencies we're using. Update your Dockerfile with the following section and run `docker build` twice again, with a small change to one of the code files in-between:

```Dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS build-stage
WORKDIR /SpeedTestLogger

COPY /SpeedTestLogger/SpeedTestLogger.csproj ./
RUN dotnet restore

COPY /SpeedTestLogger ./
RUN dotnet publish \
    --output ./PublishedApp \
    --configuration Release \
    --no-restore

FROM mcr.microsoft.com/dotnet/core/aspnet:3.1
LABEL repository="github.com/k8s-101/speedtest-logger"
WORKDIR /SpeedTestLogger

COPY --from=build-stage /SpeedTestLogger/PublishedApp ./
ENTRYPOINT ["dotnet", "SpeedTestLogger.dll"]
```

```shell
$ speedtest-logger> docker build -f Dockerfile -t speed-test-logger:0.0.1 ./
...
Step 4/11 : RUN dotnet restore
 ---> Using cache
 ---> 96393a32a3b4
Step 5/11 : COPY /SpeedTestLogger ./
 ---> Using cache
 ---> 6a6f39992822
Step 6/11 : RUN dotnet publish     --output ./PublishedApp     --configuration Release     --no-restore
 ---> Running in 7b7b5441eb60
Microsoft (R) Build Engine version 16.0.450+ga8dc7f1d34 for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.

  SpeedTestLogger -> /SpeedTestLogger/bin/Release/netcoreapp2.1/SpeedTestLogger.dll
  SpeedTestLogger -> /SpeedTestLogger/PublishedApp/
...
```

Notice how we're no longer spending any time restoring unchanged packages the second time we build? Success!

_Whether or not we can benefit from improved caching usually depends on other factors as well. If we mostly run `docker build` on a build server, and it doesn't support Docker caching between builds, we won't see any improvements._

### Using .dockerignore to only move the files you really need into the build context
Since Docker tracks changes in all files that are part of the context, and spends time moving them into the build context, we can improve speed by ignoring unused files in the build context. This can be done by more consciously selecting a build context, or by using a `.dockerignore`-file. The `.dockerignore`-file is similar to a `.gitignore`-file, in that you can specify file or folders -patterns that you want to ignore. We've already included a basic example that ignores `/bin` and `/obj` folders.


Dockerizing speedtest-api and speedtest-web
-------------------------------------------
Now we leave you on your own to dockerize speedtest-api and speedtest-web. We've included finished Dockerfiles, and it's up to you if you want to delete them and create your own, or use them as-is.

When you're done, you should be able to build an image for speedtest-api with the following command:
```shell
$ speedtest-api> docker build -f Dockerfile -t speed-test-api:0.0.1 ./
```

And you should be able to build an image for speedtest-web with this command:
```shell
$ speedtest-web> docker build -f Dockerfile -t speed-test-web:0.0.1 ./
```


What now?
---------
We've created a lot of images, but they're currently only available on our computer. Join us in the next section, where we'll [share our images with a Docker registry](3-create-a-docker-registry).
