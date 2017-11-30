# TCP half open connections and the perils of not using heartbeats

We are living interesting times in the tech world: cloud computing, containers, schedulers, serverless, etc. All these technologies being the trendiest. All is rainbows and unicorns when everything is working, we also start to forget about the systems that support our current abstractions until they broke and we have to give our best to get them fixed.

Some time ago in one of the multiple pub-sub system we have, where there are some processes publishing messages to a message broker. Those messages are later consumed by a process (running inside a container). So far this shouldn’t be exotic (view image 1). This system  worked as expected, except for some alerts from the monitoring system. Those alerts reported that the broker had a lot of  queued messages and no consumers connected.

Image 1

![Pub Sub](https://github.com/aebm/sysadvent_2017/blob/master/images/pub_sub.png)

The first reaction was to restart the consumer container and call it a day. Yet the error keep happening at the worsts times possible, it was like an Edgar Alan Poe story.

We decided to take a closer look at the problem, we confirm that the broker has no consumer to deliver the messages. The  surprise come when we inspect the consumer container. It is still running and is blocked waiting on the socket it uses to communicate with the broker. When we inspect the socket statistics inside the containers net namespace (view command 1), it  shows an established connection to the broker.

command 1
```
# ss -tpno
ESTAB      0      0               <container ip>:<some port>         <broker ip>:<broker port>
```
![](https://media.giphy.com/media/l3q2K5jinAlChoCLS/giphy.gif)


The problem seems that the connection state between broker and consumer is not synchronized. In the host network namespace, (where the broker is running) it shows that the are not any TCP connection from the container. Instead in the container network namespace there is an established connection to the broker. Our dear [RFC 793](https://tools.ietf.org/html/rfc793#page-33) mentions this situation at **Half-Open Connections and Other Anomalies** section. I proceed to quote, emphasis mine:

> An established connection is said to be  "half-open" if one of the TCPs has closed or aborted the connection at its end
> without the knowledge of the other, or if the two ends of the connection have become desynchronized owing to a crash that
> resulted in loss of memory.  **Such connections will automatically become reset if an attempt is made to *send data* in
> either direction**. However, half-open connections are expected to be unusual, and the recovery procedure is mildly
> involved.
>
> If at site A the connection no longer exists, then an attempt by the user at site B to **send** any data on it will 
> result in the site B TCP receiving a **reset** control message.  Such a message indicates to the site B TCP that something
> is wrong, and it is expected to abort the connection.

After that nice lecture, we start to think in possible causes for that desynchronization, some options being:

* broker restarted or crashed.
* MITM attack
* Kernel being mean (iptables, ebtables, bridges, etc)
* Container engine being mean
* Some Lovecraftian stuff

First we check if the broker has crashed or has been restarted. It has been running for a long time and other systems using it are working normally, so we discard this option.
The kernel iptables and bridge doesn’t show anything weird, the MITM attack seems weird and the other options are hard to prove, also it would be unprofessional to blame the container system without evidence ;-). 
Trying to check those weird hypothesis, we have tcpdump running on one of those systems. Tcpdump captured a RST message sent from the container IP to the broker just after the broker had sent a data message to the container after a long period of inactivity. The weird thing is that network traffic never reached the container, neither the RST originated from the container. A MITM attack??

Meanwhile we were trying to recreate the problem end state and check if we could make our systems resilient to this situation. We used iptables to drop or reset the traffic from the broker to the container after the container connected the broker. Both methods allowed us to end in the same state that we were getting on the wild, confirming that the container never finds out that the connection with the broker is lost. Trying to find how to find out that your peer if down even if your TCP connection state is established. After some Internet searching, we found   [RFC 1122](https://tools.ietf.org/html/rfc1122#page-101) at **TCP Keep-Alives** which I will proceed to quote a fragment, again emphasis mine:

> Implementors **MAY** include "keep-alives" in their TCP implementations, although **this practice is not universally
> accepted**.  If keep-alives are included, the application MUST be able to turn them on or off for each TCP connection, and
> they MUST default to off.
>
> Keep-alive packets MUST only be sent when **no data or acknowledgement packets have been received for the connection within
> an interval***.  This interval MUST be configurable and MUST default to no less than two hours.
>
> ...
>
> DISCUSSION:
>
>   A "keep-alive" mechanism periodically probes the other end of a connection when the connection is otherwise idle, even 
>   when there is no data to be sent.  The TCP specification does not include a keep-alive mechanism because it could:  
>   (1) cause perfectly good connections to **break during transient Internet failures;** (2) consume unnecessary bandwidth 
>   ("if no one is using the connection, who cares if it is still good?"); and (3) cost money for an Internet path that
>   charges for packets.
>
> ...
>
>   A TCP keep-alive mechanism should only be invoked in server applications that might otherwise **hang indefinitely** and
>   consume resources unnecessarily if a client crashes or aborts a connection during a network failure.

As you have read, having a distributed system is fun, and determining if a connection is still valid is the cherry on top.

Before trying to poke TCP, we kept investigating. We found out that AMQP supports heartbeats, and they are used to check if a connection is still valid see [post](https://www.rabbitmq.com/heartbeats.html).  The library we were using have this option deactivated by default. This explains why the container was blocking waiting instead of trying to reconnect to the broker. To make things worse because the container is a consumer, it never sends data to the broker. If the container has sent data using the same socket it could detect if the connection is still valid.

We decided to try the two alternatives: the TCP keep-alive version was the fastest to implement but we didn’t like it because it was delegating the detection of the broken connection to the kernel TCP implementation, also we didn’t want to mess with socket options. we ran some tests with other applications and they handled it at the application level (thank you Bandwagon effect). For the other alternative, we changed the library to activate the AMQP heartbeats, it took more time but it felt better to use the mechanisms given by the AMQP protocol.

I guess you want to know who was doing the MITM. First some context. We run periodic short lived helper containers, we have observed with tcpdump that when a container starts it announces some ICMPV6 memberships. Also by default the container network namespace is attached to a network bridge. The network bridge use a cache table to associate address its respective port (like a network switch), it populates its table it sees network traffic, if for some time there isn’t any traffic the data in the table becomes stale.

The MITM happens when in a period without traffic between the broker and the container, the bridge cache is stale and a short live container is launched, there is a chance for it to get the same IP address the consumer container has. This new container changes the bridge cache, then if the broker sends a message to the consumer, the bridge delivers it to the new container, who then answers with a TCP RST because it doesn’t have a TCP connection with the broker. Finally the broker receives the TCP RST and aborts its connection with the consumer. The magic of giving the same IP address to different containers.

A picture is worth a thousand words.

Without MITM

![Ok](https://github.com/aebm/sysadvent_2017/blob/master/images/working.png)

With MITM

![Fail](https://github.com/aebm/sysadvent_2017/blob/master/images/mitm.png)

![](https://media.giphy.com/media/l378rrt5tAawaCQ9i/giphy.gif)
 
## Conclusion

As we can see, our programs must be prepared to handle network disruptions even if the network traffic doesn’t leave the host and you are using containers (remember **the network is not reliable**), else we will have a lot of fun with distributed systems bugs. The main takeaway is that if your program never sends traffic using a socket, you can’t be sure if the connection is valid or if you will end waiting for message that will never arrive. There are two ways of avoid this situation: the first is using TCP keepalives and delegating the detection to the stale connection to the OS and the other one is to implement or use some heartbeat mechanism in an upper protocol layer. Both alternatives have their pros and cons YMMV.

### Extras

When I was preparing this post, I found this [article](https://blog.stephencleary.com/2009/05/detection-of-half-open-dropped.html) that talks about half open connections, it would have been handy when I was explaining the problem to my coworkers.
