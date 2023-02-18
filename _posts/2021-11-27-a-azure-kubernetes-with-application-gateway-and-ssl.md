---
layout: post
title: Using Azure Application Gateway Ingress Controller in AKS and Let's Encrypt Certificates to securely expose applications
categories: [Public Cloud, Microsoft Azure Kubernetes]
---

How to deploy an AKS cluster with a managed instance of __Application Gateway__ and __Application Gateway Ingress Controller__ and obtain __free__ and __automated__ Let's Encrypt certificates. We will be doing this with Terraform and Helm.

## Introduction

When using Azure's __AKS__ (Managed Kubernetes Engine) together with workloads that have to be accessible from outside your cluster, you will sooner or later be challenged with choosing a proper ingress, SSL certificates and a well maintainable way of managing all of this with minimum effort.

What we want to build is a:

- solution that needs __minimum effort__ (with reasonable _initial_ effort for setup)
- infrastructure that is mainly managed not by you, but by the platform you use and the services you deploy
- overall system that is __secure__ and __hardened__ by default

## Prerequisites

We are going to perform those tasks mainly via HashiCorp Terraform. If you want to follow this step-by-step, you will need some basic Terraform skills and a proper setup, we won't discuss this here. I recently wrote a blog entry on [How To Start With Terraform](https://torbentechblog.com/a-how-to-start-with-terraform/), you might check this out if you lack some skills on this spot.

Otherwise, what you are going to need is:

