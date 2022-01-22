---
layout: post
title: Manage recurring resources with Terraform and YAML
categories: [Cloud, Infrastructure]
---

Managing many cloud resources of the same type? Use Terraform `for_each` directive in combination with YAML templating!

In this post we're going to elaborate on how to structure your Terraform resources in a more readbable and easy changeable way.

## Introduction to Terraform

Terraform is an open-source infrastructure as code software tool created by HashiCorp. Users define and provide data center infrastructure using a declarative configuration language known as HashiCorp Configuration Language (HCL), or optionally JSON. Since JSON is made for machines and not for humans, we're going to focus on the very readable HCL.

If you didn't catch the grasp with Terraform yet, you might look into an [easy introduction to Terraform](https://torbentechblog.com/a-how-to-start-with-terraform/) I wrote recently.

## YAML

YAML is a **human-readable data-serialization language**. It is commonly used for _configuration files_ and in _applications_ where data is being stored or transmitted. YAML targets many of the same communications applications as the _Extensible Markup Language (XML)_ but with a **much** easier to read format.

Here's a little example for you, if you are not familiar with YAML yet:

```yaml
---
students:
  - name: John Doe
    major: Geography
  - name: Maria Schmitt
    major: Mathematics
  - name: Torben Dury
    major: Computer Science
```

As you can see, it is stupid-easy to describe our `students` and their properties which remind us of object-oriented programming.

Terraform uses declared resources like objects, so why shouldn't we declare them in YAML lists directly?

## Utilizing YAML with Terraform-managed resources

Before we get our hands dirty, let's describe our working directory structure. I like to keep the YAMLs separated from `*.tf` files, so create a new directory.

```bash
.
├── ip.tf
├── loadbalancer.tf
├── main.tf
├── virtualmachine.tf
└── yaml/
```

Now, let's say we want to create ServiceAccounts (in GCP - they're called _Service Principals_, in Azure).

The code looks like this, according to the documentation:

```hcl
resource "google_service_account" "service_account" {
  account_id   = "service-account-id"
  display_name = "My Service Account"
}
```

If we want to have multiple service accounts, we need to copy-paste these 4 lines for every single account, huh? Not really. Let me show you how to accomplish this. First, in your `yaml/` directory, create a file called `serviceaccounts.yaml`.

Fill the file like this:

```yaml
---
serviceaccounts:
  - my-account-1
  - my-account-2
  - my-account-3
```

Now, in your Terraform working directory, create a file `serviceaccounts.tf`. Paste the following code into it:

```hcl
locals {
  serviceaccounts = 
}
```

## Summary

