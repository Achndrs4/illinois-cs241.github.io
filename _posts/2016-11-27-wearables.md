---
layout: doc
title: "Wearables"
permalink: wearables
submissions:
- title: Entire Assignment
  due_date: 12/05 11:59pm
  graded_files:
  - wearable_server.c
learning_objectives:
  - Client/Server Model
  - Networking
  - Streaming Data
wikibook:
  - "Networking, Part 2: Using getaddrinfo"
  - "Networking, Part 3: Building a simple TCP Client"
  - "Networking, Part 4: Building a simple TCP Server"
---
## Wearables

It's Friday, and you've sent your employees home for the weekend. Your startup just hit a major milestone, and you're relaxing at the Thomas M. Siebel Bar for Computer Science. You end up sitting next to a scruffy-looking chap, him turning out to be the smartest you've met so far, and you've talked to him for hours when he drops the bomb:

"By the way, my name is Terry Paige." The founder of Broogle.

Terry continues. "What was your startup called, again?"

"Uh, Tearable Inc."

"Well, Tearable is working on exactly the kind of clothing sensors we're interested in. We also just funded 241andme, an interesting healthcare startup that I think can synergize with Tearable. Let's schedule a meeting for next Monday, December 5th at 11:59pm--show me what you've got, and we may be interested in bringing you and your team on board. Sound good?"

"Y-y-y-y-yes."

You've worked for years for this opportunity. On top of this, your biotech wearable startup just struck a deal with manufacturers and hardware engineers to create affordable wearables at massive scale, so you have the hardware.

All that's left before the demo is the software that connects everything together. Let's get rolling.

## Information

The purpose of this MP is to scratch the surface of networking and introduce the client-server programming model.

Every network application is based on the client-server model. With this model, an application consists of a server process and one or more client processes. The server manages some resource, and it provides some service for its clients by manipulating that resource.

The data your wearables collect are limited to three different types:

* Heartbeats per minute
* Blood sugar
* Body temperature

Your wearables can already collect and transmit this data. You need the server to hold a user's personal data as well as run some analytics on this data.

Luckily, you've kept some consultants on retainer. They've drafted out some tasks for you:

1. Setup your server to listen for incoming connections.
2. Collect streaming data from the wearable devices.
3. Analyze the health data and report back statistics when the user queries.
4. Close the server on command and clean up resources used.

## Server

Your server will be provided two port numbers as command line arguments when it runs. One port is reserved for your server to receive data from wearables and the other one is used for clients to request data from your server. These ports will be called the `wearable port` and the `request port` respectively; you'll need to accept connections from both of these.

There could be numerous connections running simultaneously on the wearable port, and each of which could last an arbitrary amount of time. As such, you should create a thread for each new connection. For the `request port`, however, only one client will connect at any single time.

## Data

Once a connection has been established between a wearable and your server, the wearable send data to your server which is in the form `<timestamp>:<value>:<type>`,where `<timestamp>` and `<value>` are integer values, while `<type>` will correspond to one of the three strings: `heart_beat`, `blood_sugar`, `body_temp`.

You are responsible for saving this data for later processing and analysis.
We have provided you with the same vector you are familiar with from earlier in the semester.
You can refresh yourself on the functions available in `includes/vector.h`.
Please note that like the vector you implemented, this vector is **not thread safe** so you will be responsible for synchronization on it.
We also provide you with a few functions that will likely be useful as you're writing your server, which can be found in `utils.c`.
Be sure to read `utils.h` for more details!

A slight simplification for this MP: you can assume that data will always arrive in 64 byte packets because the wearable clients are actually running on the same localhost. Thus, you can read socket data in in blocks of 64 bytes and parse each block as a new data item. A real server might have to stitch multiple blocks together because the packets can arrive in arbitrary sizes as they are routed around the Internet.

## Analyze

Your server is useless if others cannot query it, so we will also open up a port (the given `request port`) where a client can send a request to. A request to your server will be done in the following steps:

* A client opens a socket on the request port which is handled in its own thread. For simplicity, there will be only one connection to the request socket, so you can safely close the server socket after the client closes the connection. This portion of code is already provided (wearables.c).

* A request that the client sends follows the format `<timestamp_start>:<timestamp_end>`, where the timestamps are both integers.

* Your server parses the request and collects all data received from wearables between those two timestamps specified in the request. Data should be collected in the inclusive-exclusive range i.e. [<timestamp_start>:<timestamp_end>). To gather information about events within such timestamp range you can use the given function `gather_timestamps()` (see documentation in `utils.h` for more details).

* If any active wearable connection has not received data up to the end timestamp (`<timestamp_end>` in the request), you must wait until that wearable thread receives data that has a timestamp greater than or equal to the end timestamp before replying the client's request. Furthermore, if a wearable thread has finished sending data and closed its connection, you still need to include its data in your reply. You will need to use some sort of synchronization to ensure that you do not send out the data too early (before you satisfy the above requirements).

* While you will receive no credit for sending data out too early, you may be thinking that you can just reply all requests from the client after all wearables have finished, eliminating the need for advanced synchronization. Solutions that do this will only receive 25% of the maximum possible score. The data should be sent as soon as it is available, but we provide a small tolerance for network delay.

