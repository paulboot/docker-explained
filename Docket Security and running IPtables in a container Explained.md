# Docket Security and running IPtables in a container Explained

[toc]



# Secure Docker with iptables firewall and Ansible

* Post authorBy Ryan Daniels

**Source:** https://ryandaniels.ca/blog/secure-docker-with-iptables-firewall-and-ansible/#problems-i-need-to-solve

Out of the box, security with Docker (and Docker Swarm) over the network is bad. Okay, that’s not entirely true. Out of the box when you have no containers started, it’s fine. But after you start a container, and if you publish a port, they are exposed to the outside world by default. And it’s not easy to fix. You need to create a custom Docker firewall with iptables.

Let’s discuss the background of firewall issues with Docker, and a working solution for my use case (either setup manually or using Ansible). By the end we will use a firewall on the server to lock down everything by default, only allowing my trusted IPs! With the option to open specified ports publicly (like SSH).

Note: This solution works with CentOS 7, RHEL 7, Ubuntu 18.04, and Ubuntu 20.04.



Table Of Contents

1. [Background](https://ryandaniels.ca/blog/secure-docker-with-iptables-firewall-and-ansible/#background)
2. [Roll your own solution](https://ryandaniels.ca/blog/secure-docker-with-iptables-firewall-and-ansible/#roll-your-own-solution)
3. [Problems I need to solve](https://ryandaniels.ca/blog/secure-docker-with-iptables-firewall-and-ansible/#problems-i-need-to-solve)
4. Solution – Docker firewall with iptables and ipset
   * [High level summary](https://ryandaniels.ca/blog/secure-docker-with-iptables-firewall-and-ansible/#high-level-summary)
5. [Warnings](https://ryandaniels.ca/blog/secure-docker-with-iptables-firewall-and-ansible/#warnings)
6. The manual way
   * [Prep](https://ryandaniels.ca/blog/secure-docker-with-iptables-firewall-and-ansible/#prep)
   * [Configure ipset](https://ryandaniels.ca/blog/secure-docker-with-iptables-firewall-and-ansible/#configure-ipset)
   * [Configure iptables](https://ryandaniels.ca/blog/secure-docker-with-iptables-firewall-and-ansible/#configure-iptables)
7. [The Automatic way with Ansible](https://ryandaniels.ca/blog/secure-docker-with-iptables-firewall-and-ansible/#the-automatic-way-with-ansible)
8. [Conclusion](https://ryandaniels.ca/blog/secure-docker-with-iptables-firewall-and-ansible/#conclusion)
9. References
   * [Background for Docker’s undocumented use of iptables INPUT chain](https://ryandaniels.ca/blog/secure-docker-with-iptables-firewall-and-ansible/#background-for-dockers-undocumented-use-of-iptables-input-chain)
   * [References and links](https://ryandaniels.ca/blog/secure-docker-with-iptables-firewall-and-ansible/#references-and-links)



## Background

![Docker punches a hole through firewall](images/Docket Security and running IPtables in a container Explained/punch_hole_eye-400x600.jpg)Image credit: [Jonathan Borba](https://www.pexels.com/photo/brown-human-eye-2873058/)

Firstly, even if you were using a firewall like iptables, Docker makes that useless. Docker punches a whole right through your firewall!

And if you [try another firewall, like firewalld](https://ryandaniels.ca/blog/ansible-manage-firewalld/)? Docker Swarm (and even regular Docker) with firewalld is a complete mess. Restart the firewalld service, or change the firewalld config, and you lost all the config that Docker needed. Now you have to restart Docker! What happens if the firewalld service failed and restarted? That Docker Swarm node is out of service.

Keep in mind, when I mention Docker, it means the regular Docker Engine. Docker Swarm means SwarmKit, which is the newer way of using Swarm. (The old way is the standalone solution which is old and not referenced at all here).

There are many attempts to solve this problem by users of Docker. Unfortunately the Docker team has been pretty quiet about these issues. They recommend a manual user solution, and to disable Docker’s use of iptables. I’m speculating here, but it seems like any future change from Docker will likely be a breaking change since this is a complicated issue to fix.

The attempted solutions (from users) are not very straight forward for a normal Docker user. And that’s the real issue. On top of that, out of the box (after you start one service with a published port), there is no security at the network layer. Anyone can connect to an exposed container’s port. Most importantly, even if you think you have a firewall protecting you.. Wrong, you don’t! With normal Docker, you can bind your service to your localhost which helps. But what about Docker Swarm? Nope, that doesn’t work.

There’s a great article about “[The problem of forcing users to make choices (in security)](https://utcc.utoronto.ca/~cks/space/blog/tech/SecurityChoiceProblem).” Definitely worth the read!



## Roll your own solution

Until this is actually addressed in Docker, our only hope is to find the simplest solution possible. And turning off iptables integration in Docker is unacceptable (which is constantly recommended by the Docker developers). The other option is to move on from Docker and/or Docker Swarm. I hear this thing called Kubernetes is pretty great. Anyways, back to Docker.
Many people have spent hours trying to learn, and figure out iptables as a solution to this. You need to roll your own solution apparently! So iptables is probably the best approach, since that is what Docker needs to use to do it’s magic (and most other firewalls are just a wrapper for iptables anyway depending on your OS).

Something fun I found out while testing this, [Docker Swarm uses iptables in an undocumented way](https://ryandaniels.ca/blog/docker-iptables-input-chain/). Docker Swarm uses the iptables INPUT chain! It’s only for encrypted overlay networks. But it’s not very fun realizing that! All of a sudden rules are being appended to the INPUT chain.

Okay, enough backstory. On with my futile attempt to roll my own solution. This took way longer than I thought it would! But it does work! Currently it works at least.. (That’s why I use the word futile!)



## Problems I need to solve

1. Only allow traffic from multiple “trusted” IP addresses to my servers. Not all of these IPs will be in the same “IP block/range” either. This will be to all services running directly on the server, and also all of the Docker containers.
2. Let only specific ports be publicly accessible, like SSH.
3. I’m not managing which containers are accessible through the firewall. Meaning, I’m not manually adding ports into my firewall solution. That kind of manual work is not happening. I need a dynamic, and flexible solution that blocks by default except to my trusted IPs.
4. The firewall solution must be simple. More complex means more room for error.
5. The firewall solution must not impact performance significantly.
6. Restarting the firewall won’t break Docker.
7. Restarting Docker won’t break the firewall.
8. No impact to running server processes or Docker services when making a change. Things need to keep working!
   Firewall changes need to happen online and not impact Docker. Meaning I can’t be restarting Docker because I made a firewall change.
   Docker “changes” need to happen online. Meaning I can’t be restarting the firewall because I made a Docker change. (A Docker “change” means starting/stopping a container).

That sounds very simple! Unfortunately, it is not with Docker (and Docker Swarm).



## Solution – Docker firewall with iptables and ipset

If you don’t know much about iptables, or ipset, that’s okay. You don’t really need to know. You should have some basic understandings though, so you don’t break your servers! The [Arch Linux wiki](https://wiki.archlinux.org/index.php/Iptables#Chains) has great information about iptables. Including this helpful [visual](https://www.frozentux.net/iptables-tutorial/chunkyhtml/images/tables_traverse.jpg) about the iptables flow.

Note: This solution works with CentOS 7, RHEL 7, Ubuntu 18.04, and Ubuntu 20.04.



### High level summary

iptables with ipset will handle all of this for us. And keep our servers, and Docker locked down from the network level.

In this solution, we will use the iptables INPUT chain to jump to another chain (let’s call our custom chain FILTERS), but return if there’s some legitimate looking traffic, so the Swarm overlay can do whatever it wants in INPUT with IPSEC, or whatever it is appending to INPUT.
Inside our custom chain FILTERS, we drop everything that doesn’t match our trusted list of IPs. We also allow our SSH port and the basic default iptables stuff.. You can also add any OS port to be publicly accessible.
The DOCKER-USER chain only needs a few entries. Any internal Docker traffic is returned, and it will drop any other traffic that’s not in our allowed IP list. You can also add any container port to be publicly accessible.

One of the dangers with this approach is if Docker changes it’s behaviour our firewall could break, or our Docker services could stop working. Since Docker doesn’t offer any solution for their users, we need our own solution. So keep in mind that you need to test this when upgrading to a new version of Docker. That is the trade-off with a “roll your own” solution. But what choice do we have?

I’ve created an [Ansible Role: iptables for Docker](https://galaxy.ansible.com/ryandaniels/iptables_docker), on [GitHub](https://github.com/ryandaniels/ansible-role-iptables-docker) and [Ansible Galaxy](https://galaxy.ansible.com/ryandaniels/iptables_docker).



## Warnings

**Warning: Be sure you have everything needed in your configuration. Once the iptables firewall is started it blocks anything that wasn’t added! Don’t lock yourself out of your server. Be sure to have another way to connect, like a console.**

**Disclaimer: Keep in mind, you should test all of this in your lab or staging environments. I can’t guarantee this will be 100% safe and can’t be held responsible for anything going wrong!**

**SELinux Bug**: If using SELinux, currently there’s a bug with SELinux which prevents saving the iptables rules to the iptables.save file.
**Impact**: Saving the iptables rules a 2nd time will silently fail. Workaround has been added so SELinux allows chmod to interact with the iptables.save file. [See notes on GitHub for SELinux workaround steps](https://github.com/ryandaniels/ansible-role-iptables-docker/blob/master/README.md#selinux-manual-workaround-for-iptables-and-chmod). Alternatively you could disable SELinux, but that’s not recommended. Bug report: https://bugs.centos.org/view.php?id=12648



## The manual way

Run the commands below. These commands are only for CentOS/RHEL 7. If you don’t want to do this manually, jump to the [Automatic section, using Ansible](https://ryandaniels.ca/blog/secure-docker-with-iptables-firewall-and-ansible/#the-automatic-way-with-ansible) (which also works with Ubuntu).



### Prep

Make note of what you already have in iptables (if you are already using it). Be sure you have some background with iptables, since you could break things!

```
# iptables -nvL --line-numbers
```

Install the required packages for CentOS / RHEL:

```
# yum install iptables iptables-services ipset ipset-service
```



### Configure ipset

ipset allows you to add a list of IPs that you can use with iptables. In our case, we will add a list of IPs we want to be able to connect to our servers.

Configure ipset with a setname of `ip_allow`.
Add IPs you want to allow. Change the IPs below to your actual trusted/allowed IP ranges. Be sure to include your Docker server IPs here, because if you don’t they can’t communicate with eachother:

```
# mkdir -p /etc/sysconfig/ipset.d
# vi /etc/sysconfig/ipset.d/ip_allow.set

create -exist ip_allow hash:ip family inet hashsize 1024 maxelem 65536
add ip_allow 192.168.1.123
add ip_allow 192.168.100.0/24
add ip_allow 192.168.101.0/24
add ip_allow 192.168.102.0/24
```

Start, and Enable the ipset service:

```
# systemctl status ipset
# systemctl start ipset
# systemctl enable ipset
```

See what ipset has in it’s loaded configuration:

```
# ipset list | head
```

Important: Make note of the size of “Number of entries”. If that number is close to the maxelem size (65536), then you need to delete the ipset and re-create it with a larger max size. If you only use a few IP ranges like above, you don’t need to worry and will be well below the limit.



### Configure iptables

Next up, iptables. iptables is our solution for a firewall. We will create a file with our rules and then add those rules into iptables. The important part is to not flush the existing rules if you are already using Docker (or Docker Swarm) on your server.

Create an iptables file to use with iptables-restore, to add the rules into iptables:

```
# vi iptables-rules.txt
```

Add below to the file. There is a lot going on here..

```
*filter
:DOCKER-USER - [0:0]
:FILTERS - [0:0]
#Can't flush INPUT. wipes out docker swarm encrypted overlay rules
#-F INPUT
#Use ansible or run manually once instead to add -I INPUT -j FILTERS
#-I INPUT -j FILTERS
-F DOCKER-USER
-A DOCKER-USER -m state --state RELATED,ESTABLISHED -j RETURN
-A DOCKER-USER -i docker_gwbridge -j RETURN
-A DOCKER-USER -s 172.18.0.0/16 -j RETURN
-A DOCKER-USER -i docker0 -j RETURN
-A DOCKER-USER -s 172.17.0.0/16 -j RETURN
#Below Docker ports open to everyone if uncommented
#-A DOCKER-USER -p tcp -m tcp -m multiport --dports 8000,8001 -j RETURN
#-A DOCKER-USER -p udp -m udp -m multiport --dports 9000,9001 -j RETURN
-A DOCKER-USER -m set ! --match-set ip_allow src -j DROP
-A DOCKER-USER -j RETURN
-F FILTERS
#Because Docker Swarm encrypted overlay network just appends rules to INPUT. Has to be at top unfortunately
-A FILTERS -p udp -m policy --dir in --pol ipsec -m udp -m set --match-set ip_allow src --dport 4789 -j RETURN
-A FILTERS -m state --state RELATED,ESTABLISHED -j ACCEPT
-A FILTERS -p icmp -j ACCEPT
-A FILTERS -i lo -j ACCEPT
#Below OS ports open to everyone if uncommented
-A FILTERS -p tcp -m state --state NEW -m tcp -m multiport --dports 22 -j ACCEPT
#-A FILTERS -p udp -m udp -m multiport --dports 53,123 -j ACCEPT
-A FILTERS -m set ! --match-set ip_allow src -j DROP
-A FILTERS -j RETURN
COMMIT
```

Use iptables-restore to add the above rules into iptables. The very important flag is `-n`. That makes sure we don’t flush the iptables rules if we have rules already in Docker (or Docker Swarm).

```
# iptables-restore -n < iptables-rules.txt
```

Next, add a rule to the INPUT chain, so we start using the new rules in FILTERS. It has to be at the top, and only needs to be added once:

```
# iptables -I INPUT 1 -j FILTERS
```

Save the iptables rules:

```
# /usr/libexec/iptables/iptables.init save
```

That will save any existing and our new iptables rules to the iptables configuration file so it will be persistent after a reboot.

In addition, it was needed to run the above iptables command manually since we want to ensure it’s only inserted once. And we can’t flush the INPUT chain to ensure that since Docker Swarm could have rules there already.

Start and Enable the iptables service:

```
# systemctl status iptables
# systemctl start iptables
# systemctl enable iptables
```

If you want to customize the iptables rules to allow more ports to be open to everyone, just add the port to the appropriate rule in the iptables file (tcp or udp), then re-run the same commands from above:

```
# iptables-restore -n < iptables-rules.txt
# /usr/libexec/iptables/iptables.init save
```

**Don’t miss the [Warnings](https://ryandaniels.ca/blog/secure-docker-with-iptables-firewall-and-ansible/#warnings) from above! Especially about SELinux.**

If you don’t want to do all of that manually, and you use Ansible, then do this instead..



## The Automatic way with Ansible

Manually run all of the above, on every Docker server is not ideal. Let’s use Ansible instead!

I’ve created an [Ansible Role: iptables for Docker](https://galaxy.ansible.com/ryandaniels/iptables_docker), on [GitHub](https://github.com/ryandaniels/ansible-role-iptables-docker) and [Ansible Galaxy](https://galaxy.ansible.com/ryandaniels/iptables_docker).

This works on CentOS 7, RHEL 7, Ubuntu 18.04, and Ubuntu 20.04.

Install the Ansible Role using Ansible Galaxy:

```
$ ansible-galaxy install ryandaniels.iptables_docker
```

Or, clone the GitHub project:

```
$ git clone https://github.com/ryandaniels/ansible-role-iptables-docker.git roles/ryandaniels.iptables_docker
```

Create the Ansible Playbook, called iptables_docker.yml:

```
---
- hosts: '{{ inventory }}'
  become: yes
  vars:
    # Use this role
    iptables_docker_managed: true
  roles:
  - ryandaniels.iptables_docker
```

Make configuration changes to add desired IP addresses and ports as needed.

**Don’t miss the [Warnings](https://ryandaniels.ca/blog/secure-docker-with-iptables-firewall-and-ansible/#warnings) from above!**

Then run the playbook:

```
$ ansible-playbook iptables_docker.yml --extra-vars "inventory=centos7" -i hosts-dev
```



## Conclusion

In conclusion, now we have secured our Docker (and Docker Swarm) environments using Ansible to perform the installation and configuration of iptables! None of our Docker published ports are exposed to the world, unless we want them to be! We have created a custom Docker firewall with iptables. Hopefully, some day this will be the default behaviour and shipped with Docker out of the box! Dare to dream. Security is hard.



## References



### Background for Docker’s undocumented use of iptables INPUT chain

[See my previous post about this](https://ryandaniels.ca/blog/docker-iptables-input-chain/).



### References and links

References, notes, and links about the Docker firewall discussion:

* [Docker Documentation – Overlay Networks](https://docs.docker.com/network/overlay/#encrypt-traffic-on-an-overlay-network)
* [Docker bypasses ufw firewall rules](https://github.com/docker/for-linux/issues/690)
* [unrouted](https://unrouted.io/2017/08/15/docker-firewall/) – Solution using iptables. Not for Swarm. It clobbers the INPUT chain, which is used by encrypted overlay with Docker Swarm
* [The big thread about Docker and a firewall](https://github.com/moby/moby/issues/22054)
* [Arch Linux wiki to the rescue to show iptables](https://wiki.archlinux.org/index.php/Iptables#Chains) flow which links to a [great visual](https://www.frozentux.net/iptables-tutorial/chunkyhtml/images/tables_traverse.jpg)