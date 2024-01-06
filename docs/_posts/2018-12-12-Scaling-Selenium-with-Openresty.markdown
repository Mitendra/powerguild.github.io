---
layout: post
title:  "Scaling Selenium with Openresty"
date:   2018-12-12 08:43:56 -0800
categories: medium blogs

---  
  
![](/assets/images/highrise_one.png) 
  
Selenium has been the de facto automation standard/tool for browser/UI automation for quite sometime.  
  
Creators of selenium also created selenium grid for scaling the UI testing. The grid consists of a selenium hub and several selenium nodes which connects/hosts the browsers and thus providing a way to run several UI tests simultaneously. Selenium also provides some easy way to customize/extend existing capabilities. For most of the use cases the selenium hub /node based architecture scales pretty well and doesn’t need much customization.  
  
### Need for higher availability and scale
The architecture, although supports scaling the selenium nodes, it doesn’t make it easy to scale the hub itself. On a typical deployment there would be a single hub and several nodes attached to it.  
  
The mapping of sessions/browser instances is maintained in memory inside the hub. So, if the hub fails, the whole grid will become unusable.  
  
A simple way to enable some level of HA is to run at least 2 hubs.  
  
### Challenge with multiple hubs  
  
#### Requirement of sticky session  
Since selenium hubs can’t talk to each other, the distribution of load to different hubs need some kind of stickiness. Stickiness here needs to make sure that once the session is created on one hub, all subsequent request for that specific session comes to the same hub.  
  
Session stickiness is not a new thing. A typical stickiness for browser based clients is done via cookies. The LB looks at the cookie information being passed to decide which upstream server the request would be forwarded.  
  
#### Cookie based stickiness ruled out  
  
Passing this extra bit of information using cookie is not possible in selenium automation scenarios, so this was not an option for our use case.  
  
#### Client ip based stickiness  
  
One option that we have been using so far is to create stickiness based on client ip.  
  
So all calls originating from one client ip will always land to a given hub. While this works for most of the scenarios, the load on hubs are not distributed evenly.  
  
This particular technique solves the HA problem as such, but this may not be a great solution for scaling if the load is not even.  
  
#### Uneven load and uneven test cases in test suites  
  
A large test suite with huge number of UI tests can completely chock the one off hub where all tests are landing.  
  
For making our test infrastructure more reliable and robust, we may have to explore some different avenue.  
  
### Openresty to the rescue  
  
Openresty is a software bundle of nginx with some high quality Lua libraries included. It allows you to customize load balancing behavior and add custom behaviors.  
  
#### Design  
  
We can add openresty as a proxy layer before the hub.  
  
We can deploy couple of instances of openresty in front of hub and the LB can forwards the request to one of these openresty instances in a round robin fashion ( earlier the LB was forwarding the request to hub directly using the client ip hashing).  
  
Openresty instances works as a reverse proxy and all instances of hub are upstream server for each openresty instances.  
  
#### The Magic  
  
The way the request forwarding works is:  
  
1. Each new session request will be forwarded to one of the hub in a round robin fashion.  
   
2. Once the session is created at hub and response comes back to the openresty proxy instance, it inspects the response body, looks for session id and appends the upstream host ip(host ip in number format) to the existing session id and responds back to the client.  
   
3. The client receives a new session id which has the upstream hub node embedded in it.  
   
4. Any subsequent operation on that session from client uses this session id in the url path, which now has the upstream hub’s ip embedded in the session id.  
   
5. The proxy inspects the request url and extracts the session id from the request.  
   
6. It rewrites the url path by removing the embedded host ip part.  
   
7. It also sets the upstream host to the specific host found in the ip section of the session id.  
    
8. Then it forwards the request to the desired hub.  
   
9. The hub sees the session id in the exact same format as it had generated and continues to process as usual.  
    
**Sample nginx conf with embedded lua code:**  
  
