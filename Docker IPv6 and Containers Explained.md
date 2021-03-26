# IPv6 and Containers Explained

**How to enable IPv6 for Docker containers on Ubuntu 18.04**

**Source:** https://medium.com/@skleeschulte/how-to-enable-ipv6-for-docker-containers-on-ubuntu-18-04-c68394a219a2

In this article I will describe the steps necessary to enable IPv6 connectivity for Docker containers on an Ubuntu 18.04 host. This is based on my findings for a standalone Docker 19.03 installation on a virtual server. It should be applicable to similar setups and covers the following topics:

- Enabling IPv6 on the Docker host
- Enabling IPv6 in the Docker daemon
- Accessing containers over IPv6

# Enabling IPv6 on the Docker host

Enabling IPv6 on the host is obviously only necessary if it is not already working. An easy way to check this is to try to send IPv6 ping messages from and to the machine. To do this from an Ubuntu host use [ping6](http://manpages.ubuntu.com/manpages/bionic/man1/ping6.1.html), for example to google.com:

```
ping6 google.com -n -c 4
```

The result should look similar to this:

```
PING google.com(2a00:1450:4001:81d::200e) 56 data bytes
64 bytes from 2a00:1450:4001:81d::200e: icmp_seq=1 ttl=57 time=3.86 ms
64 bytes from 2a00:1450:4001:81d::200e: icmp_seq=2 ttl=57 time=3.88 ms
64 bytes from 2a00:1450:4001:81d::200e: icmp_seq=3 ttl=57 time=3.88 ms
64 bytes from 2a00:1450:4001:81d::200e: icmp_seq=4 ttl=57 time=3.78 ms

--- google.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3003ms
rtt min/avg/max/mdev = 3.787/3.856/3.889/0.086 ms
```

To check if the host responds to ping messages from the internet, you need to obtain its public IPv6 address, e.g. by running `ip addr show`, locating the default ethernet interface’s section and reading the IPv6 address with *scope global*. To identify the default ethernet interface, find the *default* line in the output of `ip -6 route show`. Or, to do it all in one command:

```
ip addr show $(ip -6 route show | awk '/^default/ {print $5}') | awk '/inet6 .* scope global/ {print $2}'
```

The easiest way to ping the host’s public IPv6 address probably is to [use](https://www.ipaddressguide.com/ping6) [some](https://centralops.net/co/Ping.aspx) [online](https://www.subnetonline.com/pages/ipv6-network-tools/online-ipv6-ping.php) [service](http://www.ipv6now.com.au/pingme.php). Alternatively you can do it from your local computer if it has IPv6 connectivity. On Windows use the `ping` command, on Linux use `ping6`.

If — like me — you have a server where IPv6 first needs to be manually configured, you can execute the following steps.

## Manually configure IPv6 in Netplan

By default, Ubuntu 18.04 uses [Netplan](https://netplan.io/) for the network configuration. To enable IPv6 connectivity in Netplan, add IPv6 address and gateway to the .yaml-file in the /etc/netplan directory. I have seen this file being named 01-netcfg.yaml or 50-cloud-init.yaml. Here is an example of the altered file:

<iframe src="https://medium.com/media/657f85b8f51dfe56d072101b6e942304" allowfullscreen="" frameborder="0" height="259" width="680" title="Example Netplan config file for IPv6 on Ubuntu 18.04" class="fj es eo ig w" scrolling="auto" style="box-sizing: inherit; width: 680px; left: 0px; top: 0px; position: absolute; height: 259px;"></iframe>

/etc/netplan/01-netcfg.yaml

Be careful when editing this file, it is in [YAML format](https://yaml.org/) and uses 2 or 4 spaces for indentation — no tabs! Misformatting this file will probably break the machine’s network connection at the next reboot. You should also ensure that the nameservers can resolve IPv6 queries. Then apply the new configuration:

```
netplan apply
```

That’s it — now test again if IPv6 connections are working on the host.

# Enabling IPv6 in the Docker daemon

The next step is to enable IPv6 in the Docker daemon. Looking at the [documentation for the current Docker version 19.03](https://docs.docker.com/config/daemon/ipv6/) is not very helpful: It only states to add `{ "ipv6": true }` to /etc/docker/daemon.json and `systemctl reload docker`. While reloading succeeds, it does not seem to actually enable IPv6 for Docker’s default “bridge” network (if you want to try it, look at the output of `docker network inspect bridge`). On the other hand restarting Docker with `systemctl restart docker` now entirely fails, giving the following message in /var/log/syslog:

> failed to start daemon: Error initializing network controller: Error creating default “bridge” network: could not find an available, non-overlapping IPv6 address pool among the defaults to assign to the network

Useful information can be found in the issues [moby/moby#29443](https://github.com/moby/moby/issues/29443#issuecomment-267401488) and [moby/moby#36954](https://github.com/moby/moby/issues/36954) on GitHub: In addition to setting the `ipv6`-flag in daemon.json, an IPv6 subnet must be defined in `fixed-cidr-v6`.

The subnet configured in `fixed-cidr-v6` is used to assign IPv6 addresses to containers that are connected to Docker’s default “bridge” network. It should have a prefix length of at most 80 (i.e. a size of at least 80), which is explained in [the documentation for Docker 17.09](https://docs.docker.com/v17.09/engine/userguide/networking/default_network/ipv6/#how-ipv6-works-on-docker):

> The subnet for Docker containers should at least have a size of `/80`, so that an IPv6 address can end with the container’s MAC address and you prevent NDP neighbor cache invalidation issues in the Docker layer.

Note that this prefix length requirement is specific to the default “bridge” network, induced by its MAC address based IP address generation. In manually added IPv6 enabled Docker bridge networks, IPv6 addresses are assigned sequentially so that smaller subnets are sufficient here (e.g. /112).

Next arises the question which subnet to specify in `fixed-cidr-v6`. This can either be a subnet of publicly routable addresses or a subnet defining a private address space. Using publicly routable addresses has the advantage that no further configuration is necessary on the host to enable IPv6 internet connectivity in the containers. However, this comes with a huge drawback: all containers are potentially fully accessible from the internet, regardless of which container ports are published (or not) in Docker. In addition, it requires that you have a public subnet of at least size /80 available. For this reason my recommendation is to always set `fixed-cidr-v6` to a private subnet, e.g. `fd00::/80`.

## Configuring Docker

To enable IPv6 in Docker, first create the file /etc/docker/daemon.json with the following content (or if it already exists, add the values):

<iframe src="https://medium.com/media/dd50782b404ec9868d0cb09e086ef826" allowfullscreen="" frameborder="0" height="127" width="680" title="Example Docker config file for IPv6 on Ubuntu 18.04" class="fj es eo ig w" scrolling="auto" style="box-sizing: inherit; width: 680px; left: 0px; top: 0px; position: absolute; height: 126.984px;"></iframe>

/etc/docker/daemon.json

Then restart Docker:

```
systemctl restart docker
```

## Configuring IPv6 NAT

To enable IPv6 internet access from containers, enable NAT for the private Docker subnet on the host:

```
ip6tables -t nat -A POSTROUTING -s fd00::/80 ! -o docker0 -j MASQUERADE
```

Note that this configuration does not survive a reboot of the host, so you might want to [persist it](https://askubuntu.com/a/1072948/1041210) once you confirmed it is working as expected. If, on the other hand, you decide to use docker-ipv6nat (see below), it will automatically add this ip6tables rule.

## Testing IPv6 connectivity in containers

To test if IPv6 internet connectivity is working correctly you can again run a ping command, this time from a container:

```
docker run --rm -t busybox ping6 -c 4 google.com
```

This starts a busybox container and runs ping6 targeting google.com awaiting up to 4 responses. The result should look similar to this:

```
PING google.com (2a00:1450:4001:809::200e): 56 data bytes
64 bytes from 2a00:1450:4001:809::200e: seq=0 ttl=53 time=19.234 ms
64 bytes from 2a00:1450:4001:809::200e: seq=1 ttl=53 time=19.611 ms
64 bytes from 2a00:1450:4001:809::200e: seq=2 ttl=53 time=14.860 ms
64 bytes from 2a00:1450:4001:809::200e: seq=3 ttl=53 time=12.769 ms--- google.com ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 12.769/16.618/19.611 ms
```

# Accessing containers over IPv6

Once IPv6 is enabled in Docker, its default behaviour is to expose published container ports on the host over both, IPv4 and IPv6. However, on a Linux host like Ubuntu it uses different techniques for the two protocol versions: for IPv4 it uses iptables rules, whereas for IPv6 it uses a proxy server — the Docker userland-proxy. To see this we can fire up a minimal webserver:

```
docker run --rm -d -p 8080:80 --name webtest busybox sh -c 'echo "Hello world!" > index.html && httpd -f -v'
```

This starts a Docker container named *webtest* running [httpd](https://busybox.net/downloads/BusyBox.html#httpd) serving a “Hello world!” index file at port 8080 of the Docker host.

Now list the processes currently listening for TCP connections on the host with [netstat](http://manpages.ubuntu.com/manpages/bionic/man8/netstat.8.html):

```
netstat -plnt
```

The output will show the userland-proxy (docker-proxy) listening on port 8080 for IPv6 connections (tcp6), but not for IPv4 connections. On my machine the output looks like this:

<iframe src="https://medium.com/media/f6500e177317983057dc2a3c6840ea77" allowfullscreen="" frameborder="0" height="166" width="680" title="Sample output of &quot;netstat -plnt&quot; showing docker-proxy listening only for IPv6 connections on TCP" class="fj es eo ig w" scrolling="auto" style="box-sizing: inherit; width: 680px; left: 0px; top: 0px; position: absolute; height: 166px;"></iframe>

The userland-proxy has a major downside: it hides the actual remote address from the container. So for IPv6 requests, the container only sees the proxy’s address. This can also be observed in the log of the *webtest* container. If you want to try it yourself, navigate to the website it serves — first using the Docker host’s IPv4 address:

![img](https://miro.medium.com/max/60/1*RcKZS0fx3KChD-5i9h8ZnA.png?q=20)

![img](https://miro.medium.com/max/541/1*RcKZS0fx3KChD-5i9h8ZnA.png)

And second using the Docker host’s IPv6 address:

![img](https://miro.medium.com/max/60/1*j_ZOyQvHYS2622VE8TN-iw.png?q=20)

![img](https://miro.medium.com/max/541/1*j_ZOyQvHYS2622VE8TN-iw.png)

Now get httpd’s log output from the container with:

```
docker logs webtest
```

This will reveal a log similar to this one:

```
[::ffff:198.51.100.34]:54333: response:200
[::ffff:198.51.100.34]:54334: response:404
[::ffff:172.17.0.1]:54276: response:200
[::ffff:172.17.0.1]:54278: response:404
```

The first two lines contain the actual remote IPv4 address, while the second two lines show the IPv4 address of the userland-proxy. Side note: There are two log entries for each access because in additon to the content Firefox tries to load a [favicon](https://en.wikipedia.org/wiki/Favicon) which httpd cannot find and thus logs a 404 Not Found status error.

Depending on the use case, it might not matter which remote address containers see. However, it might still be desirable to disable the userland-proxy because of other issues related to it (see [moby/moby#11185](https://github.com/moby/moby/issues/11185) and [moby/moby#14856](https://github.com/moby/moby/issues/14856)).

## Publishing ports with ip6tables

Instead of relying on the userland-proxy, container ports can be published using ip6tables rules. Unfortunately, Docker does not natively support the management of ip6tables. Manual configuration of ip6tables can be cumbersome and error-prone. This is where [docker-ipv6nat](https://github.com/robbertkl/docker-ipv6nat) steps in:

> This project mimics the way Docker does NAT for IPv4 and applies it to IPv6.

I recommend to consult the projects [documentation](https://github.com/robbertkl/docker-ipv6nat#usage). The essence is: run the following command and you’re all set:

```
docker run -d --restart=always -v /var/run/docker.sock:/var/run/docker.sock:ro --cap-drop=ALL --cap-add=NET_RAW --cap-add=NET_ADMIN --cap-add=SYS_MODULE --net=host --name ipv6nat robbertkl/ipv6nat
```

The ip6tables rules are updated immediately, which can be verified with the commands `ip6tables -S` and `ip6tables -t nat -S`. Navigating back to the *webtest* container’s website using the hosts IPv6 address and doing a force-reload (Ctrl+F5 on Windows) will add new entries to httpd’s log (`docker logs webtest`), confirming the container now sees the actual remote address when serving over IPv6:

```
[2001:db8:1081:9b1c:9d1e:f4c8:ea75:546]:55101: response:200
[2001:db8:1081:9b1c:9d1e:f4c8:ea75:546]:55102: response:404
```

It is not strictly necessary to disable the userland proxy, but as it is superfluous when using docker-ipv6nat I still recommend to do so: add `"userland-proxy": false` to /etc/docker/daemon.json and run `systemctl restart docker`.

## Exposing containers to public IPv6 access

With IPv6 eliminating the constrains imposed by the limited number of public IPv4 addresses, it gets more feasible to publish containerized microservices using distinct public (IPv6) addresses.

On Docker’s default “bridge” network, IPv6 addresses are assigned based on the containers MAC addresses, which in turn depend on the order in which the containers start. This makes it hard to impossible to assign fixed IPv6 addresses to containers in this network. In additional IPv6 enabled bridge networks, the addresses are assigned sequentially and containers can join these networks using a predefined address.

By default, Docker uses /16 sized IPv4 subnets for additional bridge networks, providing 65534 distinct IPv4 addresses. The equivalent size in IPv6 is a /112 subnet. Supposed you have a publicly routable subnet available at your Docker host (e.g. 2001:db8:1:1::/64), you could create an additional Docker bridge network for “exposed hosts”:

```
docker network create --ipv6 --subnet=2001:db8:1:1::/112 ipv6_exposed_hosts
```

To join a container to this network with a predefined IPv6 address, run:

```
docker network connect --ip6 2001:db8:1:1::ff01 ipv6_exposed_hosts [containername]
```

Note that the address …::ff01 is from the second half of the networks subnet. This is a strategy to avoid collisions with containers that are joined without explicitly specifying an IPv6 address and thus getting it sequentially from the beginning of the range. Being fixed to the container, the IPv6 address can also reliably be used in an AAAA DNS record.

By enabling Docker’s IPv6 functionality, [forwarding of IPv6 packets gets automatically enabled in the host’s kernel](https://docs.docker.com/v17.09/engine/userguide/networking/default_network/ipv6/#how-ipv6-works-on-docker). However, the containers are still not directly reachable from the internet. The reason is that the network router cannot find out how to forward IPv6 packets to the Docker containers. To change this, the host must be configured to proxy [NDP](https://en.wikipedia.org/wiki/Neighbor_Discovery_Protocol) (Neighbor Discovery Protocol) messages to the containers.

One way to achieve this is to enable NDP proxying in the kernel with the command `sysctl net.ipv6.conf.ens3.proxy_ndp=1`(replace *ens3* with the name of the hosts default ethernet interface). More information about this can be found in the [documentation for Docker 17.09](https://docs.docker.com/v17.09/engine/userguide/networking/default_network/ipv6/#using-ndp-proxying). With this approach every IPv6 address needs to be added to the NDP proxy table separately with commands like `ip -6 neigh add proxy 2001:db8:1:1::ff01 dev ens3`. It is not possible to add a complete subnet this way.

## Setting up the NDP Proxy Daemon

An alternative to manually setting up NDP proxying on the host is to install the [NDP Proxy Daemon](https://github.com/DanielAdolfsson/ndppd) (ndppd). It can be configured to proxy NDP messages to a subnet and can automatically resolve the proxy target interface from the system’s IPv6 routing table.

To **install ndppd**, run:

```
apt-get update -y
apt-get install -y ndppd
```

Then **copy the example config file** to /etc/ndppd.conf:

```
cp /usr/share/doc/ndppd/ndppd.conf-dist /etc/ndppd.conf
```

Now **edit /etc/ndppd.conf**:

- Change the line `proxy eth0 {` to contain your hosts ethernet interface, e.g.: `proxy ens3 {`.
- Change the line `rule 1111:: {` to contain the subnet you chose for the exposed hosts network, e.g. `rule 2001:db8:1:1::/112 {`.

Finally **restart ndppd**:

```
systemctl restart ndppd
```

At this point, all IPv6 capable containers that are connected to the “ipv6_exposed_hosts” network should be reachable over IPv6 from the internet.