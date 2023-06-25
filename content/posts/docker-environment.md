---
title: "Streamlining Your Development Workflow with Docker Containers"
date: 2023-06-18T10:48:56+02:00
draft: true
---

## Introduction

In today's fast-paced world of software development, it's more important than ever to streamline your workflow and eliminate any unnecessary roadblocks. One way to achieve this is by using Docker containers in your programming projects. By providing a consistent and reproducible environment for running code, Docker can help you avoid issues with dependencies and configuration differences between machines.

But that's just the beginning. In this article, we'll explore how Docker containers can make it easier to test and debug code, collaborate with others on open source projects, deploy applications across different environments, increase efficiency in programming projects by providing standardized lightweight environments that are consistently replicated across development and production environments; improve portability allowing greater flexibility and scalability when running applications; provide clean separation between components of an application making it easier to debug, test, secure applications; reduce costs over time as they require fewer resources than VMs resulting in cost savings over time; ensure stability regardless of host environment which makes them ideal for working with open source software which may not have consistent testing or support across different environments.

By the end of this article, you'll have a solid understanding of how Docker containers can revolutionize your development workflow – helping you work smarter (not harder) while delivering high-quality results faster than ever before.

## Why Docker?
Here are just some of the ways that Docker containers can help streamline your development workflow:

- Docker containers provide a consistent and reproducible environment for running code, which can help avoid issues with dependencies and configuration differences between machines.
- Docker containers can make it easier to test and debug code, as you can quickly spin up and tear down containers with different configurations.
- Docker containers can help with collaboration, as you can share container images with others to ensure they are running the same environment as you.
- Docker containers can help with deployment, as you can package your code and its dependencies into a container image that can be easily deployed to different environments.
- Last but not the least, Docker containers can help you to keep your development environment clean and tidy, as you can easily spin up and tear down containers with different configurations without touching system or any other important files on your host machine.

## What is disadvantage of this approach?
- Docker containers have no pure device access, so you cannot use it for hardware development or testing.
- Docker containers are not as secure as virtual machines, as they share the same kernel and operating system with the host machine if host machine is Linux.
- It is easy to use Docker containers for software building for Linux, but it is not so easy to use it for Windows or Mac applications.

Today I am going to show how to prepare and use docker container for building linux-based open source project.
For the demonstation I will ose OpenSSH project (https://github.com/openssh/openssh-portable)

1. Prepare docker image
We will use debian stable image as base because of it's stability and popularity.

Here is example of Dockerfile:
```dockerfile
FROM debian:stable-slim
RUN apt-get update && \
    apt-get install -y bash build-essential git bc libncurses5-dev libssl-dev autoconf automake libtool zlib1g-dev && \
    rm -rf /var/lib/apt/lists/*

RUN apt-get update && apt-get install -y locales && rm -rf /var/lib/apt/lists/* \
    && localedef -i en_US -c -f UTF-8 -A /usr/share/locale/locale.alias en_US.UTF-8
ENV LANG en_US.utf8
```

2. Build docker image
```bash
docker build -t dbe .
```

We are going to use this image for building any open-source linux-based project, so idea is following:

Use a command as a prefix to any string that will allow to execute it inside docker container on files that are mounted from host machine. So you can use any comfortable tools of yout choose to edit, analyze or version control system on your machine and the same time seamlessly build project inside docker container.

To have such seamless prefix commands I am going to introduce environment modification (zsh in my case) that will add prefix to any command that will be executed in current shell session.

```bash
DBE_IMAGE="dbe"

pwd_alias() { echo "$PWD" ;}
      cwd() { basename $(pwd_alias); }
```

We will call all of this as DBE (Docker Build Environment)
Image name then will be **dbe**.

**pwd_alias** is a function that will return current working directory path.
**cwd** is a function that will return current working directory name. We will use it to keep access to the files in current working directory both from host machine and docker container.

Next, we should use Docker Volumes to keep all files inside the container.

# Using a volume in Docker has several benefits:
1. Data persistence: When a container is deleted, any data that was stored inside the container is also deleted.
   By using a volume, you can store data outside of the container and ensure that it persists even if the container is deleted.

2. Sharing data between containers: Volumes can be shared between containers, allowing multiple containers to access the same data.

3. Improved performance: When using a volume, Docker does not need to copy data between the container and the host file
   system, which can improve performance.

4. Easier backup and restore: Since data is stored outside of the container, it is easier to backup and restore data.

The main benefits is we can easily create and remove containers, but data will be untoched.
But for us this is not so important, since our files kept on host machine, nothing will be created only inside the container. But since I am using this approach in more complicated cases where container should keep temporary files during the build process, I will use volumes everywhere to keep scripts/environments consistent.

```bash
docker volume inspect dbeVolume 1>/dev/null 2>&1

if [[ $? != 0 ]]; then
	echo "== Create volume for the project =="
	docker volume create --name dbeVolume
	docker run --privileged -it --rm -v dbeVolume:/workdir busybox chown -R 1000:1000 /workdir
	echo
fi
```

This part will check if volume exists and create it if not. Also it will change owner of the volume to the current user to avoid any permission issues.

and now the main function that our customized environment will have:
```bash
dbe() {
docker run -it --rm --name dbe_instance \
        -v dbeVolume:/workdir  \
        -v $(pwd_alias):/workdir/src \
        --workdir=/workdir/src \
        ${DBE_IMAGE} $@;
}

So, this will run container based on our image and pass argument to dbe() function as command to be executed inside the container. Our current directory will be mounted to /workdir/src inside the container, so we can access all files from host machine inside the container for building tools that continer will provide for us.

Sometimes, we will need just a shell access inside the container, so we will add another function for this:

```bash
dbesh() {
	echo "== development environment =="
	echo

	dbe "/bin/bash"
}

Please note that functions names are short enought to be memorable and easy to type in terminal.

At the last lines of our environment it will be nice to have a banner that will show us which functions are available:

```bash
echo " This environment has following functions:"
echo "============================================"
echo " * dbe      - start container with project located in current directory"
echo " * dbe_exec - execute specified command in this container environment"
echo
```

Here is the link to Dockerfile to create the image: Dockerfile
Here is the link to Environment file to customize shell environment: env

Now we can start building openssh project:

```bash
git clone
cd openssh-portable
dbe autoreconf
dbe ./configure
dbe make
```

To start key generator we still can use prefix dbe():
```bash
dbe ssh-keygen
```

That will start ssh-keygen (probably mpdified by us) inside the container and we will be able to generate keys.
More than that we can compare output of our modified software with original one (running in the host or in another development container) to see if we have any differences.