[scaling_selenium/nginx.conf](https://github.com/Mitendra/code_samples/blob/master/scaling_selenium/nginx.conf)  
  

{% highlight shell %}
worker_processes auto;
worker_rlimit_nofile 1024;
error_log logs/error.log debug;
events {
    worker_connections 1024;
}
http {
		upstream seleniumhub {
		server host1ip:4444; # replace this with actual upstream hub ip and port
		server host2ip:4444; # replace this with actual upstream hub ip and port
		}

	server {
		listen *:4444;
		listen *:5555;
		
		location ~ /wd/hub/session$ {
			proxy_pass http://seleniumhub;
			set $response_body '';

			# Reset the response's content_length, so that Lua can generate a 
			# body with a different length. 
			header_filter_by_lua_block {
				ngx.header.content_length = nil
				--ngx.log(ngx.ERR, " header filter ngx.var.upstream_addr ", ngx.var.upstream_addr) 
			}

			body_filter_by_lua_block {
			local resp_body = ngx.arg[1]
			ngx.ctx.buffered = (ngx.ctx.buffered or "") .. resp_body
			if ngx.arg[2] then  -- arg[2] is true if this is the last chunk
				ngx.var.response_body = ngx.ctx.buffered
			else
				ngx.arg[1] = nil 
				return
			end
			
			-- extract the ip in number format from upstream host
			local upstream_host = '127.0.0.1'
			if ngx.var.upstream_addr then
				local ngx_re = require "ngx.re"
				local addrs = ngx_re.split(ngx.var.upstream_addr, ",")

				if #addrs > 0 then
					--ngx.log(ngx.ERR, "multiple upstream: " .. ngx.var.upstream_addr)
					upstream_host = addrs[#addrs]
				else
					--ngx.log(ngx.ERR, "single upstream")
					upstream_host = addrs
				end
			end
			local o1,o2,o3,o4 = upstream_host:match("(%d+)%.(%d+)%.(%d+)%.(%d+)") -- this will exclude :4444 automatically
			local num = 2^24*o1 + 2^16*o2 + 2^8*o3 + o4
			local ip = math.floor(num)
			 ngx.log(ngx.ERR, "current_response " .. ngx.var.response_body)	
			local current_session_id = 	ngx.var.response_body:match('"sessionId":"(%w+)')
			local new_session_id = ip .. 'dddd' .. current_session_id

			local modified_response = ngx.var.response_body:gsub(current_session_id, new_session_id)
			ngx.log(ngx.ERR, "current_session_id " .. current_session_id .. "new id " .. new_session_id)
			
			ngx.arg[1] = modified_response	
			
			}	
		}

		location ~ /wd/hub/session/.+ {

			set $target_host '';
			set $target_uri '';
			access_by_lua_block {
				local url = ngx.var.request_uri
				ngx.log(ngx.ERR, "url " .. url)
				local session_id = url:match('/wd/hub/session/(%w+)')	
				local upstream_host_ip_in_num, upstream_session_id = session_id:match('(%w+)dddd(%w+)')

				local o1 = math.floor(upstream_host_ip_in_num/(2^24))
				local rem = upstream_host_ip_in_num % (2^24)

				local o2 = math.floor(rem/(2^16))
				rem = rem % (2^16)

				local o3 = math.floor(rem/(2^8))
				rem = rem % (2^8)

				local o4 = math.floor(rem)

			
				local upstream_host_ip =  o1 .. '.' .. o2 .. '.' .. o3 .. '.' .. o4
				ngx.var.target_host = upstream_host_ip

				ngx.var.target_uri = url:gsub(session_id, upstream_session_id)
				
			
			 }

			proxy_pass http://$target_host:4444$target_uri;
		
		}	
	}
}
{% endhighlight %}  
  

### Taking a step further  
  
While solving the scale and availability with openresty proxy, we can use this proxy for some more features built on top of Selenium hub.  
  
#### Rate limiting  
  
Even though this would help in scaling the hub and nodes, there are scenarios where the Spike in number of new sessions may exceed the number of available resources. Currently all requests are put in the queue and processed when the resources are available which results into delayed session creation. Different test cases/frameworks have different timeout settings and while some of them proceed, many of them would get stuck. Many times, while the sessions get times out at client/test case layer, the hub still will go on and create a session. This unused sessions later gets cleaned up but it makes resource crunch even worse. To solve this we can add a rate limiting feature, where based on the capacity and traffic pattern only certain number of sessions in the queue would be allowed and beyond that we can return a http 429 status with some details like current active sessions and an estimated time after which it can be retried.  
  
#### Monitoring data  
  
We can add a lot of monitoring data about sessions bring requested, time to create a new session, active session time etc from proxy layer, which can give further insight on how to optimize the infrastructure usage.  
  

### References  

[Image source](https://pixabay.com/en/skyscraper-highrise-building-tall-1209407/)  
[Selenium](https://www.seleniumhq.org/)  
[Selenium grid](https://www.seleniumhq.org/docs/07_selenium_grid.jsp)  
[Openresty](https://openresty.org/en/)  
[openresty/lua-nginx-module](https://github.com/openresty/lua-nginx-module)  