- An Azure subscription to work on
- `helm`, `kubectl`, `azure-cli` and `terraform` installed
- An example app to publish (don't worry, I'll spend one if you don't have one ready)
- A domain in your hands to play around with

## Why not use nginx ingress controller?

You may have stumbled over the thought of using the OSS and free _nginx ingress controller_. In many cases, this is absolutely fine and it often works out of the box. However, sometimes this does not meet our requirements. It brings us a piece of software that __needs to be configured, maintained and kept up-to-date__. In terms of cloud thinking, this does not seem to be as rosy as we might think (at least for production workloads).

Instead, rethink - you can pay _some_ money to your cloud provider (in this case, Microsoft Azure) and __will need much less time afterwards__ to maintain everything - the bill might soon pay itself.

## Application Gateway

Azure __Application Gateway__ basically is a __L7 load balancer__ for web traffic of any kind. It can make route decisions based on defined rule sets and much more. It is __high availabe__, has __autoscaling__ enabled on default and - for us most important - delivers an __ingress controller__ inside AKS Kubernetes clusters out of the box.

Application Gateway is also able to handle sticky sessions (cookie based) and can do automatic SSL redirects (nice!) and supports custom error pages.

If you want to put the money in it, Application Gateway also offers a __Web Application Firewall (WAF)__. It's just a checkbox away.

## Application Gateway Ingress Controller

The __ingress controller__ part of Azure Application Gateway constantly monitors your deployed workloads which are supposed to be exposed via Application Gateway, and contintuously updates the AppGW so your selected services get exposed the right way. It runs as a `Pod` inside your cluster, you can find it in `kube-system` namespace.

As always in public cloud managed clusters, you don't worry about services running in `kube-system` namespace. They're managed by the cloud provider - if you experience problems with some services in it, make it their problem.

AGIC basically eliminates the need of yet-another ingress controller and yet-another load balancer which stands in front of the cluster. Application Gateway can talk to your Pods directly using their private IP, so there is no more need for `NodePort` and `KubeProxy` stuff.

## Deploying AppGW + AGIC

There's always the possibility of deploying AppGW + AGIC _brownfield_ (to an existing cluster), but I recently had good experience with _greenfield_ deployments into fresh clusters. When creating a new cluster, __AGIC__ can be deployed as an __AddOn__ to the cluster. This also enables you to __automatically__ have an Application Gateway instance created for you without having to worry about it. Give it a `/24` subnet and let it provision.

Here's a Terraform snippet you can use:

```hcl
resource "azurerm_kubernetes_cluster" "example" {

  # [...]

  addon_profile {
    ingress_application_gateway {
      enabled = true
      gateway_name = "example-appgw"
      subnet_cidr = "somesubnet/24"
      # OR
      subnet_id = your.existing.subnet.for.appgw
    }
  }
}
```

This should be a sufficient snippet for your existing AKS Terraform resource. If you don't own one, you can take the example config from the [official azurerm Terraform documentation](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/kubernetes_cluster).

## Credentials, cert-manager and example app

Your cluster will have a public IP assigned by default. Also, your Application Gateway (it will reside in a separate Resource Group) will have a public IP. You can use this public IP to let a DNS point to. Then, use subdomains to access your specific applications.

### Get credentials

Configure your `azure-cli` to point to the right Azure subscription, obtain credentials and check access:

```bash
  $ az account set --subscription "your-subscription-id"
  [...]
  $ az aks get-credentials --name "cluster-name" --resource-group "your-resource-group"
  [...]
  $ kubectl get pods --all-namespaces
```

### Deploy cert manager

To let cert-manager obtain new SSL certificates for you, simply use the provided Helm Chart which also will deploy CRDs (CustomResourceDefinitions) for you:

```bash
  # Add Helm repository for the Chart
  $ helm repo add jetstack https://charts.jetstack.io && helm repo update
  # Create namespace
  $ kubectl create ns cert-manager
  # Install CRDs + Helm Chart
  $ helm install \
    cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --create-namespace \
    --version v1.6.1 \
    --set installCRDs=true
```

Now watch your cert-manager installation bootstrap itself. When everything is complete, you should see 3 Pods running:

```bash
  kubect get pods -n cert-manager
```

If you experience problems while deploying `cert-manager`, consult the [official docs](https://cert-manager.io/docs/installation/helm/).

#### Add a cluster issuer

In order to access Let's Encrypt and its ACME, you will need to provide the following information:

```bash
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:

    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: <YOUR.EMAIL@ADDRESS>

    # ACME server URL for Let’s Encrypt’s staging environment.
    # The staging environment will not issue trusted certificates but is
    # used to ensure that the verification process is working properly
    # before moving to production
    server: https://acme-staging-v02.api.letsencrypt.org/directory

    privateKeySecretRef:
      # Secret resource used to store the account's private key.
      name: letsencrypt-secret

    # Enable the HTTP-01 challenge provider
    # you prove ownership of a domain by ensuring that a particular
    # file is present at the domain
    solvers:
    - http01:
        ingress:
            class: azure/application-gateway
EOF
```

This should be sufficient for our example case. We use the "staging" environment of Let's Encrypt to prevent getting blocked on their production servers when doing try-and-error or just some testing.

_WARNING: Those certificates provisioned by staging environment will not be trusted publicly. When you need to access the production environment for trusted certificates, simply remove the "-staging" from the URL above._

### Deploy a sample application

Credits for this sample app go to Microsoft Azure.

```bash
  kubectl apply -f https://raw.githubusercontent.com/Azure/application-gateway-kubernetes-ingress/master/docs/examples/aspnetapp.yaml
```

This will deploy a single Pod, a Service and an Ingress for you. I will give you a sample Ingress using AGIC + cert-manager, and your task will be to modify the sample Ingress of your `aspnetapp` to provision a certificate.

```bash
kubectl apply -f - <<EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
name: example-ingress-letsencrypt-staging
annotations:
    kubernetes.io/ingress.class: azure/application-gateway
    certmanager.k8s.io/cluster-issuer: letsencrypt-staging
spec:
tls:
- hosts:
    - <PLACEHOLDERS.COM>
    secretName: example-secret-name
rules:
- host: <PLACEHOLDERS.COM>
    http:
    paths:
    - backend:
        serviceName: frontend
        servicePort: 80
EOF
```

## Renewal of certificates

`cert-manager` is going to take care of auto-renewing the certificates when needed. By default, Let's Encrypt certificate last for 3 months and then need to be updated - this is where handy `cert-manager` comes in. It constantly checks for expiring certificates and automatically renews them for you. AGIC will take care of replacing them for users which access your application.

## Securing your application with NSG

You can create a Azure `Network Security Group` and assign it to the subnet of your Application Gateway instance. With this, you can ensure control of network activity of inbound as well as outbound traffic to/from your AppGW.

__IMPORTANT__: Here is a full official and documented list of inbound/outbound rules you will __need__ in order for your AppGW and applications to work properly: [Application Gateway - Network Security Groups](https://docs.microsoft.com/en-us/azure/application-gateway/configuration-infrastructure#network-security-groups)

## Overview of what we've built

Instead of accessing your application Pods directly, users will now be redirected through Application Gateway. This routes users to the specific applications which are configured to be accessed publicly. Network Security Groups will check that incoming traffic is allowed and will drop traffic if needed. The Application Gateway Ingress Controller is translating the route to a specific set of Pods which are behind a Kubernetes Service which will distribute the load. Simply said, our incoming traffic flow looks like this (many details are kept out for simplicity):

<div style="text-align: center"><img src="/images/2021-11-27/01.png"/></div>

## Conclusion

In this post you've learned to:

- Deploy AKS with AGIC AddOn enabled (and AppGW auto-provisioning)
- Deploy `cert-manager` into your cluster
- expose your application for public traffic through AGIC
- obtain SSL certificates for your custom domain
