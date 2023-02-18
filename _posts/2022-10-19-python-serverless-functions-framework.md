---
layout: post
title: Using the Python functions-framework for serverless development
categories: [Development, Python, Public Cloud]
---

The `functions-framework` is a serverless framework published and maintained by the Google Cloud Functions team.
It is **free and publicly available** for everyone on [GitHub](https://github.com/GoogleCloudPlatform/functions-framework-python).

## Use Cases

You can use it for rapid development of lightweight and portable Python functions that (do not only) run on Google Cloud Function environments. It also runs on any Knative-based environments as well as Cloud Run and of course Cloud Run for GKE/Anthos. The best of it? It also **runs on your local machine like it would run on the public cloud**, including HTTP traffic as well as **event-driven functions**.

## Quickstart

For local development, it is recommended that you install the `functions-framework` package with pip:

```python
pip install functions-framework
```

Or add the following to your `requirements.txt`:

```text
functions-framework==3.*
```

**Note:** For Google Cloud Functions, you do not need to specify the above line in your `requirements.txt`.

### HTTP Example

Let's start with a simple HTTP example. Serverless functions typically have an entrypoint that is defined when deploying the function. **Note** that there are patterns where your entrypoint is only a relay function that parses requests to other nested functions.

The *Hello World* of the functions-framework is as easy as:

```python
import functions_framework

@functions_framework.http
def entrypoint(request):
    return "Hello world!"
```

Save this code snippet as `main.py`, spin up a terminal and type:

```bash
functions-framework --target entrypoint --debug
```

And start making requests:

```bash
curl localhost:8080/
```

You can also define **different routes** for your functions or handle request headers and body handed over as a `request` object.

### (Cloud) Event Example

A big plus for the `functions-framework` is that it **provides a decorator for reacting to events**. Event-driven architectures are being advertised and developed and find a wider audience than they used to, so with the `functions-framework` you are perfectly prepared for implementing event-driven approaches.

The `functions-framework` also implements the [CloudEvents](https://cloudevents.io) specification which also is a big plus.

To consume an event, take this quickstart example:

```python
import functions_framework

@functions_framework.cloud_event
def entrypoint(cloud_event):
   print(f"Received event with id {cloud_event['id']} and data {cloud_event.data}")
```

Run your application with `functions-framework –target=entrypoint` and push an event:

```bash
curl -X POST localhost:8080 \
   -H "Content-Type: application/cloudevents+json" \
   -d '{
    "specversion" : "1.0",
    "type" : "torben.techblog.cloud.event",
    "source" : "https://torbentechblog.com/cloudevents/pull",
    "subject" : "sensor_event",
    "id" : "A234-1234-1234",
    "time" : "2022-10-19T13:37:00Z",
    "data" : "31.2"
}'
```

Your function then should log the following line: `Received event with id A234-1234-1234 and data 31.2`

#### Emulating event platforms

Using `curl` for very small tests while developing your function might be suitable for you, but most of the time you will want a **full emulation** of the cloud platform your code will be deployed to.

Let's assume that you want to deploy your function to Google Cloud Functions and it should listen to Google Pub/Sub events. Pub/Sub does not enforce CloudEvents specifications, so neither will we.

Write a function like this:

```python
def entrypoint(event, context):
     print(f"Received {context.event_id}")
```

And start it locally like the other functions.

In a second terminal, you can use the `gcloud` CLI to start a local Pub/Sub emulator (Note that it does not fully implement everything you get with Pub/Sub):

```bash
export PUBSUB_PROJECT_ID=torbentechblog
# 8080 is already taken by functions-framework
gcloud beta emulators pubsub start \
    --project=$PUBSUB_PROJECT_ID \
    --host-port=localhost:8085
```

And begin sending messages to the emulated Pub/Sub instance. If you need help with pushing messages, have a look at the [official documentation](https://cloud.google.com/pubsub/docs/publish-receive-messages-client-library#publish_messages) that has a *very* good quickstart prepared for you.

## Conclusion

The `functions-framework` by Google Cloud Functions team is still pretty young in the framework world, so there might be big changes in the future that you will or will not like. The team itself says that they do not want to be another abstraction layer for Flask applications (although the framework relies on Flask under the hood), so the framework does not implement the full feature set of Flask and the team is not planning to do so.

Still, the `functions-framework` is a very interesting framework that can really enhance your development speed when it comes to serverless functions. The way it provides you a stable base for starting your development really leaves the standard hassle behind.

In near future, I will also write a post for testing and implementing error handling on your `functions-framework` code, so stay tuned for more!
