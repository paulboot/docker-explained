# Docker Networking



## Docker and networking

Source: https://runnable.com/docker/basic-docker-networking

* `docker network ls`
* `docker network inspect <network-name>`

Networks can be configured to provide complete isolation for containers, which enable building web applications that work together *securely*.

## Default behavior

Docker creates three networks automatically on install: `bridge`, `none`, and `host`. Specify which network a container should use with the `--net` flag. If you create a new network `my_network` (more on this later), you can connect your container (`my_container`) with:

```
docker run my_container --net=my_network
```

### Bridge

All Docker installations represent the `docker0` network with `bridge`; Docker connects to `bridge` by default. Run `ifconfig` on the Linux host to view the `bridge` network.

When you run the following command in your console, Docker returns a JSON object describing the `bridge` network (including information regarding which containers run on the network, the options set, and listing the subnet and gateway).

```
docker network inspect bridge
```

Docker automatically creates a subnet and gateway for the `bridge` network, and `docker run` automatically adds containers to it. If you have containers running on your network, `docker network inspect` displays networking information for your containers.

Any containers on the same network may communicate with one another via IP addresses. Docker does not support automatic service discovery on `bridge`. You must connect containers with the `--link` option in your `docker run` command.

The Docker `bridge` supports port mappings and `docker run --link` allowing communications between containers on the `docker0` network. However, these error-prone techniques require unnecessary complexity. Just because you can use them, does not mean you *should*. It’s better to define your own networks instead.

### None

This offers a container-specific network stack that lacks a network interface. This container only has a local loopback interface (i.e., no external network interface).

### Host

This enables a container to attach to your host’s network (meaning the configuration *inside* the container **matches** the configuration *outside* the container).

### MACVLAN

See: https://www.youtube.com/watch?v=5grbXvV_DSk

* Here docker will assign first free address and conflict with a local DHCP server.
* Each IP address

```
docker network create -d macvlan
—subnet 192.168.0.0/24
—gateway 192.168.0.1
—ip-range 192.168.0.253/32
-o parent=enp6s0
```

### IPVLAN

See: https://www.youtube.com/watch?v=5grbXvV_DSk

* Only use one MAC address with several IPaddresses, mmm.



### Overlay network

* Used in docker-swarm, not relevant for Kubernetis



## Defining your own networks

You can create multiple networks with Docker and add containers to one or more networks. Containers can communicate within networks but not *across* networks. A container with attachments to multiple networks can connect with all of the containers on all of those networks. This lets you build a “hub” of sorts to connect to multiple networks and separate concerns.

### Creating a bridge network

Bridge networks (similar to the default `docker0` network) offer the easiest solution to creating your own Docker network. While similar, you do not simply clone the `default0` network, so you get some new features and lose some old ones. Follow along below to create your own `my_isolated_bridge_network` and run your Postgres container `my_psql_db` on that network:

