## Docker Network and Private ports

### Background macvlan and ipvlan

* https://docs.docker.com/network/macvlan/
* https://sreeninet.wordpress.com/2016/05/29/macvlan-and-ipvlan/
* https://sreeninet.wordpress.com/2016/05/29/docker-macvlan-and-ipvlan-network-plugins/
* https://blog.oddbit.com/post/2018-03-12-using-docker-macvlan-networks/



## Explaining the concepts

When Docker is installed a default bridge network named `docker0` is created. Each new Docker container is automatically attached to this network. Besides `docker0` , two other networks get created automatically by Docker: `host` (no isolation between host and containers on this network, to the outside world they are on the same network) and `none` (attached containers run on container-specific network stack)

Assume you have a clean Docker Host system with just 3 networks available – bridge, host and null

```
root@ubuntu:~# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
871f1f745cc4        bridge              bridge              local
113bf063604d        host                host                local
2c510f91a22d        none                null                local
root@ubuntu:~#
```

My Network Configuration is quite simple. It has eth0 and eth1 interface. I will just use eth0.

```
root@ubuntu:~# ifconfig
docker0   Link encap:Ethernet  HWaddr 02:42:7d:83:13:8e
          inet addr:172.17.0.1  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

eth0      Link encap:Ethernet  HWaddr fe:05:ce:a1:2d:5d
          inet addr:100.98.26.43  Bcast:100.98.26.255  Mask:255.255.255.0
          inet6 addr: fe80::fc05:ceff:fea1:2d5d/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:923700 errors:0 dropped:367 overruns:0 frame:0
          TX packets:56656 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:150640769 (150.6 MB)  TX bytes:5125449 (5.1 MB)
          Interrupt:31 Memory:ac000000-ac7fffff

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:45 errors:0 dropped:0 overruns:0 frame:0
          TX packets:45 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:3816 (3.8 KB)  TX bytes:3816 (3.8 KB)
```

### Creating MacVLAN network on top of eth0.

```
docker network create -d macvlan --subnet=100.98.26.43/24 --gateway=100.98.26.1  -o parent=eth0 pub_net
```

### Verifying MacVLAN network

```
root@ubuntu:~# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
871f1f745cc4        bridge              bridge              local
113bf063604d        host                host                local
2c510f91a22d        none                null                local
bed75b16aab8        pub_net             macvlan             local
root@ubuntu:~#
```

Let us create a sample Docker Image and assign statics IP (ensure that it is from free pool)

```
root@ubuntu:~# docker  run --net=pub_net --ip=100.98.26.47 -itd alpine /bin/sh
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
ff3a5c916c92: Pull complete
Digest: sha256:e1871801d30885a610511c867de0d6baca7ed4e6a2573d506bbec7fd3b03873f
Status: Downloaded newer image for alpine:latest
493a9566c31c15b1a19855f44ef914e7979b46defde55ac6ee9d7db6c9b620e0
```

**Important Point**: When using macvlan, you cannot ping or communicate with the default namespace IP address. For example, if you create a container and try to ping the Docker host’s eth0, it will not work. That traffic is explicitly filtered by the kernel modules themselves to offer additional provider isolation and security.

## Enabling Container to Host Communication

This does not seem to work on my setup because networks are overlapping.

Example: `ip link add mac0 link $PARENTDEV type macvlan mode bridge`

So, in our case, it will be:

```
ip link add mac0 link eth0 type macvlan mode bridge
ip addr add 100.98.26.38/24 dev mac0
ifconfig mac0 up
```

Let us try creating container and pinging:

```
root@ubuntu:~# docker run --net=pub_net -d --ip=100.98.26.53 -p 81:80 nginx
10146a39d7d8839b670fc5666950c0e265037105e61b0382575466cc62d34824
root@ubuntu:~# ping 100.98.26.53
PING 100.98.26.53 (100.98.26.53) 56(84) bytes of data.
64 bytes from 100.98.26.53: icmp_seq=1 ttl=64 time=1.00 ms
64 bytes from 100.98.26.53: icmp_seq=2 ttl=64 time=0.501 ms
```

### Same host access problem with Macvlan

