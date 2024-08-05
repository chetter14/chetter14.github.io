---
layout: post
title: Computer Networking. Transport Layer. Part 2.
---

I learned a lot about how reliable data transfer is implemented, and it's terrific. All the pipelined protocols: *GBN (Go-Back-N), SR (Selective Repeat), and the protocol in TCP*; estimation of RTT (Round Trip Time) and deviations from it to calculate an appropriate timeout value; *windows sizes, sequence and acknowledgments numbers*, and more.

However, *TCP also provides congestion control* in the network. I discovered that a basic congestion-control algorithm consists of 3 phases: **Slow Start, Congestion Avoidance, and Fast Recovery**. The versions of TCP that use different congestion-control algorithms are *TCP Tahoe* (no Fast Recovery), *TCP Reno* (includes Fast Recovery), and *TCP CUBIC* (like TCP Reno but faster rises to the pre-loss rate). 

The way congestion-control algorithms are implemented makes the fair sharing of the bottleneck link among many TCP connections possible (every connection gets an equal share).

Also, I read about *QUIC* (**Q**uick **U**DP **I**nternet **C**onnection) - an application layer protocol on top of the UDP developed by Google. It's secure, effective, allows many streams in one connection (i.e., to send web page objects through different streams), and provides a reliable and congestion-controlled data transfer.

To practice - I made an *SMTP client that allows to send email messages*. The SMTP protocol is used here. I'm going to cover the main parts of the client code.
```
# Connect to the mail server (google.com, yandex.ru, etc.)
mailserver = '209.85.233.108' 	
clientSocket = socket(AF_INET, SOCK_STREAM)
clientSocket.connect((mailserver, 587))			# port 587 handles SMTP connections

# Check what we have received from the server
recv = clientSocket.recv(1024).decode() 
if recv[:3] != '220': 							# if the code is not 220 (some error occurred)
    print('220 reply not received from server.') 
``` 

Then we send HELO command with the hostname of the client (as I understand it's a part after '@' in your mail address):
```
# Send HELO command 
helloFrom = "gmail.com"
heloCommand = 'HELO {helloFrom}\r\n'.format(helloFrom=helloFrom) 
clientSocket.send(heloCommand.encode())
...

# Send STARTTLS command for security issues
# TLS 1.2 is not supported by Python 2.7
startTlsCommand = 'STARTTLS'
clientSocket.send(startTlsCommand.encode())
...
```
Important to know, I *do check responses from the server after each command* I send but the code is pretty identical every time, so I don't put it into the code samples.

When HELO command was properly sent, the connection is established and you can send email messages (following the SMTP message format):
```
# Send MAIL FROM command (who the email is from)
senderMail = "artemleshchukov02@gmail.com"
mailFromCommand = 'MAIL FROM: <{mail}>\r\n'.format(mail=senderMail)
clientSocket.send(mailFromCommand.encode())
...

# Send RCPT TO command (to whom the email is addressed)
rcptMail = "artem-leshchukov@mail.ru"  
rcptToCommand = 'RCPT TO: <{mail}>\r\n'.format(mail=rcptMail)
clientSocket.send(rcptToCommand.encode())
...

# Send DATA command (to denote the start of the message body)
dataCommand = 'DATA\r\n' 
clientSocket.send(dataCommand.encode()) 
recv = clientSocket.recv(1024).decode()
...

# Now we send the message itself and the ending with a single period
messageCommand = "I love computer networks!\r\n.\r\n" 
clientSocket.send(messageCommand.encode())
...

# Send QUIT command (to close the SMTP connection)
quitCommand = "QUIT\r\n" 
clientSocket.send(quitCommand.encode())
...
```
