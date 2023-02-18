---
layout: post
title: k3d Up and Running Tutorial for Linux machines
categories: [Public Cloud, Kubernetes, Homelab]
---

From zero to local Kubernetes development in 10 minutes. Promised!

Get started with local Kubernetes clusters to enhance development speed and follow the step-by-step tutorial to even use local ingress DNS.

## Introduction to k3d

`k3d` is a containerized lightweight Kubernetes distro, based on `k3s`.

`k3s` is a certified Kubernetes distribution which was originally built for the IoT / Edge computing segment. IoT devices naturally have less power than workstations and in some use cases still need to be able to be clustered with Kubernetes. This is where `k3s` comes in handy.

We will use `k3d` to spin up a local multi-node HA Kubernetes cluster, including a sample `nginx` Deployment and some local configuration snippets to use super cool local DNS like `nginx.k3d.localhost`.

Ready? Let's get started!

## Machine requirements

This tutorial was written for Linux machines (Ubuntu, in my case). In later configuration, you might need to change some configs according to your system you're on.

Basically, you do not need anything else installed beforehand but `docker`. [Get Docker here](https://docs.docker.com/get-docker/). If you hate the new Docker ToS, any other container runtime environment (e.g. `podman`) will do the trick, too.

## Install

Since I want this tutorial to still be valid in a month, we'll default to the latest release here.

`k3d` comes with a nice wrapper CLI which will help us deploying local Kubernetes clusters in seconds. Get it here:

```bash
  $ wget -q -O install.sh https://raw.githubusercontent.com/rancher/k3d/main/install.sh
  [...]
  $ chmod +x install.sh
  $ ./install.sh
```

If you don't have `wget` installed on your machine, you can also download the installer script with `curl`.

After executing the installer script, verify your installation by typing:

```bash
  k3d --help
```

Now we're ready to rumble!

## Your first cluster - done right

You can always quickly create clusters with using the wrapper CLI flags and arguments. But we want to create reproducable environments, and maybe even want to share them with our team, right? `k3d` understands this and is able to parse YAML configs. Let's start with our first simple configuration:

```yaml
---
apiVersion: k3d.io/v1alpha3
kind: Simple
name: local
servers: 1
agents: 2
image: rancher/k3s:v1.20.4-k3s1
ports:
  - port: 8080:80
    nodeFilters:
      - loadbalancer
options:
  k3d:
    wait: true
  kubeconfig:
    updateDefaultKubeconfig: true
    switchCurrentContext: true
```

The first three lines are needed in every `k3d` cluster configuration. `name` is the name which `k3d` will give to your cluster. In your kubeconfig, the clustername will appear as `k3d-[name]`. We configure the number of master nodes with `servers` and the number of worker nodes with `agents`. This config uses a `k3s` image with includes Kubernetes `v1.20.4`.

`k3d` gives you a `traefik` loadbalancer shipping with batteries, and we want the inner port `80` to be reachable on our machine on `8080`. Thus, we'll be able to call our Kubernetes applications on `http://localhost:8080`.

At the end, there's options telling `k3d` to

- wait until the cluster is ready (which takes about 15-30 seconds!)
- update our `/.kube/config` file with a new context pointing to our cluster
- switch the current `kubectl` context

These are only some convenience features that can be turned off.

Save the above configuration file as `config.yml` and create your cluster like this:

```bash
  k3d cluster create --config config.yml
```

As already mentioned, this will be done blazingly fast. If you ever worked with _minikube_ or _kind_, you'll instantly love it.

**Pro tip:** Do you need more nodes fast? This can [be done while your cluster is running](https://k3d.io/v5.2.2/usage/multiserver/#adding-server-nodes-to-a-running-cluster)!

![k3d overview](https://www.plantuml.com/plantuml/png/POgz3S9038LxJ_4MI45O89kWKsp2YJjRSjvLYDqf81A8yljPTfRaw4sQNGa6icutGclQoXekQukXk9yL3m4yrD3BJik3ocREo-aNPtdAUyCq7SkVcULJljLYhgEt5m00)

And...basically, we're done! Have fun with your cluster! - Or read on and get application manifests and a tutorial on how to forward traffic to your cluster locally with DNS!

## Forwarding traffic

Above, I told you that your cluster is reachable via `http://localhost:8080` using the given configuration. That gets lame quite quickly, especially when you're running multiple applications. Let's change that!

### Host configuration

We want to make `k3d.localhost` available as a domain for our `traefik` ingress. This is done quickly:

```bash
  sudo echo "127.0.0.1 k3d.localhost" >> /etc/hosts
```

Now we can access the cluster using `http://k3d.localhost:8080/`. The port is kinda annoying, isn't it?

You have two options.

1. Change the forwarded port in the cluster configuration to `80` - which did not work in my case.
2. Reverse proxy requests to `http://k3d.localhost:80/` to `http://k3d.localhost:8080/`.

I'll quickly show you the second option.

### Reverse proxying traffic (also useful for HTTPS redirects)

This can easily be done with using `nginx` locally. I already have one running, but if you don't, that's not a problem:

```bash
  $ sudo apt purge apache2 && \
      sudo apt update && \
      sudo apt upgrade -y && \
      sudo apt install -y nginx && \
      sudo systemctl start nginx
```

Now replace the default config (normally located at `/etc/nginx/sites-enabled/default`) with your own:

```nginx
server {
        listen 80;

        server_name "^.*k3d\.localhost$";

        location / {
                proxy_pass http://127.0.0.1:8080/;
                proxy_set_header Connection "";
                proxy_set_header X-Forwarded-For $host;
                proxy_set_header X-Forwarded-Proto $scheme;
                proxy_set_header Connection `upgrade´;
                proxy_set_header Host $host;
        }
}
```

This will forward every traffic containing `k3d.localhost` in the domain to our `k3d` cluster.

Reload nginx and test it!

```bash
  $ sudo nginx -t && \
      sudo systemctl reload nginx
  $ curl -vL http://k3d.localhost
```

This should show you the default ingress `404` page.

Traffic is directed like this:

![Traffic direction with nginx reverse proxying](https://www.plantuml.com/plantuml/png/NOn12iCm30JlUiL-88UczvAl64j9JHpBg78fbFwzO99ISf6q8tQcXmVpjcNACZjSOMcvEpYPH4zQA4HN0yjJibOnAig2igJoefYrCTOhuqr0VxW5cTDwn53hvUyUwKC_4uRXjelwxFdexxkpBZc1aIOftBRy1G00)

If you want to read more on nginx reverse-proxying, [this post](https://torbentechblog.com/setup-nginx-as-reverse-proxy/) might be interesting for you.

## Example application to host

You've made it this far? Great! Let me provide you the last YAML for today to host your first high-available sample application.

We will deploy 4 instances of `nginx` which will, when called, return _Hello from k3d!_.

To accomplish this, we will need a `Deployment`, a `Service`, an `Ingress` and a `ConfigMap` which are all describe below.

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
  namespace: default
spec:
  rules:
    - host: nginx.k3d.localhost
      http:
        paths:
          - path: /
            pathType: ImplementationSpecific
            backend:
              service:
                name: nginx
                port:
                  number: 80
```

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: default
spec:
  selector:
    app: nginx
  type: ClusterIP
  ports:
    - name: nginx
      protocol: TCP
      port: 80
      targetPort: 80
```

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx
  namespace: default
data:
  nginx.conf: |
    user nginx;
    worker_processes  3;
    error_log  /dev/stdout info;
    events {
      worker_connections  10240;
    }
    http {
      log_format  main
              'remote_addr:$remote_addr\t'
              'time_local:$time_local\t'
              'method:$request_method\t'
              'uri:$request_uri\t'
              'host:$host\t'
              'status:$status\t'
              'bytes_sent:$body_bytes_sent\t'
              'referer:$http_referer\t'
              'useragent:$http_user_agent\t'
              'forwardedfor:$http_x_forwarded_for\t'
              'request_time:$request_time';
      access_log /dev/stdout;
      server {
          listen       80;
          server_name  _;
          location / {
              root   /var/www/html;
              add_header X-NGINX-HOST $hostname;
              index  index.html index.htm;
          }
      }
    }
  index.html: |
    <html>
      <body>
      Hello from k3d!
      </body>
    </html>
```

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: default
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 4
  strategy:
    rollingUpdate:
      maxSurge: 50%
      maxUnavailable: 50%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:stable-alpine
          resources:
            requests:
              cpu: 100m
              memory: 100Mi
            limits:
              cpu: 100m
              memory: 100Mi
          livenessProbe:
            tcpSocket:
              port: 80
            initialDelaySeconds: 5
            timeoutSeconds: 5
            successThreshold: 1
            failureThreshold: 3
            periodSeconds: 10
          readinessProbe:
            tcpSocket:
              port: 80
            initialDelaySeconds: 5
            timeoutSeconds: 2
            successThreshold: 1
            failureThreshold: 3
            periodSeconds: 10
          ports:
            - containerPort: 80
              name: nginx
          volumeMounts:
            - mountPath: /etc/nginx
              readOnly: true
              name: nginx-config
            - mountPath: /var/www/html
              readOnly: true
              name: nginx-html
            - mountPath: /var/log/nginx
              name: log
      restartPolicy: Always
      volumes:
        - name: nginx-config
          configMap:
            name: nginx
            items:
              - key: nginx.conf
                path: nginx.conf
        - name: nginx-html
          configMap:
            name: nginx
            items:
              - key: index.html
                path: index.html
        - name: log
          emptyDir: {}
```

Save those YAML files and apply them with `kubectl apply -f [my files].yaml`.

After that, try to reach your nginx!

```bash
  curl -vL http://nginx.k3d.localhost
```

## Summary

In this post, you've learned how to:

- Install the `k3d` CLI
- Create your first local Kubernetes cluster
- Forward traffic with cool local DNS to your applications
- Deploy your first sample application

I hope you enjoyed it! If you have any questions, don't hesitate to reach out to me. Happy coding!