* When you decide to reply for a request, you will send out the median and mean of the data for each type. We have provided the method `write_results()`, which (given a file descriptor, `char* type`, and results set) will write out the required statistics. You should call this method for each type in order, i.e. `write_results(fd, TYPE1 …)`, `write_results(fd, TYPE2 ....)`, etc. Further, you must write the string `"\r\n"` to the file descriptor after your 3 calls to write_results.

Please don't use busy-waiting i.e. wait for all the data and then send the information; you'll want to use a mutex and condition wait instead. The wikibook is a great resource to learn about correct use of condition variables.

## Close Gracefully

Your server will stop accepting connections when you receive the interrupt signal (SIGINT), and will exit gracefully (i.e. freeing all memory and closing all ports) after all wearable connections have closed. In all of the tests you can assume the client has connected to the request socket and the SIGINT signal is only sent after the user has finished requesting data. Make sure that your server frees all resources after receiving SIGINT!

## Building and Running Your Server

To build the server, run `make`.

We've provided a wearables simulator for your testing purposes, so you can focus on creating the server. Recall that your server takes 2 ports--wearables and request. Starting your server, where 8888 and 8889 are arbitrary ports, should look like:

```
% ./wearable_server 8888 8889
```

Once your server has started, you can begin the wearables simulator which will simulate sending data from wearables and requests. We've included some basic but insufficient test scripts to check your output against. They're human-readable, so you can create your own tests to verify your output with one command on each line. Please make sure your server is restarted before every simulator run.

```
% ./wearable_sim 8888 8889 tests/[Testname].ww
```

While the simulator is running, it will report back the data you sent from your server. Additionally, it also generates the expected output for you to compare against. After running the simulator you should see 3 files, `_expected.rp`, `_received.rp`, and `_error_report.rp`. `_received.rp` will hold the exact data sent from your server, so to see if your output is correct after running the simulator, you can use:

```
% diff _received.rp _expected.rp
```

or

```
% bash check_solution.sh
```

Please note that `_error_report.rp` will hold any errors that may have been encountered while running the simulator.

You can create your tests like this:

```
BEGIN - Signifies the beginning of a wearable definition.
START: - States how much time before this wearable begins transmitting information.
INTERVAL: - States the time delay between sending data points. <type>:<value> - One data point, where type is a string corresponding TYPE1, TYPE2, TYPE3. Value is an integer. Note all data points are transmitted sequentially by order in which they appear in a BEGIN-END wearable definition.
END - Signifies the end of a wearable definition
```

```
SAMPLE_INT:<wait time>:<timestamp1>:<timestamp2> - Simulates a user request.
```

There can be multiple commands of this time, and the are simulated sequentially. `<wait time>` represents the time (in millis) to sleep before this request is sent. The timestamp values represent the range of timestamps requested.

A basic test that creates one wearable which starts 1 second after the simulator is started, with 3 data points (send every second) and requests timestamp between 0, 2000 (with the request started 2 seconds after) would look like:

```
SAMPLE_INT:2000:0:2000
BEGIN
START:1000
INTERVAL:1000
heart_beat:100
blood_sugar:104
body_temp:104
END
```

This test script would send the data:

```
1000:heart_beat:100
2000:blood_sugar:104
3000:body_temp:104
```

and the expected output would be:

```
Results for heart_beat:
Size:1
0 100
Median:100
Average:100
Results for blood_sugar:
Size:0
Median:0
Average:0
Results for body_temp:
Size:0
Median:0
Average:0
```

If you want to kill the wearable server:
You might have to do this if you have not implemented the signal handler or if your socket isn't closed properly.

First, find the pid of the process by running `netstat -tulnap | grep :<port number>` with the appropriate port number.

Then, kill the process by running `kill -9 <pid>` with the appropriate pid.

## Input Format

You can safely assume that we will provide all input in the format stated above. However, it might be helpful for you own debugging purposes to add some error handling routines in your server program.

## Food For Thought

This portion is ungraded, but you may find it helpful for both checking understanding and for future assessments.

* How is a domain name converted to an IP address? What could be some of the benefits of caching DNS lookups?
* Describe the purposes of the pthread_join and pthread_create calls, and how threads can be used in C servers.
* What are the advantages of TCP over UDP? What are the disadvantages? Can you think of a use case where UDP would be a better choice? Think about this, then see [Quora on why Netflix uses TCP instead of UDP](https://www.quora.com/Why-does-Netflix-use-TCP-and-not-UDP-for-its-streaming-video) for streaming video if you're interested.
* What is the "TCP Handshake"?
* Explain in plain English what sockets are, and how we use them inside Unix programs (hint: think files). How are sockets different from other files you have encountered `so far? (Do they have inode metadata? Do writes on sockets result in Disk I/O?).
* How are sockets related to ports? Can you think of a way to uniquely identify a connection with sockets? What could be some reasons to allow for the reusing of ports?
* Does TCP encrypt packets that are sent over the network? If not, what are some ways in which encryption is carried out?