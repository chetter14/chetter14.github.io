---
layout: post
title: Computer Networking. Transport Layer.
---

In chapter 3 of the book - Transport Layer - I learned that *TCP and UDP are only implemented on end hosts*. While transmitting messages across the network (through routers, links, etc.), no protocol higher than IP is used. 

So, it means that the developer of TCP *has to make this protocol reliable even though there is an IP below TCP*. UDP is like IP with error-checking and process-to-process (not host-to-host) delivery.

Another interesting thing I read about is *demultiplexing in UDP and TCP*. In UDP, only the *destination IP and port are used* for demultiplexing. It implies that segments from different source hosts with the same destination IP and port will be directed to one socket. In TCP, *source IP and port are also included* in the demultiplexing. 

I understood that **UDP has an advantage over TCP in speed but is not as reliable as TCP**. Application-to-application communication can be reliable using UDP only if the reliability is provided in the application itself (with retransmission, acknowledgments, etc.)

Here I want to show and explain a bit the client code I wrote for the UDP Pinger task in chapter 2 (the complete sources can be found [here](https://github.com/chetter14/computer-networking-assignments/blob/main/udp-pinger/client.py)):
```
# in secs
min_rtt = 10.0			# set to 10.0 seconds just because it's higher that 1.0 (timeout)
max_rtt = 0.0
total_rtt_sum = 0.0		# to be used for calculation of average rtt
rtt_number = 0			# the same, + packet loss
```

At first, I send some message to the server and receive a reply:
```
send_time = datetime.now()
message = "ping {number} {time}".format(number=i, time=send_time)
client_socket.sendto(message.encode(), (server_host, int(server_port)))

# ...

reply, server_address = client_socket.recvfrom(2048)
print("From server - {reply}".format(reply=reply.decode()))
receive_time = datetime.now()
```

After that we can calculate the RTT and related values:
```
delta_time = receive_time - send_time
rtt = delta_time.seconds + delta_time.microseconds * 1e-6

if (rtt < min_rtt): min_rtt = rtt
if (rtt > max_rtt): max_rtt = rtt

total_rtt_sum = total_rtt_sum + rtt
rtt_number = rtt_number + 1
```

Another fascinating application I made is the [Heartbeat](https://github.com/chetter14/computer-networking-assignments/tree/main/udp-pinger). It checks whether the client is still connected or dropped out. Here is a rough client code where it sleeps for a while and sends the current time:
```
sleep_seconds = random.randint(0, 7)
time.sleep(sleep_seconds)

current_time = datetime.now()
message = "{sequence_number} {time}".format(sequence_number=i, time=current_time)
client_socket.sendto(message.encode(), (server_host, server_port))
```

On the server, we receive the message from the client and calculate the delta of transmission. If the timeout occurs, we assume that the client on the other side has dropped off.
```
try:
	message, address = serverSocket.recvfrom(1024) 
	receive_time = datetime.now()
	
	# parse the client time 
	client_message = message.split(' ')
	client_time_string = client_message[1] + " " + client_message[2]
	client_time = datetime.strptime(client_time_string, "%Y-%m-%d %H:%M:%S.%f")
	delta_time = receive_time - client_time
	
except timeout:
	print("The client application was closed.\n")
```
