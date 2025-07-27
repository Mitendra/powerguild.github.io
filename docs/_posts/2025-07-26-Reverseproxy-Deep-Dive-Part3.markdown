---
layout: post
title:  "Reverse Proxy Deep Dive Part 3: Service Discovery and Load Balancing in the Real World"
date:   2025-07-26 08:43:56 -0800
categories:  [reverse-proxy, networking, architecture]
tags: [load balancing, service discovery, proxies, infrastructure, dns]
summary: >
  Part 3 of the Reverse Proxy Deep Dive explores the complexities of service discovery and load balancing,how proxies maintain host lists, select backends, and cope with failure, scale, and churn.
---  

In [Part 1](https://startwithawhy.com/reverseproxy/2024/01/15/ReverseProxy-Deep-Dive.html) of this series, we covered the foundational layer of connection management. [Part 2](https://startwithawhy.com/reverseproxy/2025/07/20/ReverseProxy-Deep-Dive-Part2.html) explored the nuances of HTTP parsing and why it’s harder than it looks. Today, we’ll dive into what is arguably the most critical and visible part of a reverse proxy: **service discovery and load balancing**.

Unlike edge-specific concerns like TLS or abusive clients, **discovery and balancing must work consistently across all layers of a distributed system**. Whether it's an edge gateway or a sidecar proxy deep in the service mesh, routing requests to the right place at the right time is fundamental. But ensuring it works reliably, efficiently, and at scale is where things start to get complicated

---

## What Is Service Discovery and Load Balancing?

At its core, the job is straightforward:
1. Maintain an up-to-date list of upstream hosts (service discovery)
2. Select one host for each request (load balancing)

It sounds simple, but keeping it correct and fast across thousands of services, tens of thousands of hosts, and a constantly changing topology turns it into one of the hardest parts of any proxy.

---

## Ways to Discover Hosts

### 1. Static Host Lists

This is the most basic form. You hardcode IPs or hostnames into your proxy config. Works great if your infrastructure doesn’t change often, like dedicated on-prem clusters where host churn is rare.

**Pros:**
- Simple, predictable
- Zero dependency
- Most efficient

**Cons:**
- Not dynamic
- Requires manual coordination for scale

---

### 2. DNS-Based Discovery

Instead of using a static IP, you provide a fully qualified domain name (FQDN). The proxy resolves it to get a list of IP addresses, either at startup or during runtime.

If resolution happens at startup, it's very efficient because the proxy has less work to do later. However, you’ll need to restart proxy nodes periodically to pick up changes in the upstream hosts. This is still better than a hardcoded list, but it requires coordination and doesn’t work well if hosts change frequently.

Runtime resolution is better for dynamic workload, but it adds overhead. The proxy must:
- Parse DNS records, often mixing IPv4/IPv6
- Handle reordering (most DNS clients shuffle the host list)
- Diff the result vs existing list and identify the delta
- Remove stale entries, shut down old connections
- Add new entries into the rotation

**This gets worse when the host list is large (say, 200+ entries)**. If the algorithm isn't efficient, even something as simple as diffing host lists can become a bottleneck. HAProxy has encountered challenges with [this](https://github.com/haproxy/haproxy/issues/1185) in the past.

Most DNS traffic relies on UDP, which has a strict packet size limit, [512 bytes](https://datatracker.ietf.org/doc/html/rfc1035) for traditional and [4096 bytes](https://www.rfc-editor.org/rfc/rfc6891.txt) with EDNS(0). If the response exceeds that limit, it gets truncated. Then you need to retry over TCP, but even TCP has a limit ([65,535 bytes](https://www.rfc-editor.org/rfc/rfc6891.txt)) on how many records it can return.

This means the proxy now has to fully implement both DNS-over-UDP and DNS-over-TCP, just to get a list of hosts.

Another key concept in DNS based system is DNS [Time To Live](https://datatracker.ietf.org/doc/html/rfc2181#section-8)

## DNS TTL Tradeoffs

DNS TTL determines how long the proxy should treat the resolved host list as fresh.

### High TTL
- Fewer DNS lookups  
- Lower CPU and memory overhead in the proxy  
- Less churn in connection pools  

**But:**  
- Hosts may go down, drain, or be replaced without the proxy knowing  
- Traffic may continue flowing to broken nodes  

### Low TTL
- More up-to-date host list  
- Dead hosts removed faster  

**But:**  
- High cost from frequent DNS queries, parsing, and memory updates  
- Adds latency, especially with large upstream lists  

### Hybrid Approach
Use DNS for discovery combined with healthchecks to prune bad hosts locally.

- On each DNS update, fetch the full list  
- Periodically run healthchecks on each host (HTTP or TCP probes)  
- Temporarily remove failing nodes  
- Let DNS eventually catch up and remove bad entries  

It is a balance, reduce reliance on fast TTL but do not blindly trust DNS alone.

We will cover healthchecks in more detail later in the healthcheck section.

---

### 3. External Discovery Systems
When DNS isn’t good enough, large systems build their own discovery control planes, often using:

- ZooKeeper (classic choice for strongly consistent registries)
- HTTP/gRPC APIs (e.g. Envoy’s xDS)
- Runtime API + external sync agents (like HAProxy’s socket API)

These systems push updates only when things change, reducing unnecessary polling. They also support richer metadata (like health status or locality), which DNS can't provide.

Example (ZooKeeper-style):

1. The proxy starts by querying ZooKeeper to get the initial host list and sets a watch to monitor changes.
2. It maintains a persistent connection to ZooKeeper.
3. When there is a change, the proxy receives a notification and queries ZooKeeper again to update the host list.
4. If the connection is lost, the proxy repeats steps 1 through 3 to recover.

This approach significantly reduces unnecessary queries when there are no changes.
It’s efficient, but comes with its own set of problems:

- Integration is custom (many proxies don’t natively support ZooKeeper)
- Delete events require the client to re-fetch and compare
- These systems often favor strong consistency, which can hurt availability under load

Envoy’s xDS offers a modern approach by pushing service data over gRPC. However, unless your proxy is designed for this model like Envoy is, you will need an adapter layer to handle it.

## Healthcheck

While service discovery helps build a list of healthy nodes for routing traffic, its source of truth is not always up to date. This can happen for many reasons, such as network partitions, the high cost of healthchecks at the DNS layer, broken healthcheck setups, misbehaving components in the ecosystem, or temporary unavailability of the service.

Although the goal is to make service discovery as accurate as possible and remove bad hosts quickly, achieving that level of precision is a difficult problem.

To guard against such scenarios, proxies often perform their own healthchecks as a fallback.

### Active Health Checks

The proxy periodically probes each host, typically with a simple HTTP or TCP request, to confirm it's alive. Simple in theory, but expensive in practice:

- Thousands of hosts mean thousands of probes
- At scale, this burns CPU and network, especially if checks involve HTTPS or TLS handshakes
- Load spikes can happen if all proxies check all hosts at the same time

To reduce cost, many systems perform **lightweight HTTP checks** regularly and **heavier HTTPS checks less frequently**, such as 1 out of every 50 probes.

### Passive Health Checks

Instead of probing, the proxy watches live traffic. If a host consistently returns errors or times out, it is marked as unhealthy.

Passive checks are cheap, but reactive. They only trigger after enough failures happen. Most systems combine both approaches, tuning frequency and aggressiveness based on traffic shape and failure patterns.

## Load Balancing
Another key role of a proxy is deciding which upstream host should handle each request. The goal is to maximize host utilization without overloading or underusing any server. It should also ensure a consistent user experience, such as routing users to the same host to preserve sessions or use cached data. At the same time, the proxy must handle dynamic changes like hosts being added or removed, respect server characteristics like warmup time, and avoid sending traffic to overloaded nodes.

## Load Balancing Algorithms
Once a healthy host list is built, proxies must pick one. The balancing logic varies, but each option has tradeoffs.

### 1. Round Robin

In this approach, each request is sent to the next host in line. It is simple and fair, but only if all hosts are healthy, fully warmed up, and equally performant.

#### Challenges

1. **Maintaining a consistent view**  
   The proxy must keep a consistent view of the host list, which becomes tricky during deployments. Hosts are removed from rotation and later added back, but the order may differ. Sometimes, new hosts are entirely different from the previous ones, making consistency hard to maintain.

2. **No warm-up phase**  
   When new hosts are added, they begin receiving traffic right away, even if they are not yet fully initialized or ready to handle the load.

3. **Unequal request types**  
   Not all requests are the same. Some are CPU-intensive, others take longer to process, and some are very lightweight. If traffic is distributed purely by request count, it can lead to imbalanced load—overloading some nodes while underutilizing others.

### 2. Least Connections

The proxy sends requests to the host with the fewest active connections. This accounts for variable load per request, such as long-lived versus short-lived calls.


#### Challenges
1. **Burst traffic**
When a new node is added, it naturally has the fewest connections and starts receiving a disproportionate share of traffic. Since new connections can be expensive (e.g., due to TLS setup), this sudden spike can overwhelm the server, especially if it is not yet warmed up.

 2. **More work for proxy**
 The proxy must track all open connections, which adds operational complexity.


### 3. Consistent hashing
This strategy is useful when requests with certain properties should consistently go to the same upstream host. It's often used to improve cache hit rates or maintain session persistence. The proxy applies a hash function on a specific request attribute (like user ID or session ID) to pick a host.


#### Challenges
1. **uneven distribution**
Request volume and cost can vary based on the hashed property. For example, one user might generate significantly more traffic or make heavier API calls than another. This can overload some hosts while leaving others underused. One common mitigation is to apply consistent hashing only to request types that are more uniform in cost or volume, like payment or messaging APIs, and to use a property that is known to be evenly distributed.

2. **Burst traffic**
Adding new nodes can cause traffic to shift abruptly, leading to bursts that may overload the new host.

### 4. Random Choice of Two
Although it sounds counterintuitive, using randomness can lead to surprisingly good load balancing. In this approach, the proxy picks two random hosts and selects the one with fewer active connections. This method works well at scale and handles varied load types effectively, including long-lived, slow, or bursty requests.

### Challenges
1. **No warm-up phase** 
Like other strategies, this approach does not solve the problem of cold starts. New hosts may still receive traffic before they are fully ready.

2. **Added complexity**
The proxy must perform additional operations for random selection and comparison. However, the performance gain often justifies the cost.
