---
layout: post
title: Cloud Native Local Development of Containerized Software with Docker and Kubernetes
categories: [Cloud, Kubernetes, Homelab, Docker]
---

How to setup your local development environment to be truly _cloud native_, using Docker and Kubernetes. All together united in your local IDE (e.g. Visual Studio Code - VS Code), you can test your applications like they were running in an arbitrary cloud.

## Cloud Native Development

Lately, the term _cloud native_ has become a catch-all for all the tools that are required by software engineers to build, deploy and maintain their software on _cloud infrastructure_.

To me, _cloud native_ development starts on my machine which sits in front of me. When I work _cloud native_ I don't just write code and push it to a repository. There is much more happening on your local machine, or at least it should.

What if you could build, containerize and run your software in an environment that has little to no difference from the big public cloud environment your company is running on?

What if you could test the integration of the services that your software depends on, before your code leaves your machine?

Let's find a simple and quick way to get started.

## IDE

First, make sure you work in an IDE you feel **comfortable** using. There are few things worse than working in an IDE you hate.

Ideally, your IDE is extensible, either via public marketplaces or via dedicated shops.

For my part, I now find my way around Microsoft's [Visual Studio Code](https://code.visualstudio.com/download) very well for most things. It is free and available for the three major platforms Linux, MacOS and of course Windows. The marketplace is built right in and most extensions can be used right after installation without restarting the IDE.

## Extensions and installed software

To follow this article I recommend you to install Docker (or another container environment of course). If you don't get along with Docker Inc.'s new pricing model, there are now an almost unmanageable number of alternatives.

Next, you need (in the example of VS Code) the extensions for [Docker](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-docker) and [Kubernetes](https://marketplace.visualstudio.com/items?itemName=ms-kubernetes-tools.vscode-kubernetes-tools). They're both officially published by Microsoft itself, so you won't find a code graveyard.

If you haven't read my last article about [How do I get a local Kubernetes cluster ready to use in seconds](https://torbentechblog.com/k3d-up-and-running-tutorial-linux/), you can jump here. I will wait for you.

## Development Flow

I will first explain the workflow graphically before we go into the details.

![Development Workflow](https://www.plantuml.com/plantuml/png/PO-nJiD038PtFuKtfaPAkZ6W3Z3m59Kvkwt5vIvoV210VNToEQcXmjl_-RDb7sOdyp96U8XoSlICfkUB8wj9SCq9A7WsPFcGc2Sn26IynUD8uQ99y0TmgH1pONpVposlHMT9ZZHD_NyqhEWA6tnzVcd9N4yK7D-AHZwEuiJaTDyBcGMkSBi6TxkdkW4ViU_mqzIbEVTB_cX3XoR4g6bsA-l7CzHMLUheukoxlbs1b7Y1oKcJc7xBpQmVLt50bYdchys2WoGkO_m5)

We as a developer simply write our code. When we're done with local unit testing, we instruct the IDE to build and tag our container according to our `build_metadata` (which will be generated automatically later!) and then deploy it to a local `k3d` Kubernetes cluster where `app1` and `app2` (dependencies - that might be a database, cache, backend service, ...) are already running.

After that, we can run tests against the whole environment - all that before we even push a commit to our repository.

With these tools we can find out how our software behaves, if it works in a cloud environment, before we had to pay a dime to a cloud provider and before our CI/CD had to start a single pipeline.

**This** is where _cloud native_ should start.

## How To setup and use the extensions

### Docker

Thankfully, the above extensions are very easy to understand. Once you've written your code, open the command palette and select "Docker: Add Docker Files to Workspace".

You will be guided through an interactive menu in which you can, among other things, specify your programming language and choose whether a Docker Compose file should be generated (which we don't need in our case).

Time for your first image build, isn't it? Open the command palette, choose "Docker Image: Build" and let yourself be guided by the command prompts. Shortly, your image will be built.

#### Your own container image registry!

When you followed the instructions of [Setting up k3d](https://torbentechblog.com/k3d-up-and-running-tutorial-linux/), you will need to apply a small change to your `k3d` `config.yml`:

```yaml
registries:
  create:
    name: registry.localhost
    host: "0.0.0.0"
    hostPort: "5555"
```

And another entry in your `/etc/hosts` file: `127.0.0.1 registry.localhost`.

When you apply the `k3d` config, you will be gifted with your very own container registry to which you can push your locally built images. The registry itself is running inside your `k3d` cluster and can be referenced to when deploying your applications to it.

#### Back to Docker

To push your local container image, open the command palette again and choose "Docker Images: Push". Your image will be pushed to your local image registry.

### Kubernetes

After that you need a suitable Kubernetes Manifest. At the end of [my last blog post](https://torbentechblog.com/k3d-up-and-running-tutorial-linux/) you will find an example deployment, a service and an ingress resource. Modify it to match your needs (change the name, port, URL, ...).

When you're done, select "Kubernetes" via the extension button on the left in your IDE and switch to the context of your k3d cluster.

Open the Kubernetes Manifest, then the command palette and select "Kubernetes: Apply".

There might be a small pop-up which wants to show you what exactly is going to be applied - you can certainly read through it - and then hit "Apply".

After that, when you run `kubectl get pods`, you should already see your application running.

Done!

## Summary

In this article, I showed you how a simple approach of _cloud native_ development can work:

- Setting up your IDE to be cloud native
- Configuring your `k3d` cluster to host an image registry
- Building and pushing container images
- Deploying your application to your local Kubernetes cluster
