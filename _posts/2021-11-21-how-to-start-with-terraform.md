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

The principle of Terraform IaC is that you write understandable and readable code, which is translated by Terraform to speak with Cloud Provider APIs and deploys consistent and repeatable environments:

![Basic workflow of Terraform](https://www.plantuml.com/plantuml/svg/LP0nImD148Nx_HK3TcdiDKWa8Dgbq4gkcBkTN0RsTkERsGltxzrW3amxxxtlWzcPCtpI72S-XyttGzBnv2DawUZB167JRfUJkdHF-vAFEbQmQydqfaaiR8SIvIL4COL4qdm4cwCENY6XmLs8ZQwji7tyApytw6hgKstaJm7uM32jF0TdIzVny5yQlD3CIIEz7Zu8ybF5tCAiJ6UKMQE0UiqC5RlNDTy8aTpHeVP91zgdKgFTeaLIAfUEtfSU6k-p0iwZj1rqPfSrt4cEjxVz0W00)
