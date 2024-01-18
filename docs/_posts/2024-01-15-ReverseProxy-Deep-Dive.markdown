---
layout: post
title:  "Revere proxy deep dive"
date:   2024-01-15 08:43:56 -0800
categories: ReverseProxy

---  
## A deep dive of a reverse proxy

Reverse proxy is a critical piece of software very commonly seen in different set up in a distributed system.
You might have seen this as a proxy enabling service mesh, as a loadbalancer helping distribute the load among different database instances or  as an edge proxy hiding all the complexity behind a site.
  
![](/assets/images/reverse_proxy.png)  
  

Multiple great reverse proxy exists like HAProxy, Nginx, Envoy, caddy, traeffic, zuul, Apache Traffic Server etc. they have their own strength and weaknesses and often used in specific contexts. e.g. for service mesh Envoy/linkerd are used very often. For edge proxy Nginx/HAProxy are used. for caching proxy ATS is great. Load balancing among different database load again HAProxy is a great choice.

## Unveiling the Functions and Intricacies of Reverse Proxies

At its core, the reverse proxy facilitates communication between clients and origin hosts. The high-level workflow involves:

1. Clients connecting to the reverse proxy
2. client sending requests (http/tcp/udp/ssh etc)
3. the proxy forwarding the request to one of the origin hosts 
4. proxy awaiting the respons from the origin
5. proxy forwarding the response to the client 
6. proxy/client/origin closing the connections.
   
Note: For simplicity we'll focus on Layer 7 (http) reverse proxy.
  
Breaking down the workflow reveals several essential functionalities:
1. Connection management: 
     listening to port, accepting a connection, accepting http requests 
2. http request parsing/header manipulation: 
     parsinng http request, sanitizing the request (to make sure it’s a valid request), manipulating headers, rewriting Paths etc
3. Service discovery/downstream discovery:
     Idntifying set of valid hosts for handling the requests, identifying one of the host as a target host for the specific requests.
4. Function as a http client:
    Initiatiing connection to origins hosts and sending requests/accepting response from the origin
5.  Observability:
    Enhancing visibility into proxy operations 

Each of these steps are complicated and we can go a level deeper in each of these steps.

## Deep dive into Connection management:

On a very high level the proxy need to bind to a specifci port, and listens on it. It needs to wait for some data, read/process the data it recieves, and forward the data to the origin, and once it receives the data from origin, write it back to the connection, so that clients can read the response. It continue this in the loop.

All standard programing languges provide netwrok libraries which can help write a simple server very easily. One sample go tcp server example:

