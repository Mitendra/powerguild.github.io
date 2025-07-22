---
layout: post
title:  "Reverse Proxy Deep Dive Part 2"
date:   2025-07-20 08:43:56 -0800
categories: ReverseProxy
---  
In [Part 1](https://medium.com/@mitendra_mahto/cross-posted-from-https-startwithawhy-com-reverseproxy-2024-01-15-reverseproxy-deep-dive-html-c3443dc3e0e5) of this series, we explored a high-level overview of reverse proxies and dived deep into connection management. This post shifts our focus to the intricate world of HTTP handling within a reverse proxy.

# Deep Dive into HTTP Handling

At a high level, the HTTP workflow from a proxy’s perspective might seem straightforward:

1. Receive the request from the client
2. Parse and sanitize the request
3. Uses different requst metadata (path, headers, cookies) to select an upstream host
4. Manipulates the requests as needed
5. Send the request to the selected upstream
6. Read the response from the upstream
7. Sanitize the response
8. Forward response to the client

# What Makes It Complex

## Challenges in HTTP parsing

### The Evolution of the HTTP Protocol

While these steps are supported by many standard libraries, making them work reliably at scale while meeting stringent security and compliance requirements is a surprisingly complex task.
  
![](/assets/images/http_evolution.png)  
  

Over the years http spec has significantly evolved. It started with simple request lines like GET /index.html in HTTP/0.9, moved to newline-separated headers and persistent connections in HTTP/1.1, and later adopted full binary framing and multiplexing in HTTP/2.

Each new version brought added features, redefined earlier assumptions, and introduced additional complexity. These changes require continual updates to existing systems, libraries, and tooling.

MDN offers a detailed exploration of [HTTP’s evolution](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Evolution_of_HTTP) for those interested in a deeper dive.

### Different Development Lifecycles of Clients, Proxies, and Servers

Clients and servers are often developed using different languages, frameworks, and platforms, each with their own independent lifecycles. Some systems, like in-car music players, may remain in use for 15 years or more, while clients and proxies evolve more rapidly. This disparity places the responsibility on the proxy layer to act as an intermediary, managing compatibility between components. The proxy often must decide which parts of the RFC specifications to enforce strictly and where to allow flexibility.

Examples:

- As per the HTTP/1.1 specification, header names are case-insensitive.. However, some clients and servers expect header names in a specific case. Reverse proxies typically provide a way to preserve or normalize casing. For instance, Haproxy provises [case option](https://docs.haproxy.org/dev/configuration.html#h1-case-adjust) for preserving case and Envoy provides similar [functionality](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/header_casing).
- HAProxy has an option to accept invalid request using [accept-unsafe-violations-in-http](https://docs.haproxy.org/dev/configuration.html#4.2-option%20accept-unsafe-violations-in-http-request). 

### Performance Consideration: One Size Doesn’t Fit All

Not all requests are the same. Some are small, while others carry large payloads. Some include long URLs or large cookies, while others do not. These variations make it difficult for proxy developers to create a single, highly efficient logic that works well in all cases. As a result, proxies often need to provide flexible configuration options and write code that can adapt to different traffic profiles.

These profiles can sometimes be in conflict. For example, supporting large header sizes requires more memory to parse and process incoming requests. At high throughput, this leads to increased memory usage and can affect overall system performance. This creates a trade-off between request size, memory consumption, and throughput.

### Lack of Industry-Wide Standards for Key Limits

Limits like maximum URL length, cookie size, and header size vary widely across platforms, and in some cases, the default settings provided by frameworks or servers are surprisingly low or outdated.

Azure Front Door imposes these [limits](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits) or URL/query size:

```
Maximum URL size — 8,192 bytes — Specifies maximum length of the raw URL (scheme + hostname + port + path + query string of the URL)

Maximum Query String size — 4,096 bytes — Specifies the maximum length of the query string, in bytes.

Maximum HTTP response header size from health probe URL — 4,096 bytes — Specified the maximum length of all the response headers of health probes.
```

Past investigations into cookie size reveal similar inconsistencies:
https://support.convert.com/hc/en-us/articles/4511582623117-cookie-size-limits-and-the-impact-on-the-use-of-convert-goals

```
Google Chrome — 4096 bytes

Internet Explorer — 5117 bytes

Firefox — 4097 bytes

Microsoft Edge — 4097 bytes
```

Typically, the following are allowed:

```
300 cookies in total

4096 bytes per cookie

20 cookies per domain

81920 bytes per domain
```

Handling quotes in Set-Cookie headers, whether single, double, or escaped, adds another layer of confusion. Some libraries interpret them strictly, others ignore them, and a few may even fail altogether. This inconsistency often leads to subtle bugs or broken behavior across browsers and frameworks. Here's a great read on the challenges of cookie handling and how it impacts major websites.
https://grayduck.mn/2024/11/21/handling-cookies-is-a-minefield/

### Security: The Proxy Is the First Line of Defense

![](/assets/images/http_smuggling.png)  

Even with established standards and defined behaviors, there are always bad actors who exploit gaps to try to break the system. Proxies deployed at the edge are particularly exposed to such clients and must be prepared to handle these cases with extreme caution.

**Examples include:**
* Carefully crafted requests with conflicting Content-Length and Transfer-Encoding headers that can lead to [HTTP request smuggling](https://medium.com/r?url=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FHTTP_request_smuggling).
* Invalid http version like 0.5 causes some servers to crash

## Request Sanitization

Sanitizing incoming requests is critical for maintaining the stability, security, and performance of a proxy system. It’s more than basic filtering. Proxies need to detect malformed headers, incomplete or oversized requests, and block unwanted paths. This adds complexity and requires extra logic, memory, and CPU. Proxies must handle all of this while still performing well under pressure. Some examples:

**Validity Checks:**
Ensure the request conforms to protocol expectations. For example, reject requests with multiple Content-Length headers or conflicting Content-Length and Transfer-Encoding values.

**Completeness:**
Attackers may send data very slowly to tie up resources (e.g., [Slowloris-style](https://en.wikipedia.org/wiki/Slowloris_(cyber_attack))). Proxies often enforce timeouts and request completion checks.

**Size Limits:**
Reject requests that exceed acceptable limits for URL length, headers, or cookie size to prevent resource exhaustion.

**Access Control:**
Block access to restricted endpoints such as /admin, /healthcheck, or any routes intended only for internal or localhost use.
Many times the server expects these requests only from a certain ip ranges. e.g. if you have put your static assets in some of CDNs, you may expect the CDN to hit your server occasionaly to load the assets. These endpoints should be only accesible for the cdn and not be accessibe by other.


Similarly outbound responses from proxy also need to be sanitized. For example, the proxy often acts as a central enforcement point for GDPR compliance by ensuring only approved cookies are included in the response. It also plays a key role in setting appropriate security headers like Content-Security-Policy to protect clients from common web threats.

## Header Manipulation
Reverse proxies often need to add/remove/update headers:

* X-Forwarded-For: Track client IP
* User-Agent: Normalize or drop broken values
* Geo-IP headers: For localization
* Request-ID: For observability
* Authorization: Normalize for downstream

Adding/removing headers means the proxy needs change the incoming stream of data before forwarding it. This ofte means that the complete headers are stored in memory and easily accessible/updatable.

This adds come constrains on the memory footprint, the max length of headers that proxy can read/write and other optimization related constraints.
For easy search/configuration, one standard way of search optimization is proxy converts all the header key to lowercase. In some odd cases, this can cause trouble with clients/servers although http spec suggests it should be case insensitive

These adds additional complexity to the system

## Rewriting Paths
In many real-world scenarios, proxies need to rewrite incoming request paths before forwarding them to the upstream service. This can serve several purposes:

* Migrate /v1/* to /v2/*
* Redirect from http → https
* Hide implementation details (/api/products?id=123 → /p/123)

This is where things like prefix stripping, redirects (301/302), or even full-blown regex rewrites come in.

# Final Thoughts
While HTTP parsing might seem like a simple, “solved” problem, it’s precisely where performance, security, correctness, and compatibility often clash. A reverse proxy functions as both a vigilant guard and a skilled translator, tasked with handling messy, unexpected behavior without crashing. Understanding these underlying complexities is crucial for anyone building or operating scalable web services.





