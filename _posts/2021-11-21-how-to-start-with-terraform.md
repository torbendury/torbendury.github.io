---
layout: post
title: How To Start With Terraform
categories: [Cloud]
---

Terraform basically is a enterprise-ready deployment technology used by IT professionals who want to **provison** and **manage** their infrastructure as code (**IaC**).
In this post, you will learn:

- The basics of HCL syntax
- Core elements and _building blocks_ of Terraform
- How to set up a Terraform workspace
- Finally, configuring and creating your first _virtual machine_ on GCP (Google Cloud Platform)!

## What is Terraform?

Terraform is a complete technology for IT departments used to provision Cloud infrastructure. Cloud is by no means only limited to Public Cloud providers, as long as there is a callable API, you can call it via Terraform. Most public cloud providers, including GCP, Azure and AWS, publish so-called Terraform _providers_. Those are libraries which enable you to write Terraform code and call Cloud APIs with it. If you want to check if your cloud provider hosts a Terraform provider, check the [official Terraform registry](https://registry.terraform.io/browse/providers).

Terraform is deterministic and keeps a state of your infrastructure. If you _roll out_ your Terraform code twice, the same outcome will be present on the other side. Terraform also checks your determined state against the real-world state and keeps it in sync (and even modifies resources to match your desired state).

{% plantuml %}
[First] - [Second]
{% endplantuml %}
