---
layout: post
title: Computer Networking. Reliable Transport Protocol. 
---

One of the assignments in Chapter 3 is to implement your reliable transport protocol. There are two versions of this lab: the *Alternating-Bit-Protocol (so, stop-and-wait) and Go-Back-N*. At first, I will implement the Alternating-Bit-Protocol version and move on to the Go-Back-N (such an order is because of complexity: ABP is simpler than GBN). All the required data, resources, tasks, and details regarding this assignment can be found [here](https://gaia.cs.umass.edu/kurose_ross/programming/RDT/RDT_Implementing%20a%20Reliable%20Transport%20Protocol.html).

So, whatever the protocol is used, *these cases should be taken into consideration and handled*:
1) No corruption, no loss, no timeout (ideal case);
2) No corruption, lost packet (from sender to receiver), timeout;
3) No corruption, lost ACK (from receiver), timeout;
4) No corruption, no loss, premature timeout (timeout happened before the ACK from the receiver was accepted);
5) Corrupted packet (from sender), no loss, no timeout;
6) Corrupted ACK (from receiver), no loss, no timeout.

Complete sources and scenarios of handling 6 cases above are resided [here](https://github.com/chetter14/computer-networking-assignments/tree/main/RTP).

Now, after we know what the protocol should do and how, we can get to the code. Initially, declare and create a few variables for further usage:
```
typedef struct sender				// define the sender-side required variables
{
	int curSeqNum;
	int waitingAckNum;				// currently not used
	struct pkt packetToRetransmit;
	bool isAnyMessageInTransit;
} sender;

typedef struct receiver				// define the receiver-side required variables
{
	int curAckNum;					// currently not used
	int waitingSeqNum;
} receiver;

sender A_sender;					// create A sender object
receiver B_receiver;				// create B receiver object

const float timeout = 20;			// time value after which the timeout interrupt occurs
```

Another important functions to be used later are a *calculation and validation of the checksum* (to detect corruptions):
```
int calculateChecksum(struct pkt* packet)
{
	int seqNum = packet->seqnum;
	int ackNum = packet->acknum;
	char* payload = packet->payload;
	
	// do the addition of all the values in a packet
	int sum = seqNum + ackNum;
	for (int i = 0; i < 20; ++i)
	{
		sum += (int)payload[i];
	}
	
	// invert the sum; (~sum) + sum = 11111111 11111111 11111111 11111111 (in binary)
	// if on the receiver after doing the same additions we couldn't reproduce this "only 1s" value, 
	// then the packet was corrupted during the transit
	sum = ~sum;					
	return sum;
}

bool isPacketValid(struct pkt* packet)
{
	int seqNum = packet->seqnum;
	int ackNum = packet->acknum;
	int checksum = packet->checksum;
	
	// do the addition of all the variables
	int sum = seqNum + ackNum + checksum;
	for (int i = 0; i < 20; ++i)
	{
		sum += (int)packet->payload[i];
	}
	
	// if the packet is not corrupted, the sum would be 11111111 11111111 11111111 11111111 in binary 
	// (which is -1 in signed int)
	if (sum == -1)
		return true;
		
	return false;
}
```

So, we can get to the main functions that are required to be written by us. The sender functions are presented below:
```
/* called from layer 5, passed the data to be sent to other side */
A_output(message)
  struct msg message;
{
	if (A_sender.isAnyMessageInTransit)
		return;
	
	struct pkt packet;
	packet.seqnum = A_sender.curSeqNum; 	// current sequence number
	packet.acknum = -1;						// ack number isn't used in sender
	strcpy(packet.payload, message.data);
	packet.checksum = calculateChecksum(&packet);
	
	// make a copy of packet for possible future retransmissions
	A_sender.packetToRetransmit = packet;
	strcpy(A_sender.packetToRetransmit.payload, packet.payload);

	tolayer3(0, packet);
	starttimer(0, timeout);
	
	A_sender.isAnyMessageInTransit = true;
}

/* called from layer 3, when a packet arrives for layer 4 */
A_input(packet)
  struct pkt packet;
{
	if (!isPacketValid(&packet))					// received packet is corrupted
	{
		stoptimer(0);
		tolayer3(0, A_sender.packetToRetransmit);
		starttimer(0, timeout);
	}
	else											// packet is correct
	{
		if (packet.acknum == -1)					// packet that was sent from sender to receiver is NACKd
		{
			stoptimer(0);
			tolayer3(0, A_sender.packetToRetransmit);
			starttimer(0, timeout);
		}
		else
		{
			if (packet.acknum == A_sender.curSeqNum)	// received the expected ack
			{
				stoptimer(0);
				A_sender.curSeqNum = (A_sender.curSeqNum + 1) % 2;
				A_sender.isAnyMessageInTransit = false;
			}
			// else - not the expected ack num (i.e., because of premature timeout and doubled ack):
			// do nothing
		}
	}
}

/* called when A's timer goes off */
A_timerinterrupt()
{
	tolayer3(0, A_sender.packetToRetransmit);
	starttimer(0, timeout);
}

/* the following routine will be called once (only) before any other */
/* entity A routines are called. You can use it to do any initialization */
A_init()
{
	A_sender.curSeqNum = 0;
	A_sender.waitingAckNum = 0;
	A_sender.isAnyMessageInTransit = false;
}
```
The functions `tolayer3()`, `tolayer5()`, `stoptimer()`, and `starttimer()` are straightforward and self-explanatory.

The receiver side is smaller in code, but not any less important than the sender:
```
/* called from layer 3, when a packet arrives for layer 4 at B*/
B_input(packet)
  struct pkt packet;
{
	struct pkt reply;
	strncpy(reply.payload, "empty", 20);						// "empty" is just a stub
	
	if (!isPacketValid(&packet))								// packet is corrupted
	{
		reply.seqnum = -1;	// not used
		reply.acknum = -1;	// equivalent to NACK
	} 
	else 													// packet is correct
	if (packet.seqnum == B_receiver.waitingSeqNum)			// and that we expected
	{
		struct msg message;
		strcpy(message.data, packet.payload);
		
		tolayer5(1, message);
		
		reply.seqnum = -1;								// seq number isn't used in receiver
		reply.acknum = packet.seqnum;					// ack number of the packet received
		
		B_receiver.waitingSeqNum = (B_receiver.waitingSeqNum + 1) % 2;
	}
	else													// not what the receiver expects
	{
		reply.seqnum = -1;
		reply.acknum = (B_receiver.waitingSeqNum + 1) % 2;	// this ack number would be the last correctly received packet
	}
	
	reply.checksum = calculateChecksum(&reply);
	tolayer3(1, reply);
}

/* called when B's timer goes off */
B_timerinterrupt()
{
	// empty
}

/* the following rouytine will be called once (only) before any other */
/* entity B routines are called. You can use it to do any initialization */
B_init()
{
	B_receiver.curAckNum = 0;
	B_receiver.waitingSeqNum = 0;
}
```

These are the functions that were necessary to be filled in. All the testing functionality was already in the file. It can be find [here](https://github.com/chetter14/computer-networking-assignments/tree/main/RTP).
