---
layout: post
title: Computer Networking. Part 2.
---

To continue the previous article, I learned about video streaming and techniques related to this - for example, the **DASH** (**D**ynamic **A**daptive **S**treaming over **H**TTP). The main idea is that there are many versions of videos with different qualities on the server. The client *dynamically (by measuring the available bandwidth) selects the appropriate version of the video* for itself.

Also, I got familiar with **CDN**s (**C**ontent **D**istribution **N**etworks): pull/push caching, cluster selection strategy, etc.

A really interesting part of this chapter is socket programming, namely assignments. The programming in the chapter is straightforward - you initialize the server, run the client, send to the server some sentence, server does "uppercase" the whole sentence and sends it back to the client. The two versions are made using TCP and UDP.

Assignment #1 is to build a *web server that receives HTTP requests and sends back HTTP responses*. I will step through pieces of code that I find worth pointing out (I won't talk about how to send/receive messages with sockets, read files, and other simple functionality). The sources can be found [here](https://github.com/chetter14/computer-networking-assignments).

First, I do some initializations:
```
max_clients_number = 3						# the max number of possible clients of server
server_socket.listen(max_clients_number)
mutex = Lock()							# create a mutex to use for multithreading purposes
current_client_number = 0					# like an ID of a client
```

Then run a thread for every client that the server has accepted:
```
connection_socket, client_address = server_socket.accept()
client_thread = Thread(target=connection_handling_function, args=[connection_socket])
client_thread.start()
```

Every time a new client's thread starts and ends, we increment/decrement the current client number:
```
mutex.acquire()
current_client_number = current_client_number + 1
cur_thread_client_number = current_client_number
print("\nConnected to the " + str(cur_thread_client_number) + " client!")
mutex.release()
# ...
mutex.acquire()
current_client_number = current_client_number - 1
mutex.release()
```

Because the client requests some `.html` files we should make access to such files secure for the multithreading server:
```
mutex.acquire()
f = open(filename[1:])
output_data = f.readlines()
f.close()
mutex.release()
```

And the vital part - sending the HTTP response. The HTTP response is generated according to *the specifications of the HTTP message formats*:
```
status_line = "HTTP/1.1 200 OK\r\n"
connection_socket.send(status_line.encode())
connection_header = "Connection: close\r\n"
connection_socket.send(connection_header.encode())
content_type_header = "Content-type: text/html\r\n"
connection_socket.send(content_type_header.encode())

connection_socket.send("\r\n".encode())

print(str(cur_thread_client_number) + " Sent HTTP headers!")

# Send the content of the requested file
for i in range(0, len(output_data)):
	connection_socket.send(output_data[i].encode())
connection_socket.close()
```
If the server can't read the file that the client requested, then "404 Not Found" is sent to the client:
```
status_line = "HTTP/1.1 404 Not Found\r\n"
connection_socket.send(status_line.encode())

connection_header = "Connection: close\r\n"
connection_socket.send(connection_header.encode())
connection_socket.send("\r\n".encode())

entity = "404 Not Found"
connection_socket.send(entity.encode())
connection_socket.close()
```

Essentially, that's it about the server. Then I made my own HTTP client to test the server. The client python script expects server hostname, server port, and filename (that is being requested) as arguments. The core functionality looks like this:
```
client_socket.connect((server_host, int(server_port)))
http_request = "GET /{filename} HTTP/1.1\r\nConnection: close\r\nHost: {server_host}\r\n\r\n ".format(filename=filename, server_host=server_host)

client_socket.send(http_request.encode())
response = client_socket.recv(2048).decode()
```
I typed the HTTP request with brute force. If desired, it can be altered to a more flexible option.
