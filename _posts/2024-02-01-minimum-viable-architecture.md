---
layout: post
title: Minimum Viable Architecture
categories: [Software Architecture]
---

A minimum viable architecture is a key part of a MVP for evaluating technical innovation and making sure that a product can grow.

## Intro

Building a minimum viable product provides great speed for product teams when evaluating new technical solutions. It helps product teams focus on delivering a working solution in such an early state that they can gain faster feedback from customers and test out new market possibilities without a project becoming too expensive from the ground up.

But a MVP is very useful way beyond the start of a project. When thought, planned and built out correctly, it will save time, money and effort on shaping implementations that fulfill actual requirements of customers.

## MVPs need MVAs

Product teams focus on implementing a MVP as fast as possible to gain knowledge early. In early product phases, they do not know about any measurable quality attributes and need a simple and minimalistic design of which makes their components work together. This early goal is reached by designing a minimum viable architecture.

Early design decisions are almost always tactical, because they solve a minimal set of problems. It's almost impossible to make good strategical design decisions at that point and will often prove wrong - because you do not know yet what you are building exactly.

A MVA should be expected to be reworked when the implemented MVP turns out to be successful and evolve further. This can result in significant refactoring of the initial design which balances out with the initial speed gained by a MVA.

## How much is enough

Customers care about the sustainability of products, which transitively makes them care about architecture. There are multiple questions to (not) answer and multiple architectural decisions to (not) do. If making - or not making - a decision affects the product in its viability and its sustainability, the decision is part of a MVA.

## Typical characteristics

When making decisions about a minimum viable architecture, an architect typically has to respond to those characteristics and adjust the architecture if needed:

1. Concurrency needs - this is related to the number of users/devices/... which the product will need to handle at once.
2. Data throughput - meaning the volume of data and/or transactions over a specified time period which the product must be able to process.
3. Latency - how quickly does the product need to respond to events?
4. Scalability - increasing the cost of a system to handle increased workload. Is the product able of handling this scenario?
5. Persistence - must the product store any data? Which data storage technologies respond to those needs?
6. Security - how will unauthorized access be denied? Is the data confidential? What about integrity and availability?
7. Monitoring - Does the product need to be instrumented in a way that you can see failure and prevent system issues?
8. Interfaces - How is the product going to communicate with its users?

## Conclusion

A MVP is not a throw-away prototype. It is the first release of a product. With more iterations, not only a MVP, but also its architecture, the MVA, evolves.

Only develop just enough architecture to exactly meet the requirements. Do not overengineer. Strive for a continous architecture which evolves over time.

Following the continous architecture principles, delay design decisions until they are absolutely necessary. Design which is not needed right now may not be needed at all and is going to constraint further design evolution.

Make architectural decisions as transparent as possible. This makes the product team and everyone around them better understand why certain choices have (not) been made.
