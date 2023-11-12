# Docker Traefik Reverse Proxy Explained

[toc]

# TL;DR

* > **ANGRY:** "error while parsing rule Host('dashboard.bocuse.nl'): 1:6: illegal rune literal"
  >
  > Yes, it really means you used the wrong ticks 'bla' 
  >
  > ```
  > Host('dashboard.bocuse.nl')
  > ```
  >
  > It rather should be the ``
  >
  > ```
  > Host(`dashboard.bocuse.nl`)
  > ```

* Let's Encrypt, provides Domain Validation (DV) certificates.

  * It issues certificates for specific domains in exchange for proof of ownership for those domains.

  * To make this exchange, domains rely on the [ACME protocol](https://en.wikipedia.org/wiki/Automatic_Certificate_Management_Environment). 

  * The ACME protocol defines three challenges that must be resolved to obtain a certificate. 

    * **HTTP:** Proving ownership by serving a file containing a specific token on the specific domain (on a custom path).

    * **TLS:** Proving ownership by handling the connection to the specific domain with a temporary TLS certificate containing a specific token.

    * **DNS:** Proving ownership by creating a TXT record containing a specific token that would be then publicly available and verified by Let’s Encrypt.

  * Let’s Encrypt communicates the specific token to you after requesting a challenge in all three challenges.



**Source** 

* [DO How To Use Traefik v2 as a Reverse Proxy for Docker Containers on Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-use-traefik-v2-as-a-reverse-proxy-for-docker-containers-on-ubuntu-20-04)
*  [The official Traefik documentation](https://doc.traefik.io/traefik/)
*  https://matthiasnoback.nl/2018/03/defining-multiple-similar-services-with-docker-compose/



# Docker compose example

```
version: '3.8'

services:
    web:
        restart: always
        networks:
            - external_dev_vlan20
            #    ipv4_address: 192.168.20.10
            - net_192_168_3
        dns: 192.168.100.1
        expose:
            - "80"
        volumes:
            - ./site.conf:/etc/nginx/conf.d/default.conf
        image: nginx:1.25.3
        labels:
            - "traefik.docker.network=external_dev_vlan20"
            - "traefik.enable=true"
            - "traefik.http.services.dashboard_bocuse_nl.loadbalancer.server.port=80"
            # HTTP
            - "traefik.http.routers.dashboard_bocuse_nl_80.entrypoints=web"
            - "traefik.http.routers.dashboard_bocuse_nl_80.rule=Host(`dashboard.bocuse.nl`) || Host(`dashboard.weerboot.nl`)"
            - "traefik.http.routers.dashboard_bocuse_nl_80.middlewares=redirect-dashboard_bocuse_nl_443"
            # HTTPS
            - "traefik.http.routers.dashboard_bocuse_nl_443.entrypoints=websecure"
            - "traefik.http.routers.dashboard_bocuse_nl_443.rule=Host(`dashboard.bocuse.nl`) || Host(`dashboard.weerboot.nl`)"
            - "traefik.http.routers.dashboard_bocuse_nl_443.tls=true"
            - "traefik.http.routers.dashboard_bocuse_nl_443.tls.certresolver=lets-encrypt"
            - "traefik.http.middlewares.redirect-dashboard_bocuse_nl_443.redirectscheme.scheme=https"
            - "traefik.http.middlewares.redirect-dashboard_bocuse_nl_443.redirectscheme.port=443"
        logging:
            driver: local
            options:
                max-size: "100m"
                max-file: "3"

networks:
    external_dev_vlan20:
        external: true
    net_192_168_3:
        external: true

```

### Introduction

[Docker](https://www.docker.com/) can be an efficient way to run web applications in production, but you may want to run multiple applications on the same Docker host. In this situation, you’ll need to set up a reverse proxy. This is because you only want to expose ports `80` and `443` to the rest of the world.

[Traefik](https://traefik.io/) is a Docker-aware reverse proxy that includes a monitoring dashboard. Traefik v1 has been widely used for a while, and [you can follow this earlier tutorial to install Traefik v1](https://www.digitalocean.com/community/tutorials/how-to-use-traefik-as-a-reverse-proxy-for-docker-containers-on-ubuntu-20-04)). But in this tutorial, you’ll install and configure Traefik v2, which includes quite a few differences.

The biggest difference between Traefik v1 and v2 is that *frontends* and *backends* were removed and their combined functionality spread out across *routers*, *middlewares*, and *services*. Previously a backend did the job of making modifications to requests and getting that request to whatever was supposed to handle it. Traefik v2 provides more separation of concerns by introducing middlewares that can modify requests before sending them to a service. Middlewares make it easier to specify a single modification step that might be used by a lot of different routes so that they can be reused (such as HTTP Basic Auth, which you’ll see later). A router can also use many different middlewares.

In this tutorial you’ll configure Traefik v2 to route requests to two different web application containers: a [Wordpress](http://wordpress.org/) container and an [Adminer](https://www.adminer.org/) container, each talking to a [MySQL](https://www.mysql.com/) database. You’ll configure Traefik to serve everything over HTTPS using [Let’s Encrypt](https://letsencrypt.org/).



## Prerequisites

To complete this tutorial, you will need the following:

* Docker installed on your server, which you can accomplish by following **Steps 1 and 2** of [How to Install and Use Docker on Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04).
* Docker Compose installed using the instructions from **Step 1** of [How to Install Docker Compose on Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-compose-on-ubuntu-20-04).
* A domain and three A records, `db-admin.your_domain`, `blog.your_domain` and `monitor.your_domain`. Each should point to the IP address of your server. 



## Step 1 - Configuring and Running Traefik

The Traefik project has an [official Docker image](https://hub.docker.com/_/traefik), so you will use that to run Traefik in a Docker container.

But before you get your Traefik container up and running, you need to create a configuration file and set up an encrypted password so you can access the monitoring dashboard.

You’ll use the `htpasswd` utility to create this encrypted password. First, install the utility, which is included in the `apache2-utils` package:

```
sudo apt-get install apache2-utils
```

Then generate the password with `htpasswd`. Substitute `secure_password` with the password you’d like to use for the Traefik admin user:

```
htpasswd -nb admin secure_password
```

The output from the program will look like this:

```
Outputadmin:$apr1$ruca84Hq$mbjdMZBAG.KWn7vfN/SNK/
```

You’ll use this output in the Traefik configuration file to set up HTTP Basic Authentication for the Traefik health check and monitoring dashboard. Copy the entire output line so you can paste it later.

To configure the Traefik server, you’ll create two new configuration files called `traefik.toml` and `traefik_dynamic.toml` using the TOML format. [TOML](https://github.com/toml-lang/toml) is a configuration language similar to INI files, but standardized. [These files let us configure the Traefik server and various integrations](https://docs.traefik.io/providers/overview/), or `providers`, that you want to use. In this tutorial, you will use three of Traefik’s available providers: `api`, `docker`, and `acme`. The last of these, `acme`, supports TLS certificates using Let’s Encrypt.

Create and open `traefik.toml` using `nano` or your preferred text editor:

```
nano traefik.toml
```

First, you want to specify the ports that Traefik should listen on using the `entryPoints` section of your config file. You want two because you want to listen on port `80` and `443`. Let’s call these `web` (port `80`) and `websecure` (port `443`).

Add the following configurations:

traefik.toml

```toml
[entryPoints]
  [entryPoints.web]
    address = ":80"
    [entryPoints.web.http.redirections.entryPoint]
      to = "websecure"
      scheme = "https"

  [entryPoints.websecure]
    address = ":443"
```

Note that you are also automatically redirecting traffic to be handled over TLS.

Next, configure the Traefik `api`, which gives you access to both the API and your dashboard interface. The heading of `[api]` is all that you need because the dashboard is then enabled by default, but you’ll be explicit for the time being.

Add the following code:

`traefik.toml`

```toml
...
[api]
  dashboard = true
```

To finish securing your web requests you want to use Let’s Encrypt to generate valid TLS certificates. Traefik v2 supports Let’s Encrypt out of the box and you can configure it by creating a *certificates resolver* of the type `acme`.

Let’s configure your certificates resolver now using the name `lets-encrypt`:

`traefik.toml`

```toml
...
[certificatesResolvers.lets-encrypt.acme]
  email = "your_email@your_domain"
  storage = "acme.json"
  [certificatesResolvers.lets-encrypt.acme.tlsChallenge]
```

This section is called `acme` because [ACME](https://github.com/ietf-wg-acme/acme/) is the name of the protocol used to communicate with Let’s Encrypt to manage certificates. The Let’s Encrypt service requires registration with a valid email address, so to have Traefik generate certificates for your hosts, set the `email` key to your email address. You then specify that you will store the information that you will receive from Let’s Encrypt in a JSON file called `acme.json`.

The `acme.tlsChallenge` section allows us to specify how Let’s Encrypt can verify that the certificate. You’re configuring it to serve a file as part of the challenge over port `443`.

Finally, you need to configure Traefik to work with Docker.

Add the following configurations:

```toml
...
[providers.docker]
  watch = true
  network = "web"
```

The `docker` provider enables Traefik to act as a proxy in front of Docker containers. You’ve configured the provider to `watch` for new containers on the `web` network, which you’ll create soon.

Our final configuration uses the `file` provider. With Traefik v2, static and dynamic configurations can’t be mixed and matched. To get around this, you will use `traefik.toml` to define your static configurations and then keep your dynamic configurations in another file, which you will call `traefik_dynamic.toml`. Here you are using the `file` provider to tell Traefik that it should read in dynamic configurations from a different file.

Add the following `file` provider:

```
[providers.file]
  filename = "traefik_dynamic.toml"
```

Your completed `traefik.toml` will look like this:

```toml
[entryPoints]
  [entryPoints.web]
    address = ":80"
    [entryPoints.web.http.redirections.entryPoint]
      to = "websecure"
      scheme = "https"

  [entryPoints.websecure]
    address = ":443"

[api]
  dashboard = true

[certificatesResolvers.lets-encrypt.acme]
  email = "your_email@your_domain"
  storage = "acme.json"
  [certificatesResolvers.lets-encrypt.acme.tlsChallenge]

[providers.docker]
  watch = true
  network = "web"

[providers.file]
  filename = "traefik_dynamic.toml"
```

Copy

Save and close the file.

Now let’s create `traefik_dynamic.toml`.

The dynamic configuration values that you need to keep in their own file are the *middlewares* and the *routers*. To put your dashboard behind a password you need to customize the API’s *router* and configure a *middleware* to handle HTTP basic authentication. Let’s start by setting up the middleware.

The middleware is configured on a per-protocol basis and since you’re working with HTTP you’ll specify it as a section chained off of `http.middlewares`. Next comes the name of your middleware so that you can reference it later, followed by the type of middleware that it is, which will be `basicAuth` in this case. Let’s call your middleware `simpleAuth`.

Create and open a new file called `traefik_dynamic.toml`:

```
nano traefik_dynamic.toml
```

Add the following code. This is where you’ll paste the output from the `htpasswd` command:

traefik_dynamic.toml

```toml
[http.middlewares.simpleAuth.basicAuth]
  users = [
    "admin:$apr1$ruca84Hq$mbjdMZBAG.KWn7vfN/SNK/"
  ]
```

To configure the router for the api you’ll once again be chaining off of the protocol name, but instead of using `http.middlewares`, you’ll use `http.routers` followed by the name of the router. In this case, the `api` provides its own named router that you can configure by using the `[http.routers.api]` section. You’ll configure the domain that you plan on using with your dashboard also by setting the `rule` key using a host match, the entrypoint to use `websecure`, and the middlewares to include `simpleAuth`.

Add the following configurations:

traefik_dynamic.toml

```toml
...
[http.routers.api]
  rule = "Host(`your_domain`)"
  entrypoints = ["websecure"]
  middlewares = ["simpleAuth"]
  service = "api@internal"
  [http.routers.api.tls]
    certResolver = "lets-encrypt"
```

Copy

The `web` entry point handles port `80`, while the `websecure` entry point uses port `443` for TLS/SSL. You automatically redirect all of the traffic on port `80` to the `websecure` entry point to force secure connections for all requests.

Notice the last three lines here configure a *service*, enable tls, and configure `certResolver` to `"lets-encrypt"`. Services are the final step to determining where a request is finally handled. The `api@internal` service is a built-in service that sits behind the API that you expose. Just like routers and middlewares, services can be configured in this file, but you won’t need to do that to achieve your desired result.

Your completed `traefik_dynamic.toml` file will look like this:

traefik_dynamic.toml

```toml
[http.middlewares.simpleAuth.basicAuth]
  users = [
    "admin:$apr1$ruca84Hq$mbjdMZBAG.KWn7vfN/SNK/"
  ]

[http.routers.api]
  rule = "Host(`your_domain`)"
  entrypoints = ["websecure"]
  middlewares = ["simpleAuth"]
  service = "api@internal"
  [http.routers.api.tls]
    certResolver = "lets-encrypt"
```

Copy

Save the file and exit the editor.

With these configurations in place, you will now start Traefik.



## Step 2 - Running the Traefik Container

In this step you will create a Docker network for the proxy to share with containers. You will then access the Traefik dashboard. The Docker network is necessary so that you can use it with applications that are run using Docker Compose.

Create a new Docker network called `web`:

```
docker network create web
```

When the Traefik container starts, you will add it to this network. Then you can add additional containers to this network later for Traefik to proxy to.

Next, create an empty file that will hold your Let’s Encrypt information. You’ll share this into the container so Traefik can use it:

```
touch acme.json
```

Traefik will only be able to use this file if the root user inside of the container has unique read and write access to it. To do this, lock down the permissions on `acme.json` so that only the owner of the file has read and write permission.

```
chmod 600 acme.json
```

Once the file gets passed to Docker, the owner will automatically change to the **root** user inside the container.

Finally, create the Traefik container with this command:

```
docker run -d \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $PWD/traefik.toml:/traefik.toml \
  -v $PWD/traefik_dynamic.toml:/traefik_dynamic.toml \
  -v $PWD/acme.json:/acme.json \
  -p 80:80 \
  -p 443:443 \
  --network web \
  --name traefik \
  traefik:v2.2
```

This command is a little long. Let’s break it down.

You use the `-d` flag to run the container in the background as a daemon. You then share your `docker.sock` file into the container so that the Traefik process can listen for changes to containers. You also share the `traefik.toml` and `traefik_dynamic.toml` configuration files into the container, as well as `acme.json`.

Next, you map ports `:80` and `:443` of your Docker host to the same ports in the Traefik container so Traefik receives all HTTP and HTTPS traffic to the server.

You set the network of the container to `web`, and you name the container `traefik`.

Finally, you use the `traefik:v2.2` image for this container so that you can guarantee that you’re not running a completely different version than this tutorial is written for.

[A Docker image’s `ENTRYPOINT` is a command that always runs when a container is created from the image](https://docs.docker.com/engine/reference/builder/#entrypoint). In this case, the command is the `traefik` binary within the container. You can pass additional arguments to that command when you launch the container, but you’ve configured all of your settings in the `traefik.toml` file.

With the container started, you now have a dashboard you can access to see the health of your containers. You can also use this dashboard to visualize the routers, services, and middlewares that Traefik has registered. You can try to access the monitoring dashboard by pointing your browser to `https://monitor.your_domain/dashboard/` (the trailing `/` is required).

You will be prompted for your username and password, which are **admin** and the password you configured in Step 1.

Once logged in, you’ll see the Traefik interface:

![Empty Traefik dashboard](https://assets.digitalocean.com/articles/67541/traefik_2_empty_dashboard.1.png)

You will notice that there are already some routers and services registered, but those are the ones that come with Traefik and the router configuration that you wrote for the API.

You now have your Traefik proxy running, and you’ve configured it to work with Docker and monitor other containers. In the next step you will start some containers for Traefik to proxy.



## Step 3 - Registering Containers with Traefik

With the Traefik container running, you’re ready to run applications behind it. Let’s launch the following containers behind Traefik:

1. A blog using the [official WordPress image](https://hub.docker.com/_/wordpress/).
2. A database management server using the [official Adminer image](https://hub.docker.com/_/adminer/).

You’ll manage both of these applications with Docker Compose using a `docker-compose.yml` file.

Create and open the `docker-compose.yml` file in your editor:

```
nano docker-compose.yml
```

Add the following lines to the file to specify the version and the networks you’ll use:

docker-compose.yml

```yaml
version: "3"

networks:
  web:
    external: true
  internal:
    external: false
```

You use Docker Compose version `3` because it’s the newest major version of the Compose file format.

For Traefik to recognize your applications, they must be part of the same network, and since you created the network manually, you pull it in by specifying the network name of `web` and setting `external` to `true`. Then you define another network so that you can connect your exposed containers to a database container that you won’t expose through Traefik. You’ll call this network `internal`.

Next, you’ll define each of your `services`, one at a time. Let’s start with the `blog` container, which you’ll base on the official WordPress image. Add this configuration to the bottom of the file:

docker-compose.yml

```yaml
...

services:
  blog:
    image: wordpress:4.9.8-apache
    environment:
      WORDPRESS_DB_PASSWORD:
    labels:
      - traefik.http.routers.blog.rule=Host(`blog.your_domain`)
      - traefik.http.routers.blog.tls.enabled=true
      - traefik.http.routers.blog.tls.cert-provider=lets-encrypt
      - traefik.port=80
    networks:
      - internal
      - web
    depends_on:
      - mysql
```

Copy

The `environment` key lets you specify environment variables that will be set inside of the container. By not setting a value for `WORDPRESS_DB_PASSWORD`, you’re telling Docker Compose to get the value from your shell and pass it through when you create the container. You will define this environment variable in your shell before starting the containers. This way you don’t hard-code passwords into the configuration file.

The `labels` section is where you specify configuration values for Traefik. Docker labels don’t do anything by themselves, but Traefik reads these so it knows how to treat containers. Here’s what each of these labels does:

* `traefik.http.routers.adminer.rule=Host(`````blog.your_domain`````)` creates a new *router* for your container and then specifies the routing rule used to determine if a request matches this container.
* `traefik.routers.custom_name.tls=true` specifies that this router should use TLS.
* `traefik.routers.custom_name.tls.certResolver=lets-encrypt` specifies that the certificates resolver that you created earlier called `lets-encrypt` should be used to get a certificate for this route.
* `traefik.port` specifies the exposed port that Traefik should use to route traffic to this container.

With this configuration, all traffic sent to your Docker host on port `80` or `443` with the domain of `blog.your_domain` will be routed to the `blog` container.

You assign this container to two different networks so that Traefik can find it via the `web` network and it can communicate with the database container through the `internal` network.

Lastly, the `depends_on` key tells Docker Compose that this container needs to start *after* its dependencies are running. Since WordPress needs a database to run, you must run your `mysql` container before starting your `blog` container.

Next, configure the MySQL service:

docker-compose.yml

```yaml
services:
...
  mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD:
    networks:
      - internal
    labels:
      - traefik.enable=false
```

You’re using the official MySQL 5.7 image for this container. You’ll notice that you’re once again using an `environment` item without a value. The `MYSQL_ROOT_PASSWORD` and `WORDPRESS_DB_PASSWORD` variables will need to be set to the same value to make sure that your WordPress container can communicate with the MySQL. You don’t want to expose the `mysql` container to Traefik or the outside world, so you’re only assigning this container to the `internal` network. Since Traefik has access to the Docker socket, the process will still expose a router for the `mysql` container by default, so you’ll add the label `traefik.enable=false` to specify that Traefik should not expose this container.

Finally, define the Adminer container:

docker-compose.yml

```yaml
services:
...
  adminer:
    image: adminer:4.6.3-standalone
    labels:
      - traefik.http.routers.adminer.rule=Host(`db-admin.your_domain`)
      - traefik.http.routers.adminer.tls=true
      - traefik.http.routers.adminer.tls.certresolver=lets-encrypt
      - traefik.port=8080
    networks:
      - internal
      - web
    depends_on:
      - mysql
```

Copy

This container is based on the official Adminer image. The `network` and `depends_on` configuration for this container exactly match what you’re using for the `blog` container.

The line `traefik.http.routers.adminer.rule=Host(`````db-admin.your_domain`````)` tells Traefik to examine the host requested. If it matches the pattern of `db-admin.your_domain`, Traefik will route the traffic to the `adminer` container over port `8080`.

Your completed `docker-compose.yml` file will look like this:

docker-compose.yml

```yaml
version: "3"

networks:
  web:
    external: true
  internal:
    external: false

services:
  blog:
    image: wordpress:4.9.8-apache
    environment:
      WORDPRESS_DB_PASSWORD:
    labels:
      - traefik.http.routers.blog.rule=Host(`blog.your_domain`)
      - traefik.http.routers.blog.tls=true
      - traefik.http.routers.blog.tls.certresolver=lets-encrypt
      - traefik.port=80
    networks:
      - internal
      - web
    depends_on:
      - mysql

  mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD:
    networks:
      - internal
    labels:
      - traefik.enable=false

  adminer:
    image: adminer:4.6.3-standalone
    labels:
    labels:
      - traefik.http.routers.adminer.rule=Host(`db-admin.your_domain`)
      - traefik.http.routers.adminer.tls=true
      - traefik.http.routers.adminer.tls.certresolver=lets-encrypt
      - traefik.port=8080
    networks:
      - internal
      - web
    depends_on:
      - mysql
```

Copy

Save the file and exit the text editor.

Next, set values in your shell for the `WORDPRESS_DB_PASSWORD` and `MYSQL_ROOT_PASSWORD` variables:

```
export WORDPRESS_DB_PASSWORD=secure_database_password
export MYSQL_ROOT_PASSWORD=secure_database_password
```

Substitute `secure_database_password` with your desired database password. Remember to use the same password for both `WORDPRESS_DB_PASSWORD` and `MYSQL_ROOT_PASSWORD`.

With these variables set, run the containers using `docker-compose`:

```
docker-compose up -d
```

Now watch the Traefik admin dashboard while it populates.

![Populated Traefik dashboard](https://assets.digitalocean.com/articles/67541/traefik_2_populated_dashboard.1.png)

If you explore the **Routers** section you will find routers for `adminer` and `blog` configured with TLS:

![HTTP Routers w/ TLS](https://assets.digitalocean.com/articles/67541/traefik_2_http_routers.1.png)

Navigate to `blog.your_domain`, substituting `your_domain` with your domain. You’ll be redirected to a TLS connection and you can now complete the WordPress setup:

![WordPress setup screen](https://assets.digitalocean.com/articles/67541/traefik_2_wordpress_setup.1.png)

Now access Adminer by visiting `db-admin.your_domain` in your browser, again substituting `your_domain` with your domain. The `mysql` container isn’t exposed to the outside world, but the `adminer` container has access to it through the `internal` Docker network that they share using the `mysql` container name as a hostname.

On the Adminer login screen, enter `root` for **Username**, enter `mysql` for **Server**, and enter the value you set for `MYSQL_ROOT_PASSWORD` for the **Password**. Leave **Database** empty. Now press **Login**.

Once logged in, you’ll see the Adminer user interface.

![Adminer connected to the MySQL database](https://assets.digitalocean.com/articles/67541/traefik_2_adminer_screen.1.png)

Both sites are now working, and you can use the dashboard at `monitor.your_domain` to keep an eye on your applications.



# [Defining multiple similar services with Docker Compose](https://matthiasnoback.nl/2018/03/defining-multiple-similar-services-with-docker-compose/)

**Source:** https://matthiasnoback.nl/2018/03/defining-multiple-similar-services-with-docker-compose/

Posted on Mar 13th 2018 by Matthias Noback

For my new workshop - ["Building Autonomous Services"](https://github.com/matthiasnoback/building-autonomous-services-workshop) - I needed to define several Docker containers/services with more or less the same setup:

1. A PHP-FPM process for running the service's PHP code.
2. An Nginx process for serving static and dynamic requests (using the PHP-FPM process as backend).

To route requests properly, every Nginx service would have its own hostvcname. I didn't want to do complicated things with ports though - the Nginx services should all listen to port 80. However, on the host machine, only one service can listen on port 80. This is where reverse HTTP proxy [Traefik](https://traefik.io/) did a good job: it is the only service listening on the host on port 80, and it forwards requests to the right service based on the host name from the request.

This is the configuration I came up with, but this is only for the "purchase" service. Eventually I'd need this configuration about 4 times.

```yaml
services:
    purchase_web:
        image: matthiasnoback/building_autonomous_services_purchase_web
        restart: on-failure
        networks:
            - traefik
            - default
        labels:
            - "traefik.enable=true"
            - "traefik.docker.network=traefik"
            - "traefik.port=80"
        volumes:
            - ./:/opt:cached
        depends_on:
            - purchase_php
        labels:
            - "traefik.backend=purchase_web"
            - "traefik.frontend.rule=Host:purchase.localhost"

    purchase_php_fpm:
        image: matthiasnoback/building_autonomous_services_php_fpm
        restart: on-failure
        env_file: .env
        user: ${HOST_UID}:${HOST_GID}
        networks:
            - traefik
            - default
        environment:
            XDEBUG_CONFIG: "remote_host=${DOCKER_HOST_NAME_OR_IP}"
        volumes:
            - ./:/opt:cached
```

## Using Docker Compose's extend functionality

Even though I usually favor composition over inheritance, also for configuration, in this case I thought I'd be better of with inheriting some configuration instead of copying it. These services don't accidentally share some setting, in the context of this workshop, these services are meant to be more or less identical, except for some variables, like the host name.

So I decided to define a "template" for each service in `docker/templates.yml`:

```yaml
version: '2'

services:
    web:
        restart: on-failure
        networks:
            - traefik
            - default
        labels:
            - "traefik.enable=true"
            - "traefik.docker.network=traefik"
            - "traefik.port=80"
        volumes:
            - ${PWD}:/opt:cached

    php-fpm:
        image: matthiasnoback/building_autonomous_services_php_fpm
        restart: on-failure
        env_file: .env
        user: ${HOST_UID}:${HOST_GID}
        networks:
            - traefik
            - default
        environment:
            XDEBUG_CONFIG: "remote_host=${DOCKER_HOST_NAME_OR_IP}"
        volumes:
            - ${PWD}:/opt:cached
```

Then in `docker-compose.yml` you can fill in the details of these templates by using the [`extends`](https://docs.docker.com/compose/extends/) key (please note that you'd have to use "version 2" for that):

```yaml
services:
    purchase_web:
        image: matthiasnoback/building_autonomous_services_purchase_web
        extends:
            file: docker/templates.yml
            service: web
        depends_on:
            - purchase_php
        labels:
            - "traefik.backend=purchase_web"
            - "traefik.frontend.rule=Host:purchase.localhost"

    purchase_php_fpm:
        extends:
            file: docker/templates.yml
            service: php-fpm
```

We only define the things that can't be inherited (like `depends_on`), or that are specific to the actual service (host name).

## Dynamically generate Nginx configuration

Finally, I was looking for a way to get rid of specific Nginx images for every one of those "web" services. I started with a `Dockerfile` for every one of them, and a specific Nginx configuration file for each:

```
server {
    listen 80 default_server;
    index index.php;
    server_name purchase.localhost;
    root /opt/src/Purchase/public;

    location / {
        # try to serve file directly, fallback to index.php
        try_files $uri /index.php$is_args$args;
    }

    location ~ ^/index\.php(/|$) {
        fastcgi_pass purchase_php_fpm:9000;
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT $realpath_root;

        # Prevents URIs that include the front controller. This will 404:
        # https://domain.tld/index.php/some-path
        # Remove the internal directive to allow URIs like this
        internal;
    }
}
```

To reuse the Nginx image for every "web" service, I needed a way to use variables in this configuration file. The solution was documented in the description of the official `nginx` image: [Using environment variables in nginx configuration](https://hub.docker.com/_/nginx/). The trick is to use environment variables in the configuration file, and replace them with their real values when you start the Nginx container. The template configuration file could look something like this:

```
server {
    listen 80 default_server;
    index index.php;
    server_name ${SERVER_NAME};
    root ${ROOT};

    location / {
        # try to serve file directly, fallback to index.php
        try_files $uri /index.php$is_args$args;
    }

    location ~ ^/index\.php(/|$) {
        fastcgi_pass ${PHP_BACKEND}:9000;
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT $realpath_root;

        # Prevents URIs that include the front controller. This will 404:
        # https://domain.tld/index.php/some-path
        # Remove the internal directive to allow URIs like this
        internal;
    }
}
```

The problem is, the configuration file itself contains many strings that *look like environment variables* (e.g. `$realpath_root`). When using the proposed solution, all those variables were replaced by empty strings.

After some fiddling (and looking up configuration options for `envsubst`), I found the solution: you can explicitly mention *which* variables should be replaced. The only other thing I needed to do is properly escape the names of these variables, to prevent them from being replaced on the spot:

```docker
FROM nginx:1.13-alpine
COPY template.conf /etc/nginx/conf.d/site.template
...
CMD sh -c "envsubst '\$SERVER_NAME \$ROOT \$PHP_BACKEND' < /etc/nginx/conf.d/site.template > /etc/nginx/conf.d/default.conf && nginx -g 'daemon off;'"
```

At this point, I was able to build only one Docker image that could be reused by all the "web" services. I only had to set the correct environment variables in `docker-compose.yml`:

```yaml
services:
    purchase_web:
        # ...
        environment:
            - SERVER_NAME=purchase.localhost
            - PHP_BACKEND=purchase_php
            - ROOT=/opt/src/Purchase/public

    sales_web:
        # ...
        environment:
            - SERVER_NAME=sales.localhost
            - PHP_BACKEND=sales_php
            - ROOT=/opt/src/Sales/public

    # ...
```

You'll find the complete configuration in the workshop project's repository: ["Building Autonomous Services"](https://github.com/matthiasnoback/building-autonomous-services-workshop).