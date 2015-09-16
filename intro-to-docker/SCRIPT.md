# Introduction to Docker

This document is meant to be a quick introduction to docker, and will cover the basic architecture, challenges, managing containers and images, and through to building and running a static website on docker.

[TOC]

## Slides

This document only contains the show-and-tell portion. Architectural overview slides can be found [here](https://github.com/ripta/decks.js/blob/master/intro-to-docker/index.html).

If you'd like to follow, you'll need access to a docker host.

## Installation

If you don't have anything setup on your OS X machine, expect to download about 200 MB of data, though the exact amount depends on what prerequisites you already have on your machine.

**I highly recommend the CLI way.**

Doing so requires more time to get up to speed and deals with lower level commands. While there's more to learn, I think it's more useful in the long run to know the concepts behind new technology, how to operate it, and how to fix things should they break.

The GUI way is easily, but abstracts away most concepts. If things go haywire—especially considering that docker isn't the simplest of technologies and that the GUI isn't mature (yet), they *will* go haywire—you're better off learning from the ground up.

### The CLI Way (Recommended)

On OS X, you'll need a hypervisor, e.g., VirtualBox, VMWare Fusion, or Parallels Desktop. If you don't have one already the easiest way to go is by using Homebrew.

This portion assumes you already have Homebrew installed and running.

Install the `caskroom` extension to homebrew if you haven't already, which will allow you to install Application binaries from the command line:

```
$ brew update
$ brew install caskroom/cask/brew-cask
```

Install one of VirtualBox (free and recommended by *Docker, Inc*), VMWare Fusion (not-free, but Ripta's preference), or Parallels Desktop:

```
$ brew cask install virtualbox
$ brew cask install vmware-fusion
$ brew cask install parallels-desktop
```

Get the `docker` client, `docker-machine`, and `docker-compose`:

```
$ brew install docker docker-machine docker-compose
```

We won't be using `docker-compose` today, but it can be used to define and run multiple containers as one unit.

### The Easy, GUI Way

In August 2015, Docker, Inc. announced the release of [Docker Toolbox](https://www.docker.com/toolbox), which provides a one-click install for the docker client, docker machine, docker compose, a hypervisor, and a front-end GUI.

I won't be covering the GUI way.

## Create New Machine

Now that you have a hypervisor and the docker command line tools installed, we can start our first docker host.

It doesn't matter (at least not at this point) the directory from which you issue these commands.

I'm going to first create a new docker host running on VMWare. I'll name it `intro`, but you can choose any name you'd like; a name that resembles a hostname would be a good candidate, because it will actually be used as the hostname of the docker host:

```
$ docker-machine create -d vmwarefusion intro
Creating SSH key...
Creating VM...
Starting intro...
Waiting for VM to come online...
To see how to connect Docker to this machine, run: docker-machine env intro
```

If you're running on a different hypervisor, you'll need to lookup the driver name under the `--driver` option in the output of:

```
$ docker-machine create -h
```

It should take a minute to boot up your new docker host; longer if `docker-machine` needs to download the images, which it will do automatically.

Once your docker host is running, you'll now have a set of environment variables needed to access the new docker host:

```
$ docker-machine env intro
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://172.16.241.137:2376"
export DOCKER_CERT_PATH="/Users/rpasay/.docker/machine/machines/intro"
export DOCKER_MACHINE_NAME="intro"
```

Evaluating the output of the above command will set your terminal up:

```
$ eval `docker-machine env intro`
```

After setting up the environment variables, your `docker` client will now have access to your docker host (and docker instance running on said host).

As with all git-like CLI, `docker help` should get you started. A useful command is `docker info`, which provides information about the host:

```
$ docker info
Containers: 0
Images: 0
Storage Driver: aufs
 Root Dir: /mnt/sda1/var/lib/docker/aufs
 Backing Filesystem: extfs
 Dirs: 0
 Dirperm1 Supported: true
Execution Driver: native-0.2
Logging Driver: json-file
Kernel Version: 4.0.9-boot2docker
Operating System: Boot2Docker 1.8.1 (TCL 6.3); master : 7f12e95 - Thu Aug 13 03:24:56 UTC 2015
CPUs: 1
Total Memory: 996.2 MiB
Name: intro
ID: YK3Z:WC3H:AV2M:UZQC:5GXE:V7OK:CYZW:7HOH:RRJM:SDFJ:YYJY:RIW3
Debug mode (server): true
File Descriptors: 9
Goroutines: 16
System Time: 2015-09-14T02:59:59.584647298Z
EventsListeners: 0
Init SHA1:
Init Path: /usr/local/bin/docker
Docker Root Dir: /mnt/sda1/var/lib/docker
Labels:
 provider=vmwarefusion
```

Of note in the output are:

* number of containers, which are separate "things" running;
* number of images, which are snapshots of things that *could* be running, stored on disk;
* storage driver, which determine *how* images are stored on disk, usually **aufs** (advanced multilayered unification filesystem); and
* the kernel version running on the docker host.

The version of docker client (running on your local machine) and server (running on the docker host) can be retrieved using:

```
$ docker version
Client version: 1.7.0
Client API version: 1.19
Go version (client): go1.4.2
Git commit (client): 0baf609
OS/Arch (client): darwin/amd64

Server version: 1.8.1
Server API version: 1.20
Go version (server): go1.4.2
Git commit (server): d12ea79
OS/Arch (server): linux/amd64
```

Obviously, my client version is a little outdated. Not a problem in this case, but if you can, I'd recommend running the same version.

**Aside:** You'll probably never need to do so—and in fact, doing this could mess up your setup if you don't know what you're doing—however, you can access the docker host itself using `docker-machine ssh intro`. When inside the docker host, the amount of space available to your AUFS mount, which is used for images is useful to know. There are options under `docker-machine create` to expand this size if you're building a lot of images.

## Finding Images

Official images maintained by Docker, Inc are available under the `library` username:

```
$ docker search library
```

Alternatively, open [https://hub.docker.com/r/library/](https://hub.docker.com/r/library/) in your browser. Tags show different versions of images in the repo, and also the image size.

While it may be a matter of preference on which image to use as a base, I prefer smaller, slimmer base images: you can fit more per host and it's more portable. I've found Alpine Linux to be a good balance between size and feature.

One reason for a larger image is if it contains management tools. It's up for debate, but some folks have advocated for treating images as full OSes, each with potentially its own ssh daemon, for instance.

Let's use the smaller `alpine` image for now:

```
$ docker pull -a alpine
$ docker images alpine
REPOSITORY      TAG             IMAGE ID            CREATED             VIRTUAL SIZE
alpine          2.6             6e25877bc8bc        3 months ago        4.502 MB
alpine          2.7             c22deeac7b13        3 months ago        4.711 MB
alpine          3.1             878b6301beda        3 months ago        5.041 MB
alpine          3.2             31f630c65071        3 months ago        5.254 MB
alpine          latest          31f630c65071        3 months ago        5.254 MB
alpine          edge            5e704a9ae9ac        3 months ago        5.31 MB
```

Multiple tags can point to the same image ID, e.g., 3.2 and latest above.

### Running Container

Running a container with alpine v3.2:

```
$ docker run -i -t alpine:3.2 /bin/sh
```

where `-i` keeps STDIN open even when we're not attached to the container, `-t` allocates a pseudo TTY, and `/bin/sh` being the init process to run inside the container.

Inside shell, the hostname of the container corresponds to the container ID:

```
/ # hostname
3747ada8f970
```

There aren't many processes running. In fact, `/bin/sh`, init process (PID 1), is the only thing normally running:

```
/ # ps -ef
PID   USER     TIME   COMMAND
    1 root       0:00 /bin/sh
   11 root       0:00 ps -ef
```

This is important to note, because just like a full-blown OS:

1. when the init process exits, the container stops;
2. the init process should be prepared to own any orphaned processes; and
3. the init process should be prepared to reap any adopted processes.

We can also view disk and memory usage:

```
/ # df -h
Filesystem                Size      Used Available Use% Mounted on
none                     18.2G      1.6G     15.6G   9% /
tmpfs                   498.1M         0    498.1M   0% /dev
shm                      64.0M         0     64.0M   0% /dev/shm
tmpfs                   498.1M         0    498.1M   0% /sys/fs/cgroup
/dev/sda1                18.2G      1.6G     15.6G   9% /etc/resolv.conf
/dev/sda1                18.2G      1.6G     15.6G   9% /etc/hostname
/dev/sda1                18.2G      1.6G     15.6G   9% /etc/hosts
tmpfs                   498.1M         0    498.1M   0% /proc/kcore
tmpfs                   498.1M         0    498.1M   0% /proc/timer_stats

/ # free -m
             total         used         free       shared      buffers
Mem:           996          379          617          112           13
-/+ buffers:                366          630
Swap:         1108            0         1108
```

Because we have not restricted resources on the container, it has access to all the host's resources.

You can also view routing information:

```
/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
5: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:11:00:01 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe11:1/64 scope link
       valid_lft forever preferred_lft forever
```

Try making a curl request to `ifconfig.co`, which returns your public IP address:

```
/ # curl ifconfig.co
/bin/sh: curl: not found
```

Thankfully, Alpine Linux comes with a lightweight package manager: apk.

```
/ # apk update
/ # apk add curl
```

After much output, you'll hopefully have `curl` available to you.

```
/ # curl ifconfig.co
12.174.123.130
```

Awesome. Let's exit the container (`exit` works, as does `C-D`). 

Let's start a new container using the same image:

```
$ docker run -i -t alpine:3.2 /bin/sh
/ # curl ifconfig.co
/bin/sh: curl: not found
```

Note, however, that curl is no longer available to us. It's important to note that when a container is run from an image:

1. Changes inside the container is invisible to any other container running the same image.
2. Changes to the image is not automatically kept.

**Observations:**

1. Once created, images are read-only. When a container is started using an image, a writable file system is created on top of the read-only image (thus, references to filesystem layers when doing a `docker pull` and the use of AUFS as a backing store to docker).
2. A command, `docker commit`, exists to create a new image based on changes introduced by a container. See its help page for information.


## Listing Containers

If you issued `docker ps` while a container is running, you'll see an entry for every running container in its output. However, `docker ps` only lists running containers.

When I exit the running container, it can still be queried with `-a`:

```
$ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                          PORTS               NAMES
3747ada8f970        alpine:3.2          "/bin/sh"           23 minutes ago      Exited (1) About a minute ago                       sick_einstein
```

Image and container IDs go hand-in-hand and are both 64-character identifiers, but refer to different objects.

## Restarting Containers

You can restart a stopped container, after which it will show as running again:

```
$ docker restart 3747ada8f970
3747ada8f970

$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
3747ada8f970        alpine:3.2          "/bin/sh"           34 minutes ago      Up 2 seconds                            sick_einstein
```

However, it runs the container in detached mode, meaning that no TTY is allocated against it. You could manually attach to a running container using:

```
$ docker attach 3747ada8f970
/ # %
```

Keep in mind that when you attach to a container, you are now interacting with its init process. Depending on the process, interrupting it (`C-C`) could kill the process and stop the container.

When you exit the container, it once again becomes stopped. Stopped containers always appear in `docker ps -a` and are not automatically removed. You can remove it manually using `docker rm`:

```
$ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                     PORTS               NAMES
3747ada8f970        alpine:3.2          "/bin/sh"           35 minutes ago      Exited (0) 3 seconds ago                       sick_einstein

$ docker rm 3747ada8f970
```

## Detached Containers

Because containers exit when the init process exits, the init process would need to be foregrounded. Take for example a forever process that loops and prints "Hello World!" once a second:

```
$ docker run alpine:3.2 /bin/sh -c 'while true; do echo "Hello World!"; sleep 1; done'
```

However, doing so keeps a terminal occupied. We've seen how to restart a container in detached mode, but when running a new container, the answer would be to explicitly detach the container with `-d`:

```
$ docker run -d alpine:3.2 /bin/sh -c 'while true; do echo "Hello World!"; sleep 1; done'
032b0875083eadcc20c21987ffbb04ca4c6b0f7dd89d1740ea5e4be0ea9ea583

$ docker ps
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS               NAMES
032b0875083e        alpine:3.2          "/bin/sh -c 'while t   2 minutes ago       Up 3 seconds                            reverent_engelbart
```

Viewing logs, both STDERR and STDOUT, which are still separate streams:

```
$ docker logs 032b0875083e
$ docker logs -f 032b0875083e
$ docker logs -f -t 032b0875083e
```

Viewing all running processes:

```
$ docker top 032b0875083e
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
root                1906                1530                0                   05:09               ?                   00:00:00            /bin/sh -c while true; do echo "Hello World!"; sleep 1; done
root                2009                1906                0                   05:10               ?                   00:00:00            sleep 1
```

Stopping a container (`docker stop ID`) sends a SIGTERM, while (`docker kill ID`) sends a SIGKILL. It's up to the init process on how it handles these signals; hopefully predictably.

## Removing Containers

Docker doesn't provide one command to remove all containers, but this should get you going:

`docker ps -a -q | xargs docker rm`

**Notes:**

1. This command will attempt to remove *all* containers, regardless of whether they're running or stopped. Thankfully, docker will prevent you from deleting running containers; you'll just get a lot of warning messages.
2. The `-q` option will print out only the container ID column, without a column header. It's very useful for shell magickery, and many commands support this option.

## Sample Application

You can get the project from [https://github.com/ripta/dockerfell](https://github.com/ripta/dockerfell):

```
$ cd ~/Projects
$ git clone git@github.com:ripta/dockerfell.git
```

The repository contains several docker projects. The one you want is `nginx-static-test`:

```
$ cd dockerfell/nginx-static-test
```

You may wish to edit the `Dockerfile` to have your name and email address.

Once ready, build the image, giving it the tag `$DOCKER_HUB_USER/nginx-static-test:1.0`, where `$DOCKER_HUB_USER` is your docker hub username if you have one.

If you don't plan on pushing the image to docker hub later—and you don't have to—you can give it a local tag, e.g., `nginx-static-test:1.0`. 

```
$ docker build -t $DOCKER_HUB_USER/nginx-static-test:1.0 .
Sending build context to Docker daemon 224.3 kB
Sending build context to Docker daemon
Step 0 : FROM alpine:3.2
 ---> 31f630c65071
Step 1 : MAINTAINER Ripta Pasay <ripta+docker@pasay.name>
 ---> Running in 4d28a4beff1f
 ---> 17b2ddda00c7
Removing intermediate container 4d28a4beff1f
Step 2 : ENV META_IMAGE_VERSION 1.0
 ---> Running in 10322b11c0a6
 ---> bc6ca695a5c7
Removing intermediate container 10322b11c0a6
Step 3 : RUN apk update
 ---> Running in eea0d955c8f2
fetch http://dl-4.alpinelinux.org/alpine/v3.2/main/x86_64/APKINDEX.tar.gz
v3.2.3-35-g583a384 [http://dl-4.alpinelinux.org/alpine/v3.2/main]
OK: 5295 distinct packages available
 ---> 517497e53894
Removing intermediate container eea0d955c8f2
Step 4 : RUN apk add nginx
 ---> Running in 1ed0c47d698a
(1/2) Installing pcre (8.37-r1)
(2/2) Installing nginx (1.8.0-r1)
Executing nginx-1.8.0-r1.pre-install
Executing busybox-1.23.2-r0.trigger
OK: 7 MiB in 17 packages
 ---> 10c88301b95d
Removing intermediate container 1ed0c47d698a
Step 5 : COPY . /usr/share/nginx/html/
 ---> 1bb05fedcd31
Removing intermediate container 01c1298bb966
Step 6 : RUN echo "Generated at $(date)" > /usr/share/nginx/html/VERSION
 ---> Running in 88287bcff994
 ---> 606891c0f07a
Removing intermediate container 88287bcff994
Step 7 : EXPOSE 80
 ---> Running in 64f4670e03c4
 ---> e1df01d77395
Removing intermediate container 64f4670e03c4
Successfully built e1df01d77395
```

You should have a new image, tagged with your tag, although the image ID will likely be different as you build at different times:

```
$ docker images
REPOSITORY                TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
ripta/nginx-static-test   1.0                 e1df01d77395        2 days ago          7.639 MB
```

Now try running nginx:

```
$ docker run -d -p 80 $DOCKER_HUB_USER/nginx-static-test:1.0 nginx
eedf6027be4bf66a20317265154ae857f4cae74ebab3d9ae151d61c20065d555
```

Even though you get a container ID, if you check `docker ps`, you'll notice that the container is not running.

Something is wrong, because the container has exited. What went wrong?

Let's spawn a shell against the image, and run nginx manually:

```
$ docker run -i -t $DOCKER_HUB_USER/nginx-static-test:1.0 /bin/sh
/ # which nginx
/usr/sbin/nginx
```

You'll notice that because the image already has nginx on it, we don't need to reinstall it.

```
/ # nginx
/ #
```

Looks like nginx got backgrounded. Uh-oh.

More importantly, if you're familiar with how processes are daemonized, you'll know that applications will frequently [fork or double-fork](http://stackoverflow.com/a/16317668), with the intention that the child process is orphaned, then adopted by the init process.

If you recall, the process we start is actually the init process. When the init process exits, *regardless of whether there are other processes running inside the container* or not, the container also exits, thus terminating all processes in the container.

So, you'll note that nothing has been started, because nginx is automatically backgrounded. If you refer to the nginx docs, you'll note the [`-g` flag](http://wiki.nginx.org/CommandLine), which allows you to [disable daemonization](http://nginx.org/en/docs/ngx_core_module.html#daemon).

Let's try that again:

```
$ docker run -d -p 80 ripta/nginx-static-test:1.0 nginx -g 'daemon off;'
e4359bc5905ee4432e38a109770efb44497aa6caf13c67d50ab90054ff781664

$ docker ps
```

You'll see docker's host port in the output of `docker ps` above. So how do we connect to it?

Normally, you would most likely know the IP address of the docker host. In our case, however, `docker-machine` provides a simple way to query the IP address of the docker host:

```
$ docker-machine ip intro
172.16.241.128
```

and if you have the container ID of the nginx container, you can retrieve the host's port that is mapped to the container port 80:

```
$ docker port e4359bc5905e 80/tcp
0.0.0.0:32769
```

To connect to the web server, use curl or open the URL in a browser, subtituting for the IP and ports above:

```
$ curl -i http://172.16.241.128:32769/
$ curl -i http://172.16.241.128:32769/VERSION
```

## Cleaning Up

You can stop your docker host using `docker-machine stop intro`, which will also stop all containers running in it.

Docker does not keep track of the state of the containers during shut down, and so it will *not* automatically restart containers when you start the docker host back up again using `docker-machine start intro`.
