Sockets and Server Testing
===========================
Sockets and HTTP
---------------------------
### What is a socket?

A socket is a software construct used for many modes of communication between processes. The mode of communication that this recitation will focus on is network communication. In particular, stream sockets represent an endpoint for reliable, bidirectional connections such as TCP connections. This allows for two processes, on separate computers, to communicate over a TCP/IP network connection.

Sockets have:
- an IP address, to (typically) identify the computer that the socket endpoint belongs to
- a port number, to identify which process running on the computer the socket endpoint belongs to
- a protocol, such as TCP or UDP. TCP is for reliable transport, UDP is unreliable. HTTP/1.0 uses TCP.

An IP address and port number are both required in order for a computer to communicate with a specific process on a remote computer. Think about would happen if a port number wasn't necessary.

### Simple socket communication
A simple way to demonstrate the bidirectional and network-based communcation of sockets is with `netcat`. `netcat` is a bare-bones program to send streams of binary data over the network.

Imagine we have two computers that can communicate over the internet, with the IP addresses `clac.cs.columbia.edu` and `clorp.cs.nyu.edu`.

Connecting two socket endpoints to each other is not a symmetrical process. One socket needs to recieve connections on a specific port number, and the other socket needs to connect to that socket with the port number it specified. In `netcat`, the fist part happens through the `-l` flag as such:
```bash
joy@clac.cs.columbia.edu:~$ nc -l 10000
```

The `netcat` program on `clac.cs.columbia.edu` will create a socket and wait for connections on the port 10000. On the other server:
```bash
jeremy@clorp.cs.nyu.edu:~$ nc clac.cs.columbia.edu 10000
```
Notice the differences between these two commands. The first command only requires a port number, and doesn't need to know the IP address of the other server. The second command requires knowledge of both the IP address -- what computer to connect to; and the port number -- which process to connect to on that computer. This asymmetry is what's known as the client-server model.

### The client-server model
The two endpoints in a socket connection serve different roles. One end acts as a server: 
- It tells the operating system that it should recieve incoming connections on a port number
- It waits for incoming connections
- When it recieves a connection, it creates a *new socket* for each client which will be used for communication to that client

The other end is a client:
- It "connects" to the server using the server’s IP address and
 the port number

After a client connects to a server, there is bidirectional communication between the two processes, often with I/O system calls such as `read()` and `write()`, or their wrappers `recv()` and `send()`. 

### Sockets API Summary
![](client-server.png)
**`socket()`**
- Called by both the client and the server
- On the server-side, a listening socket is created; a connected socket will be created later by `accept()`

**`bind()`**
- Usually called only by the server
- Binds the listening socket to a specific port that should be known to the client

**`listen()`**
- Called only by the server
- Sets up the listening socket to accept connections

**`accept()`**
- Called only by the server
- By default blocks until a connection request arrives
- Creates and returns a new socket for each client

**Listening socket vs connected socket**

**`send()` and `recv()`**
- Called by both the client and server
- Reads and writes to the other side
- Message boundaries may not be preserved

### HTTP 1.0
- Client sends a **HTTP request** for a resource on the server
- The server sends a **HTTP response**

**HTTP request**
- First line: method, request-URI, version
    - Ex: "GET /index.html HTTP/1.0\r\n"
- Followed by 0 or more headers
    - Ex: "Host: www.google.com\r\n"
- Followed by an empty line
    - "\r\n"

Example:

**HTTP response**
- First line: response status
    - Success: "HTTP/1.0 200 OK\r\n"
    - Failure: "HTTP/1.0 404 Not Found\r\n"
- Followed by 0 or more response headers
- Followed by an empty line
- Followed by the content of the response
    - Ex: image file or HTML file

Example:

Testing your multi-server
---------------------------
### Siege
Siege is a command-line tool that allows you to benchmark your webserver using load testing. Given a few parameters, Siege tells you things like the number of successful transactions to your website, percent availability, and latency/throughput of your server, 

To install Siege, type the following command:

`sudo apt install siege`

To use siege with your webserver in HW3, run your server and test with the following command:

`siege http://<hostname>:<port>/<url>`

This will run for an infinite amount of time. When you CTRL-C out of the command, a list of statistics will be outputted on your terminal.

A better way to test with siege is using its options. The `-c` and `-r` options are particularly useful, as they allow you to specify the number of concurrent "users" and repetitions per user, respectively. For example, the following command will create 25 concurrent users that will each attempt to hit the server 50 times, resulting in 1250 hit attempts:

`siege -c 25 -r 50 http://<hostname>:<port>/<URI>`

There are many other options, specified in the siege man page. These include `-t`, which specifies how long each user should run (rather than how many times), and `-f`, which specifies a file path that contains a list of urls to test. 

### Additional guidance on testing/benchmarking

When grading, we're going to test your implementation using a mix of manual connections (e.g. using tools like netcat) and stress testers like siege.

You should use netcat to make sure that basic functionality works and that you can indeed have more than 1 connection to the server at any given time. netcat is nice because it allows you to establish a connection and then prompts you for the data to send. You should also use netcat to test that your cleanup logic is correct, as you can control exactly when connections start/terminate.

Once you've tested the basic functionality, use a stress tester to make sure that your server handles concurrent hoards of requests in a reasonable amount of time. Since we're all on VMs running on different host machines, we can't really say "X requests must finish in Y seconds". We're just looking to make sure that your server doesn't take years (e.g. because it is actually serializing requests).

Our grading scripts make heavy use of siege url files. siege will basically make requests using the URLs specified in this file. Use this to make sure your server can concurrently handle all kinds of requests and correctly respond to all of them (2xx, 4xx, 5xx, multi-server never responds with 3xx).

Regarding benchmarking, the assignment prompt occasionally instructs you to compare the performance of the implementation of one part with another. However, since you are testing multi-server in a virtual machine, the performance isn’t guaranteed to be significantly better. As such, don’t worry too much about the benchmarking instructions - it’s not a hard and fast requirement.
