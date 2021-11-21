---
layout: post
title: How Chaos Engineering May Save Your Infrastructure
categories: [Cloud]
---

A short story about how you can specifically inject errors into a running system - and how you will improve your system with it.

You have probably already developed a software artifact that you were afraid of being touched by the wrong developer afterwards. There is probably a component in the overall system in your project or company that no one likes to do support or operating. And most likely you are afraid that your system will fail or malfunction. Did you nod your head while reading? Then stick with it and read about how to build more confidence in your system - **by breaking it**.

## Prelude

**Netflix, LinkedIn, and probably dozens of other companies have done the trick: They destroy their systems frequently and continuously to make them more resilient.**

Netflix' Simian Army has taken big steps forward by developing various "monkeys". You can read that on the [Netflix TechBlog](https://netflixtechblog.com/the-netflix-simian-army-16e57fbab116). LinkedIn has also developed several tools that they write about in various articles, for example in the infoQ journal.

<div style="text-align: center;">
  <picture>
    <source srcset="/images/2021-10-24/01.webp" type="image/webp">
  </picture>
</div>

**When developing software, we are too afraid that something might break.** We trust the operations department or a dedicated team that takes care of architecture, infrastructure and day-to-day operations. That will be fine, the team will bend it. Well, that's where it's going to explode!

Werner Vogels, CTO at Amazon Web Services, put it aptly: **Everything fails, all the time.** If we take a look at the monitoring of our systems, we see what he means by this: We experience crashes every day. Pods are constantly exploding in our Kubernetes clusters, but Kubernetes takes care that they start up again, right? Our database is very shaky, we take a complete snapshot of it before each risky transaction (which is generally not a bad thing). It's just like that, we've got used to it.

{% include pullquote.html quote="It is high time that we come out of this ball pool of good weather development and start taking care of making our systems resistant." %}

## Start Breaking Stuff

**It is not difficult at all to get into Chaos Engineering.** The "hello world" of it is to simply push a component over the cliff. The shutdown tells us if the rest of our system is able to handle the situation and continue to operate normally.

All of our components are disposable and we have to live with them dying. The rest of our system has to gracefully handle it and continue operating - otherwise the customers will run away.

The churn rate of customers has been firmly anchored in our business metrics for ages, we calculate them and know that every missing customer makes us fluctuate. The competition is simply too big for such failures.

![code picture](images/2021-10-24/02.webp)

_But how can you make your system more stable?_ First, get an overview of the current status of the overall system. This is relatively easy if you already have a graphic or something similar. If not, make one. Your team, your department and everyone else involved will thank you! In the next step, you make a list of which components are critical (Tier-1) and which are non-critical (Tier-2). The Tier-2 components will be the ones we'll experiment with first. Do you have a test environment? Use it! Do you not have? Put one on! Nobody likes people who smash the castle with the sandpit shovel in the productive environment.

Once you've put your sandbox on, it's time to start thinking about what tests you want to do. We have already discussed the simple shutdown of components above. Here is a short list of some key points you should tap:

- shutdown of a component
- pause a component
- cause network latency
- cause packet loss in communication
- CPU & memory overload
- desynchronize system clock
- fork bombs

Do these ideas sound too simple for you? Think about your own experiments. To start, you can use the experiment description template at the bottom.

## Chaos Done Easily

By the way, if you are not sure how to inject errors into the system, there is a remedy. There is now a wide range of open source tools, for example the [Chaos Toolkit](https://chaostoolkit.org/), with which you can automatically create and carry out experiments on native systems, but also in the Docker and Kubernetes environment.

There are many other tools, I personally like to work with [Pumba](https://github.com/alexei-led/pumba) (docker-based chaos) and [Kube Monkey](https://github.com/asobti/kube-monkey) (Kubernetes-based chaos). But don't let that stop you from writing your own scripts. For CPU stressing and many many more, there are beautiful shell scripts out there that you can use as an idea and inject into your applications.

News: There's also whole Chaos-as-a-Service platforms available which are ready to startup e.g. in your Kubernetes clusters. They ship ready and with batteries included. Good examples for those platforms (free!) are [Litmus Chaos](https://litmuschaos.io/) and [Chaos Mesh](https://chaos-mesh.org/).

**Have you ever thought about how your system will behave if your database thinks it is in the Asia/Pyongyang time zone?**

Above all, it is important to choose a fixed target for your tests. Be it a database, a Docker container, a VM or an entire data center. That is entirely up to you and your requirements. At the beginning, you should keep the blast radius very small: Make sure that a component failure does not hurt, causes no data loss and is easy to re-roll. (Did I mention that you shouldn't be testing on prod in the beginning?)

## Improve

**Realize that simply destroying your component won't help anyone.** No maintainer will thank you if you have proven that his service is useless and unstable. It is therefore important that you analyze some useful data from the system during the experiment. These depend on the experiment you are performing and not every type of data set will be suitable for analyzing each experiment.

![code picture](images/2021-10-24/03.webp)

After you have identified suitable system and business metrics and can monitor them, it is time to establish a hypothesis. What can we expect how the system will behave during the experiment? Usually, assume that the system will continue to run normally. If the hypothesis is that the system crashes or trips over its own feet, you don't have the necessary confidence in your system. Did you predict a mistake in the hypothesis? First, take care that this error will not occur before you run the experiment. Experiments are not there to confirm that an error is occurring, but rather that a system is able to deal with an error.

Once the target has been identified, the hypothesis established, the experiment performed, and the blast radius as small as possible, it is time to conduct the experiment. Switch on the monitoring and carry out the action. Watch the events live and record the data that has arisen, they will be useful for the analysis afterwards.

## Analyze

If the experiment is finished because it is finished, had to be canceled or other events have occurred, the analysis begins. Has the hypothesis been confirmed? If not, why not? At least specialists for the component that was tested should be part of the team's initial analysis. Think about how the system can be improved to successfully complete the experiment in the next run. This is usually easier than expected. Did a fork bomb cause the VM below to say goodbye? Then you define a fixed number of processes for the component that it can start. Has the component turned out to be critical and must it not crash because the system can then no longer work? Then you should negotiate to activate high availability for the component and take other precautions.

**Now you have learned how to identify your target, describe your hypothesis, perform an action and analyze the whole thing afterwards. What are you waiting for? Start breaking your system and make it more resilient!**

Did I mention that there should be a small MarkDown template for a chaos experiment here?

```markdown
# EXPERIMENT TITLE

## 1. SHORT DESCRIPTION

** What is the initial state? **

** What action is being taken? **

** What are the effects of this action? **

## 2. HYPOTHESIS

** What do we expect, how the system will react? **

## 3. METRICS

** What data do we pull from the system to observe the experiment? **

** Which business-relevant data do we consider? **

## 4. NOTIFY

** Who needs to be informed before the experiment is carried out? **

## 5. IMPLEMENTATION

** How do we carry out the action? **

** How long does the experiment take? **

## 6. ANALYSIS

** What system and business data has been accumulated? **

** What do we conclude from the individual data sets? **

## 7. SCALING

** How can we roll out the experiment more broadly? **

** What other components can this experiment be applied to? **

** What other components is the experiment already applied to? **

## 8. AUTOMATION

** How can the experiment be automated? **
```

Note: You might've already stumbled across this article on Medium. The reason I repost it here is that you'll be able to read as many of my articles as you want to - while not seeing a Medium-paywall or having to wait to read more Medium stories.
