---
layout: post
title: Computer Networking. Network Layer. 
---

I learned a lot about the network layer: *IP, data and control planes, middleboxes, routing algorithms, and protocols*. Now, I understand how forwarding works in routes with its queues, switch fabric, and other features. Finally, I got a grasp of *IPv4 addressing* - what 223.1.1.0/24 means and how all the networks are built around it. 

Also, I became familiar with *general forwarding (implemented via SDN) and middleboxes*: how they operate, and what their pros and cons are. I think this topic is incredibly captivating and nowadays becomes more and more important because of the continuous steady growth of the Internet the whole.

The topics above are related to the data plane of the 3rd layer. So, to the control plane, I learned about intra-AS and inter-AS routing and appropriate protocols, namely *OSPF (**O**pen **S**hortest **P**ath **F**irst) and BGP (**B**order **G**ateway **P**rotocol)*. 

I dove into the *SDN control plane*: how the SDN controller sits between the network-control applications and switches and acts as a glue - accesses packet switches via the *Southbound API* (to get info or set something) and provides the functionality to the network-control applications to control packet switches via the *Northbound API*. Northbound and Southbound APIs use different protocols for message exchange. 

That's it for now, but the following 2-3 posts will be solely on programming computer networking projects.
