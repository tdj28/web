---
title: "Abstracting Networking with Docker Containers"
date: 2016-03-03T19:21:05-07:00
draft: false
comments: false
summary: Docker comes with a default network, but you can make your own network of unlimited complexity.
tags:
  - devops
  - sre
  - networking
categories:
  - deep-dive
  - software
---

# Goal:

Network Namespace is a Linux tool that allows for the easy virtualization of network models. While it has plenty of [practical uses directly on the hardware](http://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/), this will be a quick introduction to how it can be used to create more complex Docker networks that could be extended to modeling, say, the infrastructure of a data center or cloud provider.

[Brandon Rhodes created](https://github.com/brandon-rhodes/fopnp/tree/m/playground) a great "playground" for his book [Foundations of Python Network Programming](https://github.com/brandon-rhodes/fopnp). Inspired by this example, we will create a small simple netns/docker example that is more bite-sized for those who aren't familiar with these tools.

#### What is NetNS?
Very briefly, netns is a tool that allows us to create virtual network namespaces that are isolated from each other.

Let's let its man page speak for itself:

>A network namespace is logically another copy of the network stack, with its own routes, firewall rules, and network devices.

>By default a process inherits its network namespace from its parent.Initially all the processes share the same default network namespace from the init process.

One's imagination can fly away at this point thinking of the multitude of possibilities this offers for security, but that should be a focus of a separate entry.

Creating a new namespace, here let's call it newtonsapple for the sake of example, is as easy as

    :::bash
    ip netns add newtonsapple

Of course there is much, much more to it than that, but we will see more details about what netns can do below.
#### What is Docker?
Chances are pretty high that if you found this page, you have a pretty good idea of what Docker is. I will put it in the context of our netns discussion. In some ways, Docker is to the OS what netns is the machine's networking configuration. Docker is a way to run processes in an isolated environment that does not use virtualization of hardware, but rather has direct access to the hardware via having direct access to the kernel. There are plenty of places out there that will provide fantastic introductions to Docker, but for the sake of this entry, it helps to think of netns and Docker as sort of cousins in that they both create isolated sandboxes which can interact in controlled ways with the _real_ OS/network outside of their bubbles (in the case of Docker this bubble is called a **container** and in case of netns this bubble is called a **namespace**).

###### Docker Networking

Docker has a great [tutorial on Docker Networking](https://docs.docker.com/engine/userguide/networking/) and it would be inefficient to recreate that here in any way. Instead I want to highlight the key parts that are relevant to this entry.

The first thing to note is that Docker automatically creates its own network. If you do an `ifconfig` or an `ip a` on a machine running a Docker server, you will find an entry corresponding to this network listed in the output:

```bash
docker0   Link encap:Ethernet  HWaddr 56:84:7a:fe:97:99  
          inet addr:172.17.42.1  Bcast:0.0.0.0  Mask:255.255.0.0
 ...
```
Docker also creates a network internal to each container (loopback) and you can launch a container that hooks directly in to the host network (`docker run --net=host`). The default is to launch it on the docker0 network, though it is often useful to run it on the host network if, for example, it needs access to VPN connections and it isn't worth the effort to create further bridges. However, an interesting option is `--net=none`. This tells Docker to not touch the networking of the container and allows us to create our own networking for the containers.

#### Using NetNS with Docker to Model Your Home Network


###### Preliminary setup
This post will assume you are working from Ubuntu/Debian, though any Linux distribution will have these tools.

On Ubuntu, you have to create a directory `/var/run/netns` in order to use netns:
```bash
sudo mkdir -p /var/run/netns
```
Though this wasn't my experience, you may have to enable two Linux kernel modules as well:

```bash
sudo modprobe ip_nat_ftp nf_conntrack_ftp
```

You may also need to install bridge-utils if it isn't already installed:

```bash
sudo apt-get update
sudo apt-get install bridge-utils
```

###### Bringing up the containers and linking netns

```bash
docker run --net=none --dns=8.8.8.8 --name=verizon -d ubuntu /bin/sh -c "while true; do echo ""; done"
pid=$(docker inspect -f '{{.State.Pid}}' verizon)
sudo ln -s /proc/$pid/ns/net /var/run/netns/verizon

docker run --net=none --dns=8.8.8.8 --name=router -d ubuntu /bin/sh -c "while true; do echo ""; done"
pid=$(docker inspect -f '{{.State.Pid}}' router)
sudo ln -s /proc/$pid/ns/net /var/run/netns/router

docker run --net=none --dns=8.8.8.8 --name=laptop -d ubuntu /bin/sh -c "while true; do echo ""; done"
pid=$(docker inspect -f '{{.State.Pid}}' laptop)
sudo ln -s /proc/$pid/ns/net /var/run/netns/laptop
```


###### Creating Network Interfaces


```bash
root@vagrant-ubuntu-trusty-64:~/code# ip netns exec verizon ip link list
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
10: eth1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether f6:8b:e9:66:00:b0 brd ff:ff:ff:ff:ff:ff
root@vagrant-ubuntu-trusty-64:~/code# ip netns exec router ip link list                                                                                        
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
4: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 36:7a:68:d8:a5:84 brd ff:ff:ff:ff:ff:ff
9: eth0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast state DOWN mode DEFAULT group default qlen 1000
    link/ether fe:60:62:9d:1b:7b brd ff:ff:ff:ff:ff:ff
root@vagrant-ubuntu-trusty-64:~/code# ip netns exec laptop ip link list                                                                                        
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
7: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether f2:ee:9e:56:55:d4 brd ff:ff:ff:ff:ff:ff
root@vagrant-ubuntu-trusty-64:~/code# brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.56847afe9799       no
home            8000.821c2640a1ea       no              laptop_eth1

```

```bash
### Create network namespaces
ip netns add levela
ip netns add levelb
ip netns add levelc

### Create peer devices
ip link add veth0a type veth peer name veth1a
ip link add veth0b type veth peer name veth1b
ip link add veth0c type veth peer name veth1c

### Put these devices in namespaces
# veth0a remains in globalspace
ip link set veth1a netns levela
ip link set veth0b netns levela
ip link set veth1b netns levelb
ip link set veth0c netns levelb
ip link set veth1c netns levelc

### Set up Networks
ip netns exec levela ifconfig veth1a 172.16.1.0/24 up
ip netns exec levelb ifconfig veth1b 10.1.1.1/24 up
ip netns exec levelc ifconfig veth1c 192.168.1.1/24 up
```

At thist point we should be able to observer network namespaces and corresponding devices with their networks:

```bash
# ip netns exec levela ip addr list

1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
6: veth1a: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast state DOWN group default qlen 1000
    link/ether ba:8c:ec:6e:71:db brd ff:ff:ff:ff:ff:ff
    inet 172.16.1.0/24 brd 172.16.1.255 scope global veth1a
       valid_lft forever preferred_lft forever
9: veth0b: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 82:88:3d:09:e7:1f brd ff:ff:ff:ff:ff:ff

# ip netns exec levelb ip addr list

1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
8: veth1b: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast state DOWN group default qlen 1000
    link/ether c2:c4:9c:2b:41:a5 brd ff:ff:ff:ff:ff:ff
    inet 10.1.1.1/24 brd 10.1.1.255 scope global veth1b
       valid_lft forever preferred_lft forever
11: veth0c: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 96:a5:1c:5b:e3:8c brd ff:ff:ff:ff:ff:ff

# ip netns exec levelc ip addr list

1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
10: veth1c: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast state DOWN group default qlen 1000
    link/ether 66:d7:ff:9b:65:63 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.1/24 brd 192.168.1.255 scope global veth1c
       valid_lft forever preferred_lft forever
```

and corresponding routes:

```bash
# ip netns exec levela ip route list

172.16.1.0/24 dev veth1a  proto kernel  scope link  src 172.16.1.0

# ip netns exec levelb ip route list

10.1.1.0/24 dev veth1b  proto kernel  scope link  src 10.1.1.1

# ip netns exec levelc ip route list

192.168.1.0/24 dev veth1c  proto kernel  scope link  src 192.168.1.1
```
