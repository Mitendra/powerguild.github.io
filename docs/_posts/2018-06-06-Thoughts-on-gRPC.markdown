---
layout: post
title:  "Thoughts on gRPC"
date:   2018-06-06 08:43:56 -0800
categories: medium blogs

---  
  
![](/assets/images/light_bulb.png)  
  
Go and gRPC have become a popular choice for microservices of late. gRPC is an efficient over the wire communication protocol. Built on top of http/2 and protobuf (in theory other serialization protocols can also be supported), this is supposed to be faster, with lesser network overhead and an efficient protocol.  
  
### What is http/2
  
Http 2 is the next version of http protocol(http 1.1 is the most commonly used current version). Few points to note about http 2.0  
  
1. It supports connection multiplexing. So if there are multiple parallel request from one server to another, all the responses can be sent together, interleaved, over a single connection, making it faster when compared to responses over separate connections.  
2. Headers are compressed and transferred as binary data rather than plain text in http 1.1.  
3. Connection reuse also means less overhead of ssl handshake.  
4. Most of the current implementations supports only https mode(although this doesn’t seem to be a requirement , but just a practice of the current implementations in browsers), making it more secure.  
    
If http 2.0 is so good, why does everyone not use it?  
  
### Pitfalls  
  
1. Mostly for web use cases. The RFC talks about this, core focus is on browser use cases. Although can be used for non browser scenarios as well.  
2. Multiplexing actually adds overhead if there is a single request response involved. In case of proxy or dns based round robin scenarios, each request may be routed to a different downstream and multiplexing doesn’t work very well. Multiple simultaneous downstream call to the same service within the same request context is also less likely. So multiplexing may not be efficient in such scenarios.  
3. Number of headers in non browser use cases are typically limited. Larger header size could mostly be attributed to cookies in browser use cases. With smaller number of headers, compression gain is not that significant.  
4. A compressed and binary header also means inspection becomes difficult and an upgrade of tool sets is required.  
     
### What is protobuf  
  
Protobuf is a messaging protocol for over the wire communication. Being a binary transfer protocol, it packs the data in a compact format for efficient and faster data transfer. The client library to work with protobuf is available in several languages. Golang has a native support of protobuf.  
  
### A brief history of serialization protocols  
  
Long back, XML was kind of default serialization format and increasing popularity of Java could be attributed to its rise.  

While xml was easy to read/write/manipulate for both human and mahcine, yet it was too verbose and for a larger payload it had a high overhead.  
  
With javascript’s rise in popularity another cool serialization format started becoming popular, i.e. JSON. JSON was much smaller in size and still completely readable by human and was less verbose.  
  
Although JSON is quite readable, but it seems the specs are not very clear and there are times when different libraries would interpret it differently. Parsing JSON seems to be much more CPU intensive as well. The default JSON parser in golang actually seems to be on a slower side.  
  
Since, bigger payload was leading to higher network transfer cost, one alternative to that was compressing the data before transferring and decompressing it at the other end. This technique is quite popular and infact most of the modern browsers support gzip compression and its enabled by default which helps in reducing the payload size.so with json/xml based services, typically compression is enabled which reduces the payload size to a great extent.  
  
For a more efficient packing of data, the need of better options arose. Focused on the core needs instead of being a generic serlization solutions, many companies developed their own protocols which had less metadata attached, and thus reducing the payload size.  
  
Protobuff is one such option. It was developed within Google and later opensourced. Thrift is another which was developed by Facebook and later opensourced.  
  
### Few questions to ask  
  
1. Does the protocol have library support for the language we are interested in(consider both client and server languages)  
2. Does it have the tooling for manual request response from command line if the need arises(mostly for debugging and testing)  
3. Will the users of the API be comfortable using it? Browsers, generally, have good support for json/xml as they handle gzip. Similarly other popular tools like curl, postman supports json/xml and yet need a good support of gRPC.  
   
**References:**  
  
[NGINX HTTP2 White Paper](https://cdn-1.wp.nginx.com/wp-content/uploads/2015/09/NGINX_HTTP2_White_Paper_v4.pdf)  
[HTTP/2 Frequently Asked Questions](https://http2.github.io/faq/)  
[grpc / Guides](https://grpc.io/docs/guides/)  
[Parsing JSON is a Minefield | Hacker News](news.ycombinator.com)  

