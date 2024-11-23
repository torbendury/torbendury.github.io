---
layout: post
title: Working around Google Kubernetes Preemptible VM limitations
categories: [Development, Open Source, Public Cloud, Kubernetes]
---

In honor of an abandoned tool, I wrote [my own controller](https://github.com/torbendury/gke-preemptible-sniper) to deal with the technical limitation of preemptible VMs in Google Kubernetes Engine.

## Problem Statement

Google Kubernetes Engine (GKE) preemptible VMs offer very cost-effective solutions for running your container workloads, but they come with a significant drawback: After approximately 24 hours, they are automatically preempted and abruptly shut down. This can disrupt workloads, leading to unexpected downtime and bad customer experience. As organizations rely more on Kubernetes, dealing with such limitations is quite a common challenge.

One possible way could be to switch over to on-demand or spot VMs, however they do not offer the same price-performance ratio as preemptible VMs, which can in some cases save you up to 90% of your Compute Engine bill.

## Solution

[`gke-preemptible-sniper`](https://github.com/torbendury/gke-preemptible-sniper) to the rescue! This is a small controller I developed in my spare time, which mainly fulfills these requirements:

1. **Low resource consumption** - I don't want to implement a solution that eats up the savings from preemptible VMs
2. **Stability** - When it is deployed and configured, I don't want to have to do greater operation tasks. Minor updates are acceptable, however I want stable software that does its job and nothing more.
3. **Graceful shutdown of VMs** - This is the main technical requirement to fulfill.
4. **Trustworthy dependencies with regular updates** - Since this tool is a replacement for another one that isn't maintained anymore, I want to build something simple with the least amount of dependencies which are trustworthy.
5. **Observability** - The last tool I've been using didn't inform me about any action taken, neither logs nor metrics were supplied to the end user. I want to see what is happening and want to be able to show it to others.

## Implementation

[`gke-preemptible-sniper`](https://github.com/torbendury/gke-preemptible-sniper) is a simple but quite powerful Kubernetes application, designed to handle this exact challenges. By interacting with the Kubernetes API, the tool proactively marks preemptible VMs for graceful removal - including workloads - before they are forcibly being terminated by Google Cloud. This ensures minimal downtimes.

The tool continuously checks for all preemptible VMs running in the same cluster as itself in a configurable interval. For every node found, it calculates a timestamp - at least 12 hours from now, plus a random buffer. This ensures that each node in the cluster will be gracefully removed well before it hits the 24-hour mark, preventing unexpected terminations.

I wrote the application in Golang because of the great standard library features, the lovely libraries available for Kubernetes interaction and its low resource consumption as well as offering the most simple way of running tasks concurrently that I've ever seen.

## Low Resource Consumption

One of the requirements to fulfill is overall low resource consumption. I need such tools to provide maximum efficiency because in the end they serve one task only and this one should be served with as minimal impact as possible. This means that it should be able to start up without any waiting time, run basically forever without any performance impact on other applications on the same VM and keep the bill low and leave space for other applications.

All in all, the CPU and memory consumption is kept very low (about 0.0005 CPU cores and 120MiB memory), while most of the memory consumption is due to the Prometheus instrumentation.

### Container Image

With this in mind, I started out by running a full-blown Golang (1.20 as of time, recently updated to 1.23) image based on Debian on a local [Kind cluster](https://kind.sigs.k8s.io/) and saw that they took way too much space. The compressed size was already at around 300MB, the unpacked image on the node was even bigger. Even the oh-so-small Alpine-based images already were at around 70MB without(!) having my application binary in it. This is where `scratch` containers join the chat.

tl;dr: [Here](https://github.com/torbendury/gke-preemptible-sniper/blob/main/Dockerfile) is my Dockerfile.

So what are `scratch` images? `scratch` is **the** most minimal image in the world of Docker. It basically is the ancestor for all other images that exist. Actually, the `scratch` image itself is empty and does not contain any folders, files or operating systems. Operating system images as Debian or Alpine build - for example - their rootfs blobs and unpack them into those empty `scratch` images to have a clean environment ready for their OS. I wanted to leverage this empty image, so I put my application into it.

A quite important fact to understand and remember is that the `scratch` image really does not contain everything. If you build an application and it needs dependencies, you must bring them yourself. In terms of Golang, this is quite easy, because normally we produce binaries that already contain everything that our code needs to be able to run.

And how do we produce a binary that can be put into an empty base image? Multi-stage builds coming in. With multi-stage builds, we can use multiple `FROM` statements in our Dockerfiles. Each `FROM` instruction *can* use a different base, and each of them starts a new build stage. An important fact with multi-stage builds is that we can selectively copy artifacts from one stage to another and leave everything behind that we don't want in our final image. You might already smell it, but let's take apart [my Dockerfile](https://github.com/torbendury/gke-preemptible-sniper/blob/main/Dockerfile).

Everything starts out very basic:

```Dockerfile
ARG GOLANG_VERSION=1.23.2-alpine

# Dev Stage
FROM golang:${GOLANG_VERSION} AS dev
WORKDIR /app
CMD ["sh"]
```

I love to have a very clean environment which I can work with, and I love to work as near to the build environment as I can. This is why I provide myself a development environment called `dev` which is based on the Golang + OS version which my CI (GitHub Actions) is going to use.

To get my development environment running, I run:

```sh
docker build --target dev -t gke-preemptible-sniper:dev
docker run -ti --rm -v $(PWD):/app gke-preemptible-sniper:dev
```

With these two commands, I instruct `docker` to execute a build based on my Dockerfile and stop at the target `dev`. Right now, our Dockerfile only contains `dev`, but now we're going to get a `build` stage running which will be put directly under the code above:

```Dockerfile
FROM golang:${GOLANG_VERSION} AS build

ENV USER=appuser
ENV UID=10001

RUN adduser \
    --disabled-password \
    --gecos "" \
    --home "/nonexistent" \
    --shell "/sbin/nologin" \
    --no-create-home \
    --uid "${UID}" \
    "${USER}"

RUN apk update && apk add --no-cache git
WORKDIR /src
COPY . .
RUN go mod download && go mod verify
RUN mkdir /app && CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags="-w -s" -o /app/app cmd/main.go
```

In this `build` stage, I create a build environment which has its own least-privileged user and updates the system before anything else happens. After that, we download and verify any Golang modules and start our build:

```sh
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags="-w -s" -o /app/app cmd/main.go
```

`CGO_ENABLED=0` disabled the use of Cgo, which is good for cross-compilation, portability, compile time, security and safety reasons. Also, it is critical for our goal to create a self-sufficient binary that is statically linked with all dependencies needed. I plan to run the binary in a Linux 64 bit environment, which is why `GOOS=linux GOARCH=amd64` is set. Also, we pass some settings to the linker, allowing us to directly control the linking process. `-w` tells the linker to omit the DWARD symbol table. This is usually quite nice for debugging, but since we want to build releasable binaries, we might also exclude it and save some symbols. `-s` omits the symbol table as well as the string table - again, quite nice for debugging purposes, but I don't need it in a release binary which has to fulfill the goal of minimum size (Also, it makes it harder to reverse engineer the binary. In my case this is not interesting since the code can be found on GitHub, but you might be interested in it when you distribute binaries commercially).

If we now build our container, we have a runnable version of our application ready:

```sh
docker build --target build -t gke-preemptible-sniper:build
docker run -ti --rm -v $(PWD):/app gke-preemptible-sniper:build
```

But there's still one part missing - we still have the full Alpine OS underneath! Lets get rid of it:

```Dockerfile
# Release Stage
FROM scratch AS release

COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=build /etc/passwd /etc/passwd
COPY --from=build /etc/group /etc/group

USER appuser:appuser

COPY --from=build /app/app /go/bin/gke-preemptible-sniper
ENTRYPOINT ["/go/bin/gke-preemptible-sniper"]
```

Now here is where the magic happens. You remember that `scratch` containers contain absolutely nothing, and now we need to fill it with the necessary things:

- `ca-certificates.crt` for a set of root certificates that allow our application to validate SSL connections. Since we're going to call the Kubernetes API as well as the Google Cloud API (for Compute Engine), we need those root certificates for secure HTTPS connections.
- `/etc/passwd` and `/etc/group` to ensure the system in the release image has necessary group and user information. Our `USER` directive specifies as whom the application inside the image is going to run. If this information wouldn't be reflected by the system itself, the directive would fail.
- `/app/app` which is our statically linked binary that contains our release application.

It is built by the CI with the target `release` and pushed to DockerHub afterwards.

## Stability

Kubernetes environments and especially the ones with preemptible or spot VMs can become turbulent, and thus errors can happen. In the case of `gke-preemptible-sniper`, errors can happen at multiple places.

- Kubernetes API misbehaving
  - Pod Eviction not happening
  - Nodes not being retrieved / annotated / cordoned / drained / removed
- Google Compute Engine API misbehaving
  - VMs not being deleted properly

And those are only the specific errors, not even taking into account that network is always going to be faulty at some point and many more. This is where the application keeps track of an error budget.

For every loop cycle - checking nodes, draining and deleting them, updating metrics ... - an error budget is calculated. There's an [initial error budget of 10](https://github.com/torbendury/gke-preemptible-sniper/blob/main/cmd/main.go#L41). Why? Because that's how many errors I wanted to be allowed before I think somethings' horribly wrong. I also could've taken 11 or 42.

For every error that happens, the error budget decreases. If the error budget reaches 0, the application halts and gets itself some fresh air before starting the reconciliation loop again. After every successful loop cycle, the error budget is increased up to its maximum. This allows for *some* errors to happen, while utilizing Kubernetes readiness / liveness probes make sure that `gke-preemptible-sniper` is being restarted automatically if the situation goes south.

If a node can't be processed correctly, that's not a problem. The logic behind the `gke-preemptible-sniper` is to look if the timestamp of an annotated node is in the past, it does not care how far back in the past it is. Thus, in the next loop cycle, the `gke-preemptible-sniper` is going to automatically retry the node without any additional logic needed. I might call this passive stateless retry and put a patent on it. Stay tuned.

## Graceful Shutdown

Kubernetes workloads deserve to die a dignified death. So we're going to dig a little deeper into the Kubernetes bag of tricks and use the [Eviction API](https://kubernetes.io/docs/concepts/scheduling-eviction/api-eviction/). This is actually mainly used by Kubernetes itself to relieve nodes when workloads require too much CPU or RAM and the node can no longer cope. When these spikes are detected, Kubernetes itself takes care of the health of the node and uses the Eviction API to terminate the pod.

We can also make personal use of this and [call the Eviction API ourselves](https://github.com/torbendury/gke-preemptible-sniper/blob/main/k8s/k8s.go#L138). This allows us to let Kubernetes do its work and have a good time ourselves.

However, in some cases, Pod eviction does not work out on itself. These include the following reasons:

- `PodDisruptionBudgets` (PDB) - if a Pod has a PDB configured, eviction might not be able to be performed if the disruption (removing one Pod) would cause the number of total available replicas to drop below the minimum specified in the PDB. If you run into one of those cases in a normal everyday scenario (no other disruptions happening at the time being, e.g. updates being deployed), you might need to adjust your PDB to match realistic criteria. With PDBs, you can configure a minimum number (or percentage) of Pods that should always be available. If you have 3 Pods of your Deployment running, and a configured `minAvailable` of `80%`, you won't be able to remove even a single Pod of those three because it would cause the available replicas to drop below `80%` of the wanted 3 replicas.
- `DaemonSet` Pods - Pods created by DaemonSets are typically used to run essential services on all nodes in the cluster (networking, logging, monitoring, ... you name it) - such Pods are generally not evicted. If you want to get rid of them, you need to call the Delete API.
- Pods with `emptyDir` volume - Kubernetes typically evicts Pods primarily to manage resources or handle failures. When evicting a Pod with an `emptyDir` volume, the system has to ensure that is doesn't leave any side effects that might compromise stability.

You can see this in reallife when you try to run `kubectl drain <nodeName>` on a node which has Pods running that fulfill above criteria. If you want to play around with draining nodes, you might be interested in the `--disable-eviction` flag. It forces the draining process to use the Delete API, even if Eviction would work out. This would at least bypass the checking of PDBs.

As soon as no more application Pods are running on the node, we are safe to remove the Kubernetes node itself. Since GKE does not automatically register that the Kubernetes cluster does no longer have the node in its registry, we have to help out by removing the Google Compute Engine (GCE) instance via the Compute Engine API.

Tip: You might experience that GCP provisions a node after this process which has the exact same name. According to GCP Support and Technical Account Managers, this is intended behavior when deleting preemptible machines. It's not dangerous, however it might make metrics and monitoring look weird so watch out for it.

## Ease of Deployment

To deploy `gke-preemptible-sniper`, I went with the most straightforward and least error-prone way I could imagine: Releasing and deploying it via [Helm](https://helm.sh/).

The release process is quite straightforward and in my case completely free. I use [GitHub Pages](https://pages.github.com/) for it. It is a service by GitHub intended to make content for programming projects available as a web page. This includes blogs (like the one you're reading right now!), technical documentation - and release pages.

Since a [Helm repository](https://helm.sh/docs/topics/chart_repository/) is (simplified) a file server that serves a repository `index.yaml` which contains all versions of a Helm Chart and the links at which you can download them, it's quite easy to setup. In my case, I have a separate branch on the repository called `gh-pages` to which this `index.yaml` is being pushed by GitHub Actions. See the `index.yaml` [here](https://github.com/torbendury/gke-preemptible-sniper/blob/gh-pages/index.yaml) and the GitHub Action [here](https://github.com/torbendury/gke-preemptible-sniper/blob/main/.github/workflows/release.yml#L74).

The Helm Chart itself can be deployed by following the [README.md](https://github.com/torbendury/gke-preemptible-sniper/tree/main?tab=readme-ov-file#helm) which mainly consists of

1) adding the Helm repository
2) Creating a custom `values.yaml` to override necessary values
3) Installing the Helm Chart

Users are, of course, not limited to installing this manually, but can integrate it into their CI to be deployed and managed automatically (ArgoCD and Flux entering the chat).

## Trustworthy Dependencies

`gke-preemptible-sniper` is built on top of three core dependencies:

1) [kubernetes/client-go](https://github.com/kubernetes/client-go) which is the official Kubernetes Golang client library. It is a great layer in between applications and Kubernetes cluster API servers, takes care of authentication and secure communication and offers many Golang-native features to be able to work concurrently with Kubernetes.
2) [prometheus/client_golang](https://github.com/prometheus/client_golang) which is the official Prometheus Golang client library. It is made up of two separate libraries: One for instrumenting application code, and another one for creating client which then talk to the Prometheus API. If you use `net/http` or a compatible HTTP server implementation, `promhttp` really is your friend because it natively integrates with it and offers a HTTP handler func which makes offering a `/metrics` endpoint [require one line of code](https://github.com/torbendury/gke-preemptible-sniper/blob/main/cmd/main.go#L144) in the end. Also, `promauto` really gets you on the fast lane because it takes great care of provisioning metrics and the data objects needed behind it, taking care of your metric registry and makes it really easy to update metrics regularly. You can have a look at [my botchy implementation](https://github.com/torbendury/gke-preemptible-sniper/blob/main/stats/stats.go) to get a grasp on it.
3) [googleapis/google-cloud-go](https://github.com/googleapis/google-cloud-go) which contains a truckload of client libraries that enable you to talk to Google Cloud services. I use it to [talk to the Compute Engine API](https://github.com/torbendury/gke-preemptible-sniper/blob/main/gcloud/gcloud.go) and had a blast on how easy this was in the end. The client libraries might look a bit overwhelming and heavy weight at the first glance, but you're free to pick up just the features you need and leave the rest being.

As those three projects are maintained just perfectly fine, come directly from the product authors and have a great community backing, I can make sure that I will be able to contribute `gke-preemptible-sniper` for a very long time. I didn't need to wrap my head around hacking through API documentations since for everything I needed there are hardened and Go-"native" client libraries ready to plug in. This also saved me lots of time because especially the Google Cloud APIs tend to change a lot on customer facing side.

## Observability

One thing that bugs me is applications that are supposed to do something but don't tell me *what* they're doing exactly and which steps they have to take to get there. Long story short: We need more observable applications out there.

I want to contribute to observable applications, so I went with the most low hanging fruits that were available right away: Logging structured information in a structured format using the Golang standard library `slog` logger together with a JSON formatter, and instrumenting my application with Prometheus which still remains one of the most widely used monitoring tools for metrics, particularly in cloud-native and Kubernetes environments.

Setting up the `slog` logger is done quite simply:

```go
// ...
logger = slog.New(slog.NewJSONHandler(os.Stdout, nil))
// ...
logger.Info("node has time to live left", "node", node, "left", fmt.Sprintf("%vh%vm", int(duration.Hours()), int(duration.Minutes())%60))
// ...
```

And allows for a nicely structured log output which can be consumed by a logging stack of your choice.

As I described above, I instrumented metrics using the Prometheus client library. I know that OpenTelemetry is gaining some real traction as a unified framework for metrics, logs and traces which is very attractive for organizations adopting distributed architectures (including microservices, serverless, ...), I simply didn't have a stack ready to use and integrate with OpenTelemetry. Nevertheless, if you're developing an application and want to integrate those observability information into a single glass of pane, you should have a look at the OpenTelemetry instrumentation library for Golang which works just great.

I offer two metrics, one which takes count of the nodes that will be sniped within the next hour, and one which takes count of the nodes that have been sniped within the last hour. Since `gke-preemptible-sniper` is intended to run inside Kubernetes, I included batteries into the Helm Chart to enable metric scraping by Kubernetes-native Prometheus installations [here](https://github.com/torbendury/gke-preemptible-sniper/blob/main/helm/gke-preemptible-sniper/templates/monitoring.yaml). You might notice that a `PodMonitoring` as well as a `ServiceMonitor` are available, which is because Google Cloud offers a [Managed Prometheus](https://cloud.google.com/stackdriver/docs/managed-prometheus/setup-managed?hl=en) installation with custom resources that do not match the original Prometheus operator API and thus need some extra configuration. Thanks for that, Google Cloud!

## Conclusion

`gke-preemptible-sniper` was a fun project to get going and I really learned a lot more about the Kubernetes API and Prometheus instrumentation. I learn best by building real worl applicable projects, and it's great that I was able to solve a real problem and see my application running in production seamlessly. Of course, my Golang skills are not expert level and some parts of the code might spaghetti to more sophisticated developers, but I don't develop applications on my everyday job and I'm just starting out getting more knowledge on actual coding, so have mercy :-) I'm always open to learn more, so if you find any bugs or have any tips for me to restructuring and improving `gke-preemptible-sniper`, I'm always happy to hear from you! Open a GitHub Issue and let's discuss a bit!
