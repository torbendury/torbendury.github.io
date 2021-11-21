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
resource "google_compute_instance" "tutorial" {
  name         = "terraform-example-01"
  machine_type = "f1-micro"
  zone         = "us-central1-c"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-10"
    }
  }

  network_interface {
    network = "default"

    access_config {
      // Ephemeral public IP
    }
  }
}
```

_Content of `vm.tf`_

With this example, you will create a Google Compute Engine instance called `terraform-example-01` with machine type `f1-micro`. The zone (same as in our provider config) is `us-central1-c`. It will run a basic Debian 10 image (provided by GCP) and run in our `default` network. This network gets created when you create a new project in GCP. It will get an ephemeral public IP from which you will be able to connect to it.

Terraform resources are by far the most important elements of Terraform. What you see above is `HCL`, the HashiCorp Language. We declared a `resource` of type `google_compute_instance` and an identifier `tutorial`.

**NOTE: The identifier is used by Terraform. With this, we can refer to this VM instance from other points in our Terraform workspace. It will not show up in your deployed infrastructure and it may differ from your declared `name` inside the resource.**

There's several input arguments shown in the resource, such as a `boot_disk` block which (in our case) sets the base image to deploy, and a `network_interface` block to assing the VM instance to a specific Virtual Private Cloud (VPC).

If you are unsure which arguments are needed to deploy a resource, you can always look it up in the documentation of each provider. For example, to explore the `google_compute_instance` further, you may head to the [google_compute_instance Documentation](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_instance). Always make sure to use the version-specific documentation as the provider and its APIs may change in different versions.

### Initialize Terraform

Now that you have filled up your `terraform-example-01` directory, you need to let Terraform initialize the workspace. This is a one-liner and it will let Terraform download the Google provider from the registry. You need to initialize every workspace _at least_ once.

Use the following command to initialize your workspace:

```bash
$ terraform init
```

### Planning and Applying resources

What is planning? Planning is a so-called "dry-run" of your workspace against the cloud provider API. It will show you what resources it will create, update or delete.

**IMPORTANT**: To plan and apply resources, we must authenticate against our cloud platform. To authenticate against GCP, we can modify `main.tf` and add a `credentials` argument to it:

```hcl
provider "google" {
  project     = "my-project-id"
  region      = "us-central1"
  zone        = "us-central1-c"
  // Path to or the content of a JSON service account key file
  credentials = "mysecretkey.json"
}
```

#### Planning

To see which resources will be created, updated or deleted before really touching them, you can run a `terraform plan` and see which resources will be touched by Terraform:

```bash
$ terraform plan
```

Terraform is going to refresh its state and lookup if there are any conflicts or errors in your code which may result in failure. If everything is alright, you are ready to head to the next step.

#### Applying

Now that we've configured our cloud provider and declared a VM instance, let's wait no longer and finally create it with `terraform apply:`

```bash
$ terraform apply

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
    + create

Terraform will perform the following actions:

    # google_compute_instance.tutorial will be created
    + resource "google_compute_instance" "tutorial" {
        [...]
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
    Terraform will perform the actions described above.
    Only 'yes' will be accepted to approve.

Enter a value:
```

This output is called an _execution plan_. In general, it's a good idea to review the output before proceeding. When you're done and ready to move on, type `yes` and hit `Enter`.

You're done! Now, locate to [GCP Console](https://console.cloud.google.com) and review the instance created.

#### Destroying

Nothing in life is free, neither are cloud resources (except the `f1-micro` instance we just created). When you no longer need cloud resources, you'll always want to destroy them. Terraform can take care of cleaning up your cloud project (as long as it was created - or imported - by Terraform)! Just hit in the following command and confirm when prompted:

```bash
$ terraform destroy
```

This will show you another CLI output of the resources that will be deleted.

## Sum up

In this post, you learned how to:

- Install Terraform
- Configure a cloud provider and authenticate it
- Creating resources on your cloud provider
- Destroying resources when you no longer need them

This is a very good starter project to get a little taste of Terraform. But it does by no mean do justice to the Terraform technology as a whole. In Terraform, you can do much more then simply provision cloud resources from boring static code. You can provision resources completely dynamically, based on the results of queries or lookups, which I will show you in an upcoming post. Stay tuned!
