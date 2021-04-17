# Docker Daemon on Demand with Socket Activation Explained

written on Friday, February 03, 2017 by [Danilo Bargen](https://blog.dbrgn.ch/about/)

## TL;DR

* Enable socket activation

```
$ sudo systemctl disable docker.service
$ sudo systemctl stop docker.service
$ sudo systemctl enable docker.socket
$ sudo systemctl start docker.socket
```

On my home computer, I use [Docker](https://www.docker.com/) from time to time. Since I don't need it every day,  the daemon must be started on demand. Rhe people behind [systemd](https://freedesktop.org/wiki/Software/systemd/) had some better ideas on how to solve the issue of network services that are only needed infrequently.

## Socket Activation

Systemd implements a technique called [socket activation](http://0pointer.de/blog/projects/socket-activation.html). It basically means that instead of starting a service that listens on a socket, a supervisor daemon listens on behalf of those services. When a new connection comes in that wants to talk to that socket, the service is started transparently in the background. The connection is kept alive and handed over to the service as soon as the service is up and running.

One of the downsides of this approach is that the service needs to be aware of socket activation. Luckily support for systemd socket activation was implemented in the Docker daemon [in 2014](https://github.com/docker/docker/pull/3105). (Unfortunately it's [still missing](https://plus.google.com/+ceciletonglet_intergalactic_bounty_hunter/posts/V2PF6JeuVNa) in PostgreSQL...)

## Starting Docker on Demand

Arch Linux ships with appropriate unit files for a socket activated Docker daemon.

The first requirement for socket activation to work, is that the daemon is started appropriately. For Docker, the daemon needs to be started with the `-H fd://` argument. This is being done correctly, as you can see in the service file shipped with the `docker` package:

```
# /usr/lib/systemd/system/docker.service

[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network.target docker.socket
Requires=docker.socket

[Service]
ExecStart=/usr/bin/dockerd -H fd://
# ...
```

The second component that needs to be present is the [socket unit configuration](https://www.freedesktop.org/software/systemd/man/systemd.socket.html). This is what the file looks like on Arch Linux:

```
#  /usr/lib/systemd/system/docker.socket

[Unit]
Description=Docker Socket for the API
PartOf=docker.service

[Socket]
ListenStream=/var/run/docker.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker

[Install]
WantedBy=sockets.target
```

To start Docker only on demand, we need to disable the service (to prevent it for starting on boot) and to enable the socket.

```
$ sudo systemctl disable docker.service
$ sudo systemctl stop docker.service
$ sudo systemctl enable docker.socket
$ sudo systemctl start docker.socket
```

Now if you run any command that tries to connect to the Docker daemon, the service should be started automatically!

```
$ docker info
Containers: 52
 Running: 2
 Paused: 0
 Stopped: 50
Images: 323
Server Version: 1.12.6
...
```