```
$ docker network create --driver bridge my_isolated_bridge_network
3b7e1ad19ee8bec9628b18f9f3691adecd2ea3395ec248f8fa57a2ec85aa71c1
$ docker network inspect my_isolated_bridge_network
[
    {
        "Name": "my_isolated_bridge_network",
        "Id": "3b7e1ad19ee8bec9628b18f9f3691adecd2ea3395ec248f8fa57a2ec85aa71c1",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1/16"
                }
            ]
        },
        "Internal": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
$ docker network ls
NETWORK ID          NAME                         DRIVER
fa1ff6106123        bridge                       bridge
803369ddc1ae        host                         host
3b7e1ad19ee8        my_isolated_bridge_network   bridge
01cc882aa43b        none                         null
$ docker run --net=my_isolated_bridge_network --name=my_psql_db postgres
$ docker network inspect my_isolated_brige_network
[
    {
        "Name": "my_isolated_bridge_network",
        "Id": "3b7e1ad19ee8bec9628b18f9f3691adecd2ea3395ec248f8fa57a2ec85aa71c1",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1/16"
                }
            ]
        },
        "Internal": false,
        "Containers": {
            "b4ba8821a2fa3d602ebf2ff114b4dc4a9dbc178784dad340e78210a1318b717b": {
                "Name": "my_psql_db",
                "EndpointID": "4434c2c253afed44898aa6204a1ddd9b758ee66f7b5951d93ca2fc6dd610463c",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

Any other container you create on this network can immediately connect to any other container on this network. The network isolates containers from other (including external) networks. However, you can expose and publish container ports on the network, allowing portions of your `bridge` access to an outside network.

### Creating an overlay network

If you want native multi-host networking, you need to create an overlay network. These networks require a valid key-value store service, such as Consul, Etcd, or ZooKeeper. You must install and configure your key-value store service **before** creating your network. Your Docker hosts (you can use multiple hosts with overlay networks) must communicate with the service you choose. Each host needs to run Docker. You can provision the hosts with Docker Machine.

Open the following ports between each of your hosts:

| Protocol | Port | Purpose |
| :------- | :--- | :------ |
| udp      | 4789 | data    |
| tcp/udp  | 7946 | control |

Check your key-value store service documentation; your service may need more ports open.

Create an overlay network by configuring options on each Docker daemon you wish to use with the network. You may set the following options:

| Option                                                       | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `--cluster-store=PROVIDER://URL`                             | Describes the location of the key-value store service        |
| `--cluster-advertise=HOST_IP` or `--cluster-advertise=HOST_IFACE:PORT` | The IP address or interface corresponding to the clustering host |
| `--cluster-store-opt=KEY-VALUE OPTIONS`                      | Additional options, like a TLS certificate                   |

1. Create the overlay network in a similar manner to the bridge network (network name `my_multi_host_network`):

   ```
    docker network create --driver overlay my_multi_host_network
   ```

2. Launch containers on each host; make sure you specify the network name:

   ```
    docker run -itd -net=my_multi_host_network my_python_app
   ```

Once you connect, every container on the network has access to all the other containers on the network, *regardless of the Docker host* serving the container.

## Further information

Normally when you detach from a container, the container stops. You can leave it running by pressing `CTRL-p + CTRL-q`.

The Linux `screen` tool might become your new best friend. You can use it to start containers in other “windows” (in your terminal) and jump between the action in one window and the next. Try out these really helpful commands:

| Command             | Description                                                  |
| :------------------ | :----------------------------------------------------------- |
| `screen -S my_name` | Creates a new screen called “my_name” (so you can later reference it by name rather than ID) |
| `screen -x`         | If you have only one screen open, jumps to that screen; if you have multiple screens open, lists available screens |
| `screen -x my_name` | Switches to screen with name `my_name`                       |

### None and Host

These have specific uses for the Docker installation. Do not bind containers to these networks.

### Custom Bridge Networks

Use bridge networks when you need a relatively small network on a single host. Containers you create on your custom bridge network must exist on the same host. Docker *does not* support linking on bridge networks you define.

### Custom Overlay Networks

You must *install and configure* your key-value store service (either Consul, Etcd, or ZooKeeper) **before** creating your network.

## Docker-compose and networking

Source: https://runnable.com/docker/docker-compose-networking

Docker Compose sets up a single network for your application(s) by default, adding each container for a service to the default network. Containers on a single network can reach and discover every other container on the network.

## Networking Basics

Running the command `docker network ls` will list out your current Docker networks; it should look similar to the following:

```
$ docker network ls
NETWORK ID          NAME                         DRIVER
17cc61328fef        bridge                       bridge
098520f7fce0        composedjango_default        bridge
1ce3c572afc6        composeflask_default         bridge
8fd07d456e6c        host                         host
3b578b919641        none                         null
```

You can alter the network name with the `-p` or `--project-name` flags or the `COMPOSE_PROJECT_NAME` environment variable. (In the event you need to run multiple projects on a single host, it’s recommended to set project names via the flag.)

