# Systemd Containers and systemd-nspawn Explained

[toc]

# TL;DR



Source: https://blog.selectel.com/systemd-containers-introduction-systemd-nspawn/

# Introduction

Containerization has become an increasingly relevant topic. There are already thousands, if not tens of thousands, of articles and posts written about popular solutions like LXC and Docker.
In today’s article, we’d like to discuss [systemd-nspawn](https://www.freedesktop.org/software/systemd/man/systemd-nspawn.html), a systemd component for creating isolated environments. Systemd is already a standard in the world of Linux and in light of this, it wouldn’t be unfounded to suggest that the potential for systemd-nspawn will significantly expand in the near future. For this reason, we think now would be a good time to better acquaint ourselves with this tool.

## systemd-nspawn: General Information

The name systemd-nspawn is an abbreviation of *namespaces spawn*. From this name alone we can see that systemd-nspawn only manages isolated processes; it cannot isolate resources (however, this can be done with systemd, which we’ll talk about later on).

Using systemd-nspawn, we can create a fully isolated environment, which will automatically monitor the /proc and /sys pseudo-file systems and create an isolated loopback interface and separate name space for process identifiers (PID). Inside these spaces, we can launch Linux-based operating systems.

Unlike Docker, systemd-nspawn does not have a special image repository, but images can be created and uploaded using any third-party program. tar, raw, qcow2, and dkr (the Docker image format; this isn’t written anywhere in the systemd-nspawn documentation and its author made quite an effort to avoid using the word *Docker*) image formats are supported. Images are managed based on the [btrfs file system](https://btrfs.wiki.kernel.org/index.php/Main_Page).

## Launching Debian in a Container

In this introduction to systemd-nspawn, we’ll start with a simple, yet extremely practical example.
We’ll create an isolated environment for launching Debian on a server running Fedora. All commands given below are for Fedora 22 and version 219 of systemd; commands may be different for different Linux distributions and versions of systemd.

We start by installing the necessary dependencies:

```
# sudo dnf install debootstrap bridge-utils
```

Then we create a file system for the future container:

```
# debootstrap --arch=amd64 jessie /var/lib/machines/container1/
```

Once we’ve finished our prep, we can launch the container:

```
# systemd-nspawn -D /var/lib/machines/container1/ --machine test_container
```

A prompt for the guest operating system will appear in the console:

```
# root@test_container
```

We set the root password:

```
# passwd
Enter new UNIX password: 
Retype new UNIX password: 
passwd: password updated successfully
```

Now we leave the container by entering the keyboard combination Ctrl+ ]]] (some keyboards will use % instead of ]), and then run the following command:

```
# systemd-nspawn -D /var/lib/machines/container1/ --machine test_container  -b
```

Here we have the -b (or –boot) flag, which indicates that when an operating system is launched in the container, init should be run each time a daemon is launched. This flag can only be used if the OS launched within the container supports systemd. If it doesn’t, there’s no guarantee the system will load.

After this, the system will prompt you for the login and password.

And there you have it! A complete operating system has just been launched in an isolated environment. Now we need to configure the network. We leave the container and build a bridge for connecting to the interface on the main host:

```
# brctl addbr cont-bridge
```

We assign the bridge an IP address:

```
# ip a a [IP Address] dev cont-bridge
```

Afterwards, we run the command:

```
# systemd-nspawn -D /var/lib/machines/container1/ --machine test_container --network-bridge=cont-bridge -b
```

We can also set up a network using the option –network-ipvlan, which connects the container to a given interface on the primary host using ipvlan:

```
# systemd-nspawn -D /var/lib/machines/container1/ --machine test_container -b --network-ipvlan=[network interface]
```

## Launching a Container as a Service

With systemd, containers can be configured to automatically launch with the system. To do this, we add the following configuration file to the directory /etc/systemd/system:

```
[Unit]
Description=Test Container

[Service]
LimitNOFILE=100000
ExecStart=/usr/bin/systemd-nspawn --machine=test_container --directory=/var/lib/machines/container1/ -b --network-ipvlan=[network interface] 
Restart=always

[Install]
Also=dbus.service
```

