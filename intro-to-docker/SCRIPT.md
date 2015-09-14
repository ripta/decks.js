### Introduction

I'll spend about fifteen minutes to cover the basics, and then jump into show-and-tell.

If you'd like to follow, you'll need access to a docker host. If you don't have anything setup, expect to download about 200 MB of data.

### Installation

On OS X, you'll need a hypervisor. If you don't have one already—i.e., VirtualBox, VMWare Fusion, or Parallels Desktop—the easiest way to go is by using Homebrew.

Install the `caskroom` extension to homebrew if you haven't already, which will allow you to install Application binaries from the command line:

```
$ brew update
$ brew install caskroom/cask/brew-cask
```

Install one of VirtualBox (recommended by *Docker, Inc*), VMWare Fusion (Ripta's preference), or Parallels Desktop:

```
$ brew cask install virtualbox
$ brew cask install vmware-fusion
$ brew cask install parallels-desktop
```

Get the `docker` client, `docker-machine`, and `docker-compose`:

```
$ brew install docker docker-machine docker-compose
```

We won't be using `docker-compose` today, but it can be used to define and run multiple containers in one.

### Create New Machine

It doesn't **yet** matter your location, but I'm issuing these commands under a project directory.

I'm going to first create a new docker host running on VMWare. I'll name it `intro`, but you can choose any name you'd like; a name that resembles a hostname would be a good candidate:

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
$ docker-machine create -h | grep -- --driver
```

It should take a minute to boot up your new docker host; longer if `docker-machine` needs to download the images.

You'll now have a set of environment variables needed to access the new docker host:

```
$ docker-machine env intro
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://172.16.241.137:2376"
export DOCKER_CERT_PATH="/Users/rpasay/.docker/machine/machines/intro"
export DOCKER_MACHINE_NAME="intro"
```

Evaluating the output of the above command will set your terminal up, after which you should be able to issue any docker command:

```
$ eval `docker-machine env intro`
```

As with git-like CLI, `docker help` should get you started. A useful command is `docker info`, which provides information about the host:

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
* number of images, which are snapshots stored on disk;
* storage driver, which determine how images are stored on disk, usually **aufs** (advanced multilayered unification filesystem); and
* the kernel version.

The version of docker client and server can be retrieved using:

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

Obviously, my client version is a little outdated. Not a problem in this case.

**One side note:** You'll probably never need to do so—and in fact, doing this could mess up your setup if you don't know what you're doing—however, you can access the docker host itself using `docker-machine ssh intro`. Of note is the amount of space available to your AUFS mount, which is used for images. There are options under `docker-machine create` to expand this size if you're building a lot of images.

### Finding Images

Official images maintained by Docker, Inc are available under the `library` username:

```
$ docker search library
```

Alternatively, open [https://hub.docker.com/r/library/](https://hub.docker.com/r/library/) in your browser. Tags show different versions of images in the repo, and also the image size.

While it may be a matter of preference on which image to use as a base, I prefer smaller, slimmer base images: you can fit more per host and it's more portable. I've found Alpine Linux to be a good balance between size and feature.

Let's use that for now:

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

Running alpine v3.2:

```
$ docker run -i -t alpine:3.2 /bin/sh
/ # 
```

where `-i` keeps STDIN open even when we're not attached to the container, and `-t` allocates a pseudo TTY.

Inside shell, the hostname of the container corresponds to the container ID:

```
/ # hostname
3747ada8f970
```

There aren't many processes running. In fact, `/bin/sh` is the init process (PID 1):

```
/ # ps -ef
PID   USER     TIME   COMMAND
    1 root       0:00 /bin/sh
   11 root       0:00 ps -ef
```

This is important to note, because:

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

View routing information:

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

Alpine Linux comes with a lightweight package manager: apk.

### Listing Containers

`docker ps` lists all running containers. When I exit the running container, it can still be queried with `-a`:

```
$ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                          PORTS               NAMES
3747ada8f970        alpine:3.2          "/bin/sh"           23 minutes ago      Exited (1) About a minute ago                       sick_einstein
```

Image and container IDs go hand-in-hand, but are different. Both are 128-bit (32-character) identifier.

### Restarting Containers

```
$ docker restart 3747ada8f970
3747ada8f970

$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
3747ada8f970        alpine:3.2          "/bin/sh"           34 minutes ago      Up 2 seconds                            sick_einstein

$ docker attach 3747ada8f970
/ # %

$ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                     PORTS               NAMES
3747ada8f970        alpine:3.2          "/bin/sh"           35 minutes ago      Exited (0) 3 seconds ago                       sick_einstein

$ docker rm 3747ada8f970
```

### Detached Containers

Because containers exit when the init process exits, the init process would need to be foregrounded:

```
$ docker run alpine:3.2 /bin/sh -c 'while true; do echo "Hello World!"; sleep 1; done'
```

However, doing so keeps a terminal occupied. The answer would be to detach the container with `-d`:

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

Stopping a container (`docker stop 032b0875083e`) sends a SIGTERM, while (`docker kill 032b0875083e`) sends a SIGKILL.

### Removing Containers

`docker ps -a -q | xargs docker rm`

### Sample Application

```
$ cd ~/Projects/dockerfell/nginx-static-test
$ docker build -t ripta/nginx-static-test:1.0 .
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

Try running nginx:

```
$ docker run -d -p 80 ripta/nginx-static-test:1.0 nginx
$ docker ps
```

Something is wrong, because the container isn't executing. `docker ps -n 1` will show that the container exited. What went wrong?

```
$ docker run -i -t ripta/nginx-static-test:1.0 /bin/sh
/ # nginx
/ #
```

Looks like nginx got backgrounded. Uh-oh. More importantly, if you're familiar with how processes are daemonized, you'll know that applications will frequently [fork or double-fork](http://stackoverflow.com/a/16317668), with the intention that the child process is orphaned, then adopted by the init process.

If you recall, the process we start is actually the init process. When it exits, the container exits.

So, you'll note that nothing has been started, because nginx is automatically backgrounded. If you refer to the nginx docs, you'll note the `-g` flag:

```
$ docker run -d -p 80 ripta/nginx-static-test:1.0 nginx -g 'daemon off;'
$ docker ps
```

You'll see docker's host port in the output of `docker ps` above. So how do we query it?

```
$ docker-machine ip intro
$ curl -i http://172.16.241.128:32773/
$ curl -i http://172.16.241.128:32773/VERSION
```
