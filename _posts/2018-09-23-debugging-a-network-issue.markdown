---
layout: post
title:  "Investigating a container network issue"
date:   2018-09-23 11:57:48 +0530
published: false
categories: docker networking
---

At my workplace, we are transitioning from running services under [supervisor](http://supervisord.org/index.html) to [docker](https://www.docker.com/). I have also been tasked to move some of the services and had to resolve a network issue which turned out to be non-trivial. The service in question had two requirements as far as our discussion is concerned -
* all the network traffic inside the container needed to be routed through a VPN server
* the service should be able to forward logs to the host machine at a particular port.

A simple duckduckgo search returned several results and I was able to set it up in couple of hours. I used [dperson/openvpn-client](https://github.com/dperson/openvpn-client) to setup the VPN client. I figured out that I could use docker's default bridge network to connect to the host. On this network, the host is available at the first address, this was `172.17.0.1` on my computer.

Tested in my dev setup, submitted for review, and got it approved. All good. Except it wasn't when I tried it on one of the staging machine. The VPN part was fine but the logging part was failing causing weird issues to pop up. Connections to the host machine were timing out. I cross-checked everything to see if I made a typo somewhere and everything was fine.

Then I ran traceroute inside the container to the host
```
$ traceroute 172.17.0.1
traceroute to 172.17.0.1 (172.17.0.2), 30 hops max, 60 byte packets
 1  * * *
 2  * * *
 3  * * *
 4  * * *
...
```

This means that the IP address is not reachable from the current device. This was weird because docker sets up the host on the bridge network. I didn't know how to debug further and trying out random ideas. Then I saw that the `docker0` on the host by running `ifconfig` which is the bridge network docker created.

```
$ ifconfig
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:af:da:ba:dc  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
....
```
I tried to `ping` the host from the container and found that it was working fine. This means that the host is present but it was not accepting TCP connections but because `ping` uses ICMP, it was fine.
```
$ ping 172.17.0.1
64 bytes from 172.17.0.1: seq=0 ttl=64 time=0.061 ms
64 bytes from 172.17.0.1: seq=1 ttl=64 time=0.060 ms
64 bytes from 172.17.0.1: seq=2 ttl=64 time=0.055 ms
64 bytes from 172.17.0.1: seq=3 ttl=64 time=0.055 ms
...
```

At this point I was pretty sure that the issue was either -
* something wrong with the iptables inside the container or
* host machine which was not allowing connections

I started investaging inline with the first possibility; may be a bug in the openvpn config I used is causing the requests to host to route through the VPN connection and causing connections to time out.
```
# ip route
0.0.0.0/1 via 10.8.0.9 dev tun0
default via 172.17.0.1 dev eth0
10.8.0.1 via 10.8.0.9 dev tun0
172.17.0.0/16 dev eth0 proto kernel scope link src 172.17.0.3
...
```

But that turned out not to be the case. The default route was set to pass through the `eth0` interface which refers to the `docker0` interface on the host which should be fine.

So, the only possibility is that the host is blocking connections from container. I started searching along those lines and stumbled upon [this](https://stackoverflow.com/a/31328031/4129373) answer's last section which adds an IP table rule to the `docker0` interface to allow incoming connections and voila! Everything works as expected.

After talking to my manager, I learned that we use a firewall called UFW which doesn't allow incoming connections to all ports on the host and that was the reason why connections were failing. Adding the iptable rule bypassed the UFW rules which resulted in everything to work properly.
