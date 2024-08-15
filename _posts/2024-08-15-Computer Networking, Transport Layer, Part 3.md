---
layout: post
title: Computer Networking. Transport Layer. Part 3.
---

After a bunch of theories I learned, I was doing questions and problems at the end of the chapter. I dealt with issues related to *flow control protocols (GBN, SR, and TCP), congestion control mechanisms (TCP Reno, TCP Tahoe), speed limits, and optimal utilization of the link's bandwidth* concerning buffer size on the client and server.

I did labs in *Wireshark on HTTP and DNS*. At first, I just casually searched for web pages and scanned through the HTTP packets that the browser sends to the server and receives (various header lines that contain practical info, the objects themselves, etc.). Then I used two commands in cmd - *nslookup and ipconfig* - to get IP addresses of domain servers and websites, get network-related info of your host, and other, fascinating to me, details.

I also made a simple [web proxy](https://en.wikipedia.org/wiki/Proxy_server) in Python. It works when you type this - "localhost:8888/www.google.com/" - in your web browser. Here I am going to step through the significant parts of this program (each line of code is necessary here):
```
# Extract the filename from the given message 
full_path = message.split()[1][1:]           # with no '/' before the name of the file
print("Full path - {full_path}".format(full_path=full_path))

# ...

# Check whether such a file exists in the cache
if full_path in pages_cache:                            # if yes, then cache hit and send to the client
	tcpCliSock.send(pages_cache[full_path].encode())
	print('Read from cache\n')
```
I should note that the web proxy cache is *intentionally not persistent* in this task.

```
else:
	# Get the host and the requested file
	host, slash, file = full_path.partition("/")

	# Create a socket on the proxyserver
	proxy_client_socket = socket(AF_INET, SOCK_STREAM)
	hostn = host.replace("www.","",1)					# replace for further use in HTTP message format
	print("Hostname - {host}".format(host=hostn))
	
	try:     
		# Connect to the socket to port 80     
		proxy_client_socket.connect((hostn, 80))
		print("Successfully connected to host")

		# Redirect the request to the server
		http_request = "GET /{file} HTTP/1.0\r\nHost: {host}\r\n\r\n ".format(file=file, host=hostn)
		proxy_client_socket.send(http_request.encode())
		print("Sent the request - " + http_request)

		# Read the response into buffer
		final_response = ""
		while True:
			cur_response = proxy_client_socket.recv(2048).decode()
			if not cur_response: break
			final_response = final_response + cur_response

		print("Received the data from the server:\n" + final_response)

		# Store the response in the cache   
		pages_cache[full_path] = final_response
		print("Stored the data in the cache")

		# Send the response to the client socket     
		tcpCliSock.send(final_response.encode())

	except:     
		print("Error! Something went wrong during the interaction with the server!\n\n")
```

You may get the result like: "301 Moved Permanently" and similar stuff. It happens because the server expects the **HTTPS**, not just **HTTP**. To make the proxy work as expected, you should establish a connection with the server via **TLS/SSL** instead of **TCP**. Also, there are plenty of options to extend the functionality of the proxy: multithreading, persistent storage, TLS/SSL, etc.
