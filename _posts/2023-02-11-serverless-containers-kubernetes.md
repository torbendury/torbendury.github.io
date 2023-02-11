---
layout: post
title: Serverless, Containers, Kubernetes. An Odyssey
categories: [Architecture, Cloud, Development]
---

Which one do you need to choose? What do those technologies have in common, which drawbacks do they have? A distillate from the buzzword-driven world.

## Introduction

Since you came across this article, I'm assuming that you've already dealt with these technologies, at least superficially. Therefore, I will not go into this shallow surface any further.

I will therefore go into the technologies from the middle onwards and go into a bit more depth on various points.

Container apps are all the rage these days. They are supposed to simplify **building, distributing, deploying and operating** applications. The main takeaway, however, is that they are 100% **portable**, since they already come with all the dependencies you need as an application at runtime.

Kubernetes is the *home* of containers. The technology takes away a lot of the classic day two operations that you have to expect when dealing with containers: Rolling deployments, high availability, upscaling and outscaling.

Serverless is a hot commodity right now, there's a lot of buzz around it but few are taking advantage of the real opportunities and often just picking up the drawbacks of it. Serverless from a developer's point of view means that you no longer have to worry about the build and operating process. A framework or platform takes care of that for the developer. Serverless functions can be rolled out quickly, because the underlying platform ensures the building of the application, the resolution of dependencies and the operation.

This post is suitable for groups of people who have already dealt with both application operating options and are now walking the tightrope of deciding between the two for applications X and Y.

## Equality Promotes Simplicity: A Short Ode To Containers

Why have containers become so popular in the first place?

One of the main reasons why not only developers but also operating teams are turning to containers is that they enable the operation of heterogeneous software in part in the first place. Whether you are running a classic web server or a complex application with business logic, a cache, a database - as long as it is available as a container, it can be run like any other. Equality promotes simplicity.

Another practical advantage is the portability of containers. You don't tie yourself to a particular hyperscaler, but can switch back and forth at will, or even take a completely hybrid approach. In addition, on-premise operation on bare metal is also possible due to the FOSS state of Kubernetes.

Where we used to need dozens of people just to run and maintain servers, today all we need is a small agile team to deal with the platform itself and enable easy and smooth deployment of containers.

This is also one of the main reasons why serverless computing is becoming more popular - and that's the dangerous part.

Ill-considered decisions lead to every application being described and operated as a serverless function. But not every application is suitable for serverless operation.

## When Serverless?

The operation of business logic as a serverless function is always a case-by-case decision.

What sounds charming in any case is that with classic serverless operation in the public cloud you do not pay for idle resources. When a function is not needed, it is automatically shut down by the cloud provider and costs nothing.

During operation, you pay for the invocation of the function itself, for the CPU/memory time it needs to do its work, and for data transferred via the network.

It can take some time for ROI to occur when refactoring existing applications to serverless functions. A permanently running container sounds like a waste of resources at first, but since it often runs in an existing ecosystem and actually requires few resources in the idle state, it may even be more cost-effective than restructuring to a serverless function.

Serverless is a good way to get up to speed rapidly and drive new developments. If we don't know how often a business logic is needed, we can take advantage of the main benefit of hyperscalers: scaling to quasi-infinity. Even if we need to run a logic, e.g. short-lived data processing, once, serverless can be an advantage. Here, the point that we only need to worry about the code comes into play again. The possibly higher costs for compute resources and data transfer quickly amortize the otherwise high development costs.

Serverless can be used in countless ways. Be it for a fast implementation of middleware to let two APIs communicate with each other, recurring short-lived data processing, event-driven architectures or completely asynchronous running processes that are not time-critical.

However, you also lose a lot of control over the underlying platforms and resources. Serverless is currently one of the highest levels of abstraction at which we can combine business logic with custom development. This is a very desirable side effect with serverless, but from a developer's point of view it must always be taken into account that general-purpose hardware is to be expected.

We shouldn't have any special hardware requirements when we're on serverless platforms. With serverless functions, CPU and memory are often linked and you can only select one of the two resources. In addition, serverless platforms do not offer any persistence to the functions. In many cases, this is even desirable, since the workloads that are operated there should be stateless, but you also have to consider this decision when thinking about a serverless architecture.

If you write and operate your serverless functions, you should also keep in mind that it is not necessarily possible to deploy the application 1:1 on another hyperscaler. The technical differences are often subtle, but they should still be taken into account, as they can mean additional work in a future-proof setup with the option of failover or hybrid operation.

To sum this up in short bullet points, you should at least ask yourself these questions when operating serverless:

- Do I have special **hardware requirements**?
- Do I need a **persistence**?
- Will **cold start** times cause problems for me?
- Does my code require a special **runtime environment**?
- Does my code react to **events** or to **direct calls** via special protocols?
- How do I handle **errors**?
- How much **data** will my code process?
- **How long** will my code take for an invocation? How often is my code actually called?
- Can my code do a **more general job**?

## When Containers on Kubernetes?

Kubernetes is a FOSS platform for simplified operation of any containerized software. It has an **extremely** broad community and it has become hard to find software that isn't offered in containerized form.

Still, it's also a *strategic and architectural decision* that I have to make when looking at the platform.

Do I have enough workloads running on the Kubernetes cluster to make running and deploying an operations team worthwhile? Do I need the option of always running a Kubernetes cluster with another cloud provider or even on-premise, or do I run the risk of a lock-in?

How do I implement high security standards and encryption of data at rest and in motion? How do I handle multiple Kubernetes clusters? Can I see them as a fleet and put a common umbrella over them, or are they self-contained systems?

Does a microservice architecture possibly fit into my system image or am I trying to pack fat monoliths into containers to apparently simplify operation?

As you can see, many questions arise when taking on the burden of Kubernetes clusters. Here, too, it is always a case-by-case decision as to which path to take, I can only give you a few questions that you should ask yourself and give plenty of thought to.

## Conclusion

There is no simple decision tree where you can make a few turns and then make the right decision between operating software as a container or as a serverless function. You always have to consider many points, some of which I have mentioned here as neutrally as possible, and you can either put your foot in it or find the holy grail with both decisions.

When making architectural decisions about containers and serverless, we always have to ask ourselves many questions (and not just once!) and in the end find the best possible way.

What you should at least take away from this post is not to blindly fall for colorful and shiny marketing slides which try to sell you something.
