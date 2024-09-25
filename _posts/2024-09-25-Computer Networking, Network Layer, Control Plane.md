---
layout: post
title: Computer Networking. Network Layer. Control Plane.
---

I have dealt with few tasks related to the topic of Network Layer (on OSI) here: *ICMP ping* and routing via the *distance-vector algorithm*. At first, about the ICMP ping program. My task is to write a piece of code that processes the received message from 'ping' command - prints out the delay time or that the request is timed out. I am talking about this part:
```
...
echo_reply = struct.unpack("qqibbHHhff", recPacket)			# the received packet is in binary format
# print(echo_reply)

icmp_header_index = 3                                    # at the 3rd index of received message starts ICMP type and code 
if isErrorInEcho(echo_reply[icmp_header_index], echo_reply[icmp_header_index + 1]):
	return "Error occurred."

# in binary format we read from recPacket into 2 floats but we need 1 double, so we convert it from one type to another
double_struct = struct.pack("ff", echo_reply[icmp_header_index + 5], echo_reply[icmp_header_index + 6])
time_in_packet = struct.unpack("d", double_struct)[0]

delta = timeReceived - time_in_packet

timeLeft = timeLeft - howLongInSelect
if timeLeft <= 0:
	return "Request timed out."
else:
	return delta
...
```
Also, I added: 1) an extra function to handle ICMP erros, and 2) printing out of min, max, average RTT (Round Trip Time) and the percentage of packet loss:
```
...
def isErrorInEcho(type, code):
    if type == 3:                       # means ICMP error
        if code == 0:
            print("Destination network unreachable")
        elif code == 1:
            print("Destination host unreachable")
        elif code == 2:
            print("Destination protocol unreachable")
        elif code == 3:
            print("Destination port unreachable")
        elif code == 6:
            print("Destination network unknown")
        elif code == 7:
            print("Destination host unknown")
        return True
    return False

...

pings_number = 10

min_rtt = 10
max_rtt = 0
total_rtt_sum = 0
rtt_number = 0
packets_lost = 0

# Send ping requests to a server separated by approximately one second
for i in range(pings_number) :
	delay = doOnePing(dest, timeout)
	print("RTT - " + str(delay) + "s")

	if isinstance(delay, str):
		packets_lost = packets_lost + 1
	else:
		if delay < min_rtt:
			min_rtt = delay
		elif delay > max_rtt:
			max_rtt = delay
		total_rtt_sum = total_rtt_sum + delay
		rtt_number = rtt_number + 1

	time.sleep(1)# one second

print("")
print("Max RTT - " + str(max_rtt) + "s")
print("Min RTT - " + str(min_rtt) + "s")
print("Average RTT - " + str(total_rtt_sum / rtt_number) + "s")
print("Packets lost - " + str(packets_lost / pings_number * 100) + "%")

...
```

So, to the "Distributed Asynchronous Distance Vector Routing" application. The whole task description can be found [here](https://gaia.cs.umass.edu/kurose_ross/programming/DV/Programming%20Assignment%201.html). Although I developed 2 functions - 'rtinit()' and 'rtupdate()' - for each node in the graph, I will show the code of the 0 node because the logic stays the same despite the node number (except for distance values):
```
static void notifyNeighboringNodes()
{
	// Send the distance vector to 1, 2, and 3 nodes:
	
	struct rtpkt updatePacket;
	updatePacket.sourceid = 0;
	for (int i = 0; i < 4; ++i)
	{
		updatePacket.mincost[i] = dt0.costs[i][0];
	}
	
	for (int i = 1; i < 4; ++i)
	{
		updatePacket.destid = i;
		tolayer2(updatePacket);
	}
}

void rtinit0() 
{
	// Initialize a distance vector:
	
	// destination 0:
	dt0.costs[0][0] = 0;
	dt0.costs[0][1] = 999;		// source 0 and dest 0 through 1/2/3 makes no sense, so assign "infinity"
	dt0.costs[0][2] = 999;
	dt0.costs[0][3] = 999;
	
	// destination 1:
	...
	
	// destination 2:
	...
	
	// destination 3:
	...
	
	// printf("Node 0 initialization at %f\n\n", clocktime);
	
	printdt0(&dt0);
	
	notifyNeighboringNodes();	// to spread the info about "this node"s distance values to its neighbors
}

void rtupdate0(rcvdpkt)
  struct rtpkt *rcvdpkt;
{
	int srcNode = rcvdpkt->sourceid;
	
	// iterate over min costs of another node:
	
	printf("\nBefore update:\n");
	printdt0(&dt0);

	
	bool wasUpdated = false;
	for (int i = 0; i < 4; ++i)
	{
		if (dt0.costs[i][0] > rcvdpkt->mincost[i] + dt0.costs[srcNode][0])
		{
			dt0.costs[i][0] = rcvdpkt->mincost[i] + dt0.costs[srcNode][0];
			wasUpdated = true;
		}
	}
	
	// printf("Node 0 update at %f\n\n", clocktime);
		
	if (wasUpdated)
	{
		printf("\nAfter update:\n");
		printdt0(&dt0);
		printf("\nCosts to other nodes: 1 - %d, 2 - %d, 3 - %d\n", dt0.costs[1][0], dt0.costs[2][0], dt0.costs[3][0]);
		notifyNeighboringNodes();
	}
}
```
I should note that 'rtinit()' and 'rtupdate()' functions are written separetely for each node number because it's the way the task asks to do it. In real circumstances, the code should be written once to avoid duplication of code and improve scalability of the project.
