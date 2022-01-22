---
layout: post
title: Setup Nginx As A Reverse Proxy (on your Raspberry Pi)
categories: [Infrastructure]
---

In a previous post, we discussed on [using `nginx` as a load balancer](https://torbentechblog.com/b-how-to-configure-nginx-as-a-loadbalancer/) for various protocols, as well as [setting up a Raspberry Pi as a VPN](https://torbentechblog.com/setup-raspberry-pi-as-vpn/). 

Those use cases are already very interesting, but what if one of the two cases happen?

1. You need to forward traffic to more than one application inside your network
2. On the same host, you host two applications serving clients

This is where `nginx` again comes in handy for utilizing it as a reverse proxy. Let's have a look into it, and see how we can set it up easy and secure!


## Reverse Proxies in General

When looking at computet networks, a _reverse proxy_ simply is an application that sits in front of one or more backend applications and forwards clients (browsers, other applications, ...) to those applications. In that way, reverse proxies tremendously increase scalability, resilience, performance and security! And the best of it? The client applications don't know it. Resources returned by the reverse proxy appear to the client as if they come from the reverse proxy itself, so the client does not need to know that its traffic was only forwarded.

![nginx forwarding traffic from multiple clients](http://www.plantuml.com/plantuml/png/SoWkIImgAStDuShBJqbLI2hABozEBO9maaE3V22iyjIa-CI20WWdBpqphmAgF34vEpKlnH25PuJ2C-RYWXggeAiBrGio6C636euG09D0Bjnu314Zk0Z26WSW2VG70000)
_nginx forwarding traffic from arbitrary clients to several backend services_

## But does it run on my Pi?

Yes! I've been running it for quite a time now on my [Raspberry Pi Zero](https://www.berrybase.de/raspberry-pi/raspberry-pi-computer/boards/raspberry-pi-zero-w) and never ran into any resource or performance problems. My installation is bare-metal (no containerization for keeping the resource overhead as low as possible) and is receiving quite some load. It forwards traffic in my home network for 10-15 backend applications which are accessed by around 10-30 devices constantly. `nginx` barely needs any RAM and perfectly utilizes your CPU power. Also, it's got its extra version for ARM processors which comes in handy for Raspberry Pi installations.

## Installation

The installation is quite simple, lets get started. Take this as an example for Debian'ish systems with `apt` installed.

First, update your local package index and upgrade any packages which are already installed on your machine:

```bash
  $ sudo apt update && sudo apt upgrade -y
```

Now, we want to get rid of any existing `apache2` installations. `apache2` is another web server which might come already installed on your machine (e.g. when you start up a server distro) and it interferes with our `nginx` installation, so let's remove it:

```bash
  $ sudo apt purge apache2
``` 

This will remove the web server itself including its configurations.

And now, finally install `nginx`!

```bash
  $ sudo apt install -y nginx
```

Depending on your machine and your network, this might take a while. When you're done, head over to the next step.

## Configuration (the easy way)

Normally, your `nginx` installation should start up right away and you can access the default page at `http://<your-ip>:80`. If not, start it up like so:

```bash
  $ sudo systemctl start nginx
```

### Configuration directory

Per default, `nginx` stores its configuration at `/etc/nginx/`, so let's have a look at it:

```bash
  $ cd /etc/nginx/sites-available/
```

and create your first configuration file called `example.conf`.

```bash
  $ sudo vim example.conf
```

Add the following configuration (adjust the host and proxy_pass target to match your needs):

```nginx
server {
  listen 80;
  server_name my.local.dns;
  location / {
    proxy_pass http://localhost:3000;
  }
}
```

`listen` is going to tell `nginx` to _listen_ on port `80` which is the standard HTTP port. `server_name` is going to be the domain of the website which clients will type into their browser when they want to reach a certain backend. For our example, we will forward `/` which basically means **every** request that reaches our web server. Then, we will forward clients to the URL behind `proxy_pass`. This can be any URL, IP or even another `nginx` location.

Hit save, and now we will check the config, enable it and reload nginx.

```bash
  $ ln -s /etc/nginx/sites-available/example.conf /etc/nginx/sites-enabled/example.conf

  $ sudo nginx -t

  $ sudo systemctl reload nginx
```

And that's it! First we symlink our fresh configuration to the `sites-enabled/` directory, so `nginx` knows that it needs to respect that configuration as active. Then, we test the config for being syntactically correct with `sudo nginx -t` and then we reload our `nginx` service so the new config gets loaded.

We're done! Our nginx installation is now forwarding traffic from `my.local.dns:80` to a backend application running on the same host as nginx which is listening on port `3000`.


## Summary

In this post, you learned:

- What reverse proxying is
- How you can utilize it for your home or enterprise environments
- How requests are distributed
- How to simply and securely configure it even on a Raspberry Pi
