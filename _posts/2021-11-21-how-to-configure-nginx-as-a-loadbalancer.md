---
layout: post
title: How To Configure Nginx As A Load Balancer
categories: [Infrastructure]
---

Software-based horizontal scaling infrastructure is popular because of its flexibility and cost-effectiveness. You need a load balancer that behaves as dynamic as your infrastructure.

## Prelude

NGINX fills our need as a load balancer in a number of ways. It can be configured as a HTTP, TCP and UPD load balancer with different balancing behaviors, which I will cover in this blog post.

Regardless of the stateless-ness of modern web applications, reality shows that we might still need to let clients stick to certain servers. NGINX can handle that. When we balance load, the impact to users may **only** be positive.

You also need to ensure that the backends you are balancing load to are halthy - NGINX offers health checks for this, passive and active ones. We will discuss them later in this post.

## HTTP Load Balancing

Let's get to the most basic use case: _You want to distribute request load between >=2 HTTP web servers._

NGINX offers a `HTTP module` to balance load via a `upstream` block, as shown here:

```nginx.conf
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
