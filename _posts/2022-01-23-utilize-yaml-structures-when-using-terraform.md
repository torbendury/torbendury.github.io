---
layout: post
title: Manage Recurring Resources With Terraform and YAML
categories: [Public Cloud, Terraform]
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
  serviceaccounts = yamldecode(file("${path.module}/yaml/serviceaccounts.yaml"))["serviceaccounts"]
}

resource "google_service_account" "service_account" {
  for_each     = toset(local.serviceaccounts)
  account_id   = each.key
  display_name = each.key
}
```

Let's look through this line for line. First, we declare a `locals` block to create a local variable. The inner function `file()` allows us to read a files' content, while the variable `path.module` points to our Terraform workspace to provide a full qualified path. `yamldecode()` reads and validates YAML content and converts it to objects. In our case, we have an object called `serviceaccounts` to which we point. Boom, we have our list of Service Accounts ready to go.

In our Terraform resource, we call the `for_each` directive so we can iterate over the content of our variable. Because `for_each` expects sets/maps, we explicitly convert it to one using `toset()`.

Then, we iterate over the list and access the content of the elements with `each.key` (`each.value`, respectively) and fill up our resource arguments.

## More complex data structures

The above example is just a simple proof of concept. Since YAML allows us to use arbitrarily complex data structures, can we also use them in Terraform? The answer is an absolutely clear YES!

Think of an imaginary firewall resource which has the following structure:

```yaml
- name: default-deny
  priority: 1337
  direction: "Inbound"
  access: "Deny"
  traffic:
    protocol: "*"
    source_port_range: "*"
    destination_port_range: "*"
    source_address_prefix: "*"
    destination_address_prefix: ${my_kubernetes_api_server}
```

This would per default deny every incoming traffic that wants to reach out to the IP address which lies behind `${my_kubernetes_api_server}`.

When we want to access this object, we would need to re-structure our above Terraform code. I will explain this in two parts

### Importing the YAML into a variable

```hcl
locals {
  fw_rules = yamldecode(templatefile("${path.module}/yaml/firewall-rules.yaml", { my_kubernetes_api_server = my_provider.kubernetes-cluster.ip_address }))["serviceaccounts"]
}
```

In this example, we utilize the built-in `templatefile()` function. It allows us to use variables in our YAML when we don't want to hard-code anything into it and barely know anything (or use multi-staging or multi-tenancy). And we never want to hard-code, right? `templatefile()` lets us replace the variables in any file with actual content known from Terraform resources.

Next, we define an imaginary firewall resource:

```hcl
resource "my_cloud_firewall" "kubernetes-fw" {
  for_each  = { for firewall_rule in local.fw_rules : firewall_rule.name => firewall_rule }
  name      = each.key
  priority  = each.value["priority"]
  direction = each.value["direction"]
  access    = each.value["access"]

  traffic {
    protocol = each.value["traffic"]["protocol"]
    # ...
  }
  # ...
}
```

This example is incomplete regarding showing everything in the YAML, but you get the trick. You can easily access nested structures from your YAML template.

When your YAML is filling up with 20-40 firewall rules, you will still not need to define the resource `my_cloud_firewall` again. It'll simply feed itself from the YAML file and enlarge the list of single resources.

## Summary

In this post, you've learned:

- How you can combine YAML structures with Terraform resources
- How to access nested structures
- How to re-use Terraform resources with iterations (`for_each`) instead of hard-coding everything
- Replacing variables in YAML when reading them into Terraform local variables
