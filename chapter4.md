.output chapter4.wd
.bookmark reliable-request-reply
+ Reliable Request-Reply Patterns

[#advanced-request-reply] covered advanced uses of ZeroMQ's request-reply pattern with working examples. This chapter looks at the general question of reliability and builds a set of reliable messaging patterns on top of ZeroMQ's core request-reply pattern.

In this chapter, we focus heavily on user-space request-reply //patterns//, reusable models that help you design your own ZeroMQ architectures:

* The //Lazy Pirate// pattern: reliable request-reply from the client side
* The //Simple Pirate// pattern: reliable request-reply using load balancing
* The //Paranoid Pirate// pattern: reliable request-reply with heartbeating
* The //Majordomo// pattern: service-oriented reliable queuing
* The //Titanic// pattern: disk-based/disconnected reliable queuing
* The //Binary Star// pattern: primary-backup server failover
* The //Freelance// pattern: brokerless reliable request-reply

##  What is "Reliability"?

Most people who speak of "reliability" don't really know what they mean. We can only define reliability in terms of failure. That is, if we can handle a certain set of well-defined and understood failures, then we are reliable with respect to those failures. No more, no less. So let's look at the possible causes of failure in a distributed ZeroMQ application, in roughly descending order of probability:

* Application code is the worst offender. It can crash and exit, freeze and stop responding to input, run too slowly for its input, exhaust all memory, and so on.

* System code--such as brokers we write using ZeroMQ--can die for the same reasons as application code. System code //should// be more reliable than application code, but it can still crash and burn, and especially run out of memory if it tries to queue messages for slow clients.

* Message queues can overflow, typically in system code that has learned to deal brutally with slow clients. When a queue overflows, it starts to discard messages. So we get "lost" messages.

* Networks can fail (e.g., WiFi gets switched off or goes out of range). ZeroMQ will automatically reconnect in such cases, but in the meantime, messages may get lost.

* Hardware can fail and take with it all the processes running on that box.

* Networks can fail in exotic ways, e.g., some ports on a switch may die and those parts of the network become inaccessible.

* Entire data centers can be struck by lightning, earthquakes, fire, or more mundane power or cooling failures.

To make a software system fully reliable against //all// of these possible failures is an enormously difficult and expensive job and goes beyond the scope of this book.

Because the first five cases in the above list cover 99.9% of real world requirements outside large companies (according to a highly scientific study I just ran, which also told me that 78% of statistics are made up on the spot, and moreover never to trust a statistic that we didn't falsify ourselves), that's what we'll examine. If you're a large company with money to spend on the last two cases, contact my company immediately! There's a large hole behind my beach house waiting to be converted into an executive swimming pool.

##  Designing Reliability

So to make things brutally simple, reliability is "keeping things working properly when code freezes or crashes", a situation we'll shorten to "dies". However, the things we want to keep working properly are more complex than just messages. We need to take each core ZeroMQ messaging pattern and see how to make it work (if we can) even when code dies.

Let's take them one-by-one:

* Request-reply: if the server dies (while processing a request), the client can figure that out because it won't get an answer back. Then it can give up in a huff, wait and try again later, find another server, and so on. As for the client dying, we can brush that off as "someone else's problem" for now.

* Pub-sub: if the client dies (having gotten some data), the server doesn't know about it. Pub-sub doesn't send any information back from client to server. But the client can contact the server out-of-band, e.g., via request-reply, and ask, "please resend everything I missed". As for the server dying, that's out of scope for here. Subscribers can also self-verify that they're not running too slowly, and take action (e.g., warn the operator and die) if they are.

* Pipeline: if a worker dies (while working), the ventilator doesn't know about it. Pipelines, like the grinding gears of time, only work in one direction. But the downstream collector can detect that one task didn't get done, and send a message back to the ventilator saying, "hey, resend task 324!" If the ventilator or collector dies, whatever upstream client originally sent the work batch can get tired of waiting and resend the whole lot. It's not elegant, but system code should really not die often enough to matter.

In this chapter we'll focus just on request-reply, which is the low-hanging fruit of reliable messaging.

The basic request-reply pattern (a REQ client socket doing a blocking send/receive to a REP server socket) scores low on handling the most common types of failure. If the server crashes while processing the request, the client just hangs forever. If the network loses the request or the reply, the client hangs forever.

Request-reply is still much better than TCP, thanks to ZeroMQ's ability to reconnect peers silently, to load balance messages, and so on. But it's still not good enough for real work. The only case where you can really trust the basic request-reply pattern is between two threads in the same process where there's no network or separate server process to die.

However, with a little extra work, this humble pattern becomes a good basis for real work across a distributed network, and we get a set of reliable request-reply (RRR) patterns that I like to call the //Pirate// patterns (you'll eventually get the joke, I hope).

There are, in my experience, roughly three ways to connect clients to servers. Each needs a specific approach to reliability:

* Multiple clients talking directly to a single server. Use case: a single well-known server to which clients need to talk. Types of failure we aim to handle: server crashes and restarts, and network disconnects.

* Multiple clients talking to a broker proxy that distributes work to multiple workers. Use case: service-oriented transaction processing. Types of failure we aim to handle: worker crashes and restarts, worker busy looping, worker overload, queue crashes and restarts, and network disconnects.

* Multiple clients talking to multiple servers with no intermediary proxies. Use case: distributed services such as name resolution. Types of failure we aim to handle: service crashes and restarts, service busy looping, service overload, and network disconnects.

Each of these approaches has its trade-offs and often you'll mix them. We'll look at all three in detail.

##  Client-Side Reliability (Lazy Pirate Pattern)

We can get very simple reliable request-reply with some changes to the client. We call this the Lazy Pirate pattern[figure]. Rather than doing a blocking receive, we:

* Poll the REQ socket and receive from it only when it's sure a reply has arrived.
* Resend a request, if no reply has arrived within a timeout period.
* Abandon the transaction if there is still no reply after several requests.

If you try to use a REQ socket in anything other than a strict send/receive fashion, you'll get an error (technically, the REQ socket implements a small finite-state machine to enforce the send/receive ping-pong, and so the error code is called "EFSM"). This is slightly annoying when we want to use REQ in a pirate pattern, because we may send several requests before getting a reply.

The pretty good brute force solution is to close and reopen the REQ socket after an error:

[[code type="example" title="Lazy Pirate client" name="lpclient"]]
[[/code]]

Run this together with the matching server:

[[code type="example" title="Lazy Pirate server" name="lpserver"]]
[[/code]]

[[code type="textdiagram" title="The Lazy Pirate Pattern"]]
#-----------#   #-----------#   #-----------#
|  Client   |   |  Client   |   |  Client   |
+-----------+   +-----------+   +-----------+
|   Retry   |   |   Retry   |   |   Retry   |
+-----------+   +-----------+   +-----------+
|    REQ    |   |    REQ    |   |    REQ    |
'-----------'   '-----------'   '-----------'
      ^               ^               ^
      |               |               |
      '---------------+---------------'
                      |
                      v
               .-------------.
               |     REP     |
               +-------------+
               |             |
               |    Server   |
               |             |
               #-------------#
[[/code]]

To run this test case, start the client and the server in two console windows. The server will randomly misbehave after a few messages. You can check the client's response. Here is typical output from the server:

[[code]]
I: normal request (1)
I: normal request (2)
I: normal request (3)
I: simulating CPU overload
I: normal request (4)
I: simulating a crash
[[/code]]

And here is the client's response:

[[code]]
I: connecting to server...
I: server replied OK (1)
I: server replied OK (2)
I: server replied OK (3)
W: no response from server, retrying...
I: connecting to server...
W: no response from server, retrying...
I: connecting to server...
E: server seems to be offline, abandoning
[[/code]]

The client sequences each message and checks that replies come back exactly in order: that no requests or replies are lost, and no replies come back more than once, or out of order. Run the test a few times until you're convinced that this mechanism actually works. You don't need sequence numbers in a production application; they just help us trust our design.

The client uses a REQ socket, and does the brute force close/reopen because REQ sockets impose that strict send/receive cycle. You might be tempted to use a DEALER instead, but it would not be a good decision. First, it would mean emulating the secret sauce that REQ does with envelopes (if you've forgotten what that is, it's a good sign you don't want to have to do it). Second, it would mean potentially getting back replies that you didn't expect.

Handling failures only at the client works when we have a set of clients talking to a single server. It can handle a server crash, but only if recovery means restarting that same server. If there's a permanent error, such as a dead power supply on the server hardware, this approach won't work. Because the application code in servers is usually the biggest source of failures in any architecture, depending on a single server is not a great idea.

So, pros and cons:

* Pro: simple to understand and implement.
* Pro: works easily with existing client and server application code.
* Pro: ZeroMQ automatically retries the actual reconnection until it works.
* Con: doesn't failover to backup or alternate servers.

##  Basic Reliable Queuing (Simple Pirate Pattern)

Our second approach extends the Lazy Pirate pattern with a queue proxy that lets us talk, transparently, to multiple servers, which we can more accurately call "workers". We'll develop this in stages, starting with a minimal working model, the Simple Pirate pattern.

In all these Pirate patterns, workers are stateless. If the application requires some shared state, such as a shared database, we don't know about it as we design our messaging framework. Having a queue proxy means workers can come and go without clients knowing anything about it. If one worker dies, another takes over. This is a nice, simple topology with only one real weakness, namely the central queue itself, which can become a problem to manage, and a single point of failure.

[[code type="textdiagram" title="The Simple Pirate Pattern"]]
#-----------#   #-----------#   #-----------#
|  Client   |   |  Client   |   |  Client   |
+-----------+   +-----------+   +-----------+
|   Retry   |   |   Retry   |   |   Retry   |
+-----------+   +-----------+   +-----------+
|    REQ    |   |    REQ    |   |    REQ    |
'-----+-----'   '-----+-----'   '-----+-----'
      |               |               |
      '---------------+---------------'
                      |
                      v
                .-----------.
                |  ROUTER   |
                +-----------+
                |   Load    |
                |  balancer |
                +-----------+
                |  ROUTER   |
                '-----------'
                      ^
                      |
      .---------------+---------------.
      |               |               |
.-----+-----.   .-----+-----.   .-----+-----.
|    REQ    |   |    REQ    |   |    REQ    |
+-----------+   +-----------+   +-----------+
|   Worker  |   |   Worker  |   |   Worker  |
#-----------#   #-----------#   #-----------#
[[/code]]

The basis for the queue proxy is the load balancing broker from [#advanced-request-reply]. What is the very //minimum// we need to do to handle dead or blocked workers? Turns out, it's surprisingly little. We already have a retry mechanism in the client. So using the load balancing pattern will work pretty well. This fits with ZeroMQ's philosophy that we can extend a peer-to-peer pattern like request-reply by plugging naive proxies in the middle[figure].

We don't need a special client; we're still using the Lazy Pirate client. Here is the queue, which is identical to the main task of the load balancing broker:

[[code type="example" title="Simple Pirate queue" name="spqueue"]]
[[/code]]

Here is the worker, which takes the Lazy Pirate server and adapts it for the load balancing pattern (using the REQ "ready" signaling):

[[code type="example" title="Simple Pirate worker" name="spworker"]]
[[/code]]

To test this, start a handful of workers, a Lazy Pirate client, and the queue, in any order. You'll see that the workers eventually all crash and burn, and the client retries and then gives up. The queue never stops, and you can restart workers and clients ad nauseam. This model works with any number of clients and workers.

##  Robust Reliable Queuing (Paranoid Pirate Pattern)

[[code type="textdiagram" title="The Paranoid Pirate Pattern"]]
#-----------#   #-----------#   #-----------#
|  Client   |   |  Client   |   |  Client   |
+-----------+   +-----------+   +-----------+
|   Retry   |   |   Retry   |   |   Retry   |
+-----------+   +-----------+   +-----------+
|    REQ    |   |    REQ    |   |    REQ    |
'-----+-----'   '-----+-----'   '-----+-----'
      |               |               |
      '---------------+---------------'
                      |
                      v
                .-----------.
                |  ROUTER   |
                +-----------+
                |   Queue   |
                +-----------+
                | Heartbeat |
                +-----------+
                |  ROUTER   |
                '-----------'
                      ^
                      |
      .---------------+---------------.
      |               |               |
.-----+-----.   .-----+-----.   .-----+-----.
|  DEALER   |   |  DEALER   |   |  DEALER   |
+-----------+   +-----------+   +-----------+
| Heartbeat |   | Heartbeat |   | Heartbeat |
+-----------+   +-----------+   +-----------+
|   Worker  |   |   Worker  |   |   Worker  |
#-----------#   #-----------#   #-----------#
[[/code]]

The Simple Pirate Queue pattern works pretty well, especially because it's just a combination of two existing patterns. Still, it does have some weaknesses:

* It's not robust in the face of a queue crash and restart. The client will recover, but the workers won't. While ZeroMQ will reconnect workers' sockets automatically, as far as the newly started queue is concerned, the workers haven't signaled ready, so don't exist. To fix this, we have to do heartbeating from queue to worker so that the worker can detect when the queue has gone away.

* The queue does not detect worker failure, so if a worker dies while idle, the queue can't remove it from its worker queue until the queue sends it a request. The client waits and retries for nothing. It's not a critical problem, but it's not nice. To make this work properly, we do heartbeating from worker to queue, so that the queue can detect a lost worker at any stage.

We'll fix these in a properly pedantic Paranoid Pirate Pattern.

We previously used a REQ socket for the worker. For the Paranoid Pirate worker, we'll switch to a DEALER socket[figure]. This has the advantage of letting us send and receive messages at any time, rather than the lock-step send/receive that REQ imposes. The downside of DEALER is that we have to do our own envelope management (re-read [#advanced-request-reply] for background on this concept).

We're still using the Lazy Pirate client. Here is the Paranoid Pirate queue proxy:

[[code type="example" title="Paranoid Pirate queue" name="ppqueue"]]
[[/code]]

The queue extends the load balancing pattern with heartbeating of workers. Heartbeating is one of those "simple" things that can be difficult to get right. I'll explain more about that in a second.

Here is the Paranoid Pirate worker:

[[code type="example" title="Paranoid Pirate worker" name="ppworker"]]
[[/code]]

Some comments about this example:

* The code includes simulation of failures, as before. This makes it (a) very hard to debug, and (b) dangerous to reuse. When you want to debug this, disable the failure simulation.

* The worker uses a reconnect strategy similar to the one we designed for the Lazy Pirate client, with two major differences: (a) it does an exponential back-off, and (b) it retries indefinitely (whereas the client retries a few times before reporting a failure).

Try the client, queue, and workers, such as by using a script like this:

[[code]]
ppqueue &
for i in 1 2 3 4; do
    ppworker &
    sleep 1
done
lpclient &
[[/code]]

You should see the workers die one-by-one as they simulate a crash, and the client eventually give up. You can stop and restart the queue and both client and workers will reconnect and carry on. And no matter what you do to queues and workers, the client will never get an out-of-order reply: the whole chain either works, or the client abandons.

##  Heartbeating

Heartbeating solves the problem of knowing whether a peer is alive or dead. This is not an issue specific to ZeroMQ. TCP has a long timeout (30 minutes or so), that means that it can be impossible to know whether a peer has died, been disconnected, or gone on a weekend to Prague with a case of vodka, a redhead, and a large expense account.

It's not easy to get heartbeating right. When writing the Paranoid Pirate examples, it took about five hours to get the heartbeating working properly. The rest of the request-reply chain took perhaps ten minutes. It is especially easy to create "false failures", i.e., when peers decide that they are disconnected because the heartbeats aren't sent properly.

We'll look at the three main answers people use for heartbeating with ZeroMQ.

###  Shrugging It Off

The most common approach is to do no heartbeating at all and hope for the best. Many if not most ZeroMQ applications do this. ZeroMQ encourages this by hiding peers in many cases. What problems does this approach cause?

* When we use a ROUTER socket in an application that tracks peers, as peers disconnect and reconnect, the application will leak memory (resources that the application holds for each peer) and get slower and slower.

* When we use SUB- or DEALER-based data recipients, we can't tell the difference between good silence (there's no data) and bad silence (the other end died). When a recipient knows the other side died, it can for example switch over to a backup route.

* If we use a TCP connection that stays silent for a long while, it will, in some networks, just die. Sending something (technically, a "keep-alive" more than a heartbeat), will keep the network alive.

###  One-Way Heartbeats

A second option is to send a heartbeat message from each node to its peers every second or so. When one node hears nothing from another within some timeout (several seconds, typically), it will treat that peer as dead. Sounds good, right? Sadly, no. This works in some cases but has nasty edge cases in others.

For pub-sub, this does work, and it's the only model you can use. SUB sockets cannot talk back to PUB sockets, but PUB sockets can happily send "I'm alive" messages to their subscribers.

As an optimization, you can send heartbeats only when there is no real data to send. Furthermore, you can send heartbeats progressively slower and slower, if network activity is an issue (e.g., on mobile networks where activity drains the battery). As long as the recipient can detect a failure (sharp stop in activity), that's fine.

Here are the typical problems with this design:

* It can be inaccurate when we send large amounts of data, as heartbeats will be delayed behind that data. If heartbeats are delayed, you can get false timeouts and disconnections due to network congestion. Thus, always treat //any// incoming data as a heartbeat, whether or not the sender optimizes out heartbeats.

* While the pub-sub pattern will drop messages for disappeared recipients, PUSH and DEALER sockets will queue them. So if you send heartbeats to a dead peer and it comes back, it will get all the heartbeats you sent, which can be thousands. Whoa, whoa!

* This design assumes that heartbeat timeouts are the same across the whole network. But that won't be accurate. Some peers will want very aggressive heartbeating in order to detect faults rapidly. And some will want very relaxed heartbeating, in order to let sleeping networks lie and save power.

###  Ping-Pong Heartbeats

The third option is to use a ping-pong dialog. One peer sends a ping command to the other, which replies with a pong command. Neither command has any payload. Pings and pongs are not correlated. Because the roles of "client" and "server" are arbitrary in some networks, we usually specify that either peer can in fact send a ping and expect a pong in response. However, because the timeouts depend on network topologies known best to dynamic clients, it is usually the client that pings the server.

This works for all ROUTER-based brokers. The same optimizations we used in the second model make this work even better: treat any incoming data as a pong, and only send a ping when not otherwise sending data.

###  Heartbeating for Paranoid Pirate

For Paranoid Pirate, we chose the second approach. It might not have been the simplest option: if designing this today, I'd probably try a ping-pong approach instead. However the principles are similar. The heartbeat messages flow asynchronously in both directions, and either peer can decide the other is "dead" and stop talking to it.

In the worker, this is how we handle heartbeats from the queue:

* We calculate a //liveness//, which is how many heartbeats we can still miss before deciding the queue is dead. It starts at three and we decrement it each time we miss a heartbeat.
* We wait, in the {{zmq_poll}} loop, for one second each time, which is our heartbeat interval.
* If there's any message from the queue during that time, we reset our liveness to three.
* If there's no message during that time, we count down our liveness.
* If the liveness reaches zero, we consider the queue dead.
* If the queue is dead, we destroy our socket, create a new one, and reconnect.
* To avoid opening and closing too many sockets, we wait for a certain interval before reconnecting, and we double the interval each time until it reaches 32 seconds.

And this is how we handle heartbeats //to// the queue:

* We calculate when to send the next heartbeat; this is a single variable because we're talking to one peer, the queue.
* In the {{zmq_poll}} loop, whenever we pass this time, we send a heartbeat to the queue.

Here's the essential heartbeating code for the worker:

[[code type="fragment" name="heartbeats"]]
#define HEARTBEAT_LIVENESS  3       //  3-5 is reasonable
#define HEARTBEAT_INTERVAL  1000    //  msecs
#define INTERVAL_INIT       1000    //  Initial reconnect
#define INTERVAL_MAX       32000    //  After exponential backoff

...
//  If liveness hits zero, queue is considered disconnected
size_t liveness = HEARTBEAT_LIVENESS;
size_t interval = INTERVAL_INIT;

//  Send out heartbeats at regular intervals
uint64_t heartbeat_at = zclock_time () + HEARTBEAT_INTERVAL;

while (true) {
    zmq_pollitem_t items [] = { { worker,  0, ZMQ_POLLIN, 0 } };
    int rc = zmq_poll (items, 1, HEARTBEAT_INTERVAL * ZMQ_POLL_MSEC);

    if (items [0].revents & ZMQ_POLLIN) {
        //  Receive any message from queue
        liveness = HEARTBEAT_LIVENESS;
        interval = INTERVAL_INIT;
    }
    else
    if (--liveness == 0) {
        zclock_sleep (interval);
        if (interval < INTERVAL_MAX)
            interval *= 2;
        zsocket_destroy (ctx, worker);
        ...
        liveness = HEARTBEAT_LIVENESS;
    }
    //  Send heartbeat to queue if it's time
    if (zclock_time () > heartbeat_at) {
        heartbeat_at = zclock_time () + HEARTBEAT_INTERVAL;
        //  Send heartbeat message to queue
    }
}
[[/code]]

The queue does the same, but manages an expiration time for each worker.

Here are some tips for your own heartbeating implementation:

* Use {{zmq_poll}} or a reactor as the core of your application's main task.

* Start by building the heartbeating between peers, test it by simulating failures, and //then// build the rest of the message flow. Adding heartbeating afterwards is much trickier.

* Use simple tracing, i.e., print to console, to get this working. To help you trace the flow of messages between peers, use a dump method such as zmsg offers, and number your messages incrementally so you can see if there are gaps.

* In a real application, heartbeating must be configurable and usually negotiated with the peer. Some peers will want aggressive heartbeating, as low as 10 msecs. Other peers will be far away and want heartbeating as high as 30 seconds.

* If you have different heartbeat intervals for different peers, your poll timeout should be the lowest (shortest time) of these. Do not use an infinite timeout.

* Do heartbeating on the same socket you use for messages, so your heartbeats also act as a //keep-alive// to stop the network connection from going stale (some firewalls can be unkind to silent connections).

##  Contracts and Protocols

If you're paying attention, you'll realize that Paranoid Pirate is not interoperable with Simple Pirate, because of the heartbeats. But how do we define "interoperable"? To guarantee interoperability, we need a kind of contract, an agreement that lets different teams in different times and places write code that is guaranteed to work together. We call this a "protocol".

It's fun to experiment without specifications, but that's not a sensible basis for real applications. What happens if we want to write a worker in another language? Do we have to read code to see how things work? What if we want to change the protocol for some reason? Even a simple protocol will, if it's successful, evolve and become more complex.

Lack of contracts is a sure sign of a disposable application. So let's write a contract for this protocol. How do we do that?

There's a wiki at [http://rfc.zeromq.org rfc.zeromq.org] that we made especially as a home for public ZeroMQ contracts.
To create a new specification, register on the wiki if needed, and follow the instructions. It's fairly straightforward, though writing technical texts is not everyone's cup of tea.

It took me about fifteen minutes to draft the new [http://rfc.zeromq.org/spec:6 Pirate Pattern Protocol]. It's not a big specification, but it does capture enough to act as the basis for arguments ("your queue isn't PPP compatible; please fix it!").

Turning PPP into a real protocol would take more work:

* There should be a protocol version number in the READY command so that it's possible to distinguish between different versions of PPP.

* Right now, READY and HEARTBEAT are not entirely distinct from requests and replies. To make them distinct, we would need a message structure that includes a "message type" part.

##  Service-Oriented Reliable Queuing (Majordomo Pattern)

[[code type="textdiagram" title="The Majordomo Pattern"]]
#-----------#   #-----------#   #-----------#
|  Client   |   |  Client   |   |  Client   |
'-----+-----'   '-----+-----'   '-----+-----'
      |               |               |
      '---------------+---------------'
"Give me coffee"      |         "Give me tea"
                      v
                .-----------.
                |  Broker   |
                '-----------'
                      ^
                      |
      .---------------+---------------.
      |               |               |
.-----+-----.   .-----+-----.   .-----+-----.
|  "Water"  |   |   "Tea"   |   | "Coffee"  |
+-----------+   +-----------+   +-----------+
|  Worker   |   |  Worker   |   |  Worker   |
#-----------#   #-----------#   #-----------#
[[/code]]

The nice thing about progress is how fast it happens when lawyers and committees aren't involved. The [http://rfc.zeromq.org/spec:7 one-page MDP specification] turns PPP into something more solid[figure]. This is how we should design complex architectures: start by writing down the contracts, and only //then// write software to implement them.

The Majordomo Protocol (MDP) extends and improves on PPP in one interesting way: it adds a "service name" to requests that the client sends, and asks workers to register for specific services. Adding service names turns our Paranoid Pirate queue into a service-oriented broker. The nice thing about MDP is that it came out of working code, a simpler ancestor protocol (PPP), and a precise set of improvements that each solved a clear problem. This made it easy to draft.

To implement Majordomo, we need to write a framework for clients and workers. It's really not sane to ask every application developer to read the spec and make it work, when they could be using a simpler API that does the work for them.

So while our first contract (MDP itself) defines how the pieces of our distributed architecture talk to each other, our second contract defines how user applications talk to the technical framework we're going to design.

Majordomo has two halves, a client side and a worker side. Because we'll write both client and worker applications, we will need two APIs. Here is a sketch for the client API, using a simple object-oriented approach:

[[code type="fragment" name="mdclient"]]
mdcli_t *mdcli_new     (char *broker);
void     mdcli_destroy (mdcli_t **self_p);
zmsg_t  *mdcli_send    (mdcli_t *self, char *service, zmsg_t **request_p);
[[/code]]

That's it. We open a session to the broker, send a request message, get a reply message back, and eventually close the connection. Here's a sketch for the worker API:

[[code type="fragment" name="mdworker"]]
mdwrk_t *mdwrk_new     (char *broker,char *service);
void     mdwrk_destroy (mdwrk_t **self_p);
zmsg_t  *mdwrk_recv    (mdwrk_t *self, zmsg_t *reply);
[[/code]]

It's more or less symmetrical, but the worker dialog is a little different. The first time a worker does a recv(), it passes a null reply. Thereafter, it passes the current reply, and gets a new request.

The client and worker APIs were fairly simple to construct because they're heavily based on the Paranoid Pirate code we already developed. Here is the client API:

[[code type="example" title="Majordomo client API" name="mdcliapi"]]
[[/code]]

Let's see how the client API looks in action, with an example test program that does 100K request-reply cycles:

[[code type="example" title="Majordomo client application" name="mdclient"]]
[[/code]]

And here is the worker API:

[[code type="example" title="Majordomo worker API" name="mdwrkapi"]]
[[/code]]

Let's see how the worker API looks in action, with an example test program that implements an echo service:

[[code type="example" title="Majordomo worker application" name="mdworker"]]
[[/code]]

Here are some things to note about the worker API code:

* The APIs are single-threaded. This means, for example, that the worker won't send heartbeats in the background. Happily, this is exactly what we want: if the worker application gets stuck, heartbeats will stop and the broker will stop sending requests to the worker.

* The worker API doesn't do an exponential back-off; it's not worth the extra complexity.

* The APIs don't do any error reporting. If something isn't as expected, they raise an assertion (or exception depending on the language). This is ideal for a reference implementation, so any protocol errors show immediately. For real applications, the API should be robust against invalid messages.

You might wonder why the worker API is manually closing its socket and opening a new one, when ZeroMQ will automatically reconnect a socket if the peer disappears and comes back. Look back at the Simple Pirate and Paranoid Pirate workers to understand. Although ZeroMQ will automatically reconnect workers if the broker dies and comes back up, this isn't sufficient to re-register the workers with the broker. I know of at least two solutions. The simplest, which we use here, is for the worker to monitor the connection using heartbeats, and if it decides the broker is dead, to close its socket and start afresh with a new socket. The alternative is for the broker to challenge unknown workers when it gets a heartbeat from the worker and ask them to re-register. That would require protocol support.

Now let's design the Majordomo broker. Its core structure is a set of queues, one per service. We will create these queues as workers appear (we could delete them as workers disappear, but forget that for now because it gets complex). Additionally, we keep a queue of workers per service.

And here is the broker:

[[code type="example" title="Majordomo broker" name="mdbroker"]]
[[/code]]

This is by far the most complex example we've seen. It's almost 500 lines of code. To write this and make it somewhat robust took two days. However, this is still a short piece of code for a full service-oriented broker.

Here are some things to note about the broker code:

* The Majordomo Protocol lets us handle both clients and workers on a single socket. This is nicer for those deploying and managing the broker: it just sits on one ZeroMQ endpoint rather than the two that most proxies need.

* The broker implements all of MDP/0.1 properly (as far as I know), including disconnection if the broker sends invalid commands, heartbeating, and the rest.

* It can be extended to run multiple threads, each managing one socket and one set of clients and workers. This could be interesting for segmenting large architectures. The C code is already organized around a broker class to make this trivial.

* A primary/failover or live/live broker reliability model is easy, as the broker essentially has no state except service presence. It's up to clients and workers to choose another broker if their first choice isn't up and running.

* The examples use five-second heartbeats, mainly to reduce the amount of output when you enable tracing. Realistic values would be lower for most LAN applications. However, any retry has to be slow enough to allow for a service to restart, say 10 seconds at least.

We later improved and extended the protocol and the Majordomo implementation, which now sits in its own Github project. If you want a properly usable Majordomo stack, use the GitHub project.

##  Asynchronous Majordomo Pattern

The Majordomo implementation in the previous section is simple and stupid. The client is just the original Simple Pirate, wrapped up in a sexy API. When I fire up a client, broker, and worker on a test box, it can process 100,000 requests in about 14 seconds. That is partially due to the code, which cheerfully copies message frames around as if CPU cycles were free. But the real problem is that we're doing network round-trips. ZeroMQ disables [http://en.wikipedia.org/wiki/Nagles_algorithm Nagle's algorithm], but round-tripping is still slow.

Theory is great in theory, but in practice, practice is better. Let's measure the actual cost of round-tripping with a simple test program. This sends a bunch of messages, first waiting for a reply to each message, and second as a batch, reading all the replies back as a batch. Both approaches do the same work, but they give very different results. We mock up a client, broker, and worker:

[[code type="example" title="Round-trip demonstrator" name="tripping"]]
[[/code]]

On my development box, this program says:

[[code]]
Setting up test...
Synchronous round-trip test...
 9057 calls/second
Asynchronous round-trip test...
 173010 calls/second
[[/code]]

Note that the client thread does a small pause before starting. This is to get around one of the "features" of the router socket: if you send a message with the address of a peer that's not yet connected, the message gets discarded. In this example we don't use the load balancing mechanism, so without the sleep, if the worker thread is too slow to connect, it will lose messages, making a mess of our test.

As we see, round-tripping in the simplest case is 20 times slower than the  asynchronous, "shove it down the pipe as fast as it'll go" approach. Let's see if we can apply this to Majordomo to make it faster.

First, we modify the client API to send and receive in two separate methods:

[[code type="fragment" name="mdclient-async"]]
mdcli_t *mdcli_new     (char *broker);
void     mdcli_destroy (mdcli_t **self_p);
int      mdcli_send    (mdcli_t *self, char *service, zmsg_t **request_p);
zmsg_t  *mdcli_recv    (mdcli_t *self);
[[/code]]

It's literally a few minutes' work to refactor the synchronous client API to become asynchronous:

[[code type="example" title="Majordomo asynchronous client API" name="mdcliapi2"]]
[[/code]]

The differences are:

* We use a DEALER socket instead of REQ, so we emulate REQ with an empty delimiter frame before each request and each response.
* We don't retry requests; if the application needs to retry, it can do this itself.
* We break the synchronous {{send}} method into separate {{send}} and {{recv}} methods.
* The {{send}} method is asynchronous and returns immediately after sending. The caller can thus send a number of messages before getting a response.
* The {{recv}} method waits for (with a timeout) one response and returns that to the caller.

And here's the corresponding client test program, which sends 100,000 messages and then receives 100,000 back:

[[code type="example" title="Majordomo client application" name="mdclient2"]]
[[/code]]

The broker and worker are unchanged because we've not modified the protocol at all. We see an immediate improvement in performance. Here's the synchronous client chugging through 100K request-reply cycles:

[[code]]
$ time mdclient
100000 requests/replies processed

real    0m14.088s
user    0m1.310s
sys     0m2.670s
[[/code]]

And here's the asynchronous client, with a single worker:

[[code]]
$ time mdclient2
100000 replies received

real    0m8.730s
user    0m0.920s
sys     0m1.550s
[[/code]]

Twice as fast. Not bad, but let's fire up 10 workers and see how it handles the traffic

[[code]]
$ time mdclient2
100000 replies received

real    0m3.863s
user    0m0.730s
sys     0m0.470s
[[/code]]

It isn't fully asynchronous because workers get their messages on a strict last-used basis. But it will scale better with more workers. On my PC, after eight or so workers, it doesn't get any faster. Four cores only stretches so far. But we got a 4x improvement in throughput with just a few minutes' work. The broker is still unoptimized. It spends most of its time copying message frames around, instead of doing zero-copy, which it could. But we're getting 25K reliable request/reply calls a second, with pretty low effort.

However, the asynchronous Majordomo pattern isn't all roses. It has a fundamental weakness, namely that it cannot survive a broker crash without more work. If you look at the {{mdcliapi2}} code you'll see it does not attempt to reconnect after a failure. A proper reconnect would require the following:

* A number on every request and a matching number on every reply, which would ideally require a change to the protocol to enforce.
* Tracking and holding onto all outstanding requests in the client API, i.e., those for which no reply has yet been received.
* In case of failover, for the client API to //resend// all outstanding requests to the broker.

It's not a deal breaker, but it does show that performance often means complexity. Is this worth doing for Majordomo? It depends on your use case. For a name lookup service you call once per session, no. For a web frontend serving thousands of clients, probably yes.

##  Service Discovery

So, we have a nice service-oriented broker, but we have no way of knowing whether a particular service is available or not. We know whether a request failed, but we don't know why. It is useful to be able to ask the broker, "is the echo service running?" The most obvious way would be to modify our MDP/Client protocol to add commands to ask this. But MDP/Client has the great charm of being simple. Adding service discovery to it would make it as complex as the MDP/Worker protocol.

Another option is to do what email does, and ask that undeliverable requests be returned. This can work well in an asynchronous world, but it also adds complexity. We need ways to distinguish returned requests from replies and to handle these properly.

Let's try to use what we've already built, building on top of MDP instead of modifying it. Service discovery is, itself, a service. It might indeed be one of several management services, such as "disable service X", "provide statistics", and so on. What we want is a general, extensible solution that doesn't affect the protocol or existing applications.

So here's a small RFC that layers this on top of MDP: [http://rfc.zeromq.org/spec:8 the Majordomo Management Interface (MMI)]. We already implemented it in the broker, though unless you read the whole thing you probably missed that. I'll explain how it works in the broker:

* When a client requests a service that starts with {{mmi.}}, instead of routing this to a worker, we handle it internally.

* We handle just one service in this broker, which is {{mmi.service}}, the service discovery service.

* The payload for the request is the name of an external service (a real one, provided by a worker).

* The broker returns "200" (OK) or "404" (Not found), depending on whether there are workers registered for that service or not.

Here's how we use the service discovery in an application:

[[code type="example" title="Service discovery over Majordomo" name="mmiecho"]]
[[/code]]

Try this with and without a worker running, and you should see the little program report "200" or "404" accordingly. The implementation of MMI in our example broker is flimsy. For example, if a worker disappears, services remain "present". In practice, a broker should remove services that have no workers after some configurable timeout.

##  Idempotent Services

Idempotency is not something you take a pill for. What it means is that it's safe to repeat an operation. Checking the clock is idempotent. Lending ones credit card to ones children is not. While many client-to-server use cases are idempotent, some are not. Examples of idempotent use cases include:

* Stateless task distribution, i.e., a pipeline where the servers are stateless workers that compute a reply based purely on the state provided by a request. In such a case, it's safe (though inefficient) to execute the same request many times.

* A name service that translates logical addresses into endpoints to bind or connect to. In such a case, it's safe to make the same lookup request many times.

And here are examples of a non-idempotent use cases:

* A logging service. One does not want the same log information recorded more than once.

* Any service that has impact on downstream nodes, e.g., sends on information to other nodes. If that service gets the same request more than once, downstream nodes will get duplicate information.

* Any service that modifies shared data in some non-idempotent way; e.g., a service that debits a bank account is not idempotent without extra work.

When our server applications are not idempotent, we have to think more carefully about when exactly they might crash. If an application dies when it's idle, or while it's processing a request, that's usually fine. We can use database transactions to make sure a debit and a credit are always done together, if at all. If the server dies while sending its reply, that's a problem, because as far as it's concerned, it has done its work.

If the network dies just as the reply is making its way back to the client, the same problem arises. The client will think the server died and will resend the request, and the server will do the same work twice, which is not what we want.

To handle non-idempotent operations, use the fairly standard solution of detecting and rejecting duplicate requests. This means:

* The client must stamp every request with a unique client identifier and a unique message number.

* The server, before sending back a reply, stores it using the combination of client ID and message number as a key.

* The server, when getting a request from a given client, first checks whether it has a reply for that client ID and message number. If so, it does not process the request, but just resends the reply.

##  Disconnected Reliability (Titanic Pattern)

Once you realize that Majordomo is a "reliable" message broker, you might be tempted to add some spinning rust (that is, ferrous-based hard disk platters). After all, this works for all the enterprise messaging systems. It's such a tempting idea that it's a little sad to have to be negative toward it. But brutal cynicism is one of my specialties. So, some reasons you don't want rust-based brokers sitting in the center of your architecture are:

* As you've seen, the Lazy Pirate client performs surprisingly well. It works across a whole range of architectures, from direct client-to-server to distributed queue proxies. It does tend to assume that workers are stateless and idempotent. But we can work around that limitation without resorting to rust.

* Rust brings a whole set of problems, from slow performance to additional pieces that you have to manage, repair, and handle 6 a.m. panics from, as they inevitably break at the start of daily operations. The beauty of the Pirate patterns in general is their simplicity. They won't crash. And if you're still worried about the hardware, you can move to a peer-to-peer pattern that has no broker at all. I'll explain later in this chapter.

Having said this, however, there is one sane use case for rust-based reliability, which is an asynchronous disconnected network. It solves a major problem with Pirate, namely that a client has to wait for an answer in real time. If clients and workers are only sporadically connected (think of email as an analogy), we can't use a stateless network between clients and workers. We have to put state in the middle.

So, here's the Titanic pattern[figure], in which we write messages to disk to ensure they never get lost, no matter how sporadically clients and workers are connected. As we did for service discovery, we're going to layer Titanic on top of MDP rather than extend it. It's wonderfully lazy because it means we can implement our fire-and-forget reliability in a specialized worker, rather than in the broker. This is excellent for several reasons:

* It is //much// easier because we divide and conquer: the broker handles message routing and the worker handles reliability.
* It lets us mix brokers written in one language with workers written in another.
* It lets us evolve the fire-and-forget technology independently.

The only downside is that there's an extra network hop between broker and hard disk. The benefits are easily worth it.

There are many ways to make a persistent request-reply architecture. We'll aim for one that is simple and painless. The simplest design I could come up with, after playing with this for a few hours, is a "proxy service". That is, Titanic doesn't affect workers at all. If a client wants a reply immediately, it talks directly to a service and hopes the service is available. If a client is happy to wait a while, it talks to Titanic instead and asks, "hey, buddy, would you take care of this for me while I go buy my groceries?"

[[code type="textdiagram" title="The Titanic Pattern"]]
#-----------#   #-----------#   #-----------#
|           |   |           |   |           |
|  Client   |   |  Client   |   |  Client   |
|           |   |           |   |           |
'-----------'   '-----------'   '-----------'
      ^               ^               ^
      |               |               |
      '---------------+---------------'
"Titanic,             |         "Titanic,
 give me coffee"      |          give me tea"
                      v                                
                .-----------.     #---------#     .-------.
                |           |     |         |     |       |
                |  Broker   |<--->| Titanic |<--->| Disk  |
                |           |     |         |     |       |
                '-----------'     #---------#     '-------'
                      ^
                      |
      .---------------+---------------.
      |               |               |
      v               v               v
.-----------.   .-----------.   .-----------.
|  "Water"  |   |   "Tea"   |   | "Coffee"  |
+-----------+   +-----------+   +-----------+
|  Worker   |   |  Worker   |   |  Worker   |
#-----------#   #-----------#   #-----------#
[[/code]]

Titanic is thus both a worker and a client. The dialog between client and Titanic goes along these lines:

* Client: Please accept this request for me. Titanic: OK, done.
* Client: Do you have a reply for me? Titanic: Yes, here it is. Or, no, not yet.
* Client: OK, you can wipe that request now, I'm happy. Titanic: OK, done.

Whereas the dialog between Titanic and broker and worker goes like this:

* Titanic: Hey, Broker, is there an coffee service? Broker: Uhm, Yeah, seems like.
* Titanic: Hey, coffee service, please handle this for me.
* Coffee: Sure, here you are.
* Titanic: Sweeeeet!

You can work through this and the possible failure scenarios. If a worker crashes while processing a request, Titanic retries indefinitely. If a reply gets lost somewhere, Titanic will retry. If the request gets processed but the client doesn't get the reply, it will ask again. If Titanic crashes while processing a request or a reply, the client will try again. As long as requests are fully committed to safe storage, work can't get lost.

The handshaking is pedantic, but can be pipelined, i.e., clients can use the asynchronous Majordomo pattern to do a lot of work and then get the responses later.

We need some way for a client to request //its// replies. We'll have many clients asking for the same services, and clients disappear and reappear with different identities. Here is a simple, reasonably secure solution:

* Every request generates a universally unique ID (UUID), which Titanic returns to the client after it has queued the request.
* When a client asks for a reply, it must specify the UUID for the original request.

In a realistic case, the client would want to store its request UUIDs safely, e.g., in a local database.

Before we jump off and write yet another formal specification (fun, fun!), let's consider how the client talks to Titanic. One way is to use a single service and send it three different request types. Another way, which seems simpler, is to use three services:

* {{titanic.request}}: store a request message, and return a UUID for the request.
* {{titanic.reply}}: fetch a reply, if available, for a given request UUID.
* {{titanic.close}}: confirm that a reply has been stored and processed.

We'll just make a multithreaded worker, which as we've seen from our multithreading experience with ZeroMQ, is trivial. However, let's first sketch what Titanic would look like in terms of ZeroMQ messages and frames. This gives us the [http://rfc.zeromq.org/spec:9 Titanic Service Protocol (TSP)].

Using TSP is clearly more work for client applications than accessing a service directly via MDP. Here's the shortest robust "echo" client example:

[[code type="example" title="Titanic client example" name="ticlient"]]
[[/code]]

Of course this can be, and should be, wrapped up in some kind of framework or API. It's not healthy to ask average application developers to learn the full details of messaging: it hurts their brains, costs time, and offers too many ways to make buggy complexity. Additionally, it makes it hard to add intelligence. 

For example, this client blocks on each request whereas in a real application, we'd want to be doing useful work while tasks are executed. This requires some nontrivial plumbing to build a background thread and talk to that cleanly. It's the kind of thing you want to wrap in a nice simple API that the average developer cannot misuse. It's the same approach that we used for Majordomo.

Here's the Titanic implementation. This server handles the three services using three threads, as proposed. It does full persistence to disk using the most brutal approach possible: one file per message. It's so simple, it's scary. The only complex part is that it keeps a separate queue of all requests, to avoid reading the directory over and over:

[[code type="example" title="Titanic broker example" name="titanic"]]
[[/code]]

To test this, start {{mdbroker}} and {{titanic}}, and then run {{ticlient}}. Now start {{mdworker}} arbitrarily, and you should see the client getting a response and exiting happily.

Some notes about this code:

* Note that some loops start by sending, others by receiving messages. This is because Titanic acts both as a client and a worker in different roles.
* The Titanic broker uses the MMI service discovery protocol to send requests only to services that appear to be running. Since the MMI implementation in our little Majordomo broker is quite poor, this won't work all the time.
* We use an inproc connection to send new request data from the {{titanic.request}} service through to the main dispatcher. This saves the dispatcher from having to scan the disk directory, load all request files, and sort them by date/time.

The important thing about this example is not performance (which, although I haven't tested it, is surely terrible), but how well it implements the reliability contract. To try it, start the mdbroker and titanic programs. Then start the ticlient, and then start the mdworker echo service. You can run all four of these using the {{-v}} option to do verbose activity tracing. You can stop and restart any piece //except the client// and nothing will get lost.

If you want to use Titanic in real cases, you'll rapidly be asking "how do we make this faster?"

Here's what I'd do, starting with the example implementation:

* Use a single disk file for all data, rather than multiple files. Operating systems are usually better at handling a few large files than many smaller ones.
* Organize that disk file as a circular buffer so that new requests can be written contiguously (with very occasional wraparound). One thread, writing full speed to a disk file, can work rapidly.
* Keep the index in memory and rebuild the index at startup time, from the disk buffer. This saves the extra disk head flutter needed to keep the index fully safe on disk. You would want an fsync after every message, or every N milliseconds if you were prepared to lose the last M messages in case of a system failure.
* Use a solid-state drive rather than spinning iron oxide platters.
* Pre-allocate the entire file, or allocate it in large chunks, which allows the circular buffer to grow and shrink as needed. This avoids fragmentation and ensures that most reads and writes are contiguous.

And so on. What I'd not recommend is storing messages in a database, not even a "fast" key/value store, unless you really like a specific database and don't have performance worries. You will pay a steep price for the abstraction, ten to a thousand times over a raw disk file.

If you want to make Titanic //even more reliable//, duplicate the requests to a second server, which you'd place in a second location just far away enough to survive a nuclear attack on your primary location, yet not so far that you get too much latency.

If you want to make Titanic //much faster and less reliable//, store requests and replies purely in memory. This will give you the functionality of a disconnected network, but requests won't survive a crash of the Titanic server itself.

##  High-Availability Pair (Binary Star Pattern)

[[code type="textdiagram" title="High-Availability Pair, Normal Operation"]]
#------------#           #------------#
|            |           |            |
|  Primary   |<--------->|   Backup   |
|  "active"  |           |  "passive" |
|            |           |            |
#------------#           #------------#
      ^
      |
      |
      |
#-----+------#
|            |
|   Client   |
|            |
#------------#
[[/code]]

The Binary Star pattern puts two servers in a primary-backup high-availability pair[figure]. At any given time, one of these (the active) accepts connections from client applications. The other (the passive) does nothing, but the two servers monitor each other. If the active disappears from the network, after a certain time the passive takes over as active.

We developed the Binary Star pattern at iMatix for our [http://www.openamq.org OpenAMQ server]. We designed it:

* To provide a straightforward high-availability solution.
* To be simple enough to actually understand and use.
* To fail over reliably when needed, and only when needed.

Assuming we have a Binary Star pair running, here are the different scenarios that will result in a failover[figure]:

* The hardware running the primary server has a fatal problem (power supply explodes, machine catches fire, or someone simply unplugs it by mistake), and disappears. Applications see this, and reconnect to the backup server.
* The network segment on which the primary server sits crashes--perhaps a router gets hit by a power spike--and applications start to reconnect to the backup server.
* The primary server crashes or is killed by the operator and does not restart automatically.

[[code type="textdiagram" title="High-availability Pair During Failover"]]
#------------#           #------------#
|            |           |            |
|  Primary   |<--------->|   Backup   |
| "passive"  |           |  "active"  |
|            |           |            |
#------------#           #------------#
                                ^
                                |
      .-------------------------'
      |
#-----+------#
|            |
|   Client   |
|            |
#------------#
[[/code]]

Recovery from failover works as follows:

* The operators restart the primary server and fix whatever problems were causing it to disappear from the network.
* The operators stop the backup server at a moment when it will cause minimal disruption to applications.
* When applications have reconnected to the primary server, the operators restart the backup server.

Recovery (to using the primary server as active) is a manual operation. Painful experience teaches us that automatic recovery is undesirable. There are several reasons:

* Failover creates an interruption of service to applications, possibly lasting 10-30 seconds. If there is a real emergency, this is much better than total outage. But if recovery creates a further 10-30 second outage, it is better that this happens off-peak, when users have gone off the network.

* When there is an emergency, the absolute first priority is certainty for those trying to fix things. Automatic recovery creates uncertainty for system administrators, who can no longer be sure which server is in charge without double-checking.

* Automatic recovery can create situations where networks fail over and then recover, placing operators in the difficult position of analyzing what happened. There was an interruption of service, but the cause isn't clear.

Having said this, the Binary Star pattern will fail back to the primary server if this is running (again) and the backup server fails. In fact, this is how we provoke recovery.

The shutdown process for a Binary Star pair is to either:

# Stop the passive server and then stop the active server at any later time, or
# Stop both servers in any order but within a few seconds of each other.

Stopping the active and then the passive server with any delay longer than the failover timeout will cause applications to disconnect, then reconnect, and then disconnect again, which may disturb users.

###  Detailed Requirements

Binary Star is as simple as it can be, while still working accurately. In fact, the current design is the third complete redesign. Each of the previous designs we found to be too complex, trying to do too much, and we stripped out functionality until we came to a design that was understandable, easy to use, and reliable enough to be worth using.

These are our requirements for a high-availability architecture:

* The failover is meant to provide insurance against catastrophic system failures, such as hardware breakdown, fire, accident, and so on. There are simpler ways to recover from ordinary server crashes and we already covered these.

* Failover time should be under 60 seconds and preferably under 10 seconds.

* Failover has to happen automatically, whereas recovery must happen manually. We want applications to switch over to the backup server automatically, but we do not want them to switch back to the primary server except when the operators have fixed whatever problem there was and decided that it is a good time to interrupt applications again.

* The semantics for client applications should be simple and easy for developers to understand. Ideally, they should be hidden in the client API.

* There should be clear instructions for network architects on how to avoid designs that could lead to //split brain syndrome//, in which both servers in a Binary Star pair think they are the active server.

* There should be no dependencies on the order in which the two servers are started.

* It must be possible to make planned stops and restarts of either server without stopping client applications (though they may be forced to reconnect).

* Operators must be able to monitor both servers at all times.

* It must be possible to connect the two servers using a high-speed dedicated network connection. That is, failover synchronization must be able to use a specific IP route.

We make the following assumptions:

* A single backup server provides enough insurance; we don't need multiple levels of backup.

* The primary and backup servers are equally capable of carrying the application load. We do not attempt to balance load across the servers.

* There is sufficient budget to cover a fully redundant backup server that does nothing almost all the time.

We don't attempt to cover the following:

* The use of an active backup server or load balancing. In a Binary Star pair, the backup server is inactive and does no useful work until the primary server goes offline.

* The handling of persistent messages or transactions in any way. We assume the existence of a network of unreliable (and probably untrusted) servers or Binary Star pairs.

* Any automatic exploration of the network. The Binary Star pair is manually and explicitly defined in the network and is known to applications (at least in their configuration data).

* Replication of state or messages between servers. All server-side state must be recreated by applications when they fail over.

Here is the key terminology that we use in Binary Star:

* //Primary//: the server that is normally or initially active.

* //Backup//: the server that is normally passive. It will become active if and when the primary server disappears from the network, and when client applications ask the backup server to connect.

* //Active//: the server that accepts client connections. There is at most one active server.

* //Passive//: the server that takes over if the active disappears. Note that when a Binary Star pair is running normally, the primary server is active, and the backup is passive. When a failover has happened, the roles are switched.

To configure a Binary Star pair, you need to:

# Tell the primary server where the backup server is located. 
# Tell the backup server where the primary server is located.
# Optionally, tune the failover response times, which must be the same for both servers.

The main tuning concern is how frequently you want the servers to check their peering status, and how quickly you want to activate failover. In our example, the failover timeout value defaults to 2,000 msec. If you reduce this, the backup server will take over as active more rapidly but may take over in cases where the primary server could recover. For example, you may have wrapped the primary server in a shell script that restarts it if it crashes. In that case, the timeout should be higher than the time needed to restart the primary server.

For client applications to work properly with a Binary Star pair, they must:

# Know both server addresses.
# Try to connect to the primary server, and if that fails, to the backup server.
# Detect a failed connection, typically using heartbeating.
# Try to reconnect to the primary, and then backup (in that order), with a delay between retries that is at least as high as the server failover timeout.
# Recreate all of the state they require on a server.
# Retransmit messages lost during a failover, if messages need to be reliable.

It's not trivial work, and we'd usually wrap this in an API that hides it from real end-user applications.

These are the main limitations of the Binary Star pattern:

* A server process cannot be part of more than one Binary Star pair.
* A primary server can have a single backup server, and no more.
* The passive server does no useful work, and is thus wasted.
* The backup server must be capable of handling full application loads.
* Failover configuration cannot be modified at runtime.
* Client applications must do some work to benefit from failover.

###  Preventing Split-Brain Syndrome

//Split-brain syndrome// occurs when different parts of a cluster think they are active at the same time. It causes applications to stop seeing each other. Binary Star has an algorithm for detecting and eliminating split brain, which is based on a three-way decision mechanism (a server will not decide to become active until it gets application connection requests and it cannot see its peer server).

However, it is still possible to (mis)design a network to fool this algorithm. A typical scenario would be a Binary Star pair, that is distributed between two buildings, where each building also had a set of applications and where there was a single network link between both buildings. Breaking this link would create two sets of client applications, each with half of the Binary Star pair, and each failover server would become active.

To prevent split-brain situations, we must connect a Binary Star pair using a dedicated network link, which can be as simple as plugging them both into the same switch or, better, using a crossover cable directly between two machines.

We must not split a Binary Star architecture into two islands, each with a set of applications. While this may be a common type of network architecture, you should use federation, not high-availability failover, in such cases.

A suitably paranoid network configuration would use two private cluster interconnects, rather than a single one. Further, the network cards used for the cluster would be different from those used for message traffic, and possibly even on different paths on the server hardware. The goal is to separate possible failures in the network from possible failures in the cluster. Network ports can have a relatively high failure rate.

###  Binary Star Implementation

Without further ado, here is a proof-of-concept implementation of the Binary Star server. The primary and backup servers run the same code, you choose their roles when you run the code:

[[code type="example" title="Binary Star server" name="bstarsrv"]]
[[/code]]

And here is the client:

[[code type="example" title="Binary Star client" name="bstarcli"]]
[[/code]]

To test Binary Star, start the servers and client in any order:

[[code]]
bstarsrv -p     # Start primary
bstarsrv -b     # Start backup
bstarcli
[[/code]]

You can then provoke failover by killing the primary server, and recovery by restarting the primary and killing the backup. Note how it's the client vote that triggers failover, and recovery.

Binary star is driven by a finite state machine[figure]. Events are the peer state, so "Peer Active" means the other server has told us it's active. "Client Request" means we've received a client request. "Client Vote" means we've received a client request AND our peer is inactive for two heartbeats.

Note that the servers use PUB-SUB sockets for state exchange. No other socket combination will work here. PUSH and DEALER block if there is no peer ready to receive a message. PAIR does not reconnect if the peer disappears and comes back. ROUTER needs the address of the peer before it can send it a message.

[[code type="textdiagram" title="Binary Star Finite State Machine"]]
    Start .-----------------------.   .----------.               Start
          |                       |   |          |                   
      |   |Client Request         |   |    Client| Request         |
      v   |                       v   v          |                 v
.---------+-.                 .-----------.      |           .-----------.
|           | Peer Backup     |           +------'           |           |
|  Primary  +---------------->|  Active   |<--------------.  |  Backup   |
|           |         .------>|           |<-----.        |  |           |
'-----+-----'         |       '-----+-----'      |        |  '-----+-----'
      |               |             |            |        |        |
  Peer| Active        |         Peer| Active     |        |    Peer| Active
      |               |             v            |        |        |
      |               |       .-----------.      |        |        |
      |               |       |           |      |        |        |
      |           Peer| Backup|  Error!   |  Peer| Primary|        |
      |               |       |           |      |        |        |
      |               |       '-----------'      |        |        |
      |               |             ^            |        |        |
      |               |         Peer| Passive    |  Client| Vote   |
      |               |             |            |        |        |
      |               |       .-----+-----.      |        |        |
      |               |       |           +------'        |        |
      |               '-------+  Passive  +---------------'        |
      '---------------------->|           |<-----------------------'
                              '-----------'
[[/code]]

###  Binary Star Reactor

Binary Star is useful and generic enough to package up as a reusable reactor class. The reactor then runs and calls our code whenever it has a message to process. This is much nicer than copying/pasting the Binary Star code into each server where we want that capability.

In C, we wrap the CZMQ {{zloop}} class that we saw before. {{zloop}} lets you register handlers to react on socket and timer events. In the Binary Star reactor, we provide handlers for voters and for state changes (active to passive, and vice versa). Here is the {{bstar}} API:

[[code type="fragment" name="bstar"]]
//  Create a new Binary Star instance, using local (bind) and
//  remote (connect) endpoints to set up the server peering.
bstar_t *bstar_new (int primary, char *local, char *remote);

//  Destroy a Binary Star instance
void bstar_destroy (bstar_t **self_p);

//  Return underlying zloop reactor, for timer and reader
//  registration and cancelation.
zloop_t *bstar_zloop (bstar_t *self);

//  Register voting reader
int bstar_voter (bstar_t *self, char *endpoint, int type,
                 zloop_fn handler, void *arg);

//  Register main state change handlers
void bstar_new_active (bstar_t *self, zloop_fn handler, void *arg);
void bstar_new_passive (bstar_t *self, zloop_fn handler, void *arg);

//  Start the reactor, which ends if a callback function returns -1, 
//  or the process received SIGINT or SIGTERM.
int bstar_start (bstar_t *self);
[[/code]]

And here is the class implementation:

[[code type="example" title="Binary Star core class" name="bstar"]]
[[/code]]

This gives us the following short main program for the server:

[[code type="example" title="Binary Star server, using core class" name="bstarsrv2"]]
[[/code]]

##  Brokerless Reliability (Freelance Pattern)

It might seem ironic to focus so much on broker-based reliability, when we often explain ZeroMQ as "brokerless messaging". However, in messaging, as in real life, the middleman is both a burden and a benefit. In practice, most messaging architectures benefit from a mix of distributed and brokered messaging. You get the best results when you can decide freely what trade-offs you want to make. This is why I can drive twenty minutes to a wholesaler to buy five cases of wine for a party, but I can also walk ten minutes to a corner store to buy one bottle for a dinner. Our highly context-sensitive relative valuations of time, energy, and cost are essential to the real world economy. And they are essential to an optimal message-based architecture.

This is why ZeroMQ does not //impose// a broker-centric architecture, though it does give you the tools to build brokers, aka //proxies//, and we've built a dozen or so different ones so far, just for practice.

So we'll end this chapter by deconstructing the broker-based reliability we've built so far, and turning it back into a distributed peer-to-peer architecture I call the Freelance pattern. Our use case will be a name resolution service. This is a common problem with ZeroMQ architectures: how do we know the endpoint to connect to? Hard-coding TCP/IP addresses in code is insanely fragile. Using configuration files creates an administration nightmare. Imagine if you had to hand-configure your web browser, on every PC or mobile phone you used, to realize that "google.com" was "74.125.230.82".

A ZeroMQ name service (and we'll make a simple implementation) must do the following:

* Resolve a logical name into at least a bind endpoint, and a connect endpoint. A realistic name service would provide multiple bind endpoints, and possibly multiple connect endpoints as well.

* Allow us to manage multiple parallel environments, e.g., "test" versus "production", without modifying code.

* Be reliable, because if it is unavailable, applications won't be able to connect to the network.

Putting a name service behind a service-oriented Majordomo broker is clever from some points of view. However, it's simpler and much less surprising to just expose the name service as a server to which clients can connect directly. If we do this right, the name service becomes the //only// global network endpoint we need to hard-code in our code or configuration files.

[[code type="textdiagram" title="The Freelance Pattern"]]
#-----------#   #-----------#   #-----------#
|  Client   |   |  Client   |   |  Client   |
'-----------'   '-----------'   '-----------'
   connect         connect         connect
      |               |               |
      |               |               |
      +---------------+---------------+
      |               |               |
      |               |               |
    bind            bind            bind
.-----------.   .-----------.   .-----------.
|  Server   |   |  Server   |   |  Server   |
#-----------#   #-----------#   #-----------#
[[/code]]

The types of failure we aim to handle are server crashes and restarts, server busy looping, server overload, and network issues. To get reliability, we'll create a pool of name servers so if one crashes or goes away, clients can connect to another, and so on. In practice, two would be enough. But for the example, we'll assume the pool can be any size[figure].

In this architecture, a large set of clients connect to a small set of servers directly. The servers bind to their respective addresses. It's fundamentally different from a broker-based approach like Majordomo, where workers connect to the broker. Clients have a couple of options:

* Use REQ sockets and the Lazy Pirate pattern. Easy, but would need some additional intelligence so clients don't stupidly try to reconnect to dead servers over and over.

* Use DEALER sockets and blast out requests (which will be load balanced to all connected servers) until they get a reply. Effective, but not elegant.

* Use ROUTER sockets so clients can address specific servers. But how does the client know the identity of the server sockets? Either the server has to ping the client first (complex), or the server has to use a hard-coded, fixed identity known to the client (nasty).

We'll develop each of these in the following subsections.

###  Model One: Simple Retry and Failover

So our menu appears to offer: simple, brutal, complex, or nasty. Let's start with simple and then work out the kinks. We take Lazy Pirate and rewrite it to work with multiple server endpoints.

Start one or several servers first, specifying a bind endpoint as the argument:

[[code type="example" title="Freelance server, Model One" name="flserver1"]]
[[/code]]

Then start the client, specifying one or more connect endpoints as arguments:

[[code type="example" title="Freelance client, Model One" name="flclient1"]]
[[/code]]

A sample run is:

[[code]]
flserver1 tcp://*:5555 &
flserver1 tcp://*:5556 &
flclient1 tcp://localhost:5555 tcp://localhost:5556
[[/code]]

Although the basic approach is Lazy Pirate, the client aims to just get one successful reply. It has two techniques, depending on whether you are running a single server or multiple servers:

* With a single server, the client will retry several times, exactly as for Lazy Pirate.
* With multiple servers, the client will try each server at most once until it's received a reply or has tried all servers.

This solves the main weakness of Lazy Pirate, namely that it could not fail over to backup or alternate servers.

However, this design won't work well in a real application. If we're connecting many sockets and our primary name server is down, we're going to experience this painful timeout each time.

###  Model Two: Brutal Shotgun Massacre

Let's switch our client to using a DEALER socket. Our goal here is to make sure we get a reply back within the shortest possible time, no matter whether a particular server is up or down. Our client takes this approach:

* We set things up, connecting to all servers.
* When we have a request, we blast it out as many times as we have servers.
* We wait for the first reply, and take that.
* We ignore any other replies.

What will happen in practice is that when all servers are running, ZeroMQ will distribute the requests so that each server gets one request and sends one reply. When any server is offline and disconnected, ZeroMQ will distribute the requests to the remaining servers. So a server may in some cases get the same request more than once.

What's more annoying for the client is that we'll get multiple replies back, but there's no guarantee we'll get a precise number of replies. Requests and replies can get lost (e.g., if the server crashes while processing a request).

So we have to number requests and ignore any replies that don't match the request number. Our Model One server will work because it's an echo server, but coincidence is not a great basis for understanding. So we'll make a Model Two server that chews up the message and returns a correctly numbered reply with the content "OK". We'll use messages consisting of two parts: a sequence number and a body.

Start one or more servers, specifying a bind endpoint each time:

[[code type="example" title="Freelance server, Model Two" name="flserver2"]]
[[/code]]

Then start the client, specifying the connect endpoints as arguments:

[[code type="example" title="Freelance client, Model Two" name="flclient2"]]
[[/code]]

Here are some things to note about the client implementation:

* The client is structured as a nice little class-based API that hides the dirty work of creating ZeroMQ contexts and sockets and talking to the server. That is, if a shotgun blast to the midriff can be called "talking".

* The client will abandon the chase if it can't find //any// responsive server within a few seconds.

* The client has to create a valid REP envelope, i.e., add an empty message frame to the front of the message.

The client performs 10,000 name resolution requests (fake ones, as our server does essentially nothing) and measures the average cost. On my test box, talking to one server, this requires about 60 microseconds. Talking to three servers, it takes about 80 microseconds.

The pros and cons of our shotgun approach are:

* Pro: it is simple, easy to make and easy to understand.
* Pro: it does the job of failover, and works rapidly, so long as there is at least one server running.
* Con: it creates redundant network traffic.
* Con: we can't prioritize our servers, i.e., Primary, then Secondary.
* Con: the server can do at most one request at a time, period.

###  Model Three: Complex and Nasty

The shotgun approach seems too good to be true. Let's be scientific and work through all the alternatives. We're going to explore the complex/nasty option, even if it's only to finally realize that we preferred brutal. Ah, the story of my life.

We can solve the main problems of the client by switching to a ROUTER socket. That lets us send requests to specific servers, avoid servers we know are dead, and in general be as smart as we want to be. We can also solve the main problem of the server (single-threadedness) by switching to a ROUTER socket.

But doing ROUTER to ROUTER between two anonymous sockets (which haven't set an identity) is not possible. Both sides generate an identity (for the other peer) only when they receive a first message, and thus neither can talk to the other until it has first received a message. The only way out of this conundrum is to cheat, and use hard-coded identities in one direction. The proper way to cheat, in a client/server case, is to let the client "know" the identity of the server. Doing it the other way around would be insane, on top of complex and nasty, because any number of clients should be able to arise independently. Insane, complex, and nasty are great attributes for a genocidal dictator, but terrible ones for software.

Rather than invent yet another concept to manage, we'll use the connection endpoint as identity. This is a unique string on which both sides can agree without more prior knowledge than they already have for the shotgun model. It's a sneaky and effective way to connect two ROUTER sockets.

Remember how ZeroMQ identities work. The server ROUTER socket sets an identity before it binds its socket. When a client connects, they do a little handshake to exchange identities, before either side sends a real message. The client ROUTER socket, having not set an identity, sends a null identity to the server. The server generates a random UUID to designate the client for its own use. The server sends its identity (which we've agreed is going to be an endpoint string) to the client.

This means that our client can route a message to the server (i.e., send on its ROUTER socket, specifying the server endpoint as identity) as soon as the connection is established. That's not //immediately// after doing a {{zmq_connect[3]}}, but some random time thereafter. Herein lies one problem: we don't know when the server will actually be available and complete its connection handshake. If the server is online, it could be after a few milliseconds. If the server is down and the sysadmin is out to lunch, it could be an hour from now.

There's a small paradox here. We need to know when servers become connected and available for work. In the Freelance pattern, unlike the broker-based patterns we saw earlier in this chapter, servers are silent until spoken to. Thus we can't talk to a server until it's told us it's online, which it can't do until we've asked it.

My solution is to mix in a little of the shotgun approach from model 2, meaning we'll fire (harmless) shots at anything we can, and if anything moves, we know it's alive. We're not going to fire real requests, but rather a kind of ping-pong heartbeat.

This brings us to the realm of protocols again, so here's a [http://rfc.zeromq.org/spec:10 short spec that defines how a Freelance client and server exchange ping-pong commands and request-reply commands].

It is short and sweet to implement as a server. Here's our echo server, Model Three, now speaking FLP:

[[code type="example" title="Freelance server, Model Three" name="flserver3"]]
[[/code]]

The Freelance client, however, has gotten large. For clarity, it's split into an example application and a class that does the hard work. Here's the top-level application:

[[code type="example" title="Freelance client, Model Three" name="flclient3"]]
[[/code]]

And here, almost as complex and large as the Majordomo broker, is the client API class:

[[code type="example" title="Freelance client API" name="flcliapi"]]
[[/code]]

This API implementation is fairly sophisticated and uses a couple of techniques that we've not seen before.

* **Multithreaded API**: the client API consists of two parts, a synchronous {{flcliapi}} class that runs in the application thread, and an asynchronous //agent// class that runs as a background thread. Remember how ZeroMQ makes it easy to create multithreaded apps. The flcliapi and agent classes talk to each other with messages over an {{inproc}} socket. All ZeroMQ aspects (such as creating and destroying a context) are hidden in the API. The agent in effect acts like a mini-broker, talking to servers in the background, so that when we make a request, it can make a best effort to reach a server it believes is available.

* **Tickless poll timer**: in previous poll loops we always used a fixed tick interval, e.g., 1 second, which is simple enough but not excellent on power-sensitive clients (such as notebooks or mobile phones), where waking the CPU costs power. For fun, and to help save the planet, the agent uses a //tickless timer//, which calculates the poll delay based on the next timeout we're expecting. A proper implementation would keep an ordered list of timeouts. We just check all timeouts and calculate the poll delay until the next one.

##  Conclusion

In this chapter, we've seen a variety of reliable request-reply mechanisms, each with certain costs and benefits. The example code is largely ready for real use, though it is not optimized. Of all the different patterns, the two that stand out for production use are the Majordomo pattern, for broker-based reliability, and the Freelance pattern, for brokerless reliability.
