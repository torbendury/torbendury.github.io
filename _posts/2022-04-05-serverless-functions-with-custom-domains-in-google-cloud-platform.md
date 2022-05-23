---
layout: post
title: Serverless Functions with Custom Domain in Google Cloud
categories: [Cloud, GCP, Serverless, Development, Web Development, Batch Processing, GCF, IaC]
---

When you host a simple web application, e.g. an API with serverless functions, you most certainly want your custom DNS on top of it. Here's how to do it without Firebase.

## Overview

When serving *Cloud Functions* via HTTP triggers in GCP, we get a domain that looks something like this:

```text
         # REGION       PROJECT                          FUNCTION
  https://europe-west1-sf3m-d0239d8m.cloudfunctions.net/helloworld
```

Of course, this is not an API endpoint we like to share with our developer colleagues. This is where custom domains come in. In fact, it is not *that* trivial to host your Cloud Function via Custom DNS. Of course, you could just redirect to this URL, or add a CNAME, but that would lead to a) having a server that redirects to serverless (and needs to be maintained) or b) certificate issues.

This is why we take the easy, cloud native approach and utilize the Google Cloud Platform as much as possible by specifying a *Serverless network endpoint group (NEG)*. Basically, this only specifies a group of backend endpoints (our Cloud Functions) behind a load balancer. When using serverless NEG, you're not bound to only use Cloud Functions. You may also use *Cloud Run* applications or *App Engine* apps. Our NEG will be behind a load balancer which points to our external IP address we reserved for our GCF.

## The Cloud Function

In case you do not already have a HTTP triggered function, take this example here:

```python
import functions_framework
@functions_framework.http
def hello_http(request):
    return 'Hello World!', 200
```

This is going to return a HTTP status code `200` when a client sends a `GET` request to our Cloud Function.

## IP address

We are going to need a global static external IP address. It will be used by clients to reach out to our load balancer (which forwards to the Cloud Function):

```bash
  gcloud compute addresses create example-ip --network-tier=PREMIUM --ip-version=IPV4 --global
  # note down the address
  gcloud compute addresses describe example-ip --global
```

## SSL certificates

When you want to use a custom domain, you'll also need your own SSL certificate that is being delivered to clients by the load balancer. You can create SSL certificates either as Google-managed certificates or you can bring your own self-managed SSL certificate. For our use case, we are fine with Google-managed SSL certificates, since I manage my domain inside GCP anyway. To manage your domain inside GCP, you need a DNS zone. Inside your zone, you will need to create an A-record to point to the above created IPv4 address.

## Load balancer

Our architecture will look something like this:

![External HTTPS Load Balancing architecture for serverless apps](https://cloud.google.com/load-balancing/images/lb-serverless-run-ext-https.svg)

Since we are using serverless backends that do not support load balancer health checks, we do not need to create a Cloud Armor rule to allow health checks.

To create above resources, run the following commands:

```bash
# create the serverless NEG
   gcloud compute network-endpoint-groups create SERVERLESS_NEG_NAME \
       --region=europe-west1 \
       --network-endpoint-type=serverless  \
       --cloud-run-service=CLOUD_RUN_SERVICE_NAME
# create a backend service for the LB
   gcloud compute backend-services create BACKEND_SERVICE_NAME \
       --load-balancing-scheme=EXTERNAL \
       --global
# add the created serverless NEG as a backend
   gcloud compute backend-services add-backend BACKEND_SERVICE_NAME \
       --global \
       --network-endpoint-group=SERVERLESS_NEG_NAME \
       --network-endpoint-group-region=europe-west1
# create an URL map to route incoming requests
   gcloud compute url-maps create URL_MAP_NAME \
       --default-service BACKEND_SERVICE_NAME
```

When we're done with that, we can create SSL certificates and let a HTTP(S) proxy target our URL map:

```bash
# create a Google-managed certificate
   gcloud compute ssl-certificates create SSL_CERTIFICATE_NAME \
       --domains DOMAIN

# create a HTTPS proxy
   gcloud compute target-https-proxies create TARGET_HTTPS_PROXY_NAME \
      --ssl-certificates=SSL_CERTIFICATE_NAME \
      --url-map=URL_MAP_NAME

# finally, create a forwarding rule
   gcloud compute forwarding-rules create HTTPS_FORWARDING_RULE_NAME \
       --load-balancing-scheme=EXTERNAL \
       --network-tier=PREMIUM \
       --address=example-ip \
       --target-https-proxy=TARGET_HTTPS_PROXY_NAME \
       --global \
       --ports=443
```

When we're done, we need to note down the IP address that is associated with the load balancer. Add an A-record in your DNS zone which lets your custom domain point to that IP address.

## Additions

When you have a use-case for multi-region load balancing (e.g. multiple GCF hosted in multiple regions) and need to serve traffic close to your clients, have a look at [these considerations](https://cloud.google.com/load-balancing/docs/https/setting-up-https-serverless#additional_configuration_options). There's also hints on using Cloud CDN, IAP and Cloud Armor with serverless NEGs.
