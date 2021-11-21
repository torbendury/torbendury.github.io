---
layout: post
title: How To Configure Nginx As A Load Balancer
categories: [Infrastructure]
---

Software-based horizontal scaling infrastructure is popular because of its flexibility and cost-effectiveness. You need a load balancer that behaves as dynamic as your infrastructure.

## Prelude

NGINX fills our need as a load balancer in a number of ways. It can be configured as a HTTP, TCP and UPD load balancer with different balancing behaviors, which I will cover in this blog post.

Regardless of the stateless-ness of modern web applications, reality shows that we might still need to let clients stick to certain servers. NGINX can handle that. When we balance load, the impact to users may **only** be positive.

You also need to ensure that the backends you are balancing load to are halthy - NGINX "Plus" offers health checks for this, passive and active ones. We will discuss them later in this post.

## Load Balancing Protocols

### HTTP Load Balancing

Let's get to the most basic use case: _You want to distribute request load between >=2 HTTP web servers._

NGINX offers a `HTTP module` to balance load via a `upstream` block, as shown here:

```nginx
    upstream myBackend {
        server 13.37.42.42:80       weight=1;
        server myapp.example.com    weight=2;
    }
    server {
        location / {
            proxy_pass http://myBackend;
        }
    }
```

This configuration snippet will balance any loads across the two declared HTTP servers on port `80`. The `weight` parameter defined will tell NGINX to pass **twice** as many connections to the second server (`weight=2`). `weight` defaults to `1`.

The shown `upstream` module controls HTTP load balancing for us. We can define a pool of destination servers (by IP, DNS records - yes, even UNIX sockets) and can control balancing mechanisms by declaring `weight` on certain upstreams. Each `upstream` destination may be defined in the `server` directive block. You can even control balancing algorithms (NGINX Plus feature).

To make this part complete, NGINX "Plus" (paid version) offers you some more convenient parameters as connection limits, thresholds, advanced DNS resolution and even some load ramping mechanisms.

**NGINX "Plus" features will not be discussed in this post.**

### TCP Load Balancing

Let's head away from HTTP and go to more backend-ish things, based on TCP connections. With NGINX, you can provide a HTTP frontend to speak with TCP backends, such as a MySQL database cluster.

Define an example cluster like this:

```nginx
    stream {
        upstream mysql_readonly_instances {
            server read1.mysqldb.example.com:3306   weight=5;
            server read2.mysqldb.example.com:3306;
            server 13.37.42.42:3306                 backup;
        }
        server {
            listen 3306;
            proxy_pass mysql_readonly_instances;
        }
    }
```

This one is a little bit more complicated. The `server` block instrcuts NGINX to listen on `TCP:3306` and balance incoming load between two MySQL servers (`read1` and `read2`) - and even lists another one as a pure backup, just in case that `read1` and `read2` die.

**IMPORTANT:** This config should not be added to `conf.d/`, as those configs will be used within `http` blocks - **instead**, you will create a folder, e.g. `stream.conf.d`, open a `stream` block inside your `nginx.conf` file and include the folder with the snippet above.

With the `stream` module, you may (just like in HTTP load balancing) define upstream server. When defining your server, you need to pass a port to listen on (optional: address + port). The `upstream` block is pretty much like the HTTP load balancer `upstream` block.

### UDP Load Balancing

Let's get to the last kind of load balancing that NGINX provides us out-of-the-box: UDP load balancing. Take a simple use case: You have several NTP (Network Time Protocol) servers which keep your applications in time sync. Your apps periodically check for the time to ensure that everything is in sync. You need to provide a high available backend for NTP.

Take the following example:

```nginx
    stream {
        upstream ha_ntp {
            server ntp1.myntphosts.com:1337 weight=2;
            server ntp2.myntphosts.com:1337;
        }
        server {
            listen 123 udp;
            proxy_pass ha_ntp;
        }
    }
```

In this config example, we simply tell NGINX to use the UDP protocol by appending `udp` to our `server` block. Again, we use `stream`, which will be included from a `stream.conf.d` folder inside `nginx.conf`.
NTP is a very basic example. If we need to pass multiple requests and answers back and forth, we can declare the `reuseport` parameter. You definitely want to use this if you implement services like VPNs, VoIP, virtual desktops or DTLS.

Take this as an example: You are running a NGINX instance and have a OpenVPN server running locally. You want to use this code snippet:

```nginx
    stream {
        server {
            listen 1195 udp reuseport;
            proxy_pass 127.0.0.1:1194;
        }
    }
```

Client which connect to your NGINX instance via port `UDP:1195` will then be forwarded to your OpenVPN server which (per default) listens on `UDP:1194`.

## Passive Health Checks

tbd

## Slow Start

tbd
