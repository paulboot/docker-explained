# Docker-compose Explained

**Sources:**

* https://gabrieltanner.org/blog/docker-compose



## Remember

|                                                           |                                                              |
| --------------------------------------------------------- | ------------------------------------------------------------ |
| `docker compose exec -T xxxx`                             | **NOTE:** spawn een terminal anders geen output ook niet na een \| pipe in een cronjob! |
| `docker compose pull`                                     | Only pull updated images from repro, check to see double images `docker images` |
| `docker compose build <service_name>`                     | Do a local build of an image using the mages from the docker repro, in the compose file the `dockerfile: <dockerfile_name>`<br /><br />`NOTICE: PHP message: PHP Fatal error:  Call to undefined function mysqli_connect()`<br />Rebuild the PHP image with lcoa added Myslq functions! |
| `docker compose stop <service_name>`                      |                                                              |
| `docker compose up -d`                                    | Depending on the images available this command builds a container or pulls and builds/updates an container. The `-d ` detaches from the log output. Use `docker compose logs -f ` to tail the logs of all services (or also called apps). |
| `docker compose up -d --no-deps <service_name>`           |                                                              |
| `docker compose up -d --no-deps --build <service_name>`   |                                                              |
| `docker compose up --force-recreate --build -d --no-deps` |                                                              |
| `docker image prune -f`                                   |                                                              |
| `docker compose stop`                                     | Command will stop your containers, but it won’t remove them. |
| `docker compose down`                                     | Command will stop your containers, but it also removes the stopped containers as well as any networks that were created. Will not remove images. |
| `docker compose down -v`                                  | The `-v` flag to remove all volumes too.                     |
| `docker compose config --services`                        |                                                              |
| `docker compose logs -f <service-name>`                   | Use  THIS `tail -f` like command to debug all or a single service. |
| `docker compose ps`                                       |                                                              |
| docker info \| grep Swarm                                 |                                                              |
| docker prune XXX (cleanup)                                |                                                              |
| docker system prune -a                                    |                                                              |
| docker-compose pull <service-name>                        |                                                              |
| docker-compose build -pull                                |                                                              |
| docker search <name>                                      |                                                              |
| docker image ls -a --digests                              |                                                              |
| docker info                                               |                                                              |
| docker network ls                                         |                                                              |
| docker network inspect <my-macvlan-net>                   |                                                              |


## Introduction

Docker Compose lets you bring up a complete development environment with only one command: `docker-compose up`, and tear it down just as easily using `docker-compose down`. This allows us developers to keep our development environment in one central place and helps us to easily deploy our applications.

### Create Multiple Isolated Environments

Project names allow us to isolate environments from one another. For example, when working on a specific code branch of your service, you’ll want to create an independent copy of your environment on a development host. On a continuous integration (CI) server, you can use Compose to keep builds from interfering with one another (similar to branching on git repositories).

Compose uses the basename of the project directory for the default project name. Pass the `-p` command line option (or set the `COMPOSE_PROJECT_NAME` environment variable) to manually set the project name.

### Preserve Volume Data

Compose preserves all volumes (and therefore your data) for all of your services. When Docker Compose runs, it looks for containers from previous runs and copies their volumes to the new containers, enabling you to pick up where you left off.

### Recreate Updated Containers

Docker Compose caches your configuration; when you restart a service that hasn’t been updated, Compose will reuse the existing containers. Reusing containers decreases build and load time.

### Variables and Moving Compositions

You can use variables in your compositions to customize environments or users. You can *even extend* compositions from other Compose files.

#### Variable Substitution

Compose uses variable values from the shell environment from which you run `docker-compose`. Suppose you set `EXTERNAL_PORT=8000` in the shell environment and then run the following configuration:

```
web:
    build:
        ports:
            - "${EXTERNAL_PORT}:5000"
```

