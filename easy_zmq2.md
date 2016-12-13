# A beginner's guide to ZeroMQ 

Note: This is my own cheat sheet guide to learning ZeroMQ. I have picked out key examples and concepts necessary for a beginner to grasp from [ZeroMQ official guide](http://zguide.zeromq.org/page:all) and [Learning ØMQ with pyzmq](https://learning-0mq-with-pyzmq.readthedocs.io/en/latest/). All the source code included here are in python3. 
---
[Korean] [English]

###### Table of Contents
------
[Exclusive PAIR Pattern](#PAIR)
[Request/Reply Pattern](#REQREP)
[Publish/Subscribe Pattern](#REQREP)
[Push/Pull Pattern](#PUSHPULL)
[Request/Reply Pattern](#REQREP)
---

## Four main types of messaging patterns
There are four main types of messaging patterns in ZeroMQ. 

<a name="PAIR">### Exclusive PAIR Pattern</a>
---
In Exclusive PAIR Pattern, there is only one connected peer. So a client can't connect to many servers or vice versa. However, pair can send any number of sequential messages to each other unlike REQ-REP, which we will talk about later, which has to wait for the response before sending out each message. 

Illustratin of Exclusive PAIR Pattern is shown below as follows: 

Client sends messages every second, and independent of this, server sends its own message every second.

```python
# client.py
import zmq
import random
import sys
import time

port = "5556"
context = zmq.Context()
socket = context.socket(zmq.PAIR)
socket.connect("tcp://localhost:%s" % port)

while True:
    msg = socket.recv()
    print(msg)
    socket.send_string("client message to server1")
    socket.send_string("client message to server2") 
    # sending multiple messages allowed in PAIR pattern
    time.sleep(1)


# server.py
import zmq
import random
import sys
import time

port = "5556"
context = zmq.Context()
socket = context.socket(zmq.PAIR)
socket.bind("tcp://*:%s" % port)

while True:
    socket.send_string("Server message to client3")
    msg = socket.recv()
    print(msg)
    time.sleep(1)
```

<a name="REQREP">### Request/Reply Pattern</a>
---
In a Request/Reply pattern, client sends a `request` and server `replies` to the request.
The communication is sequential in that client will block on *send* unless it has successfully received a reply back. Likewise, server will block on *receive* unless it has received a request from the client. 

Unlike Exclusive PAIR Pattern, client can connect to many servers and server can connect to many clients. 

As a general rule of thumb, the node that does `bind()` is a "server", and the node which does `connect()` is a "client".

Below is an illustration of a client connected to various servers at once:

Client sends requests to two different servers with port 5556 and 5550, respectively 

```python
# client.py

import zmq
import sys

port = "5556"
if len(sys.argv) > 1:
    port =  sys.argv[1]
    int(port)

if len(sys.argv) > 2:
    port1 =  sys.argv[2]
    int(port1)


context = zmq.Context()
print("Connecting to server...")
socket = context.socket(zmq.REQ)
socket.connect ("tcp://localhost:%s" % port)
if len(sys.argv) > 2: # enables client to send reply to one after another
    socket.connect ("tcp://localhost:%s" % port1)


while True: 
    socket.send_string ("What can I get for you?")
    message = socket.recv()
    print("CUSTOMER: ", message)

```

Server1 is binded to port 5000
```python
# server.py

import zmq
import time
import sys

port = "5000"

context = zmq.Context() 
socket = context.socket(zmq.REP) # create a Server socket 
socket.bind("tcp://*:%s" % port) # bind the Server to a port 

while True:
    message = socket.recv() #  Wait for next request from client
    print("Received request: ", message)
    time.sleep (1)  
    socket.send_string("%s need some coffee" % port) # server1 responde with coffee
```

Server2 is binded to port 6000
```python
# server2.py

import zmq
import time
import sys

port = "6000"

context = zmq.Context() 
socket = context.socket(zmq.REP) # create a Server socket 
socket.bind("tcp://*:%s" % port) # bind the Server to a port 

while True:
    message = socket.recv() #  Wait for next request from client
    print("Received request: ", message)
    time.sleep (1)  
    socket.send_string("%s need some hot chocolate" % port) # server2 responde with hot chocolate
```


* Executing the scripts by running the following commands: 
python server1.py
python server2.py
python client.py 5000 6000 


* Output from running `python client.py 5000 6000`
```
Connecting to server...
CUSTOMER:  b'5000 need some coffee'
CUSTOMER:  b'6000 need some hot chocolate'
CUSTOMER:  b'5000 need some coffee'
CUSTOMER:  b'6000 need some hot chocolate'
CUSTOMER:  b'5000 need some coffee'
CUSTOMER:  b'6000 need some hot chocolate'
...
```


Now let's look at a case where multiple clients talk to a single server, and do this asynchronously. 

![multiple clients to single server](./images/multiple-clients-single-server.png)

* How to:
Clients connect to the server and send requests.
For each request, the server sends 0 or more replies.
Clients can send multiple requests without waiting for a reply.
Servers can send multiple replies without waiting for new requests.



<a name="PUBSUB">### Publish/Subscribe Pattern</a>


Publishers, do not program the messages to be sent directly to specific receivers, called subscribers. Messages are published without the knowledge of what or if any subscriber of that knowledge exists. It's the role of subscriber to listen, filter out messages, and stop listening. 


Below is a case where a *subscriber* is connected to more than one *publisher*. 
Two publishers binded to port 5556 and 5546 respectively, and subscriber makes connections to these ports.  

```python
# pub_server.py

import zmq
import random
import sys
import time

port = "5556"
if len(sys.argv) > 1:
    port =  sys.argv[1]
    int(port)

context = zmq.Context()
socket = context.socket(zmq.PUB)
socket.bind("tcp://*:%s" % port)

while True:
    topic = random.randrange(9999,10005)
    messagedata = random.randrange(1,215) - 80
    print("%d %d" % (topic, messagedata))
    socket.send_string("%d %d" % (topic, messagedata))
    time.sleep(1)
```


A subscriber can connect to many publishers as below:

```python
# sub_client.py

import sys
import zmq

port = "5556"
if len(sys.argv) > 1:
    port =  sys.argv[1]
    int(port)
    
if len(sys.argv) > 2:
    port1 =  sys.argv[2]
    int(port1)

# Socket to talk to server
context = zmq.Context()
socket = context.socket(zmq.SUB)

print("Collecting updates from weather server...")
socket.connect ("tcp://localhost:%s" % port)

if len(sys.argv) > 2:
    socket.connect ("tcp://localhost:%s" % port1)

# Subscribe to zipcode, default is NYC, 10001
topicfilter = "10001"
socket.setsockopt_string(zmq.SUBSCRIBE, topicfilter)

# Process 5 updates
total_value = 0
for update_nbr in range (5):
    string = socket.recv()
    topic, messagedata = string.split()
    total_value += int(messagedata)
    print(topic, messagedata)

print("Average messagedata value for topic '%s' was %dF" % (topicfilter, total_value / update_nbr))
```
* Execute as below
python sub_client.py 5556 6000
python pub_server.py 5556
python pub_server.py 6000

* Output 

from pub_server.py 
```
inside pub_server 10003 70
inside pub_server 10000 56
inside pub_server 10002 -3
inside pub_server 10004 2
inside pub_server 10003 53
inside pub_server 10002 87
inside pub_server 10002 -18
inside pub_server 9999 103
inside pub_server 10000 -31
inside pub_server 10001 -61
inside pub_server 10004 35
inside pub_server 10001 84
inside pub_server 9999 -61
inside pub_server 10002 -79
inside pub_server 10001 -52
inside pub_server 10001 23
inside pub_server 10004 101
inside pub_server 10003 104
inside pub_server 10002 22
inside pub_server 10003 -6
inside pub_server 9999 1
inside pub_server 10001 73
inside pub_server 10004 -27
```

Subscriber exits at a point where the publisher publishes "10001" five times

``` 
Collecting updates from weather server...
-- update nbr -- 0
b'10001' b'-61'
-- update nbr -- 1
b'10001' b'84'
-- update nbr -- 2
b'10001' b'-52'
-- update nbr -- 3
b'10001' b'23'
-- update nbr -- 4
b'10001' b'73'
```

<skip this part>
* Note on pub-sub pattern 
-  you do not know precisely when a subscriber starts to get messages. Even if you start a subscriber, wait a while, and then start the publisher, the subscriber will always miss the first messages that the publisher sends. This is because as the subscriber connects to the publisher (something that takes a small but non-zero time), the publisher may already be sending messages out.

- To fix this, learn how to synchronize a publisher and subscribers so that you don't start to publish data until the subscribers really are connected and ready. There is a simple and stupid way to delay the publisher, which is to sleep. Don't do this in a real application, though, because it is extremely fragile as well as inelegant and slow.


Conversely, to illustrate a case where multiple subscribers subscribes to messages/topics being published by a publisher, follow the code below. 
![pub sub](./images/pub-sub.png)


```python
# pub_server.py

import zmq
import random
import sys
import time

port = "5556"
if len(sys.argv) > 1:
    port =  sys.argv[1]
    int(port)

if len(sys.argv) > 2:
    port1 =  sys.argv[2]
    int(port1)

context = zmq.Context()
socket = context.socket(zmq.PUB)
socket.bind("tcp://*:%s" % port)

if len(sys.argv) > 2: 
    socket.bind("tcp://*:%s" % port)

while True:
    topic = random.randrange(9999,10005)
    messagedata = random.randrange(1,215) - 80
    print("%d %d" % (topic, messagedata))
    socket.send_string("%d %d" % (topic, messagedata))
    time.sleep(1)
```

Two *subscribers* receive the same result that the pub_server sends. 
Yet `sub_client1.py` has topic filter set at "10001", while 'sub_client2.py' has topic filter set at "10000". 

```python
# sub_client1.py

import sys
import zmq

port = "5556"
if len(sys.argv) > 1:
    port =  sys.argv[1]
    int(port)
    
# Socket to talk to server
context = zmq.Context()
socket = context.socket(zmq.SUB)

print("Collecting updates from weather server...")
socket.connect ("tcp://localhost:%s" % port)

# Subscribe to zipcode, default is NYC, 10001
topicfilter = "10001"
socket.setsockopt_string(zmq.SUBSCRIBE, topicfilter)

# Process 5 updates
total_value = 0
for update_nbr in range (5):
    string = socket.recv()
    topic, messagedata = string.split()
    total_value += int(messagedata)
    print(topic, messagedata)

print("Average messagedata value for topic '%s' was %dF" % (topicfilter, total_value / update_nbr))

```

Second subscriber has topic filter set at "10001" 
```python
# sub_client2.py

import sys
import zmq

port = "5556"
if len(sys.argv) > 1:
    port =  sys.argv[1]
    int(port)
    
# Socket to talk to server
context = zmq.Context()
socket = context.socket(zmq.SUB)

print("Collecting updates from weather server...")
socket.connect ("tcp://localhost:%s" % port)

# Subscribe to zipcode, default is NYC, 10001
topicfilter = "10000" # different filter!!!
socket.setsockopt_string(zmq.SUBSCRIBE, topicfilter)

# Process 5 updates
total_value = 0
for update_nbr in range (5):
    string = socket.recv()
    topic, messagedata = string.split()
    total_value += int(messagedata)
    print(topic, messagedata)

print("Average messagedata value for topic '%s' was %dF" % (topicfilter, total_value / update_nbr))

```

execute as such: 

python pub_server.py 5556 5546
python sub_client.py 5556
python sub_client.py 5546


<a name="PUBSUB">### Push/Pull Pattern</a>

![push pull](./images/push-pull.png)

Push and Pull sockets let you distribute messages to multiple workers, arranged in a pipeline as shown above. A Push socket will distribute messages to the workers, who will then push the messages received to the final recipient.

*Producer* simply pushes messages to the consumers. 

```python
# producer.py 

import time
import zmq

def producer():
    context = zmq.Context()
    zmq_socket = context.socket(zmq.PUSH)
    zmq_socket.bind("tcp://127.0.0.1:5557")
    # Start your result manager and workers before you start your producers
    for num in xrange(20000):
        work_message = { 'num' : num }
        zmq_socket.send_json(work_message)

producer()

```

*Consumer* (worker) does two things 1) pull messages from producer, 2) push the messaged received to the result collector 

```
# consumer.py

import time
import zmq
import random

def consumer():
    consumer_id = random.randrange(1,10005)
    print("I am consumer #%s" % (consumer_id))
    context = zmq.Context()
    # recieve work
    consumer_receiver = context.socket(zmq.PULL)
    consumer_receiver.connect("tcp://127.0.0.1:5557")
    # send work
    consumer_sender = context.socket(zmq.PUSH)
    consumer_sender.connect("tcp://127.0.0.1:5558")
    
    while True:
        work = consumer_receiver.recv_json()
        data = work['num']
        result = { 'consumer' : consumer_id, 'num' : data}
        if data%2 == 0: 
            consumer_sender.send_json(result)

consumer()

```

* resultcollector.py
- receive messages pushed from the consumers

```
import time
import zmq
import pprint

def result_collector():
    context = zmq.Context()
    results_receiver = context.socket(zmq.PULL)
    results_receiver.bind("tcp://127.0.0.1:5558")
    collecter_data = {}
    for x in xrange(1000):
        result = results_receiver.recv_json()
        if result['consumer'] in collecter_data:
            collecter_data[result['consumer']] = collecter_data[result['consumer']] + 1
        else:
            collecter_data[result['consumer']] = 1
        if x == 999:
            pprint.pprint(collecter_data)

result_collector()


code execution in this order:
python resultcollector.py
python consumer.py 
python consumer.py
python producer.py



* output
python consumer1.py
```
I am consumer #824
```

python consumer1.py
```
I am consumer #567
```

python resultcollector.py
```
{824: 433, 9053: 567}
```

---



Parallel Pipeline

consists of three things:  
- A ventilator that produces tasks that can be done in parallel
- A set of workers that process tasks
- A sink that collects results back from the worker processes

Pub-Sub Synchronization



---
# Advanced usage

1. Extended Request-Reply Pattern (using broker)

![mclient mserver](./images/mclient-mserver.png)

Here, we will examine an example of *multiple clients* connecting to *multiple servers*. Doing this using brute force is one possibility, but it's not scalable. So, we can utilize a broker which binds 
to two endpoints, a frontend for clients and a backend for services.

We can use `zmq_poll()` to monitor these two sockets for activity and when it has some, it shuttles messages between its two sockets. It doesn't actually manage any queues explicitly—ZeroMQ does that automatically on each socket.

Using a request-reply broker makes your client/server architectures easier to scale because clients don't see workers, and workers don't see clients. The only static node is the broker in the middle.


`Client` makes a request. 

```python
# client.py

import zmq

#  Prepare our context and sockets
context = zmq.Context()
socket = context.socket(zmq.REQ)
socket.connect("tcp://localhost:5559")

#  Do 10 requests, waiting each time for a response
for request in range(1,11):
    socket.send(b"Hello")
    message = socket.recv()
    print("Received reply %s [%s]" % (request, message))
```

`Server` longer binds, but connects to the broker. 

```python
# server.py

import zmq

context = zmq.Context()
socket = context.socket(zmq.REP)
socket.connect("tcp://localhost:5560") # Note Server is connected to port 5560 

while True:
    message = socket.recv()
    print("Received request: %s" % message)
    socket.send(b"World")
```

Then the broker binds to both frontend and backend, and delivers messages appropriately. 

```python
# broker.py

import zmq

# Prepare our context and sockets
context = zmq.Context()
frontend = context.socket(zmq.ROUTER)
backend = context.socket(zmq.DEALER)
frontend.bind("tcp://*:5559") # bind to REQ
backend.bind("tcp://*:5560") # bind to REP

# Initialize poll set
poller = zmq.Poller()
poller.register(frontend, zmq.POLLIN) 
poller.register(backend, zmq.POLLIN)

# Switch messages between sockets
while True:
    socks = dict(poller.poll()) # built-in method which helps to read from multiple endpoints at once

    if socks.get(frontend) == zmq.POLLIN: # send message from frontend to backend 
        message = frontend.recv_multipart()
        print("message from frontend", mesasge)
        backend.send_multipart(message)

    if socks.get(backend) == zmq.POLLIN: # send message from backend to frontend
        message = backend.recv_multipart()
        print("message from backend", mesasge)
        frontend.send_multipart(message)
```

Output
from broker.py
```
msg from frontend [b'\x00k\x8bEg', b'', b'Hello']
msg from backend [b'\x00k\x8bEg', b'', b'World']
msg from frontend [b'\x00k\x8bEg', b'', b'Hello']
msg from backend [b'\x00k\x8bEg', b'', b'World']
msg from frontend [b'\x00k\x8bEg', b'', b'Hello']
msg from backend [b'\x00k\x8bEg', b'', b'World']
msg from frontend [b'\x00k\x8bEg', b'', b'Hello']
msg from backend [b'\x00k\x8bEg', b'', b'World']
msg from frontend [b'\x00k\x8bEg', b'', b'Hello']
msg from backend [b'\x00k\x8bEg', b'', b'World']
msg from frontend [b'\x00k\x8bEg', b'', b'Hello']
msg from backend [b'\x00k\x8bEg', b'', b'World']
msg from frontend [b'\x00k\x8bEg', b'', b'Hello']
msg from backend [b'\x00k\x8bEg', b'', b'World']
msg from frontend [b'\x00k\x8bEg', b'', b'Hello']
msg from backend [b'\x00k\x8bEg', b'', b'World']
msg from frontend [b'\x00k\x8bEg', b'', b'Hello']
msg from backend [b'\x00k\x8bEg', b'', b'World']
msg from frontend [b'\x00k\x8bEg', b'', b'Hello']
msg from backend [b'\x00k\x8bEg', b'', b'World']
```

from client.py
```
Received reply 1 [b'World']
Received reply 2 [b'World']
Received reply 3 [b'World']
Received reply 4 [b'World']
Received reply 5 [b'World']
Received reply 6 [b'World']
Received reply 7 [b'World']
Received reply 8 [b'World']
Received reply 9 [b'World']
Received reply 10 [b'World']
```



2. Parallel Pipeline with Kill Signaling (using Push-Pull)

![ventilator](./images/ventilator.png)

To build a parallel pipeline which knows when to kill the workers when the batch is finished, requires that the endpoint knows when to send a kill message to the workers. This incorporates both Push/Pull as well as Pub/Sub pattern.

*Ventilator* pushes tasks to *workers*. *Workers* process those tasks, and send it over to the *Sink*.
*Sink* creates a PUB socket on a new endpoint. *Workers* bind their input socket to this endpoint.
When the *sink* detects the end of the batch, it sends a kill to its PUB socket. When a *worker* detects this kill message, it exits. 

Ventilator creates a PUSH socket which is binded to port 5558, makes a direct connection to sink port.

```python
# ventilator.py

import zmq
import random
import time

try:
    raw_input
except NameError:
    # Python 3
    raw_input = input

context = zmq.Context()

# Socket to send messages on
sender = context.socket(zmq.PUSH)
sender.bind("tcp://*:5557")

# Socket with direct access to the sink: used to synchronize start of batch
sink = context.socket(zmq.PUSH)
sink.connect("tcp://localhost:5558")

print("Press Enter when the workers are ready: ")
_ = raw_input()
print("Sending tasks to workers…")

# The first message is "0" and signals start of batch
sink.send(b'0')

# Initialize random number generator
random.seed()

# Send 100 tasks
total_msec = 0
for task_nbr in range(100):

    # Random workload from 1 to 100 msecs
    workload = random.randint(1, 100)
    total_msec += workload

    sender.send_string(u'%i' % workload)

print("Total expected cost: %s msec" % total_msec)

# Give 0MQ time to deliver
time.sleep(1)
```

*Worker* pulls tasks from ventilator, pushes messages to the Sink. 

```python
# worker.py

import sys
import time
import zmq

context = zmq.Context()

# Socket to receive messages on
receiver = context.socket(zmq.PULL)
receiver.connect("tcp://localhost:5557")

# Socket to send messages to Sink
sender = context.socket(zmq.PUSH)
sender.connect("tcp://localhost:5558")

# Socket for control input 
controller = context.socket(zmq.SUB)
controller.connect("tcp://localhost:5559")
controller.setsockopt(zmq.SUBSCRIBE, b"") # filter out messages

# Process messages from receiver and controller
poller = zmq.Poller()
poller.register(receiver, zmq.POLLIN)
poller.register(controller, zmq.POLLIN)
# Process messages from both sockets
while True:
    socks = dict(poller.poll())

    if socks.get(receiver) == zmq.POLLIN: # message from Ventilator
        message = receiver.recv_string()

        # Process task
        workload = int(message)  # Workload in msecs

        # Do the work
        time.sleep(workload / 1000.0)

        # Send results to sink
        sender.send_string(message) # push the message to Sink

        # Simple progress indicator for the viewer
        sys.stdout.write(".")
        sys.stdout.flush()

    # Any waiting controller command acts as 'KILL'
    if socks.get(controller) == zmq.POLLIN:
        break

# Finished
receiver.close()
sender.close()
controller.close()
context.term()
```

*Controller* manages when to publish the 'kill' message to the Worker.

```python
# sink.py

import sys
import time
import zmq

context = zmq.Context()

# Socket to receive messages on
receiver = context.socket(zmq.PULL)
receiver.bind("tcp://*:5558")

# Socket for worker control
controller = context.socket(zmq.PUB)
controller.bind("tcp://*:5559")

# Wait for start of batch
receiver.recv()

# Start our clock now
tstart = time.time()

# Process 100 confirmiations
for task_nbr in range(100):
    receiver.recv()
    if task_nbr % 10 == 0:
        sys.stdout.write(":")
    else:
        sys.stdout.write(".")
    sys.stdout.flush()

# Calculate and report duration of batch
tend = time.time()
tdiff = tend - tstart
total_msec = tdiff * 1000
print("Total elapsed time: %d msec" % total_msec)

# Send kill signal to workers
controller.send(b"KILL")

# Finished
receiver.close()
controller.close()
context.term()
```


3. Reliable networking (using Request-Reply pattern)

Reliable networking means keepthing things working properly when code freezes or crashes.

If the server dies (while processing a request), the client can figure that out because it won't get an answer back. Then it can give up in a huff, wait and try again later, find another server, and so on. As for the client dying, we can brush that off as "someone else's problem" for now.

We can achieve this in the following ways: 

1. brute force (lazy pirate pattern)
Close and reopen the REQ socket after an error:

```python
# client.py

from __future__ import print_function

import zmq

REQUEST_TIMEOUT = 2500
REQUEST_RETRIES = 3
SERVER_ENDPOINT = "tcp://localhost:5555"

context = zmq.Context(1)

print("I: Connecting to server…")
client = context.socket(zmq.REQ)
client.connect(SERVER_ENDPOINT)

# initialize poller
poll = zmq.Poller()
poll.register(client, zmq.POLLIN)

sequence = 0
retries_left = REQUEST_RETRIES
while retries_left:
    sequence += 1
    request = str(sequence).encode()
    print("I: Sending (%s)" % request)
    client.send(request)

    expect_reply = True
    while expect_reply:
        socks = dict(poll.poll(REQUEST_TIMEOUT)) # resend a request if no reply has arrived within a timedout period
        if socks.get(client) == zmq.POLLIN:
            reply = client.recv()
            if not reply:
                break
            if int(reply) == sequence:
                print("I: Server replied OK (%s)" % reply)
                retries_left = REQUEST_RETRIES
                expect_reply = False
            else:
                print("E: Malformed reply from server: %s" % reply)

        else:
            print("W: No response from server, retrying…")
            # Socket is confused. Close and remove it.
            client.setsockopt(zmq.LINGER, 0)
            client.close()
            poll.unregister(client)
            retries_left -= 1
            if retries_left == 0:
                print("E: Server seems to be offline, abandoning") # completely exit the program
                break
            print("I: Reconnecting and resending (%s)" % request)
            # Create new connection
            client = context.socket(zmq.REQ)
            client.connect(SERVER_ENDPOINT)
            poll.register(client, zmq.POLLIN)
            client.send(request)

context.term()

```

* server-side
```python
# server.py

from __future__ import print_function

from random import randint
import time
import zmq

context = zmq.Context(1)
server = context.socket(zmq.REP)
server.bind("tcp://*:5555")

cycles = 0
while True:
    request = server.recv()
    cycles += 1

    # Simulate various problems, after a few cycles
    if cycles > 3 and randint(0, 3) == 0:
        print("I: Simulating a crash")
        break
    elif cycles > 3 and randint(0, 3) == 0:
        print("I: Simulating CPU overload")
        time.sleep(2)

    print("I: Normal request (%s)" % request)
    time.sleep(1) # Do some heavy work
    server.send(request)

server.close()
context.term()

```

* output 

server 
```

I: Normal request (b'1')
I: Normal request (b'1')
I: Normal request (b'2')
I: Normal request (b'3')
I: Normal request (b'4')
I: Simulating a crash

```

client 
```
I: Connecting to server…
I: Sending (b'1')
W: No response from server, retrying…
I: Reconnecting and resending (b'1')
W: No response from server, retrying…
I: Reconnecting and resending (b'1')
I: Server replied OK (b'1')
I: Sending (b'2')
I: Server replied OK (b'2')
I: Sending (b'3')
I: Server replied OK (b'3')
I: Sending (b'4')
I: Server replied OK (b'4')
I: Sending (b'5')
W: No response from server, retrying…
I: Reconnecting and resending (b'5')
W: No response from server, retrying…
I: Reconnecting and resending (b'5')
W: No response from server, retrying…
E: Server seems to be offline, abandoning
```

Pro: simple to understand and implement.
Pro: works easily with existing client and server application code.
Pro: ZeroMQ automatically retries the actual reconnection until it works.
Con: doesn't failover to backup or alternate servers.


2. Basic Reliable Queuing (Simple Pirate Pattern)



3. Paranoid Reliable Queuing (Paranoid Pirate Pattern)
