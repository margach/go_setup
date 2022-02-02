# How to create small and secure container images
#### Before you start
You must have installed Docker in your local environment.     
For instructions on how to do this, check out the [official Docker docs](https://docs.docker.com/get-docker/) for your operating system.

#### Introduction 
The first step to deploying to a Kubernetes cluster, you must put your app into a container image.    
In this article, you will learn how to create small and secure container images using the builder pattern and using small base images to build your container images.

#### Building Container Images
Thanks to Docker, building container images is very easy. You start with a base image and, you just add your app to the image and build the container.
If you are just starting and want to test things quickly, you can build your docker image with the default base image normally provided by the official maintainers of the language.

### Using small base images
Most docker images start with Debian or Ubuntu as a base image as their starting point and while this is great for easy onboarding and getting started, these images add hundreds of megabytes to the size of the final docker image.
The additional size is mostly the toolchain and other packages that your app does not probably need to run.

For example, the default base image for a basic Golang Hello App built using the fault base image is over *700 MB* and the basic Golang app is only a few bytes in size.

All this additional head over is wasted space and a place for bugs to hide.

The quickest and easiest way to create a small container image is to use a small image as a starting point, luckily, most programming languages have support for official docker images which are much smaller than the base default images.

For example, moving from the default base image of *node:8* to *node:8-alpine* reduces your image size from *670 MB* to just *65 MB*.

| Docker Image  | Size (MB) |
| ------------- | ------------- |
| node:8       | 670  |
| node:8-alpine  | 65  |

#### How to move your node app to a smaller image.

In your docker file. 
```
FROM node:alpine
WORKDIR /app
COPY package.json /app/package.json
RUN npm install --production
COPY server.js /app/server.js
EXPOSES 8080
CMD nmp start
```
- start with the node: alpine as your base image.
- Copy your app into the container.
- Install any dependencies that your app needs
- start the NodeJS server.

  
If your programming language does not support a small base image, you can build your container starting with raw Alpine Linux as the base image.

### The Builder pattern
In the previous section, we have dockized a NodeJS app that uses an interpreted language. What is an interpreted language?
The source code of an interpreted language is sent to an interpreter and it is executed directly without any compilation step.

For a compiled language, like Golang, the source code is first compiled into a binary which is later executed.
For the binary to run in the container, it does not need the compiler or linker that created it for it to execute successfully.

The builder pattern is for compiled languages, the source code is compiled in one docker image that has all the toolchain needed to handle the compilation step and the final compiled code (the binary) is transferred into another small base image without the compilers and linkers which are not needed to run the app.

#### Let's use Golang to show how the build pattern works.

This build step uses `GOPATH` for go versions not using modules.
For more information on Golang modules, check out the [official docs](https://go.dev/ref/mod) and this [blog post](https://go.dev/blog/using-go-modules) from the Go team.

Docker file
```
FROM golang:alpine AS build-env
WORKDIR  /app
ADD . /app
RUN cd /app && go build -o goapp

FROM alpine
RUN apk update && apk add ca-certificates && rm -rf /var/cache/apk/*
WORKDIR /app
COPY --from=build-env /app/goapp /app

EXPOSE 8080
ENTRYPOINT ./goapp

```

This is a multistep docker file that has 2 `FROM` keywords.
- In the first step, the `golang:alpine` image has all tool-chain to handle the compilation of the source code.
- The source code is added to the `/app` directory.
- The container runs the build command which compiles the code and outputs the binary.
- The next step starts with a raw Alpine Linux base image without any of the build tools in the first step.
- Alpine Linux does not come with root SSL certificates and so in the next steps, install the certificates. 
- The certificates are required in HTTPS API calls.
- The final step copies the binary from the previous step into the raw Alpine image and executes the app.

New versions of Go are using modules and this shows how to build the container image with the module's configuration.

Dockerfile
```
FROM golang:1.15.7-buster AS build-env
COPY . /go
RUN unset GOPATH && go build -o goapp

FROM ubuntu:20.04
COPY --from=build-env /go/path/goapp

EXPOSE 8080
ENTRYPOINT ./goapp
```

The principle is the same and the difference is only in the configuration of the language itself.
- The first step uses a Debian-based image as a build environment and this image has the required toolchain to compile the source code.
- The next step disables the GOPATH and this defaults to using the module's configuration which is the default for new versions of Golang.
- Ubuntu provides a minimal base image which is only about 25MB and is suitable as a starting point.
- The binary is copied from the previous step into the small base image and the application is executed.

### Conclusion
By creating small container images, you reduce the surface area of attacks for your application and your images use minimal memory during storage.

Another major advantage comes when your container image is being pulled into a cluster node for Kubernetes to start, big images take a long time to download and initialize, which translates into downtime for your app when the image is being pulled into a node.