``` golang
package main

import (
    "fmt"
    "net"
)

func main() {
    // Listen for incoming connections
    listener, err := net.Listen("tcp", "localhost:8080")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer listener.Close()

    fmt.Println("Server is listening on port 8080")

    for {
        // Accept incoming connections
        conn, err := listener.Accept()
        if err != nil {
            fmt.Println("Error:", err)
            continue
        }

        // Handle client connection in a goroutine
        go handleClient(conn)
    }
}

func handleClient(conn net.Conn) {
    defer conn.Close()

    // Read and process data from the client
    // ...

    // Write data back to the client
    // ...
}
```  
[source](https://okanexe.medium.com/the-complete-guide-to-tcp-ip-connections-in-golang-1216dae27b5a)

## what makes it complex

For one client, the above code or similar code works well. If we have 10 clients, we need to start thinking about how not to block these clients and consider concurrency. Maybe one thread for each client would make it work. Scaling it to 1000, most likely, a 1:1 threading may not be a great idea. Perhaps lightweight threads like goroutine would work great here, or some event loop-based mechanisms would be a nice way. What about 10k or 100k clients? Maybe select/ is not an option here, and we need to look for epoll/kqueue. 

Also, if there is only 1 thread listening to all connections, how would we leverage multiple cores? Maybe 1 main thread to listen to the activity and distribute the work to different threads and collect the data back and send it to the client. Will it require locks? Will it scale if we have 8 cores? What about 64 cores? And what about 128 cores?
Can kernel help? Cane we leverage Kernel level load balancing using so_resuseport

**Challenge**  
1. Concurrency: as we increase the conncurency, we need optimal way to handle inbound connections. In late 90s/early 2000s this led to a term called [C10K Problem](https://en.wikipedia.org/wiki/C10k_problem). This led to an interesting paradigm of [event driven programming](https://en.wikipedia.org/wiki/Event-driven_programming). [NodeJs event loop](https://www.builder.io/blog/visual-guide-to-nodejs-event-loop) is great example. Other examples are [Java Netty](https://livebook.manning.com/book/netty-in-action/chapter-1/), [C libevent](https://libevent.org).  
  
2. Leveraging multiple core: while event loops are great, they often are single threaded. With the [demise of Moore's law](https://cap.csail.mit.edu/death-moores-law-what-it-means-and-what-might-fill-gap-going-forward), we are in a phase where thenumber of cores are doubling every few years and 64+ cores are very common. So to truely levearge the power we need to make sure the event loops work for multiple cores as well. different proxies leverage different strategies for handling this. NGINx uses a multiple process stratgey, Envoy levarges kernal load balancing features so_reuseport for doing the load balancing at kernal level, HAProxy used to use multi process model but have since moved to multi threading model. each one of them have it's own challenges. e.g. Envoy enjoys the benefit of being simpler to handle the multiple cores by leveraging kernal load balancing but it significantly impacts its load balancing and connection management on the origin side.


Next level challenge comes if we also want to do say tls, so it not only need to finish the tcp connection we also need to handle tls handshake and all the complexity involve? Which tos library to use OpenSSL or libessl or boringssl? It has its own complexity and none of these are drop in replacement of each other so it adds some complexity. One way to simplify is just stick to one library but the challenge is we’ll also be bound by its limitations. Link to the challenges
What if we need to support different tls levels?

Another big challenge is what if we also need to support, udp, uds etc?  

How to handle timeouts?
How to handle abuser?

Http parsing
Once a tcp connection is established, next step is to actually send data over that connection and we’ll be talking about http protocol only. Over the years http protocol itself has evolved and  there a great article on Mozilla doc about the evolution. Let pick http 1.1 for simplicity. Http is text based protocol and -\r\n is used for delimiting different parts

A simple http get request would look like 

And a post request like.

So to accept a valid http request, a proxy needs to read the data from listening port, parse it and understand the request. If the request is valid forwarded the requests to the origin servers. Request could also be invalid and may be an attempt to exploit some issues/bugs (http smuggling/ddos) etc.
A get request typically ends with 2 \r\n, so one of the way to look at this is read data till you \r\n and that determines your request. Post request is a bit more complicated and it also has a payload. The size of the payload is determined by content-length. Payload can also have \r\n. So the previous strategy for looking for new lines won’t work.
So the revised strategy could be.
1. read one line, see if the request is get or post
2. If it’s get keep reading all the lines till you see 2 lines
3. If it’s post, from the read data identify the content-length by looking for content-length and then read those many bytes from port.


What if there are more than 1 content-length and with different values?
There are some special headers that proxy use for some special behavior so proxy needs to parse it as well. Proxy also modifies a few  headers. So it’s important to parse it efficiently and store it for further processing. Proxies may provide some dsl to add/remove/update some headers. So they need an optimized way to read as well write it. One way to do it could be to translate this into a hash map of key values.  While this is a good idea for simple requests with a few header and probably at a lower request rate. This may not be a good idea at a very high request rate rate.
1. it significantly increased the memory: all requests need to be stored in memory twice (one as we read and one in the hash)
2. In practice we may be getting a lot of headers but only refer to a few during processing, so it’ll be a lot of wasted cpu to create hash 
3. There is no good defined format of how big the header could be, which can make it a very easy dos attack vector.

So efficient parsing and easy access to read/write very quickly becomes a complex problem. Normally all proxies store it in an optimized way. Haproxy stores it in xyz format, ats also has custom ds, envoy as well.

These are optimized for general cases and may not be a great fit for all scenarios. Further customization may be required on top of it.

Cookie is another
HTTP has evolved over time.
On a very high level we have header, data, trailer

In http/1, 1.1 it’s text based.
So we need to look at the run coming data look for new lines,  parse, find the end of the request and store the info.

There are bad requests, so we need to some santization
Spec has complexity. Http 1.0 didn’t support reuse of connections. To optimize there a few headers keep-alive and connection:close were used. This brought many issues. You can read about these here. Http 1.1 was stable and was the longest serving http. Version. But it evolved to http 2.  And finally http 3. While fundamentally they still follow the similar semantics. Things became a little more complex/easier.  Looking for new line was not very efficient, so I. H2, they came with the concept of frames, so there was standard way to know the start/end of record. But it became much more complicated because of multiplexing support. With multiple streams came the complexity of quack. The complexity of server push became so complex that and so buggy that finallly chrome decided to get rid of that feature completely.


Header manipulation and path rewrite:
One of the most frequent work at the reverse proxy is to sanitize the headers. All untrusted headers are stripped of, a few more headers are added. And very often the urls/schemes are modified.

The goal
Here is to provide easy but flexible way to provide updates still being efficient.

Haproxy provides a very powerful header rewrite suntaxes.
One of catch here is spec compliance.
I’m http1 headers are case insensitive, but frequently servers don’t comply or the application server simply use some specific casing. In http2, default the headers are lowercase. Different proxies handle it differently. Envoy will by default downcase the headers and provide a way to keep the case. Haproxy also provide ways to keep the case as you need.