In our `compose_django` example, `web` can access the PostgreSQL database from `postgres://postgres:5432`. We can access `web` from the outside world via port 8000 on the Docker host (only because the `web` service explicitly maps port 8000.

## Updating Containers on the Network

You can change service configurations via the Docker Compose file. When you run `docker-compose up` to update the containers, Compose removes the old container and inserts a new one. The new container has a different IP address than the old one, but *they have the same name*. Containers with open connections to the old container close those connections, look up the new container by its name, and connect.

## Linking Containers

You may define additional aliases that services can use to reach one another. *Services on the same network can already reach one another.* In the example below, we allow `web` to reach `db` via one of two hostnames (`db` or `database`):

```
version: '2'
services:
    web:
        build: . 
        links: 
            - "db:database"
    db:
        image: postgres
```

If you do not specify a second hostname (for example, `- db` instead of `- "db:database"`), Docker Compose uses the service name (`db`). Links express dependency like `depends_on` does, meaning links dictate the order of service startup.

## Networking with Multiple Hosts

You may use the `overlay` driver when deploying Docker Compose to a Swarm cluster. We’ll cover more on Docker Swarm in a future article.

## Configuring the Default Network

If you desire, you can configure the default network instead of (or in addition to) customizing your own network. Simply define a `default` entry under `networks`:

```
verision: '2'

services:
    web:
        build: . 
        ports:
            - "8000:8000"
    db:
        image: postgres

networks:
    default:
        driver: custom-driver-1
```

## Custom Networks

Specify your own networks with the top-level `networks` key, to allow creating more complex topologies and specify network drivers (and options). You can also use this configuration to connect services with external networks Docker Compose does not manage. Each service can specify which networks to connect to with its service-level `networks` key.

The following example defines two custom networks. Keep in mind, `proxy` *cannot* connect to `db`, as they do not share a network; however, `app` can connect to both. In the `front` network, we specify the IPv4 and IPv6 addresses to use (we have to configure an `ipam` block defining the `subnet` and `gateway` configurations). We could customize either network or neither one, but we do want to use separate drivers to separate the networks (review [Basic Networking with Docker](https://runnable.com/docker/basic-docker-networking) for a refresher):

```
version: '2'

services:
    proxy:
        build: ./proxy
        networks: 
            - front
    app:
        build: ./app
        networks:
            # you may set custom IP addresses
            front:
                ipv4_address: 172.16.238.10 
                ipv6_address: "2001:3984:3989::10"
            - back
    db:
        image: postgres
        networks:
            - back

networks:
    front:
        # use the bridge driver, but enable IPv6
        driver: bridge
        driver_opts:
            com.docker.network.enable_ipv6: "true"
        ipam:
            driver: default
            config:
                - subnet: 172.16.238.0/24
                gateway: 172.16.238.1
                - subnet: "2001:3984:3989::/64"
                gateway: "2001:3984:3989::1"
    back:
        # use a custom driver, with no options
        driver: custom-driver-1
```

## Pre-Existing Networks

You can even use pre-existing networks with Docker Compose; just use the `external` option:

```
version: '2'

networks:
    default:
        external:
            name: i-already-created-this
```

In this case, Docker Compose never creates the default network; instead connecting the app’s containers to the `i-already-created-this` network.

## Common Issues

You’ll need to use version 2 of the Compose file format. (If you follow along with these tutorials, you already do.) *Legacy (version 1) Compose files **do not** support networking.* You can determine the version from the `version:` line in the `docker-compose.yml` file.

### General YAML

Use quotes (“” or ‘’) whenever you have a colon (:) in your configuration values, to avoid confusion with key-value pairs.

### Updating Containers

Container IP addresses change on update. Reference containers by name, not IP, whenever possible. Otherwise you’ll need to update the IP address you use.

### Links

If you define both links and networks, linked services must share *at least one network* to communicate.