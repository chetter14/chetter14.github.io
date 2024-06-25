---
layout: post
title: Computer Networking. Getting Started.
---

I've started reading a book on computer networking called "Computer Networking. A Top-Down Approach" by James F. Kurose and Keith W. Ross. From now on, I am going to tell you what I've learned from reading this book.

I've understood that everything in networking is based on protocols, and mastering them means mastering this field itself.

I've read about packet and circuit switching. By comparison, packet switching is more desired in practice because it allows to contain x3 number of users than in circuit switching.

One more important notion I've understood - to avoid packet loss, the number of packets arriving per second at some packet switch must be less/equal to the number of packets pushed out per second onto the communication link. Also, it's worth to mention about bottleneck links. I've learned that an engineer should be careful about them because the whole throughput between two hosts can be solely limited by these bottleneck links and it can cause serious problems for the infrastructure (like packet loss).

A vital part of computer networking is computer security. I've learned about DDoS attacks (Distributed Denial-of-Service), packet sniffers, and IP spoofing. In general, it's complicated to detect these types of "attacks" and it's my responsibility as an engineer to design network systems so that they can defend against such attacks. 

I've solved a bunch of exercises regarding delays: processing, queue, transmission, and propagation, how they form congestions in networks, etc. I've played a bit with WireShark Labs as well. I've looked at packets (frames, datagrams, segments, and messages): their sources and destinations, time of arrival, statuses, etc.
