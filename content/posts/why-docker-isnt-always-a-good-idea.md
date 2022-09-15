---
title: Why Docker isn't always a good idea Part 1
draft: false
date: 2022-09-15T15:00:00+02:00
tags: docker, network, haproxy
---

To briefly explain the situation:
We have a **HAProxy** running on a Debian server as a Docker container. This is the entrance node to a **Docker Swarm** cluster.

Now, in the last few days, there have been several small outages of the websites running in the **Docker Swarm** cluster. After getting an overview, we noticed that no new connections can be established.

As soon as we restarted the **HAProxy**, everything went back to normal. After that I did some research on TCP connections and found out that there is a socket limit.

In Linux we have a limit of sockets that can be opened at the same time. At this point, I unfortunately did not understand that this limit refers to a client connection. So we note that a client can establish a maximum of **65535** socket connections to a server.

This limit refers to a range of ports that you release. We had about 35k sockets available on our server (**HAProxy**). Now the pages are always down when this limit is reached. Thinking back for a moment, we should never get to that limit as it relates to a client. But the problem with us was that Docker's softlayer network didn't route the client address cleanly through the NAT, so everything was coming from one client.

After stopping the Docker container and installing **HAProxy** natively on the server, we were able to cross that boundary as well.

## Assumption

Because we NAT all the requests through the Docker network, the source address is always the same. This is how we reach the socket limit. If we omit the NAT and use HAProxy natively, we do not reach this limit, because the source address is no longer always the same.

![Network Address Translation](https://upload.wikimedia.org/wikipedia/commons/thumb/c/c7/NAT_Concept-en.svg/1920px-NAT_Concept-en.svg.png "Network Address Translation")

## Conclusion

With this setup, we get overhead into the system that we don't need. We have an extra abstraction layer, every request has to go through the Docker network and we reach the socket limit. All these points fall away when we use it natively.

If we use a lot of micro services it is important that we use something like Docker, because then we can share the kernel and it makes the deployment much easier.

But if we have only one application that is very important, it is better to keep it simple.
