---
layout: post
title: Computer Networking. Application Layer. Part 1.
---

**Application layer.**

I learned that a socket is *an interface between the application and transport layers*. It's like an API to deal with networks in your applications. 

The port is *an ID of the network connection*. It helps the server to distinguish between different network applications that the server can run.

TCP and UDP are well-known protocols, but I found out about **TLS** (**T**ransport **L**ayer **S**ecurity). It's an enhancement of TCP protocol with encryption of sent data. It's not the third transport layer protocol, it's the TCP with improvements in security in the application layer.

HTTP (HyperText Transfer Protocol) defines the structure of messages that the client and server exchange and how the client and server interpret these messages. HTTP is *a stateless protocol* because it doesn't store any information from before.

Also, I read about *persistent and non-persistent* connections. Non-persistent connections are not that great because of the additional time delay in opening a new connection each time.

I learned about HTTP Response and Request messages. The Request message looks like:
```
Request line (Method - URL - HTTP version)
Header lines
Entity body
``` 
The Response message: 
```
Status line (HTTP version - Status code - Phrase)
Header lines 
Entity body. 
```
There are many types of headers, status codes, request types, and other details.

I found out that *cookie data is stored on your host*. So *all the IDs on various websites (Amazon, YouTube, etc.) reside in your computer* and later when you access the website it knows that it's you by your ID (your ID is getting sent to the server in a request message).

Other useful technologies I got familiar with are web caches (like caches in your computer memory) and SMTP (format for exchanging Mail messages).

The thing *I find fascinating is DNS*, namely the way it works - to find another host I go to the DNS server, and this server redirects me to another one that knows where I can find the host I need. That server redirects to one more and eventually, you get an IP address of the host you want to connect to.

I read about P2P systems. Peer-to-peer systems have two advantages: 1) with the increasing number of peers *the distribution time does not grow as fast* as in the client-server architecture, and 2) it's *self-scaling* because of no (or minimal) reliance on some infrastructure. It led me to an idea that I've read in one book on algorithms and data structures - **the best one can be achieved by combining the best from a few things**. And, probably, by applying these properties of the P2P system to the client-server architecture we can get pretty good results.
