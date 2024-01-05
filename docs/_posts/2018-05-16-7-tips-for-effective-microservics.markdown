---
layout: post
title:  "7 tips for effective microservices"
date:   2018-05-16 08:43:56 -0800
categories: medium blogs
--- 

![](/assets/images/seven-lemons.png)  
  
Well designed microservices can make our life so much easier. From design to development to rolling out to production, it can help improve all the aspects. While microservices have been around for years, and best practices have been evolving over time, dealing with microservices at a larger scale has not been easy. For teams who are new to microservices, sometimes the complexity may be overwhelming.
  
Here are a few simple tips to get the best out of microservices: 
  
**1. Have a request-id/correlation-id for every request**
Request-id or correlation-id is a unique id assigned to each customer request. This helps tracing the flow end to end. While this seems to be a trivial idea ,in practice, this is very useful. Few scenarios where it really shines:  
a) Debugging  
b) Testing  
c) Making the request idempotent  
   
**2. Maintain backward compatibility of interfaces**  
  
Again, this seems to be a very trivial and common sense approach, but in practice, it becomes difficult when new features are being added at a rapid pace. A simple renaming of an element actually will break the backward compatibility.  
A few simple rules can help though:  
a) Do not allow to remove a mandatory element from request body  
b) Do not allow to make a new request element mandatory in later releases  
c) Have standard deprecation process defined for elements which are no longer valid  
  
**3. Have a centralized logging system**  
  
Testing and debugging are much harder in microservices. Having a single place to visualize what’s happening really helps. While each application may log different kind of stuff, it’s important that there is a minimum common information defined that each service must log. Having a capability of searching logs across the services using a request-id is super helpful during debugging sessions.  

**4. Implement idempotency and retries**  
  
Key to a good design is to understand the limitation of the system and embrace it. A distributed system is prone to many failure points and a simple way to deal with it is idempotency. Idempotency helps clients do a retry in case of failure without impacting the system state incorrectly. Idempotency in many cases may not be trivial and may need serious thoughts during design.  
  
**5. Be aware of language constraints** 
   
A key advantage of microservices architecture is we can write different services in different languages, which gives the flexibility to pick the right tooling for the specific work.While this works great in majority of the scenarios, we still need to consider and be aware that this also leads to different interpretation of information at each layer. Some very basic concepts like unsigned int(a very popular concept in C++ world) can become too problematic for the services written in Java, where this concept doesn’t exist.  
For example, We had our custom serialization protocol for inter service communication and it was built mostly keeping C++ in mind. Later new services were added in Java. While most of the stuffs worked pretty smoothly, we ran into issue with unsigned int, which was mapped to int in Java resulting into integer overflow(roll over).  
  
**6. Have a single service to manage the system state**  
  
Many a times, to give a faster and smoother experience, some services need to respond back with success even though actual work would be done/processed later. An example would be, to scale the system, the payment system responds back with success, although the actual charge on card would happen later. Multiple system trying to manipulate the system state sometime leads to inconsistencies. In such scenarios it’s important to manage the states correctly, and it’s wise to have only one service do the read/write/update of the system state.  
  
**7. Strike a balance between in-memory-data and db persistence**  
  
For a system that goes through different states, e.g. a payment system, multiple state changes are required to provide a specific functionality. Many of these could spread across multiple services and needs to store the modified state in some form. Using db as persistence layer may help with consistentcy and transactionality but may lead to bottleneck at a higher scale. In-memory cache may help with speed and scale but maintaining transactionality and availability may not be easy. A finer balance between these two solutions is the key to a successful scalable solution.
  