Let’s look at this fragment piece by piece. Under [Description] we enter the container name. Under [Service] we firstly set a limit on the permissible number of open files in the container (LimitNOFILE), then we enter the command to launch the container with the necessary options (ExecStart). Restart=always means the container should restart in the event of a crash. Under [Install] we given the additional unit that should be added to the host’s autolaunch (in our case, this is the inter-process communication system D-Bus).

We save our changes to the configuration file and run the following command:

```
# systecmctl start test_container
```

There are other, less complicated ways to launch a container as a service. Systemd has a configuration file template for automatically launching containers saved in the /var/lib/machines directory. We can enable a launch based on this template with the following command:

```
# systemctl enable machine.target
 # mv ~/test_container /var/lib/machines/test_container
 # systemctl enable systemd-nspawn@test_container.service
```

## Managing Containers: machinectl

Containers can be managed with the machinectl utility. We’ll take a brief look at its basic options.

To print a list of containers in the system:

```
# machinectl list
```

To view information on a container’s status:

```
# machinectl status test_container
```

To log into a container:

```
# machinectl login test_container
```

To restart a container:

```
# machinectl reboot test_container
```

To turn off a container:

```
# machinectl poweroff test_container
```

The last command will work if a systemd-compatible operating system is installed in the container. For operating systems using sysvinit, we have to use the terminate option.

We talked a bit about machinectl’s most basic features; for more detailed instructions, see [here](https://www.freedesktop.org/software/systemd/man/machinectl.html).

## Uploading Images

We’ve already mentioned that systemd-spawn can be used to load images in a variety of formats. There is, however, one important thing to remember: only images built on btrfs can be used, which must be mounted to the /var/lib/machines directory:

```
# dnf install btrfs-progs
# mkfs.btrs /dev/sdb
# mount /dev/sdb /var/lib/machines
# mount | grep btrfs

dev/sdb on /var/lib/machines type btrfs (rw,relatime,seclabel,space_cache)
```

If there are no disks available, Btrfs can write it to a file.

In newer versions of systemd, images can be uploaded out of the box without mounting Btrfs.

We upload a Docker image:

```
# machinectl pull-dkr --verify=no library/redis --dkr-index-url=https://index.docker.io
```

Afterwards, loading the container built on the uploaded image is simple:

```
# systemd-nspawn --machine-redis
```

## Viewing Container Logs

Information on the events that occur in a container are recorded in a log. Logging can be configured when a container is created using the –link-journal option. For example:

```
# systemd-nspawn -D /var/lib/machines/container1/ --machine test_container -b --link-journal=host
```

This command means container logs will be saved on the main host in the directory /var/log/journal/machine-id. If we use the option –link-journal=guest, then all of the logs will be saved in the container in the /var/log/journal/machine-id directory, and a symbolic link will be created on the main host to the directory with the saved address. The option –link-journal will only work if the system launched in the container has systemd installed; if it doesn’t, there’s no guarantee that logging will work properly.

Information on the launch and shutdown of containers can be viewed using journalctl, which we wrote about in a [previous article](https://blog.selectel.com/managing-logging-systemd/):

```
# journalctl -u .service
```

Journalctl can be used to view logs of container events. This is done using the -M option (we’ll display just a small fragment from the printout):

\# journalctl -M test_container Sep 18 11:50:21 octavia.localdomain systemd-journal[16]: Runtime journal is using 8.0M (max allowed 197.6M, trying to leave 296.4M free of 1.9G available >E28692E28692

## Dedicated Resources

We’ve already looked at the main features of systemd-nspawn. There’s only one key element left: dedicated container resources. As we’ve said, systemd-nspawn resources are not isolated. Using systemctl, limits can be set on the resources a container can consume, for example:

```
# systemctl set-property  CPUShares=200 CPUQuota=30% MemoryLimit=500M
```

Limits on the resources containers can access may be recorded in the unit file or [Slice] section.

## Conclusion

Systemd-nspawn is an interesting and prospective tool. Among its obvious advantages, it’s worth noting:

* integration with different systemd components;
* compatibility with various image formats;
* self-contained; there’s no need to install extra packets or kernel patches.

Of course it’s too early to list all of systemd-nspawn’s potential uses for production: the tool is still fairly raw and can only be used for experimenting with. However, considering how widespread systemd is going to be, it’s worth waiting for a more complete systemd-nspawn.