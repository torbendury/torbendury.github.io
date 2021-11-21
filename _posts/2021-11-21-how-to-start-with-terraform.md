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

## Getting Started

To follow this tutorial, you need Terraform installed locally on your machine. You can compile the tool from source or use pre-compiled binaries for several Linux distros, OSX (homebrew) or even Windows (via Chocolatey). Further explanation can be found [here](https://learn.hashicorp.com/tutorials/terraform/install-cli).

We expect that you have an active Google Cloud Platform account, a proper `project` setup and have access credentials (e.g. by [creating a Service Account](https://console.cloud.google.com/iam-admin/serviceaccounts) and giving it API access to GCP).

**NOTE**: Creating infrastructure resources in public clouds costs money and is usually billed by usage time + size. For this tutorial, we use a VM with `f1-micro` machine type which runs on GCP's _always free_ resources.

### Writing the configuration

As shown above, Terraform reads your desired infrastructure from configuration files to deploy the infrastructure components. So, if we want Terraform to deploy a VM instance, we need to _declare_ a VM instance as _code_. Start by creating an empty folder `terraform-example-01` and inside there a file called `main.tf`. The `.tf` file extension declares that this is a Terraform config file. When you run Terraform, it (basically) will concat every `.tf` file from your working directory and deploy the infrastructure described in there.

Let's start with configuring your GCP provider:

```hcl
provider "google" {
  project     = "my-project-id"
  region      = "us-central1"
  zone        = "us-central1-c"
}
```

_Content of `main.tf`_

The provider is responsible for understanding and translating API interactions, authenticating requests and so on. The `google` provider definition needs an assigned `project` in which infrastructure should be deployed to, and a default `region` and `zone` to deploy.

With this few lines, we told Terraform to use the "google" provider (GCP) and where to deploy our infrastructure. Easy, isn't it? Let's move on to our precious VM, where we will see how resources are defined. Create a file called `vm.tf` and paste this code into it:

```hcl
resource "google_compute_instance" "default" {
  name         = "terraform-example-01"
  machine_type = "f1-micro"
  zone         = "us-central1-c"

  tags = ["foo", "bar"]

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-10"
    }
  }

  // Local SSD disk
  scratch_disk {
    interface = "SCSI"
  }

  network_interface {
    network = "default"

    access_config {
      // Ephemeral public IP
    }
  }

  metadata = {
    foo = "bar"
  }

  metadata_startup_script = "echo hi > /test.txt"
}
```

_Content of `vm.tf`_
