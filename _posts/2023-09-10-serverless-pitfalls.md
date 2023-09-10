---
layout: post
title: Avoiding Serverless Pitfalls
categories: [Development, Architecture]
---

When it comes to web architecture, composability is one of the most important things to achieve. Lets find out the good, the bad and the ugly of serverless.

## Intro

Lets start with one of the most important bullet points to take away. Repeat after me: **Serverless is not a replacement for microservices.**

Many believe serverless to just be Functions-as-a-Service, and in its *most simple* form this is certainly correct. But FaaS is only a subet of serverless - better known as serverless functions. Such functions are triggered by events, i.e. when users click a button on your website. Cloud service providers (AWS, GCP, Azure, ...) will then take care of provisioning infrastructure + software environments to run your code snippet in - literally everything you have to do is "write some code and deploy it".

Since the intro is not supposed to deep-dive into all overviews of serverless, I'll sum it up as: **The common characteristic of all serverless services is that cloud service providers take care of all the infrastructure elements so you don't have to.** This saves time, complexity and money while shifting your focus solely on delivering business value.

## The Good

The 'Pro' list of serverless can basically be summed up to:

- **Elimination of Server Management**: No need for server management, as serverless computing heavily relies on cloud providers to handle configuration and maintenance.
- **Potential Cost Savings**: Serverless offers various cost-saving benefits. You don't need to predict capacity and load, in most cases you can scale from 0 to infinity, with 0 being the most interesting part for your wallet.
- **Scalability**: Don't worry about traffic spikes bringing down your whole site or causing poor performance. Of course, costs will rise as your user base increases, but that's a nice problem to have, because when your user base decreases, so do your costs.
- **Security**: Caution, this is a double-edged sword. Many articles list security as a disadvantage in serverless context. For cloud providers, it is a key component of their business model to gain trust of their customers, and they can only achieve that by providing high-secure environments, making it hard for you to shoot yourself in the foot.
- **Time to Market**: Development environments are set up in minutes. That's especially critical for MVPs with the bonus of everything being decoupled, so you can add/remove services around it anytime.

## The Bad

- **Vendor lock-in**: Of *course* it is possible to pick and choose services from different vendors, but most of the time you will choose one of them and stay there becase every vendor has "their" way of doing things. This will become a challenge if you decide to migrate to a different provider, while you were relying on provider-specific techniques.
- **Cold Start**: Serverless computing is not constantly running. If it does in your case, ask yourself why. It always should scale down to zero. When a function is called for the first time, it requires a cold start which is to say a container needs to spin up before your function can run. Thanks to latest developments, these cold start scenarios have incredibly improved on AWS/GCP. If you're struggling with this, a pro-tip is to keep your code/container image as small as possible and minimize the use of external dependencies.
- **Debugging/Testing**: Debugging serverless is quite complex due to the reduced visibility of background tasks which are naturally managed by your cloud provider. Everything is a black box except your code.

## The Ugly

To be fair, most of the ugly cases show up when the architecture isn't thoroughly serverless. Anyway, those cases *can* and *will* happen and you will need to fix them.

- **Cost Explosions**: When you experience great traffic spikes, bad actors join the game with DDoS or your functions get caught in an event loop, the cloud provider's bill will turn into a nightmare in no time. I have had several cases where functions did not process their events correctly, had no dead letter queues defined, and then got caught in a loop. The same applies to data processing with GCP Dataflow or AWS Data Pipelines. Always do rehearsal with potentially problematic data before you move to production, and never ignore those cases. Also, you might be interested in building a trap which shuts your services down when they let your monthly bill explode.
- **Bad Performance**: Cold start, poor performance and high latencies can become a bottleneck at critical points. Your customers don't want to wait several seconds until their item is in the shopping cart. Your partners don't want to wait half an hour until their data is available on your marketplace. The list of case studies is long. In these scenarios, it can be useful to track the route your data takes. Can a critical functionality or two be refactored from code so you can scale it independently? Do you have synchronous queries that traverse multiple services before the user gets a response of any kind? Also try to identify any scalability boundaries between services and include downstream services into your analysis.
- **Loss of Control**: Loss of control can occur in so many places. To cover a few points, here are a few cases where loss of control might bother you. It's worth noting that in these cases, serverless is probably not the right architecture for you. If you expect constant and predictable load with a reliable load pattern, you don't want the provider to manage the servers and the instances of your functions. You should take it into your own hands to get more control over scaling and cost-effectiveness. The issue of loss of control also often comes up when moving older applications to a new infrastructure with a different architecture. Serverless rarely works in lift-and-shift scenarios. Serverless is built on many cloud providers running multiple containers of multiple different applications from many different customers on shared hardware. On the part of the provider, this is for reasons of cost efficiency, as the hardware can be optimally utilized in this way. From the customer's point of view, this can be a problem: Has the provider implemented multitenancy correctly? Am I correctly shielded from other customers, can data flow to other customers?
