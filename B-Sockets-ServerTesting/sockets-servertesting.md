Sockets and Server Testing
===========================
Sockets and HTTP
---------------------------

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
