---
layout: post
title: Implementing Istio Retries Without Headaches
categories: [Development, Istio, Kubernetes]
---

Stop wrapping your head around complex retry logic. Have a look at the batteries included.

## Intro

Any application which communicates with other systems over a network must be resilient to failures. This sounds pretty funny until you are on-call and find an application in your stack at 03:00 AM which is *not* resilient in any kind of way, and you only found it because it exploded with such a blast radius that your bells started ringing.

Implementing resilience *can* be hard sometimes, because you can not - for example - simply retry everything in the real world and you will most probably hurt peoples' feelings if you do.
Of course, there are many other mechanisms apart retries that can be implemented to achieve resilience, but retries are one thing that can be implemented without headaches, and without even touching your beloved code base: Istio to the rescue.

### Istio

If you have a blog post ready definition for describing a service mesh, hit me up. Istio itself describes itself as a Kubernetes extension to establish a programmable, application-aware network. Doesn't spark joy immediately? Wait until your installation (via Helm) has completed and start having fun with the real stuff.

Since Istio is just an extension to Kubernetes, it mostly stays out of your way when you don't explicitly utilize it, and if you want, it brings you standardized traffic management, out-of-the-box telemetry (logs, metrics, traces) and several security features. Today we're going to look at retrying HTTP requests using Istio, but in the future I'll probably also note down some other nitty gritty stuff.

## Retries

Retries as a pattern aren't new, however with the popular world of microservices, software defined networks and Kubernetes clusters with limited-lifetime nodes, network failures are always present. If it, for once, isn't DNS, it is going to be "something network related". So why not just retry your last request?

The use case for retries is mostly in the context of I/O operations and quite intrinsic to the world of distributed computing (where network communicatino is inherently unreliable). I am going to repeat this for the purpose of trauma healing: Network communication is unreliable. Be prepared for it. It is going to fail at sunday 03:00 AM (when your colleague who made a 2000 lines-of-code change before has decided to be on 3 months sabbatical).

When implementing Istio retries, we utilize [VirtualServices](https://istio.io/latest/docs/reference/config/networking/virtual-service/). VirtualServices are a component used for traffic routing. You can manipulate traffic in many kind of ways, for example by re-routing it under certain conditions, inspecting and setting HTTP headers, HTTP matching and many more. What's especially tasty is the [HTTPRetry](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPRetry). This is the original example taken from the latest Istio documentation as of now, but its also the easiest one to explain:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings.prod.svc.cluster.local
  http:
  - route:
    - destination:
        host: ratings.prod.svc.cluster.local
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: gateway-error,connect-failure,refused-stream
```

In this case, if a service inside Kubernetes calls `ratings.prod.svc.cluster.local`, the Istio sidecar proxy is going to try the request `3` times before delivering an error to the client. For every attempt, the sidecar is going to wait `2` seconds before giving up. Requests will be retried if they failed to connect in any kind of way or received a `503` HTTP status code.

The conditions at which a retry is wanted might be different for you, you better look them up at the [Envoy proxy documentation](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/router_filter#x-envoy-retry-on).

> `gateway-error,connect-failure,refused-stream` is a robust example, because it only retries requests if they did not make it to the backend at all. If your backend receives the request, tries to process it and then responds with a HTTP 503, read [RFC 7231](https://datatracker.ietf.org/doc/html/rfc7231#section-6.6.4).

Okay, we have configurable retries for free, using a small YAML snippet. But isn't this a bit coarse? Let's jump back to the docs!

[HTTPRoute](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPRoute), specificly [HTTPMatchRequest](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPMatchRequest) allows configuring a quite exact scenario under which our retry should be executed. We can define a whole lot of criteria which must be met in order for our traffic rule (retries!) to be applied. For example, the following restricts the rule to match only requests where the URL path starts with `/ratings/v2/` and the request contains a custom end-user header with value `jason`:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings.prod.svc.cluster.local
  http:
  - match:
    - headers: # expect an exact request header
        end-user:
          exact: jason
      uri:
        prefix: "/ratings/v2/" # expect an exact URL path prefix
      ignoreUriCase: true
    route:
    - destination:
        host: ratings.prod.svc.cluster.local
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: gateway-error,connect-failure,refused-stream
```

And further squeezing out the `HTTPMatchRequest` we could further limit this to only be applied to `GET` request (which maniac does automatic retries for `POST` anyway?).

## Conclusion

Using `VirtualServices` (and having the Istio documentation at hand) helps us implementing simple retry logic without touching and re-deploying any kind of business logic. When you're running Kubernetes clusters with many different applications developed by many different teams, you will get to love features like this, because you can ship them fast without needing to touch anything on application-side.

Side note: Do not activate feature like this without talking to anyone. It might lead to confusion, production outages and ringing bells at 03:00 AM, closely followed by unlimited leave including a promotion to the new position "customer".