You can attach a container directly to a local network using the [Macvlan](https://docs.docker.com/network/macvlan/) network type. Macvlan lets you create “clones” of a physical interface on your host and use that to attach containers directly to your local network. For the most part it works great, but it does come with some minor caveats and limitations.

* https://collabnix.com/2-minutes-to-docker-macvlan-networking-a-beginners-guide/

* https://stackoverflow.com/questions/49600665/docker-macvlan-network-inside-container-is-not-reaching-to-its-own-host

* https://forums.docker.com/t/macvlan-networks-unable-to-connect-to-host-from-container/51435

I run my containers with a [macvlan 4](https://docs.docker.com/network/macvlan/) network, because I wanted to be able to run multiple containers on port 80 (or other ports).

The problem I have is that I’m unable to connect to the docker host from one of the containers. As an example:

```
+---------------Host (192.168.1.100)-------------+
|                                                |
| +---(192.168.1.102)--+  +--(192.168.1.103)---+ |
| |  Container 1       |  |  Container 2       | |
| +--------------------+  +--------------------+ |
+------------------------------------------------+
```

When I try to ping the Host (192.168.1.100) from Container 1 (192.168.1.102), I don’t get a response, no packets seem to reachable from any of the containers, but other machines on the network (outside of the host) work fine.

### Macvlan limitations and additional isolation

Older versions of the Docker documentation pointed it out (is removed from newer docs):

> **Note** : In Macvlan you are not able to ping or communicate with the default namespace IP address. For example, if you create a container and try to ping the Docker host’s `eth0` it will **not** work. That traffic is explicitly filtered by the kernel modules themselves to offer additional provider isolation and security.

Source: [https://docs.docker.com/v17.06/engine/userguide/networking/get-started-macvlan/#macvlan-bridge-mode-example-usage 335](https://docs.docker.com/v17.06/engine/userguide/networking/get-started-macvlan/#macvlan-bridge-mode-example-usage)

Though, there are ways to work around the limitation:
[http://blog.oddbit.com/2018/03/12/using-docker-macvlan-networks/ 1.4k](http://blog.oddbit.com/2018/03/12/using-docker-macvlan-networks/)

### Host access

The question is "a bit old", however others might find it useful. There is a workaround described in ***Host access*** section of [USING DOCKER MACVLAN NETWORKS BY LARS KELLOGG-STEDMAN](https://blog.oddbit.com/post/2018-03-12-using-docker-macvlan-networks/). I can confirm - it's working.

> Host access With a container attached to a macvlan network, you will find that while it can contact other systems on your local network without a problem, the container will not be able to connect to your host (and your host will not be able to connect to your container). This is a limitation of macvlan interfaces: without special support from a network switch, your host is unable to send packets to its own macvlan interfaces.
>
> Fortunately, there is a workaround for this problem: you can create another macvlan interface on your host, and use that to communicate with containers on the macvlan network.
>
> First, I’m going to reserve an address from our network range for use by the host interface by using the --aux-address option to docker network create. That makes our final command line look like:
>
> ```
> docker network create -d macvlan -o parent=eno1 \
>   --subnet 192.168.1.0/24 \
>   --gateway 192.168.1.1 \
>   --ip-range 192.168.1.192/27 \
>   --aux-address 'host=192.168.1.223' \
>   mynet
> ```
>
> This will prevent Docker from assigning that address to a container.
>
> Next, we create a new macvlan interface on the host. You can call it whatever you want, but I’m calling this one mynet-shim:
>
> ```
> ip link add mynet-shim link eno1 type macvlan  mode bridge
> ```
>
> Now we need to configure the interface with the address we reserved and bring it up:
>
> ```
> ip addr add 192.168.1.223/32 dev mynet-shim
> ip link set mynet-shim up
> ```
>
> The last thing we need to do is to tell our host to use that interface when communicating with the containers. This is relatively easy because we have restricted our containers to a particular CIDR subset of the local network; we just add a route to that range like this:
>
> ```
> ip route add 192.168.1.192/27 dev mynet-shim
> ```
>
> With that route in place, your host will automatically use ths mynet-shim interface when communicating with containers on the mynet network.
>
> Note that the interface and routing configuration presented here is not persistent – you will lose if if you were to reboot your host. How to make it persistent is distribution dependent.

### Experimenting with alternative host access

These all fail,:

```
docker network create -d macvlan \
--subnet=192.168.3.0/25 \
# --ip-range=192.168.3.10/32 \
--aux-address=192.168.3.10 \
-o macvlan_mode=bridge \
-o parent=enp6s0 hostvlan3

docker network create -d macvlan \
--subnet=192.168.3.0/25 \
--ip=192.168.3.10 \
-o macvlan_mode=bridge \
-o parent=enp6s0 hostvlan3

docker network create -d macvlan --subnet=192.168.3.0/25 --gateway=192.168.3.1 --ip-range=192.168.3.10/32 -o macvlan_mode=bridge -o parent=enp6s0 hostvlan3
```

Error:

```
Error response from daemon: network dm-04ccfec647c6 is already using parent interface enp6s0
```

This seems to work. To 'fix' things at 192.168.3.0/24 half of the network is access is done using a new macvlan interface running on the same interface `enp6s0` using a  smaller subnet `/25`.

```
sudo ip link add hostvlan3 link enp6s0 type macvlan mode bridge
sudo ip addr add 192.168.3.10/25 dev hostvlan3
sudo ip link set hostvlan3 up
```





* [Connection between host and container with own ip5](https://forums.docker.com/t/connection-between-host-and-container-with-own-ip/87046/2)

## Docker Expose Container on Privileged Port

Serve an endpoint on port 80. (Also the endpoint should be a grapqhl endpoint, but this is not part of this post).

There is only one problem: to run services with ports beneath 1024, which are called privileged ports, you need root privileges. Even in a restricted virtualized environment like docker you don’t want services to run with root privileges.

But there is another way: setcap. setcap is a command in Linux systems to set specific capabilities for a file. The list of capabilities starts with the permission to control kernel auditing (CAP_AUDIT_CONTROL), permission to set uid and gid of other files like the chown command (CAP_CHOWN) and goes on with the privileges to kill other processes (CAP_KILL) and a lot more. The one which we need is CAP_NET_BIND.

We use the following Dockerfile to get one of our services, written in go, running on port 80.

```dockerfile
# Setting the default version. Can be overriden via command argument
ARG GO_VERSION=1.12

FROM golang:${GO_VERSION}-alpine AS builder

# Git is necessary to install go dependencies
# ca-certificates is needed to make calls with ssl
RUN apk add --no-cache git libcap ca-certificates
RUN update-ca-certificates 2>/dev/null || true

WORKDIR /src

# We are using the new go modules (vgo), so we need to copy both mod files and install all
# dependencies
COPY ./go.mod ./go.sum ./
RUN go mod download

# Copy the source code
COPY ./ ./

RUN CGO_ENABLED=0 go build \
    -o /app \
    -installsuffix 'static' \
    main.go

# Add the goapp system group (or whatever groupname you like)
RUN addgroup -S -g 10101 goapp
# add the goapp system user, without a password and without a login shell with the goapp group created before
RUN adduser -S -D -u 10101 -s /sbin/nologin -h /goapp -G goapp goapp
# Set owner for our source and built app to the created user and group. /src is only needed, when you need to
# copy additional files besides the app executable like json files or images
RUN chown -R goapp:goapp /app /src

# Set the privileges for our built app executable to run on privileged ports
RUN setcap 'cap_net_bind_service=+ep' /app

# Set the container back to start
FROM scratch AS final

# Copy our binary and additional stuff to scratch container
COPY --from=builder /etc/group /etc/passwd /etc/
COPY --from=builder /app /app
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

VOLUME ["/cert-cache"]

# Run on privileged port 80
EXPOSE 80

ENTRYPOINT ["/app"]

USER goapp
```

The most relevant part of the dockerfile for this post is the `**RUN** setcap 'cap_net_bind_service=+ep' /app` command. As mentioned, the capability `cap_net_bind` allows our app to run with ports beneath 1024. The flags mean: `e=activated` , `p=permitted` . You can add `i` to allow the process to also spawn child processes which open privileged ports.

Finally we need an unprivileged user which runs our app which can be accomplished by these commands:

```dockerfile
...
RUN addgroup -S -g 10101 goapp
RUN adduser -S -D -u 10101 -s /sbin/nologin -h /goapp -G goapp goapp

RUN chown -R goapp:goapp /app /src
...
COPY --from=builder /etc/group /etc/passwd /etc/
...
USER goapp
```

There are some additional commands in there which I just learned recently.

`**ARG** GO_VERSION=1.12` allows you to define a default value which can be overridden from the outside like `docker build --build-arg GO_VERSION=1.11 ...` .

The `AS` in `**FROM** golang:${**GO_VERSION**}-alpine **AS** builder` defines a new named stage in the build process. In combination with `**FROM** scratch **AS** final` and `COPY --from=builder …` it is almost like resetting the whole build to start and copy the relevant parts without all the source files etc. The resulting docker image will be a lot smaller.