## Sibling Containers explained

*Source ([Andrea Colangelo](https://medium.com/@andreacolangelo?source=post_page-----27dc02ff2686----------------------)):*

Docker Containers are among the best options when you need to start a repeatable task in a clean context. This is especially common when it comes to CI/CD. As your pipeline requires a step where you run tests against your code, you want to get results from a pristine environment.

A Docker Container comes in hand in that regard. They are great for fire-and-forget tasks like that, where the container does its job and then it’s trashed immediately after. Nevertheless, the image it came from can be used to spawn a brand new container for another round of tests. *Wash, rinse, repeat*.

### Docker-in-docker

But what if you run your CI/CD pipelines with tools, like Jenkins or Travis, which are already running inside a Docker Container? If you run a container from inside another container, you would end up with a Docker-in-Docker scenario. This was a mandatory choice up to a few years ago, but it brings some bad news and some other very bad news, including:

* need to copy the images on the parent container (or mount a volume at least);
* issues with AppArmor, SELinux, and whatnot;
* issues running a copy-on-write filesystem (the child container) on top of a normal filesystem (the parent container);
* data corruption caused by dirty tricks and shenanigans to fix the problems above.

Nowadays we can start a container outside of the current container from within the container. A sibling rather than a child indeed. This magic can be done exploiting a key concept in Unix: *everything is a file*.

### Sibling Containers under the hood

***TL;DR\****: Docker CLI is connected to a server called Docker Engine via a socket, that is: a file on the filesystem. If you mount this file as a volume inside a container and run the Docker CLI there, you are talking to the Docker Engine on the host.*

Docker might seem a monolithic program, but it is comprised of two main parts instead:

* the CLI you usually interface with via the `docker` command;
* a daemon running in the background which is in charge of executing all the requests it gets via the CLI utility or any other mean (APIs, programming libraries, etc.) called Docker Engine.

So, despite you never directly use the Docker Engine, that’s where all the dirty work gets done. The Engine is responsible for managing images, containers and all that jazz in the first place. `docker` is just your dashboard to control it.

Now, the CLI connects to the Engine via the Docker APIs on a dedicated socket which is usually located on the path `/var/run/docker.sock` But we know everything is a file in Unix, right? So we could treat it like any other file, right? Even mount that path as a volume on the Container we want to start our sibling from, right?

### Issue commands to the host’s Docker from within a Container

At the end of the day, it all comes down to installing the Docker CLI (or any other tool that can talk to the Docker APIs) in our container and adding a very simple option to our `docker` invocation:

```
-v /var/run/docker.sock:/var/run/docker.sock
```

or declare a volume likewise for the parent’s service if you are orchestrating a composition of containers with Docker Compose:

```
volumes:      
  - /var/run/docker.sock:/var/run/docker.sock
```

The Docker socket inside the container is shadowed by the socket on the host, thus wiring up the encapsulated environment of the container with the outer world.

When your process inside the container sends requests to the Docker Engine, it blindly connects to the inner socket, but what is listening there is the Engine on the host. Hence your request to run a new container is sent to the host, thus creating a sibling to the container you are into.