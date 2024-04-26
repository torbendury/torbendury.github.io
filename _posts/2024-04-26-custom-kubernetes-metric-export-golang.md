---
layout: post
title: Exporting Custom Metrics vom Kubernetes using Golang
categories: [Development, Golang, Kubernetes]
---

Introducing `kube-node-status`, because I needed it and because I can

## Intro

What is my problem? What am I trying to achieve? How am I doing that?

In Google Cloud Platform, I'm running a [managed Kubernetes cluster](https://cloud.google.com/kubernetes-engine) and recently switched to a [managed version of Prometheus](https://cloud.google.com/stackdriver/docs/managed-prometheus) for scraping metrics from inside the cluster and saving it to Cloud Monitoring. So far, so expensive.

Google-Managed Prometheus also offers taking care of your [kube-state-metrics](https://cloud.google.com/stackdriver/docs/managed-prometheus/exporters/kube_state_metrics) installation, which sounds fun first, however Google also decided they know best [which components managed kube-state-metrics](https://cloud.google.com/kubernetes-engine/docs/how-to/configure-metrics#enable-ksm) is able to monitor and which you can't enable as a customer. Guess what, they don't think node metrics are interesting for you. Bummer.

However, since I'm here for learning and always ready to MacGyver some small solution, let me introduce [kube-node-status](https://github.com/torbendury/kube-node-status/)

### kube-node-status

`kube-node-status` is a containerized Golang application which continuously reports the status of Kubernetes nodes.

I wrote it for several reasons, including:

- Not having to switch back to a completely self-managed monitoring setup,
- being able to monitor my Kubernetes nodes,
- learning how to talk to the Kubernetes API and exposing Prometheus metrics myself.

If you're only feeling the first two reasons, go [here](https://github.com/torbendury/kube-node-status/), install it and live a peaceful life.

## Calling the Kubernetes API from inside Kubernetes

The heading should rather read "calling the Kubernetes API from *Golang*", because the "from inside Kubernetes" part is quite easy. But hold up.

When searching around, I found several articles trying to explain how to call the Kubernetes API. [This here](https://iximiuz.com/en/posts/kubernetes-api-go-types-and-common-machinery/) is very well-written and explains `k8s.io/api` package (only containing data structures, boring but useful for deep digging), `k8s.io/apimachinery` package (a bit more interesting and serves as a dependency for Kubernetes server as well as client implementations without any direct type dependencies), but **most interestingly** it introduces `k8s.io/client-go` - which, in the end, fulfilled my need to "get shit done".

You can find the package needed at [github.com/kubernetes/client-go](https://github.com/kubernetes/client-go) which is more or less a plug-and-play module for Kubernetes API interactions.

A very minimal usage (without any error handling) looks like this:

```go
import (
    "context"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/rest"
)

func main() {
    config, _ := rest.InClusterConfig()
    clientSet, _ := kubernetes.NewForConfig(config)
    kubernetesNodes, _ := clientSet.CoreV1().Nodes().List(context.TODO(), metav1.ListOptions{})
}
```

`List()` returns a pointer to a list of Kubernetes nodes and their spec and status which you can iterate on:

```go
// Iterate over all retrieved nodes and check for their readiness
for _, node := range nodes.Items {
    for _, condition := range nodes.Status.Conditions {
        if condition.Type == "Ready" {
            fmt.Println(condition.Status)
        }
    }
}
```

Exactly what I needed. Remember how I talked about calling the API "from inside Kubernetes" is easy? It was done in the example above by calling `rest.InClusterConfig()` which automatically tries to use the runtime `ServiceAccount` of your workload running in a Kubernetes cluster. If your `ServiceAccount` has the [permission to list nodes](https://github.com/torbendury/kube-node-status/blob/394d68e73bcfe49be96065e6159603d7537d9b47/helm/kube-node-status/templates/clusterrole.yaml), you're already done with that job. Just put in some error handling before you use this in production. Retrying an API call is okay, at least my Minikube was not always able to answer the requests when it was under heavy load, but if your application is not able to even construct a Kubernetes client, you should consider restarting all over.

## Exporting metrics with a Prometheus registry

Now that I got my numbers of nodes (both `Ready` and `NotReady`) I was able to tell the world - or at least my Prometheus server - about it. So, how does one expose Prometheus metrics?

Again, I did some quick research and found some very nice [official documentation](https://prometheus.io/docs/guides/go-application/) which directed me where to start with instrumenting my application to expose Prometheus metrics.

To get started in a quite basic way, Prometheus offers three official modules for usage. The code of those modules is quite understandable but I didn't want to re-invent the wheel so for the moment I chose to just use them rather than implementing everything on my own.

I use the `prometheus` module to create a metric *registry*. This registry is going to hold every metric that I want to expose from my application and will know which metrics exist. If you don't specify your own registry, you will be given a default one which serves Golang metrics out-of-the-box without any further configuration. Very tasty, but not needed for my use case, so I deactivated it and created my very own empty registry.

`promauto` is quite nice for handling new metrics and their types, I decided for `GAUGE` metrics and a `GAUGE` vector (for a list of all nodes and their status) - to add some sugar, `promauto` automatically adds your metrics to the default registry when it finds one. If you create your own registry, it's also your task to fill it (e.g. using `registry.MustRegister`).

`promhttp` is the third musketeer which integrates 100% with Golangs' builtin `net/http` server, and I love when something integrates with `net/http`.

Putting these three together, you're free to server your metric like this:

```go
package main

import (
    "log/slog"
    "net/http"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

func recordMetrics() {
    go func() {
        for {
            myMetric.Set(someWildValue)
            }
            time.Sleep(5 * time.Second)
        }
    }()
}

var (
    myMetric = promauto.NewGauge(prometheus.GaugeOpts{
        Name: "my_super_cool_metric",
        Help: "Beware, the coolest metric in the cluster",
    })

    reg = prometheus.NewRegistry()
)

func init() {
    reg.MustRegister(myMetric)
    logger = slog.New(slog.NewJSONHandler(os.Stdout, nil))
}

func main() {
    recordMetrics(clientset)

    http.Handle("/metrics", promhttp.HandlerFor(reg, promhttp.HandlerOpts{}))
    logger.Error("error serving", http.ListenAndServe(":2112", nil))
}
```

This spawns a separate goroutine which is going to set a new value for the registered `myMetric` gauge metric.

Very nice, so how does one deploy it?

## Build & Deploy

When I thought about how I would deploy an arbitrary application from the wild into a Kubernetes cluster, I immediately thought of a handy Helm Chart. I sandboxed around in my Minikube cluster to grasp what I really needed, and I created my BOM (bill of materials) needed to be deployed by my Helm Chart:

- A `Deployment` so the application actually runs
- A `ServiceAccount` so I have nice control over what my application has access to
- A `ClusterRole` and `ClusterRoleBinding` because Kubernetes `Nodes` are non-namespaced resources and I needed to allow the runtime identity of my application to access the nodes (RBAC)
- A `HorizontalPodAutoscaler` (keeping it basic by scaling on CPU) because scraping metrics can get heavy when the cluster grows
- A `PodMonitoring` which is the Google-Managed version of a `PodMonitor` that a Prometheus Operator would provide
  - This basically tells Prometheus to monitor a certain resource in the cluster by calling a certain HTTP endpoint
  - The only documentation on this I could find was a 7 lines example which (luckily) worked after some tweaking - seriously, Google, if you must provide your own shitty API and create `CustomResources` that do a *similar* thing to what the original does, at least document what they actually do and how they differ.
  - (Originally I would have liked a `ServiceMonitor` to not multiply the metric reporting by the number of Pods, but I couldn't find any documentation in the world of Google-Managed Prometheus and became tired after being pushed back to marketing pages continuously.)

The Helm Chart I came up with looked like [this](https://github.com/torbendury/kube-node-status/tree/394d68e73bcfe49be96065e6159603d7537d9b47/helm/kube-node-status) after some errors, it is still not really done 100% but it does the job.

To release the Helm Chart and the container, I utilized my free DockerHub account, GitHub Releases and GitHub Pages - all accessed by a [GitHub Action](https://github.com/torbendury/kube-node-status/blob/394d68e73bcfe49be96065e6159603d7537d9b47/.github/workflows/release.yml).

Since I wanted to host my own Helm repository, I looked up what the Helm Repository API looks like - and it is thankfully easy. You need an HTTP server which serves a basic YAML file at `/index.yaml`. GitHub Pages to the rescue! Serving it from a separate branch `gh-pages`, you can find it [here](https://github.com/torbendury/kube-node-status/blob/gh-pages/index.yaml) - it is automatically being filled by my above mentioned GitHub Actions pipeline at the end of a successful run.

This makes installing `kube-node-status` as easy as:

```bash
$ helm repo add kube-node-status https://torbendury.github.io/kube-node-status
$ helm repo update
$ helm install kube-node-status kube-node-status/kube-node-status \
    --create-namespace \
    --namespace kube-node-status
```

## Conclusion

Getting started with calling the Kubernetes API using Golang is pretty easy and works very well and stable. You have the option to handle custom errors returned by Kubernetes API calls. I guess the `client-go` module is very well written - probably because Kubernetes itself is also written in Go.

Also, instrumenting your application using Prometheus client modules is also a joyful ride, did I mention Prometheus is written in Go, too? The modules provided by Prometheus make it really easy and fun to expose your very own custom metrics and I can recommend using it.

I also recommend thinking twice if you really want Google to manage your `prometheus` and `kube-state-metrics` installation. This does not spark joy.
