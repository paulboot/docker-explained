# Docker names hostname net-alias Explained

[toc]

## Docker vs Docker-compose and DNS

Docker-compose defines a service and docker defines names. If you connect containers to the same network, this service name is defined in DNS and must be unique. The internal autoconfig DNS that runs on `127.0.0.11` knows how to resolve the service and names which is confusing.

See: the `--name` resolves:

```
dig @127.0.0.11 devweerbootnl_dev_weerboot_php_1

; <<>> DiG 9.16.6-Ubuntu <<>> @127.0.0.11 devweerbootnl_dev_weerboot_php_1
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 41092
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;devweerbootnl_dev_weerboot_php_1. IN   A

;; ANSWER SECTION:
devweerbootnl_dev_weerboot_php_1. 600 IN A      192.168.20.5
```

The shorter `service` name from the docker-file also resolves there is an second A record:


```
^Croot@9d24514301e6:/# dig @127.0.0.11 dev_weerboot_php

; <<>> DiG 9.16.6-Ubuntu <<>> @127.0.0.11 dev_weerboot_php
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 25434
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;dev_weerboot_php.              IN      A

;; ANSWER SECTION:
dev_weerboot_php.       600     IN      A       192.168.20.5
```

The `dig -x` reverse lookup:

```
root@9d24514301e6:/# dig @127.0.0.11 -x 192.168.20.5

; <<>> DiG 9.16.6-Ubuntu <<>> @127.0.0.11 -x 192.168.20.5
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 36272
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;5.20.168.192.in-addr.arpa.     IN      PTR

;; ANSWER SECTION:
5.20.168.192.in-addr.arpa. 600  IN      PTR     devweerbootnl_dev_weerboot_php_1.external_dev_vlan20.
```

### Docker inspect

I think docker inspect is the only place to see what docker-compose creates a container name and alias DNS entries:

```
docker inspect devweerbootnl_dev_weerboot_php_1
```

It labels the container: with the `service` entry: `"com.docker.compose.service": "dev_weerboot_php"`

```
            "Labels": {
                "com.docker.compose.config-hash": "19a8ae54cefc6b4423248ac41aecd630f53fb2d662a0ee3437d52b505713edf8",
                "com.docker.compose.container-number": "1",
                "com.docker.compose.oneoff": "False",
                "com.docker.compose.project": "devweerbootnl",
                "com.docker.compose.project.config_files": "docker-compose.yml",
                "com.docker.compose.project.working_dir": "/home/paulb/.docker/weerboot.nl/dev.weerboot.nl",
                "com.docker.compose.service": "dev_weerboot_php",
                "com.docker.compose.version": "1.27.4"

```

And in network it creates an alias: `"dev_weerboot_php"`

```
            "Networks": {
                "external_dev_vlan20": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": [
                        "dev_weerboot_php",
                        "ed2a4a0e803f"
                    ],
                    "NetworkID": "b232d010d65b87c53252b391d34df1fb3d04c03e62e55c4269b253e0f7ff03e8",
                    "EndpointID": "f2d194f86c1eb038266513fcd45efb06d92fce94b48acaa8723f28051851f22a",
                    "Gateway": "192.168.20.1",
                    "IPAddress": "192.168.20.5",
                    "IPPrefixLen": 24,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:c0:a8:14:05",
                    "DriverOpts": null
                }
```



## The difference between --name and --hostname

When we use `docker run` command docker creates a container and assigns a Container Id of type `UUID` to it. Now this Container Id can be used to refer to the container created. But remembering this Container Id can be difficult.

So we can use `--name` in docker run command. Now you can either use the Container Id to refer to the container created or can use the container name for the same.

Similarly when docker container is created the hostname defaults to be the container’s ID in Docker. You can override the hostname using `--hostname`. I have taken this from [Docker docs](https://docs.docker.com/config/containers/container-networking/#dns-services).

Now consider a scenario where you are making use of docker containers through code and you want to refer to docker. Since docker id is generated at the time of creation you can't know it in advance so you can use --name. To know when to use --hostname in docker run read from [this stackoverflow post](https://stackoverflow.com/a/43033828/6407858)

The `--hostname` flag only changes the hostname inside your container. This may be needed if your application expects a specific value for the hostname. It does not change DNS outside of docker, nor does it change the networking isolation, so it will not allow others to connect to the container with that name.

You can use the container `--name` or the container's (short, 12 character) id to connect from container to container with docker's embedded dns as long as you have both containers on the same network and that network is not the default bridge.

### --net-alias

If you need to change the hostname in a way that other containers from the same network will see it, just use `--net-alias=${MY_NEW_DNS_NAME}`

For example:

```
docker run -d --net-alias=${MY_NEW_DNS_NAME} --net=my-test-env --name=my-docker-name-test <dokcer-contanier>
```

### --link and --alias

Please see: [Difference between --link and --alias in overlay docker network?](https://stackoverflow.com/questions/36048897/difference-between-link-and-alias-in-overlay-docker-network)

>  **NOTE:** Do NOT use `--link` anymore!

To get containers to talk to each other,

1. Create a non default network:

```
docker network create MyNetwork
```

1. Connect containers to this network at run time:

```
docker run --network MyNetwork --name Container1 Image1
docker run --network MyNetwork --name Container2 Image2
```

Now, if `Container1` is for example a web server running on port 80, Processes inside `Container2` will be able to resolve it using a host name of `Container1` and port 80

Further if Container1 is set up like this:

```
docker run --network MyNetwork --name Container1 -p 8080:80 Image1
```

Then

* Container1 can see Container2:80
* the Host can see 127.0.0.1:8080