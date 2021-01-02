# Docker Components Explained

**Enhanced Source:** 

* https://www.digitalocean.com/community/tutorials/the-docker-ecosystem-an-introduction-to-common-components
* https://phoenixnap.com/kb/docker-image-vs-container

### Introduction

Containerization is the process of distributing and deploying applications in a portable and predictable way. It accomplishes this by packaging components and their dependencies into standardized, isolated, lightweight process environments called containers.  Many organizations are now interested in designing applications and services that can be easily deployed to distributed systems, allowing the system to scale easily and survive machine and application failures.  Docker, a containerization platform developed to simplify and standardize deployment in various environments, was largely instrumental in spurring the adoption of this style of service design and management.  A large amount of software has been created to build on this ecosystem of distributed container management.

## Docker and Containerization

Docker is the most common containerization software in use today. While other containerizing systems exist, Docker makes container creation and management simple and integrates with many open source projects.

![Docker containerization](https://assets.digitalocean.com/articles/docker_ecosystem/Container-Overview.png)

In this image, you can begin to see (in a simplified view) how containers relate to the host system. Containers isolate individual applications and use operating system resources that have been abstracted by Docker. In the exploded view on the right, we can see that containers can be built by “layering”, with multiple containers sharing underlying layers, decreasing resource usage.

Docker’s main advantages are:

* **Lightweight resource utilization**: instead of virtualizing an entire operating system, containers isolate at the process level and use the host’s kernel.
* **Portability**: all of the dependencies for a containerized application are bundled inside of the container, allowing it to run on any Docker host.
* **Predictability**: The host does not care about what is running inside of the container and the container does not care about which host it is running on.  The interfaces are standardized and the interactions are predictable.

Typically, when designing an application or service to use Docker, it works best to break out functionality into individual containers, a design decision known as service-oriented architecture. This gives you the ability to easily scale or update components independently in the future. Having this flexibility is one of the many reasons that people are interested in Docker for development and deployment.

To find out more about containerizing applications with Docker, click [here](https://www.digitalocean.com/community/tutorials/the-docker-ecosystem-an-overview-of-containerization).



## What is the difference between a Docker image and a container?

### What is a Docker Image?

A **Docker image** is an immutable (unchangeable) file that contains the source code, libraries, dependencies, tools, and other files needed for an application to run.

Due to their **read-only** quality, these images are sometimes referred to as snapshots. They represent an application and its virtual environment at a specific point in time. This consistency is one of the great features of Docker. It allows developers to test and experiment software in stable, uniform conditions.

Since images are, in a way, just **templates**, you cannot start or run them. What you can do is use that template as a base to build a container. A container is, ultimately, just a running image. Once you create a container, it adds a writable layer on top of the immutable image, meaning you can now modify it.

The image-based on which you create a container exists separately and cannot be altered. When you run a [containerized environment](https://phoenixnap.com/kb/how-to-containerize-applications), you essentially create a **read-write copy** of that filesystem (docker image) inside the container. This adds a **container layer** which allows modifications of the entire copy of the image.

![Brief explanation of Container Layer and Image layer](https://phoenixnap.com/kb/wp-content/uploads/2019/10/container-layers.png)

You can create an unlimited number of Docker images from one **image base**. Each time you change the initial state of an image and save the existing state, you create a new template with an additional layer on top of it.

Docker images can, therefore, consist of a **series of layers**, each differing but also originating from the previous one. Image layers represent read-only files to which a container layer is added once you use it to start up a virtual environment.

### What is a Docker Container?

A **Docker container** is a virtualized run-time environment where users can isolate applications from the underlying system. These containers are compact, portable units in which you can start up an application quickly and easily.

A valuable feature is the **standardization** of the computing environment running inside the container. Not only does it ensure your application is working in identical circumstances, but it also simplifies sharing with other teammates.

As containers are autonomous, they provide strong isolation, ensuring they do not interrupt other running containers, as well as the server that supports them. Docker claims that these units “provide the strongest isolation capabilities in the industry”. Therefore, you won’t have to worry about keeping your machine **secure** while developing an application.

Unlike virtual machines (VMs) where virtualization happens at the hardware level, containers virtualize at the app layer. They can utilize one machine, share its kernel, and virtualize the operating system to run isolated processes. This makes containers extremely **lightweight**, allowing you to retain valuable resources.

![The difference in structure between containers and virtual machines](https://phoenixnap.com/kb/wp-content/uploads/2019/10/container-vs-virtual-machine.png)

### Docker Images vs Containers

When discussing the difference between images and containers, it isn’t fair to contrast them as opposing entities. Both elements are closely **related and are part of a system** defined by the Docker platform.

If you have read the previous two sections that define docker images and docker containers, you may already have some understanding as to how the two establish a relationship.

Images can exist without containers, whereas a container needs to run an image to exist. Therefore, containers are dependent on images and use them to construct a run-time environment and run an application.

The two concepts exist as essential components (or rather phases) in the process of running a Docker container. Having a running container is the final “phase” of that process, indicating it is dependent on previous steps and components. That is why docker images essentially govern and shape containers.

#### From `Dockerfile` to Image to Container

It all starts with a script of instructions that define how to build a specific Docker image. This script is called a [`Dockerfile`](https://phoenixnap.com/kb/create-docker-images-with-dockerfile). The file automatically executes the outlined commands and creates a **Docker image**.

The command for creating an image from a `Dockerfile` is **`docker build`**.

The image is then used as a template (or base), which a developer can copy and use it to run an application. The application needs an isolated environment in which to run – a **container**.

This environment is not just a virtual “space”. It entirely relies on the image that created it. The source code, files, dependencies, and binary libraries, which are all found in the Docker image, are the ones that make up a container.

To create a container layer from an image, use the command **`docker create`**.

Finally, after you have launched a container from an existing image, you start its service and run the application.

![how a docker container is created](https://phoenixnap.com/kb/wp-content/uploads/2019/10/crating-a-docker-container.png)

#### Creating a Docker Image from a Container

If you make changes to the initial image and you want to preserve it for future work, you can save the modified image by taking a screenshot of the current state of the container. By doing so, you [attach a container](https://phoenixnap.com/kb/how-to-ssh-into-docker-container) layer on top of the image, ultimately building a new immutable image. As a result, you end up with two Docker images derived from the same filesystem.



## Service Discovery and Global Configuration Stores

Service discovery is one component of an overall strategy aimed at making container deployments scalable and flexible. Service discovery is used so that containers can find out about the environment they have been introduced to without administrator intervention. They can find connection information for the components they must interact with, and they can register themselves so that other tools know that they are available. These tools also typically function as globally distributed configuration stores where arbitrary config settings can be set for the services operating in your infrastructure.

![Docker service discovery](https://assets.digitalocean.com/articles/docker_ecosystem/Discover-Flow.png)

In the above image, you can see an example flow in which one application registers its connection information with the discovery service system. Once registered, other applications can query the discovery service to find out how to connect to the application.

These tools are often implemented as simple key-value stores that are distributed among the hosts in a clustered environment. Generally, the key-value stores provide an HTTP API for accessing and setting values. Some include additional security measures like encrypted entries or access control mechanisms. The distributed stores are essential for managing the clustered Docker hosts in addition to their primary function of providing self-configuration details for new containers.

Some of the responsibilities of service discovery stores are:

* Allowing applications to obtain the data needed to connect with to the services they depend on.
* Allowing services to register their connection information for the above purpose.
* Providing a globally accessible location to store arbitrary configuration data.
* Storing information about cluster members as needed by any cluster management software.

Some popular service discovery tools and related projects are:

* **[etcd](https://www.digitalocean.com/community/tutorials/how-to-use-etcdctl-and-etcd-coreos-s-distributed-key-value-store)**: service discovery / globally distributed key-value store
* **[consul](https://www.digitalocean.com/community/tutorials/an-introduction-to-using-consul-a-service-discovery-system-on-ubuntu-14-04)**: service discovery / globally distributed key-value store
* **[zookeeper](https://www.digitalocean.com/community/tutorials/an-introduction-to-mesosphere#a-basic-overview-of-apache-mesos)**: service discovery / globally distributed key-value store
* **[crypt](http://xordataexchange.github.io/crypt/)**: project to encrypt etcd entries
* **[confd](https://www.digitalocean.com/community/tutorials/how-to-use-confd-and-etcd-to-dynamically-reconfigure-services-in-coreos)**: watches key-value store for changes and triggers reconfiguration of services with new values

To learn more about service discovery with Docker, visit our guide [here](https://www.digitalocean.com/community/tutorials/the-docker-ecosystem-service-discovery-and-distributed-configuration-stores).

## Networking Tools

Containerized applications lend themselves to a service-oriented design that encourages breaking out functionality into discrete components. While this makes management and scaling easier, it requires even more assurance regarding the functionality and reliability of networking between the components. Docker itself provides the basic networking structures necessary for container-to-container and container-to-host communication.

Docker’s native networking capabilities provide two mechanisms for hooking containers together. The first is to expose a container’s ports and optionally map to the host system for external routing. You can select the host port to map to or allow Docker to randomly choose a high, unused port. This is a generic way of providing access to a container that works well for most purposes.

The other method is to allow containers to communicate by using Docker “links”. A linked container will get connection information about its counterpart, allowing it to automatically connect if it is configured to pay attention to those variables. This allows contact between containers on the same host without having to know beforehand the port or address where the service will be located.

This basic level of networking is suitable for single-host or closely managed environments. However, the Docker ecosystem has produce a variety of projects that focus on expanding the networking functionality available to operators and developers. Some additional networking capabilities available through additional tools include:

* Overlay networking to simplify and unify the address space across multiple hosts.
* Virtual private networks adapted to provide secure communication between various components.
* Assigning per-host or per-application subnetting
* Establishing macvlan interfaces for communication
* Configuring custom MAC addresses, gateways, etc. for your containers

Some projects that are involved with improving Docker networking are:

* **flannel**: Overlay network providing each host with a separate subnet.
* **weave**: Overlay network portraying all containers on a single network.
* **pipework**: Advanced networking toolkit for arbitrarily advanced networking configurations.

For a more in-depth look at the different approaches to networking with Docker, click [here](https://www.digitalocean.com/community/tutorials/the-docker-ecosystem-networking-and-communication).

## Native Docker Networking Implementation

Docker itself provides many of the networking fundamentals necessary for container-to-container and container-to-host communication.

When the Docker process itself is brought up, it configures a new virtual bridge interface called `docker0` on the host system. This interface allows Docker to allocate a virtual subnet for use among the containers it will run. The bridge will serve as the main point of interface between networking within a container and networking on the host.

When a container is started by Docker, a new virtual interface is created and given an address within the bridge’s subnet range. The IP address is hooked up to the container’s internal networking, providing the container’s network a path to the `docker0` bridge on the host system. Docker automatically configure `iptables` rules to allow for forwarding and configures NAT masquerading for traffic originating on `docker0` destined for the outside world.

### How Do Containers Expose Services to Consumers?

Other containers on the same host are able to access services provided by their neighbors without any additional configuration. The host system will simply route requests originating on and destined for the `docker0` interface to the appropriate location.

Containers can expose their ports to the host, where they can receive traffic forwarded from the outside world. Exposed ports can be mapped to the host system, either by selecting a specific port or allowing Docker to choose a random, high, unused port. Docker takes care of any forwarding rules and `iptables` configuration to correctly route packets in these situations.

### What is the Difference Between Exposing and Publishing a Port?

When creating container images or running a container, you have the option to expose ports or publish ports. The difference between the two is significant, but may not be immediately discernible.

Exposing a port simply means that Docker will take note that the port in question is used by the container. This can then be used for discovery purposes and for linking. For instance, inspecting a container will give you information about the exposed ports. When containers are linked, environmental variables will be set in the new container indicating the ports that were exposed on the original container.

By default, containers will be accessible to the host system and to any other containers on the host regardless of whether ports are exposed. Exposing the port simply documents the port use and makes that information available for automated mappings and linkings.

In contrast, publishing a port will map it to the host interface, making it available to the outside world. Container ports can either be mapped to a specific port on the host, or Docker can automatically select a high, unused port at random.

### What Are Docker Links?

Docker provides a mechanism called “Docker links” for configuring communication between containers. If a new container is linked to an existing container, the new container will be given connection information for the existing container through environmental variables.

This provides an easy way to establish communication between two containers by giving the new container explicit information about how to access its companion. The environmental variables are set according to the ports exposed by the other container. The IP address and other information will be filled in by Docker itself.

## Projects to Expand Docker’s Networking Capabilities

The networking model discussed above provides a good starting point for networking construction. Communication between containers on the same host is fairly straight-forward and communication between hosts can occur over regular public networks as long as the ports are mapped correctly and the connection information is given to the other party.

However, many applications require specific networking environments for security or functionality purposes. The native networking functionality of Docker is somewhat limited in these scenarios. Because of this, many projects have been created to expand the Docker networking ecosystem.

### Creating Overlay Networks to Abstract the Underlying Topology

One functional improvement that several projects have focused on is that of establishing overlay networks. An overlay network is a virtual network built on top of existing network connections.

Establishing overlay networks allows you to create a more predictable and uniform networking environment across hosts. This can simplify networking between containers regardless of where they are running. A single virtual network can span multiple hosts or specific subnets can be designated to each host within a unified network.

Another use of an overlay network is in the construction of fabric computing clusters. In fabric computing, multiple hosts are abstracted away and managed as a single, more powerful entity. The implementation of a fabric computing layer allows the end user to manage the cluster as a whole instead of individual hosts. Networking plays a large part of this clustering.

### Advanced Networking Configuration

Other projects expand Docker’s networking capabilities by providing more flexibility.

Docker’s default network configuration is functional, but fairly simple. These limitations express themselves most fully when dealing with cross-host networking but can also impede more customized networking requirements within a single host.

Additional functionality is provided through additional “plumbing” capabilities. These projects do not provide an out-of-the-box configuration, but they allow you to manually hook together pieces and create complex network scenarios. Some of the abilities you can gain range from simply establishing private networking between certain hosts, to configuring bridges, vlans, custom subnetting and gateways.

There are also a number of tools and projects that, while not developed with Docker in mind, are often used in Docker environments to provided needed functionality. In particular, mature private networking and tunneling technologies are often utilized to provide secure communication between hosts and among containers.



## What Are Some Common Projects for Improving Docker Networking?

There are a few different projects focused on providing overlay networking for Docker hosts. The common ones are:

* **flannel**: Developed by the CoreOS team, this project was initially developed to provide each host system with its own subnet of a shared network. This is a condition necessary for Google’s kubernetes orchestration tool to function, but it is useful in other situations.
* **weave**: Weave creates a virtual network that connects each host machine together. This simplifies application routing as it gives the appearance of every container being plugged into a single network switch.

In terms of advanced networking, the following project aims to fill that vacancy by providing additional plumbing:

* **pipework**: Constructed as a stop-gap measure until Docker native networking becomes more advanced, this project allows for easy configuration of arbitrarily advanced networking configurations.

One relevant example of existing software co-opted to add functionality to Docker is:

* **tinc**: Tinc is a lightweight VPN software that is implemented using tunnels and encryption. Tinc is a robust solution that can make the private network transparent to any applications.



## Scheduling, Cluster Management, and Orchestration

Another component needed when building a clustered container environment is a scheduler. Schedulers are responsible for starting containers on the available hosts.

![Schedule applications](https://assets.digitalocean.com/articles/docker_ecosystem/Example-Schedule-App-F.png)

The image above demonstrates a simplified scheduling decision. The request is given through an API or management tool. From here, the scheduler evaluates the conditions of the request and the state of the available hosts. In this example, it pulls information about container density from a distributed data store / discovery service (as discussed above) so that it can place the new application on the least busy host.

This host selection process is one of the core responsibilities of the scheduler. Usually, it has functions that automate this process with the administrator having the option to specify certain constraints. Some of these constraints may be:

* Schedule the container on the same host as another given container.
* Make sure that the container is not placed on the same host as another given container.
* Place the container on a host with a matching label or metadata.
* Place the container on the least busy host.
* Run the container on every host in the cluster.

The scheduler is responsible for loading containers onto relevant hosts and starting, stopping, and managing the life cycle of the process.

Because the scheduler must interact with each host in the group, cluster management functions are also typically included. These allow the scheduler to get information about the members and perform administration tasks. Orchestration in this context generally refers to the combination of container scheduling and managing hosts.

Some popular projects that function as schedulers and fleet management tools are:

* **[fleet](https://www.digitalocean.com/community/tutorials/how-to-use-fleet-and-fleetctl-to-manage-your-coreos-cluster)**: scheduler and cluster management tool.
* **[marathon](https://www.digitalocean.com/community/tutorials/an-introduction-to-mesosphere#a-basic-overview-of-marathon)**: scheduler and service management tool.
* **[Swarm](https://github.com/docker/swarm/)**: scheduler and service management tool.
* **[mesos](https://www.digitalocean.com/community/tutorials/an-introduction-to-mesosphere#a-basic-overview-of-apache-mesos)**: host abstraction service that consolidates host resources for the scheduler.
* **[kubernetes](https://www.digitalocean.com/community/tutorials/an-introduction-to-kubernetes)**: advanced scheduler capable of managing container groups.
* **[compose](https://github.com/docker/docker/issues/9694)**: container orchestration tool for creating container groups.

To find out more about basic scheduling, container grouping, and cluster management software for Docker, click [here](https://www.digitalocean.com/community/tutorials/the-docker-ecosystem-scheduling-and-orchestration).