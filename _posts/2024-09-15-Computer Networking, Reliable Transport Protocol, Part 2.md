---
layout: post
title: Computer Networking. Reliable Transport Protocol. Part 2.
---

So, after completing an RTP using the *Alternating-Bit-Protocol*, the next assignment is to complete it via the **GBN** (*Go-Back-N*) protocol. I won't delve into the details of GBN but I'll list a few *important properties of this version* of the lab:
1. Packets are sent in the amount of window size (at least, certainly can be sent like that);
2. If the window of packets is full, then upcoming packets will be stored in a buffer. Later on, packets from the buffer are going to be fetched and sent to the client.

It's a brief explanation, the full description of the task you can find [here](https://gaia.cs.umass.edu/kurose_ross/programming/RDT/RDT_Implementing%20a%20Reliable%20Transport%20Protocol.html).

I won't tell about the logic and algorithms of the GBN implementation here because it's larger than in the ABP approach, but you can read it [here](https://github.com/chetter14/computer-networking-assignments/tree/main/RTP), as well as the source code. Also, I won't pass through each piece of code because of its complexity and size, and will step through the main parts of it. So, to the code of the A-side:
```
/* called from layer 5, passed the data to be sent to other side */
A_output(message)
  struct msg message;
{	
	struct pkt packet;
	packet.acknum = -1;						// ack number isn't used in the sender
	strcpy(packet.payload, message.data);
		
	if (A_sender.nextSeqNum < A_sender.base + N)	// there is room for a packet in window
	{
		packet.seqnum = A_sender.nextSeqNum;
		packet.checksum = calculateChecksum(&packet);
		A_sender.packets[A_sender.nextSeqNum] = packet;		// store the packet for possible retransmission
		tolayer3(0, packet);
		
		if (A_sender.nextSeqNum == A_sender.base)
			starttimer(0, timeout);
		A_sender.nextSeqNum++;
	}
	else if (!isBufferFull())						// there is room for a packet in buffer
	{
		addPktToBuffer(packet);
	}
	else 											// no room anywhere
		exit(0);									// ! INTENTIONAL EXIT(), REQUIRED BY THE TASK !
}

...

void sendPacketsFromBuffer()
{
	while (A_sender.nextSeqNum < A_sender.base + N && !isBufferEmpty())		// until window is full and there are left packets in buffer
	{																			// take packets from buffer and send them
		struct pkt tempPacket = getPktFromBuffer();
		tempPacket.seqnum = A_sender.nextSeqNum;
		tempPacket.checksum = calculateChecksum(&tempPacket);
		A_sender.packets[A_sender.nextSeqNum] = tempPacket;			// for possible retransmission
		tolayer3(0, tempPacket);
		
		A_sender.nextSeqNum++;
	}
}

/* called from layer 3, when a packet arrives for layer 4 */
A_input(packet)
  struct pkt packet;
{
	if (isPacketValid(&packet))
	{
		A_sender.base = packet.acknum + 1;
		if (A_sender.base == A_sender.nextSeqNum)		// window is completely empty
		{
			if (!isBufferEmpty())						// there are packets in buffer
			{
				sendPacketsFromBuffer();
				stoptimer(0);
				starttimer(0, timeout);
			}
			else										// no packets in buffer
				stoptimer(0);
		}
		else											// window is not totally full
		{
			if (!isBufferEmpty())						// there are packets in buffer
			{
				sendPacketsFromBuffer();
			}
			// No timer reset because packets that were sent first are going to be delayed for retransmission even further 
			// so, I am trying to avoid it
			// stoptimer(0);
			// starttimer(0, timeout);
		}
	}
	else
		;// received packet is corrupted - do nothing
}
```

As you can see, the buffer is used there, so I've created a buffer header file that contains all the required logic. *The buffer is implemented via the circular queue - the FIFO approach on the array of data*:
```
// stores packets to be sent
typedef struct Buffer
{
	struct pkt packets[50];			// size of buffer - 50 packets
	int start;
	int size;
} Buffer;

Buffer buffer; 

void initBuffer()
{
	buffer.start = 0;
	buffer.size = 0;
}

bool isBufferFull()
{
	return buffer.size == 50;
}

bool isBufferEmpty()
{
	return buffer.size == 0;
}

void addPktToBuffer(struct pkt packet)
{
	int newPacketIndex = (buffer.start + buffer.size) % 50;
	buffer.packets[newPacketIndex] = packet;
	buffer.size++;
	printf("addPktToBuffer: %s\n", packet.payload);
}

struct pkt getPktFromBuffer()
{
	struct pkt packet = buffer.packets[buffer.start];
	printf("getPktFromBuffer: %s\n", packet.payload);
	buffer.start = (buffer.start + 1) % 50;
	buffer.size--;
	return packet;
}
```

And to the B side that is light and straighforward:
```
B_input(packet)
  struct pkt packet;
{
	if (isPacketValid(&packet))
	{
		if (packet.seqnum != B_receiver.expectedSeqNum)
		{
			B_receiver.packet.acknum = B_receiver.expectedSeqNum - 1;	// the last correctly received
		}
		else
		{
			struct msg message;
			strcpy(message.data, packet.payload);
			tolayer5(1, message);
			
			B_receiver.packet.acknum = B_receiver.expectedSeqNum;
			B_receiver.expectedSeqNum++;
		}
	}
	else	// packet is corrupted
	{
		B_receiver.packet.acknum = B_receiver.expectedSeqNum - 1;
	}
	
	B_receiver.packet.checksum = calculateChecksum(&B_receiver.packet);
	tolayer3(1, B_receiver.packet);
}
```