Compose will look for the shell’s `EXTERNAL_PORT` environment variable and substitute its value. *Then*, Compose will resolve the port mapping to “8000:5000” before creating the `web` container. You may use either `#VARIABLE` or `${VARIABLE}` syntax in your composition. *However*, you may not use “extended-shell” style features like `${VARIABLE-default}` or `${VARIABLE/foo/bar}`. If you need a literal dollar sign, you may use `$$`; you can also use this syntax to prevent Compose from processing environment variables:

```
    web:
        build: .
        command: "$$COMPOSE_DOES_NOT_PROCESS_THIS_VARIABLE"
```

If you use a single dollar sign, Compose assumes an environment variable and warns you:

```
COMPOSE_DOES_NOT_PROCESS_THIS_VARIABLE is not set. Substituting an empty string.
```

## Controlling Startup Order

Docker Compose starts containers in dependency order by using the `depends_on` option to dictate when a service should start. Compose uses `depends_on`, `links`, `volumes_from`, and `network_mode: "service: ..."` to determine startup order.

If you need a container to wait on another container’s “ready” state, you can use tools like [wait-for-it](https://github.com/vishnubob/wait-for-it) or [dockerize](https://github.com/jwilder/dockerize). These tools will poll the hosts and ports until TCP connections can be confirmed. You can append an `entrypoint` to your composition to force the wait:

```
version: '2'

services:
    web:
        build: .
        ports:
            - "80:8000"
        depends_on:
            - db
        entrypoint: "./wait-for-it.sh db:5432"
    db:
        image: postgres
```

You’re also able to write your own wrapper script as well if you need more granular control.

## Running Multiple Copies of a Single Compose Project

For times when you need multiple copies of environments with the same composition (or `docker-compose.yml` file), simply run `docker-compose up -p new_project_name`.

## Environment Variables

We can use shell environment variables to set values in our compositions:

1. Set the environment variables:

   ```
   $ TAG="latest"
   $ echo $TAG
   latest
   $ DB="postgres"
   $ echo $DB
   postgres
   ```

2. Then, use the environment variable in our Docker Compose file:

   ```
   db:
       image: "${DB}:$TAG"
   ```

Docker Compose accepts both `${DB}` and `$TAG`.

We can also set environment variables in our containers:

```
web:
    environment:
        - PRODUCTION=1
```

We can even pass environment variables *through* to our containers:

```
$ PRODUCTION=1
$ echo $PRODUCTION
1
web:
    environment:
        - PRODUCTION
```

## The Environment File

Recall how Docker Compose [passes an empty string](https://runnable.com/docker/introduction-to-docker-compose) to any environment variable you do not define. You can ensure your environment variables are always passed by storing them in an environment file. Call this file `.env` and store it in your working directory. Docker Compose ignores blank lines (use them for readability) and code beginning with `#` (comments). You may assign variables for later variable substitution, and you may set a handful of Compose CLI variables:

* `COMPOSE_API_VERSION`
* `COMPOSE_FILE`
* `COMPOSE_HTTP_TIMEOUT`
* `COMPOSE_PROJECT_NAME`
* `DOCKER_CERT_PATH`
* `DOCKER_HOST`
* `DOCKER_TLS_VERIFY`

Here is an example of an environment file:

```
# ./.env 
# for our staging environment

COMPOSE_API_VERSION=2
COMPOSE_HTTP_TIMEOUT=45
DOCKER_CERT_PATH=/mycerts/docker.crt
EXTERNAL_PORT=5000
```

Docker Compose ignores the first three lines (two comments and a blank line), then reads the following four lines and sets the environment variables accordingly.

## Using Multiple Docker Compose Files

Use multiple Docker Compose files when you want to change your app for different environments (e.g., dev, staging, and production) or when you want to run admin tasks against a Compose application. This gives us one way to share common configurations.

Docker Compose *already* reads two files by default: `docker-compose.yml` and `docker-compose.override.yml`. The `docker-compose-override.yml` file can be used to store overrides for existing services or define new services. Use multiple files (or an override file with a different name) by passing the `-f` option to `docker-compose up` (**order matters**):

```
$ docker-compose up -f my-override-1.yml my-overide-2.yml
```

When two configuration options match, the most recent value either replaces or extends the first.

In the following example, the new value overrides the old, and `command` runs `my_new_app.py`:

```
# original service
command: python my_app.py

# new service
command: python my_new_app.py
```

When you use options with multiple values (`ports`, `expose`, `external_links`, `dns`, `dns_search`, and `tmpfs`), Docker Compose concatenates the values (in the example below, Compose exposes ports `5000` and `8000`):

```
# original service
expose:
    - 5000
    
# new service
expose:
    - 8000
```

If you use `environment`, `labels`, `volumes`, or `devices`, Docker Compose merges your results. In the following example, the three environment variables become `FOO=Hello` and `BAR="Python Dev!"`:

```
# original service
environment:
    - FOO=Hello
    - BAR=World

# new service
environment:
    - BAR="Python Dev!"
```

### Different Environments

Start with your base Docker Compose file for your application (`docker-compose.yml`):

```
web:
    image: "my_dockpy/my_django_app:latest"
    links:
        - db
        - cache

db:
    image: "postgres:latest"

cache:
    image: "redis:latest"
```

On our development server, we want to expose some ports, mount our code as a volume, and build our web image (`docker-compose.override.yml`):

```
web:
    build: .
    volumes:
        - ".:/code"
    ports:
        - "8883:80"
    environment:
        DEBUG: "true"

db:
    command: "-d"
    ports:
        - "5432:5432"

cache:
    ports:
        - "6379:6379"
```

`docker-compose up` *automatically* reads the override file and applies it. We also need a production version of our Docker Compose app, and we want to call that `docker-compose.production.yml`:

```
web:
    ports:
        - "80:80"
    environment:
        PRODUCTION: "true"

cache:
    environment:
        TTL: "500"
```

When you want to deploy your production file, simply run the following:

```
$ docker-compose -f docker-compose.yml -f docker-compose.production.yml up -d
```

Note: Docker Compose reads `docker-compose.production.yml` *but not* `docker-compose.override.yml`.

### Administrative Tasks

We want to run an administrative copy of our application so we can perform tasks such as backing up our database. Using the same `docker-compose.yml` file above, we create the following `docker-compose.admin.yml` file:

```
dbadmin:
    build: database_admin/
    links:
        - db
```

Then we run this with the following command:

```
$ docker-compose -f docker-compose.yml -f docker-compose.admin.yml run dbadmin db-backup
```

## Extending Services

Alternately, we can share common configurations by using the `extends` field. `extends` even allows us to share service options between different projects.

1. Create a `common-services.yml` file (you may name it whatever you like):

   ```
   webapp:
       build: .
       ports:
           - "8000:8000"
       volumes:
           - "/data"
   ```

2. Create a basic `docker-compose.yml`. For example:

   ```
   web:
       extends:
           file: common-services.yml
           service: webapp
   ```

You may also define (or re-define) your configuration locally (and even add other services):

```
web:
    extends:
        file: common-services.yml
        service: webapp
    environment:
        - DEBUG=1
    cpu_shares: 5
    links:
        - db

important_web:
    extends: web
    cpu_shares: 10

db: 
    image: postgres
```

## Common Issues

### Controlling Startup Order

*Docker Compose **only** waits for a container to run before starting the next.* In the event one part of your application becomes unavailable, Docker Compose expects resiliency from the rest of your application.

### The Environment File

If you define an environment variable in the shell or via the command line when running `docker-compose`, *those variables will take precedence* over the `.env` file.

It’s considered bad practice to store environment variables on a version control system. If you use an environment file, add it to your local ignore file and create a sample, `env.sample`, similar to the following (assuming you use the `.env` file above):

```
COMPOSE_API_VERSION=    # should be 2
COMPOSE_HTTP_TIMEOUT=   # we use 30 on production and 120 on development
DOCKER_CERT_PATH=       # store your certificate path here
EXTERNAL_PORT=          # set the external port here (remeber, 5000 for Flask and 8000 for Django)
```

### Using Multiple Docker Compose Files

Note, Docker Compose merges files in the **order you specify**.

### Extending Services

Services never share `links`, `volumes_from`, or `depends_on` using `extends`; `links` and `volumes_from` must always be defined locally.

## Structure of the Compose file

Compose allows us developers to easily handle multiple docker containers at once by applying many rules which are declared in a `docker-compose.yml` file.

It consists of multiple layers that are split using tab stops or spaces instead of the braces we know in most programming languages. There are four main things almost every Compose-File should have which include:

* The version of the compose file
* The services which will be built
* All used volumes
* The networks which connect the different services

A sample file could look like this:

```yaml
version: '3.3'

services:
   db:
     image: mysql:5.7
     volumes:
       - db_data:/var/lib/mysql
     restart: always
     environment:
       MYSQL_ROOT_PASSWORD: somewordpress
       MYSQL_DATABASE: wordpress
       MYSQL_USER: wordpress
       MYSQL_PASSWORD: wordpress

   wordpress:
     depends_on:
       - db
     image: wordpress:latest
     ports:
       - "8000:80"
     restart: always
     environment:
       WORDPRESS_DB_HOST: db:3306
       WORDPRESS_DB_USER: wordpress
       WORDPRESS_DB_PASSWORD: wordpress
       WORDPRESS_DB_NAME: wordpress
volumes:
    db_data: {}
```

As you can see this file contains a whole Wordpress application including the MySQL database. Each of these services is treated as a separate container that can be swapped in and out when you need it.

Now that we know the basic structure of a Compose file let's continue by looking at the important concepts.

## Concepts / Keywords

The core aspects of the Compose file are its concepts which allow it to manage and create a network of containers. In this section, we will explore these concepts in detail and take a look at how we can use them to customize our Compose configuration.

### Services:

The services tag contains all the containers which are included in the Compose file and acts as their parent tag.

```yaml
services:
  proxy:
    build: ./proxy
  app:
    build: ./app
  db:
    image: postgres
```

Here you can see that the services tag contains all the containers of the Compose configuration.

### Base image (Build):

The base image of a container can be defined by either using a preexisting image that is available on DockerHub or by building images using a Dockerfile.

Here are some basic examples:

```yaml
version: '3.3'

services:
    alpine:
        image: alpine:latest
        stdin_open: true
        tty: true
        command: sh
```

Here we use a predefined image from DockerHub using the image tag.

```yaml
version: '3.3'
services:
    app:
        container_name: website
        restart: always
        build: .
        ports:
            - '3000:3000'
    command:
        - 'npm run start'
```

In this example, we define our images using the build tag which takes the destination of our Dockerfile as a parameter.

The last option of defining the base image is to use a Dockerfile with a custom name.

```yaml
build:
    context: ./dir
    dockerfile: Dockerfile.dev
```

### Ports:

Exposing the ports in Compose works similarly as in the Dockerfile. We differentiate between two different methods of exposing the port:

**Exposing the port to linked services:**

```yaml
expose:
 - "3000"
 - "8000"
```

Here we publish the ports to the linked services of the container and not to the host system.

**Exposing the port to the host system:**

```yaml
ports:
  - "8000:80"  # host:container
```

In this example, we define which port we want to expose and the host port it should be exposed to.

You can also define the port protocol which can either be UDP or TCP:

```yaml
ports:
  - "8000:80/udp"
```

### Commands:

Commands are used to execute actions once the container is started and act as a replacement for the CMD action in your Dockerfile.

The CMD action is the first command that gets executed when the container is started and is therefore mostly used to start a process e.g. start your website through a CLI command like npm run start.

```yaml
app:
        container_name: website
        restart: always
        build: ./
        ports:
            - '3000:3000'
        command:
            - 'npm run start'
```

Here we create a service for a website and add the starting command using the command tag. This command will be executed after the container has started and will then start the website.

For more information about CMD, RUN, and Entrypoint you can read [this article](https://goinbigdata.com/docker-run-vs-cmd-vs-entrypoint/) which discusses the details and compares their functionality.

### Volumes:

Volumes are Docker's preferred way of persisting data which is generated and used by Docker containers. They are completely managed by Docker and can be used to share data between containers and the Host system.

They do not increase the size of the containers using it, and their context is independent of the lifecycle of the given container.

![Docker Volumes](https://cms.gabrieltanner.org/content/images/2019/10/grafik.png)Src: https://docs.docker.com/storage/volumes/

There are multiple types of volumes you can use in Docker. They can all be defined using the volumes keyword but have some minor differences which we will talk about now.

**Normal Volume:**

The normal way to use volumes is by just defining a specific path and let the Engine create a volume for it. This can be done like this:

```yaml
volumes:
  # Just specify a path and let the Engine create a volume
  - /var/lib/mysql
```

**Path mapping:**

You can also define absolute path mapping of your volumes by defining the path on the host system and mapping it to a container destination using the: operator.

```yaml
volumes:
  - /opt/data:/var/lib/mysql
```

Here you define the path of the host system followed by the path of the container.

**Named volume:**

Another type of volume is the named volume which is similar do the other volumes but has it's own specific name that makes it easier to use on multiple containers. That's why it's often used to share data between multiple containers and services.

```yaml
volumes:
  - datavolume:/var/lib/mysql
```

### Dependencies:

Dependencies in Docker are used to make sure that a specific service is available before the dependent container starts. This is often used if you have a service that can't be used without another one e.g. a CMS (Content Management System) without its database.

```yaml
   ghost:
        container_name: ghost
        restart: always
        image: ghost
        ports:
            - 2368:2368
        environment:
            - .
        depends_on: [db]
    db:
        image: mysql
        command: --default-authentication-plugin=mysql_native_password
        restart: always
        environment:
            MYSQL_ROOT_PASSWORD: example
```

Here we have a simple example of a Ghost CMS which depends on the MySQL database to work and therefore uses the depends_on command. The depends_on command takes an array of string which defines the container names the service depends on.

### Environment variables:

Environment variables are used to bring configuration data into your applications. This is often the case if you have some configurations that are dependent on the host operating system or some other variable things that can change.

There are many different options of passing environment variables in our Compose file which we will explore here:

**Setting an environment variable:**

You can set environment variables in a container using the "environment" keyword, just like with the normal docker container run --environment command in the shell.

```yaml
web:
  environment:
    - NODE_ENV=production
```

In this example, we set an environment variable by providing a key and the value for that key.

**Passing an environment variable:**

You can pass environment variables from your shell straight to a container by just defining an environment key in your Compose file and not giving it a value.

```yaml
web:
  environment:
    - NODE_ENV
```

Here the value of *NODE_ENV* is taken from the value from the same variable in the shell which runs the Compose file.

**Using an .env file:**

Sometimes a few environment variables aren't enough and managing them in the Compose file can get pretty messy. That is what .env files are for. They contain all the environment variables for your container and can be added using one line in your Compose file.

```yaml
web:
  env_file:
    - variables.env
```

### Networking

Networks define the communication rules between containers, and between containers and the host system. They can be configured to provide complete isolation for containers, which enables building applications that work together securely.

By default Compose sets up a single network for each container. Each container is automatically joining the default network which makes them reachable by both other containers on the network, and discoverable by the hostname defined in the Compose file.

### Specify custom networks:

Instead of only using the default network you can also specify your own networks within the top-level *networks* key, allowing to create more complex topologies and specifying network drivers and options.

```yaml
networks:
  frontend:
  backend:
    driver: custom-driver
    driver_opts:
      foo: "1"
```

Each container can specify what networks to connect to with the service level "networks" keyword, which takes a list of names referencing entries of  the top-level "networks" keyword.

```yaml
services:
  proxy:
    build: ./proxy
    networks:
      - frontend
  app:
    build: ./app
    networks:
      - frontend
      - backend
  db:
    image: postgres
    networks:
      - backend
```

You can also provide a custom name to your network (since version 3.5):

```yaml
version: "3.5"
networks:
  webapp:
    name: website
    driver: website-driver
```

For a full list of the network configuration options, see the following references:

* [Top-level network key](https://docs.docker.com/compose/compose-file/compose-file-v2/#network-configuration-reference)
* [Service-level network key](https://docs.docker.com/compose/compose-file/compose-file-v2/#networks)

### External (Pre-existing) networks:

You can use pre-existing networks with Docker Compose using the external option.

```yaml
networks:
  default:
    external:
      name: pre-existing-network
```

In this example, Docker never creates the default network and just uses the pre-existing network defined in the external tag.

### Configure the default networks:

Instead of defining your own networks you could change the settings of the default network by defining an entry with the name default under the "networks" keyword.

```yaml
version: "3"
services:
  web:
    build: .
    ports:
      - "8000:8000"
  db:
    image: postgres

networks:
  default:
    driver: custom-driver
```

### Linking containers:

You may also define extra aliases for your containers that services can use to communicate with each other. Services in the same network can already reach one another. Links then only define other names under which the container can be reached.

```yaml
version: "3"
services:
    web:
        build: . 
        links: 
            - "db:database"
    db:
        image: mongo
```

In this example, the web container can reach the database using one of the two hostnames (db or database).

## CLI

All the functionality of Docker-Compose is executed through its build in CLI, which has a very similar set of commands to what is offered by Docker.

```bash
build    Build or rebuild services 
help     Get help on a command 
kill     Kill containers 
logs     View output from containers 
port     Print the public port for a port binding 
ps       List containers 
pull     Pulls service images 
rm       Remove stopped containers 
run      Run a one-off command 
scale    Set number of containers for a service 
start    Start services 
stop     Stop services 
restart  Restart services 
up       Create and start containers
down     Stops and removes containers
```

They are not only similar but also behave like their Docker counterparts. The only difference is that they affect the entire multi-container architecture which is defined in the docker-compose.yml file instead of a single container.

Some Docker commands are not available anymore and have been replaced with other commands that make more sense in the context of a completely multi-container setup.

The most important new commands include the following:

* docker-compose up
* docker-compose down

## Using Multiple Docker Compose Files

The use of multiple Docker Compose files allows you to change your application for different environments (e.g. staging, dev, and production) and helps you run admin tasks or tests against your application.

Docker Compose reads two files by default, a *docker-compose.yml* file, and an optional *docker-compose.override.yml* file. The docker-compose.override file can be used to store overrides of the existing services or define new services.

To use multiple override files, or an override file with a different name, you can pass the -f option to the docker-compose up command. The base Compose file has to be passed on the first position of the command.

```yaml
docker-compose up -f override.yml override2.yml
```

When you use multiple configuration files, you need to make sure that the paths are relative to the base Compose file which is specified first with the -f flag.

Now let's look at an example of what can be done using this technique.

```yaml
# original service
command: npm run dev

# new service
command: npm run start
```

Here you override the old run command with the new one which starts your website in production instead of dev mode.

When you use multiple values on options like ports, expose, DNS and tmpfs, Docker Compose concatenates the values instead of overriding them which is shown in the following example.

```yaml
# original service
expose:
    - 3000
    
# new service
expose:
    - 8080
```

## Compose in production

Docker Compose allows for easy deployment because you can deploy your whole configuration on a single server. If you want to scale your app, you can run it on a Swarm cluster.

There are still things you probably need to change before deploying your app configuration to production. These changes include:

* Binding different ports to the host
* Specifying a restart policy like *restart: always* to avoid downtime of your container
* Adding extra services such as a logger
* Removing any unneeded volume bindings for application code

After you have taken these steps you can deploy your changes using the following commands:

```yaml
docker-compose build
docker-compose up --no-deps -d
```

This first rebuilds the images of the services defined in the compose file and then recreates the services.

## Example

Now that we have gone through the theory of Compose let's see some of the magic we just talked about in action. For that, we are going to build a simple Node.js application with a Vue.js frontend which we will deploy using the tools we learned about earlier.

Let's get started by cloning the repository with the finished Todo list application so we can directly jump into the Docker part.

```bash
git clone --single-branch --branch withoutDocker https://github.com/TannerGabriel/docker-node-mongodb.git
```

This should give you a project with the following folder structure:

![Folderstructure](https://cms.gabrieltanner.org/content/images/2019/10/grafik-1.png)Node-Vue Todo list folderstructure

Now that we have the project setup lets continue by writing our first Dockerfile for the Node.js backend.

```docker
FROM node:latest

# Create app directory
WORKDIR /usr/src/app

# Install app dependencies
COPY package*.json ./

RUN npm install

# Bundle app source
COPY . .

EXPOSE 3000:3000
CMD [ "node", "server.js" ]
```

All right, let's understand what's going on here by walking through the code:

* First, we define the base image using the FROM keyword
* Then we set the directory we are going to work in and copy our local package.json file into the container
* After that, we install the needed dependencies from the package.json file and expose the port 3000 to the host machine
* The CMD keyword lets you define the command which will be executed after the container startup. In this case, we use it to start our express server using the *node server.js* command.

Now that we have finished the Dockerfile of the backend lets complete the same process for the frontend.

```docker
FROM node:lts-alpine

RUN npm install -g http-server

WORKDIR /app

COPY package*.json ./
COPY .env ./

RUN npm install

COPY . .

RUN npm run build

EXPOSE 8080
CMD [ "http-server", "dist" ]
```

This file is similar to the last one but installs an HTTP server which displays the static site we get when building a Vue.js application. I will not go into further detail about this script because it isn't in the scope of this tutorial.

With the Dockerfiles in place, we can go ahead and write the docker-compose.yml file we learned so much about.

First, we define the version of our Compose file (in this case version 3)

```yaml
version: '3'
```

After that, we start defining the services we need for the project to work.

```yaml
services:
    nodejs:
        build:
            context: ./backend/
            dockerfile: Dockerfile
        container_name: nodejs
        restart: always
        environment:
            - HOST=mongo
        ports:
            - '3000:3000'
        depends_on: [mongo]
```

The nodejs service uses the Dockerfile of the backend which we created above and publishes the port 3000 to the host machine. The service also depends on the mongo service which means that it lets the database start first before starting itself.

Next, we define a basic MongoDB service which uses the default image provided on DockerHub.

```yaml
     mongo:
        container_name: mongo
        image: mongo
        ports:
            - '27017:27017'
        volumes:
            - ./data:/data/db
```

This service also publishes a port to the host system and saves the data of the database in a local folder using a volume.

The last service we need to define is the frontend which uses the frontend Dockerfile to build the image and publishes port 8080 to the host system.

```yaml
    frontend:
        build:
            context: ./frontend/
            dockerfile: Dockerfile
        container_name: frontend
        restart: always
        ports:
            - '8080:8080'
```

That is it! We have finished our Docker files and can now move on to running the application. This is done using the following two commands:

```bash
# builds the images from the dockerfiles
docker-compose build

# starts the services defined in the docker-compose.yml file
# -d stands for detached
docker-compose up -d
```

As indicated by the terminal output, your services are now running and you are ready to visit the finished website on localhost:8080 which should look something like this:

![Docker Node todo list application](https://cms.gabrieltanner.org/content/images/2019/10/Docker-Node-example.JPG)Docker Node todo list application

Now you can add todos to your list using the add button and your app should look like this.

![Todo list with items](https://cms.gabrieltanner.org/content/images/2019/10/Docker-Node-example-items.JPG)Todo list with items

If you reload the page the items should stay the same because they are saved in our database. The last thing I want to show is how you can get the debug logs of the running containers.

```bash
docker-compose logs
```

This command will display all logs of the running containers and can help you debug your errors or check the current state of your application.

That is it for the project. The source code for the whole project can be found on my [Github](https://github.com/TannerGabriel/docker-node-mongodb).



