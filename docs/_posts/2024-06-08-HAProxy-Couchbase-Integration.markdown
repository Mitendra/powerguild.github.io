---
layout: post
title:  "HAproxy and Couchbase Integration"
date:   2024-06-08 08:00:56 -0800
categories: ReverseProxy

---  
# Introduction

Couchbase is a popular distributed NoSQL database frequently used for low-latency, high-throughput, consistent key-value data store in large-scale distributed systems.

Currently, there is no open-source solution available for connecting to Couchbase from HAProxy directly. This blog post will explore how we can integrate with Couchbase for key-value use cases.

# Options for HAProxy integration
1. SPOE: HAProxy has this protocol built in to offload some parts of request handling to an external agent. As part of the request processing, we can send the required details to a sidecar (SPOE), pause the current transaction, and once the response is available, we can continue with processing the request. This SPOE agent can communicate with Couchbase and send the response back to HAProxy. More details on SPOE can be found [here](https://github.com/haproxy/haproxy/blob/master/doc/SPOE.txt). While this is a fairly good way to integrate with Couchbase, if we need to get some Couchbase data on each request (e.g., validating login information against Couchbase), performance and reliability can suffer.
      
2. Lua integration: This will be the focus of this blog. If we could integrate Couchbase directly from HAProxy: 
    
   a. It would save an additional hop to the sidecar.  
   b. Key dependency on an external process would be removed.  

# Lua integration with HAProxy:

## Current state
There are no existing options for direct integration with Couchbase (CB) in HAProxy. While CB provides REST APIs for management and query purposes, it does not offer an HTTP API for key-value (K/V) operations. SDKs are available in languages such as Go, C++, and Java, but there is no Lua SDK available. Although there are a few open-source options to connect to CB using Lua for Nginx, none exist for HAProxy.

Nginx/openresty only supports LuaJIT, which is limited to Lua 5.1/5.2. Many new features introduced in Lua 5.3 and later are not available in LuaJIT.

Moreover, the current implementations in those libraries do not seem to support mTLS-based authentication.

## High-Level Overview of Couchbase (CB)

1. CB is cluster-aware and shards data across different nodes. For easy scaling and management, it uses buckets and virtual buckets instead of mapping keys directly to hosts.
2. Buckets are logical groupings of data that can be used for access control, among other things.
3. Each bucket is divided into 1024 virtual buckets. Each virtual bucket is then mapped to a node in the cluster.
4. To access a key, we need to find the virtual bucket and the corresponding node.
5. The virtual bucket-to-node mapping is not static and can change based on the number of nodes in the cluster.

## Simplified K/V Memcached Protocol
![](/assets/images/cb_request_response.jpeg)  
1. A stateful TCP protocol. Commnads need to run in a speceific sequence/context.
2. A basic CB request consists of:
   1. A 24-byte request/response header.
   2. The header contains the request/response body length, and the body comes as raw bytes after the header.  
3. For our use case:  
**Magic Code**: 128 for request, 129 for response  
**OpCode**: 0 for GetKey command, 181 for Get Cluster Config, 137 for Select Bucket  
**ExtraLength**: 0 if no extra header is being passed, otherwise the length of the extra data  
**DataType**: 0  
**vBucketID**: Bucket ID for the key  
**OpaqueData**: Random data, which the server will send back in the response as it is  

**Lua code to encode a request in CB memecached format**

```lua
local function _encode_request_pack(opCode, key, vBucketId, uuid)
  local magicCode = 128
  local opCode = opCode or 0
  local keyLength = #key
  local extrasLength = 0
  local dataType = 0
  local bucket = vBucketId or 0
  local _value = ''
  local bodyLength = #key + #_value
  local _opaque = uuid or 0
  local _cas = 0
  return string.pack(">BBHBBHI4I4I8c" .. bodyLength,
                      magicCode, opCode, keyLength,
                      extrasLength, dataType, bucket, 
                      bodyLength, _opaque, _cas, 
                      key, _value)
end
```  
  
## High-Level Overview of the CB flow

1. Use an SRV DNS query to find the host/port for Couchbase over the Memcached protocol. Pick the scheme as couchbases (s for secure).
```shell
$dig srv _couchbases._tcp.couchbase-host.example.com +short
    0 0 11207 cb001.example.com.
    0 0 11207 cb002.example.com.
    0 0 11207 cb003.example.com.
    0 0 11207 cb004.example.com.
    0 0 11207 cb005.example.com.
```
2. Pick any host and connect over TLS. TLS will be used for authentication; no separate SASL required.
3. Get cluster configuration to obtain the configuration details, which includes:
   1. A list of servers
```shell
$cat cluster_config.json | jq .vBucketServerMap.serverList
[
        "cb001.example.com:11210",
        "cb002.example.com:11210",
        "cb003.example.com:11210",
        "cb004.example.com:11210",
        "cb005.example.com:11210"
]
```
   2. Bucket-to-server mapping
```shell
$cat cluster_config.json| jq '.vBucketServerMap.vBucketMap'
    [
        [
            0,
            1,
            4
        ],
        [
            1,
            0,
            3
        ],
        [
            1,
            0,
            4
        ],
        [
            1,
            0,
            4
        ],
        ...... 1020 more records in the array
    ]
```
4. Send the UseBucket command to set the context.    
5. Compute the vBucket for the key you want to query:
    1. Get the CRC32 hash of the key.
    2. Take the 2 most significant bytes (MSB) by right-shifting by 16.
    3. Use this number to get the vBucket ID: crcHash & (total number of buckets - 1) to limit the hash to the number of bucket hashes.   
```lua
local function _get_bucket_id(key)
    local hash = crc32(key)
    local vbucket_id = ((hash >> 16) & 0x7fff) & 1023
    return vbucket_id
end
```
   1. Get the server index for this bucket from the bucket-to-server map obtained in step 3.
   2. Use this index and the data from step 3.1 to get the target host.

## Get Key Command
1. Connect to the target host if not already connected.
2. Set the bucket context by executing the Select Bucket command.
3. Execute the GetKey command.
4. Read the response header.
6. Extract the response size from the response header and read that many bytes to get the response.  
   
**Sample lua code for querying a a sample key. Error handling removed for simplicity**

```lua
local function get_cb_key(key)
  -- crc32 hash, extract 2 MSBs, & with 1023
  local bucket_id = _get_bucket_id(key)
  -- indexes are based on 1 in lua, cluster_config is obtained from getclusterconfig command
  local bucket_server_list = cluster_config["vBucketServerMap"]["buckets_map"][vbucket_id + 1] 
  -- first one is primary server
  local primary_server_index = bucket_server_list[1] 
  -- server index are 0 based but lua indexes are 1 based
  local hostname = cluster_config["vBucketServerMap"]["serverList"][primary_server_index + 1]
  local command = _encode_request_pack(CB_CMD_GET, key, bucket_id, req_uuid)
  -- _host_client_map contains host vs active client connection (ssl tcp connection, followed by selectBucket command executed on the tcp connection)
  local cb_client = _host_client_map[cb_host]
  local bytes_sent, err = cb_client:send(command)
  -- Read response header
  local response_header, err = cb_client:receive(24)
  -- response length starts at 9th byte
  local response_length = string.unpack(">I4", response_header, 9) 
  if response_length == 0 then
    return nil
  end
  local response_data, err = cb_client:receive(response_length)
  return response_data
end
```  
  
## What Happens When a Server Node Changes
1. If a new node is added or removed, the bucket may be offloaded to a different host and the bucket-to-host mapping will change.
2. If a request is sent to a host that no longer has the corresponding bucket (key), the response will indicate that the host doesn’t have the bucket.
3. Such a response means we need to get the latest configuration by querying the Get Cluster Config.

## Pipelining
1. The CB GetKey command is context-aware, requiring the correct bucket context be set before execution.
2. CB authentication can be granular, allowing access to specific buckets. If you use a client certificate during the TCP connect, you may have access to only a specific set of buckets during the whole session.
3. The cost of opening an SSL connection is very high, much higher than serving a GetKey request.
4. For concurrent connections, CB supports request pipelining, so clients do not need to open multiple connections.
5. For pipelining, the order of responses will still be the same as the order of requests.
6. To make it easier for the client, the protocol supports opaque data, which the server returns as is in the response. This allows the client to match the response to the correct request.

## HAProxy Integration Highlights
1. mTLS support is not enabled in HAProxy Lua TCP socket. A hack to enable it is available [here](https://github.com/Mitendra/haproxy-lua-couchbase?tab=readme-ov-file#haproxy-lua-socket-patch-for-mtls), but a better fix is needed.
2. DNS support is not enabled in Lua. HttpClient has DNS resolution options, but they don’t apply to TCP sockets. The do-resolve action is present, but interaction between Lua and HTTP actions is not documented. Attempts to use it did not work.
3. Since DNS can happen over TCP, we can build a simple, limited-scope DNS solution to handle specific use cases.
4. TCP calls can only happen in the same thread, so we need to use Lua-per-thread for this kind of interaction.
5. For pipelining:
    a. We need a way to send requests from different ongoing transactions into a queue. HAProxy Lua supports a queue which can be used here.
    b. Pause the ongoing transactions. Lua has an option to yield/sleep without blocking the event loop.
    c. Read from the queue and connect to CB. This needs to be built.
    d. Read the response and share it with the paused transactions. We are not aware of a way to enable the transaction when some other action is triggered in Lua (other than IO). A workaround is to periodically wake up, try to read the data from the shared source, and if not found, pause again.
6. TCP timeouts: core.tcp only supports overall timeouts and doesn’t have individual read/write/idle timeouts. If we set the timeout very high, a connection error takes a very long time; if we set it low, the TCP connection gets closed frequently and new connections need to be opened periodically.

## Testing
1. Most of the lua code do not really depend on the HAProxy, so we should be able to test that independently.
2. We can use some way to switch between the HAProxy TCP socket and LuaSocket. An environment variable can control this behavior
3. We can capture the request/response from Couchbase once and reuse them for testing the client and server.
4. DNS resolution is an important step, and actual CB data will contain hostnames. To make it easier, we can use dnsmasq to point to localhost. This enables end-to-end testing without depending on the actual server.

## Load Testing
1. While basic testing can be done against the actual CB server, doing load testing against a production or staging CB server may impact other users. 
2. Since we already understand the basics of the CB Memcached protocol, we can use it to build a mock test CB server as well.
3. For testing, we can ignore the mTLS part.
4. For high-volume tests, we’ll need multiple connections to be open to the server. A simple native lua cb server wont work here if it's single threaded. 
5. We can leverage HAProxy itself to provide multithreaded eventloop and use it as a CB server to test end-to-end behavior locally.

# Conclusion
We are able to build a primitive lua couchbase integration. You can find the complete code with example [here](https://github.com/Mitendra/haproxy-lua-couchbase)







