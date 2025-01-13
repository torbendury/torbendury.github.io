---
layout: post
title: Collecting syslog logs in otel-collector with Azure Container Apps via TCP and TLS
categories: [Development, Public Cloud, Microsoft Azure, Container]
---

Squeezing out Azure Container Apps features to go serverless with otel-collector syslog collection via TCP+TLS.

## Problem Statement

Recently, together with a colleague, I was commissioned to set up monitoring for metrics, traces and logging for a newly purchased software that unfortunately does not come with its own monitoring. In a previous PoC with the tool, an OpenTelemetry otel-collector proved to be helpful for collecting this telemetry data. Since the data should not be isolated, but should be viewable in a larger overall context, it is important for us to send the telemetry data in a standardized format to any data sink.

As the software to be monitored ran as SaaS in Azure, we also looked around Azure ourselves and came across Azure Container Apps. As the OpenTelemetry otel-collector is available for download as a container image, the decision was quickly made that we would also like to host it in an efficient, low-overhead and cost-effective container environment.

Of course, we first had to become aware of the technical limitations. The software's documentation revealed that it offers a logging export via syslog format (RFC 3164), for example. The export can take place to a self-set host via UDP, TCP or TCP with TLS.

Unfortunately, we searched in vain for the possibility of authentication.

As the software is not managed directly by us and we were unable to establish a private network connection with it at the time, we needed a solution that would enable both secure connections via the public Internet and connections via private cloud networks.

## Solution

One software solution offered by Azure that seemed flexible enough and yet “managed” enough to us is Azure Container Apps.

Container Apps acts as a serverless container software offering from Azure, which takes the operational pressure out of container hosting. You define the containers you want to run in a Kubernetes-like YAML, configure the required storage, secret and ingress settings, and create the app on Azure.
Only the last traces remind you that Kubernetes is being used under the hood. For example, a load balancer called 'kubernetes' is automatically provisioned when a container app is created in the Azure subscription. However, this is of no further interest to us and is an unimportant implementation detail from the customer's point of view.

With Container Apps, we are able to manage custom domains (both via DNS CNAME if we are using HTTP traffic and via A-Record if we are using TCP), can enrich the Container Apps with storage for various purposes and are able to share sensitive data with the app via an Azure Key Vault. All of this shows us how highly integrated container apps are in the Azure ecosystem and how easy it is for us to quickly set up monitoring for our software.

We can also restrict ingress via IP whitelisting and can even make container apps available via private networking, which keeps us flexible for future changes to our architecture.

This results in a solution that is going to look something like this:

<div style="text-align: center"><img src="/images/2025-01-12/01.png"/></div>

## Implementation

Info: To minimize the code sample size, I will leave out the provider configuration and variable definitions here. They should be self-explaining. If not, feel free to hit me up and I will happily add the definition and explain my thoughts.

### Basics

First things first, we will need an environment inside our existing Azure subscription in which we can deploy our application.

We'll start out with Azure Blob Storage, a Container App Environment and an empty Azure Key Vault:

```hcl
locals {
    common_prefix = "otel"
}

# We will stick to one Resource Group for this use case. Might make sense to separate it later, maybe
resource "azurerm_resource_group" "this" {
  name     = "${local.common_prefix}-${terraform.workspace}-rg"
  location = "Sweden Central"
}

# Emit logs coming from the Container App into this workspace for analysis
resource "azurerm_log_analytics_workspace" "this" {
  name                = "${local.common_prefix}-${terraform.workspace}-logs"
  location            = azurerm_resource_group.this.location
  resource_group_name = azurerm_resource_group.this.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
}

# The surrounding Container App Environment which we will deploy our app to and which is connected to Azure Networking, Log Analytics and our Storage Account.
resource "azurerm_container_app_environment" "this" {
  name                       = "${local.common_prefix}-${terraform.workspace}-cae"
  location                   = azurerm_resource_group.this.location
  resource_group_name        = azurerm_resource_group.this.name
  log_analytics_workspace_id = azurerm_log_analytics_workspace.this.id
  infrastructure_subnet_id   = azurerm_subnet.this.id # I will not go into details for this due to security reasons, but you will need a `/23` subnet to let the Container App exist inside.
}

resource "azurerm_resource_provider_registration" "app" {
  name = "Microsoft.App"
}
```

Also, for security reasons, we will rely on a Azure Identity created by ourselves so we can assign permissions to it in a clean way:

```hcl
resource "azurerm_user_assigned_identity" "otel_collector" {
  location            = var.standard_region
  name                = "otel-collector"
  resource_group_name = azurerm_resource_group.this.name
  tags = merge(var.standard_labels, var.env_labels, {
    role = "identity"
  })
}
```

These are the bare minimal basics needed to get anything running. Next, we will create a Azure Blob Storage, upload our otel-collector configuration and connect it with our Container Apps Environment:

```hcl
resource "azurerm_container_app_environment_storage" "otel_config" {
  name                         = "${local.common_prefix}-otel-config-caes"
  container_app_environment_id = azurerm_container_app_environment.this.id
  account_name                 = azurerm_storage_account.otel_config.name
  share_name                   = azurerm_storage_share.otel_config.name
  access_key                   = azurerm_storage_account.otel_config.primary_access_key
  access_mode                  = "ReadOnly"
}

# Used to make the Storage Account name unique
resource "random_id" "otel_config_storage_account" {
  byte_length = 3
}

resource "azurerm_storage_account" "otel_config" {
  name                            = "otelconfig${random_id.otel_config_storage_account.hex}" # No dashes allowed
  resource_group_name             = azurerm_resource_group.this.name
  location                        = azurerm_resource_group.this.location
  account_tier                    = "Standard"
  account_replication_type        = "ZRS"
  min_tls_version                 = "TLS1_2"
  allow_nested_items_to_be_public = "false"
  shared_access_key_enabled       = true

  blob_properties {
    delete_retention_policy {
      days = 7
    }
  }
}

# Create a mountable Storage Share inside our Storage Account for the Container App Environment
resource "azurerm_storage_share" "otel_config" {
  name               = "otel-config"
  storage_account_id = azurerm_storage_account.otel_config.id
  quota              = 50
}

# Upload the otel-collector configuration file from local workspace
resource "azurerm_storage_share_file" "otel_config" {
  name             = "otel-config.yaml"
  storage_share_id = azurerm_storage_share.otel_config.url
  source           = "${path.module}/etc/otel-config.yaml"
  content_md5      = filemd5("${path.module}/etc/otel-config.yaml") # This is actually quite critical! azurerm_storage_share_file does not update automatically when the `source` is updated, so we need to actually set the `content_md5` so the file inside the Storage Share gets updated correctly when we change the config file.
}
```

And last we will create an empty Key Vault. This will contain our TLS certificate + key later on:

```hcl
resource "azurerm_key_vault" "this" {
  name                       = "${local.common_prefix}-${terraform.workspace}-kv"
  location                   = azurerm_resource_group.this.location
  resource_group_name        = azurerm_resource_group.this.name
  sku_name                   = "standard"
  tenant_id                  = var.tenant_id
  purge_protection_enabled   = true
  soft_delete_retention_days = 7
  enable_rbac_authorization  = true
}

data "azurerm_client_config" "current" {}

# Allow otel-collector to read the TLS secrets later
resource "azurerm_role_assignment" "otel_collector_secret_reader" {
  principal_id                     = azurerm_user_assigned_identity.otel_collector.principal_id
  role_definition_name             = "Key Vault Secrets User"
  scope                            = azurerm_key_vault.this.id
  skip_service_principal_aad_check = true
}
```

### TLS Certificates

Provisioning TLS certificates is itself an easy process and everyone can get certificates for free. I will rely on ACME Lets Encrypt here, and for reasons of low operational overhead I will provision and rotate them via Terraform. This way, we can later on put the TLS certificate outputs into the Key Vault, mount them in the Container App, and trigger a deployment on it so it retrieves the latest certificates.

```hcl
resource "acme_registration" "this" {
  email_address         = "XXXXXXX@XXXXXX.XXXXXX"
  account_key_algorithm = "ECDSA"
}

resource "acme_certificate" "otel_collector" {
  account_key_pem              = acme_registration.this.account_key_pem
  common_name                  = "otel-collector.XXXXXXXX" 
  min_days_remaining           = 30
  disable_complete_propagation = true

  dns_challenge {
    # https://registry.terraform.io/providers/vancluever/acme/latest/docs/guides/dns-providers-azuredns
    provider = "azuredns"

    # ARM_CLIENT_ID and ARM_CLIENT_SECRET is set by pipeline as ENV variable and retrieved by provider automagically
    config = {
      AZURE_TENANT_ID           = var.tenant_id
      AZURE_SUBSCRIPTION_ID     = local.subscription_id
      AZURE_RESOURCE_GROUP      = azurerm_resource_group.this.name
      AZURE_ENVIRONMENT         = "public"
      AZURE_TTL                 = 60
      AZURE_ZONE_NAME           = azurerm_dns_zone.XXXXXXXXX.name
      AZURE_PROPAGATION_TIMEOUT = 420
    }
  }
}

locals {
  cert_with_intermediate_ca = "${replace(acme_certificate.otel_collector.certificate_pem, "/\n/", "\n")}\n${replace(acme_certificate.otel_collector.issuer_pem, "/\n/", "\n")}"
}

#! Key Vault strips newlines!
#! https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/key_vault_secret
resource "azurerm_key_vault_secret" "otel_collector_tls_cert" {
  # checkov:skip=CKV_AZURE_41: Rotation automated, no explicit expiry needed
  name         = "tls-cert-pem"
  value        = local.cert_with_intermediate_ca
  key_vault_id = azurerm_key_vault.this.id
  content_type = "PEM"
}

resource "azurerm_key_vault_secret" "otel_collector_private_key" {
  # checkov:skip=CKV_AZURE_41: Rotation automated, no explicit expiry needed
  name         = "tls-key-pem"
  value        = replace(acme_certificate.otel_collector.private_key_pem, "/\n/", "\n")
  key_vault_id = azurerm_key_vault.this.id
  content_type = "PEM"
}
```

The ACME provider works incredibly good and can be found here: [vancluever/acme](https://registry.terraform.io/providers/vancluever/acme/latest/docs). It also supports a very big amount of DNS providers, so if Azure DNS is not your cup of tea, I am pretty sure you will find your loved provider there, too. The provider automatically takes care of solving the DNS01 challenge by propagating the ACME ownership TXT record. This is optional, so if you want to take care of this in another way, feel free to.

An important thing to know - which I learned when I implemented this - is that the server has to deliver not only the Lets Encrypt certificate but also the intermediate CA when responding to a request. Otherwise the certificate won't get verified by the client!

### Container App Deployment

Now that we have our stuff mostly set up, lets talk about the Container App deployment. It is surprisingly easy and mostly feels like defining a Kubernetes Deployment, but in HCL instead of YAML:

```hcl
locals {
  otel_config_volume_name = "otel-config"
  otel_config_mount_path  = "/mnt/config"
}

resource "azurerm_container_app" "otel_collector" {
  name                         = "${local.common_prefix}-${terraform.workspace}-otel-collector"
  container_app_environment_id = azurerm_container_app_environment.this.id
  resource_group_name          = azurerm_resource_group.this.name
  revision_mode                = "Single"

  template {
    container {
      name   = "otel-collector"
      image  = "otel/opentelemetry-collector-contrib:latest"
      cpu    = "0.5"
      memory = "1Gi"
      args   = ["--config=${local.otel_config_mount_path}/otel-config.yaml"]

      volume_mounts {
        name = local.otel_config_volume_name
        path = local.otel_config_mount_path
      }

      volume_mounts {
        name = "tls"
        path = "/mnt/tls"
      }

      # Send a prayer to the gods of the cloud, because Container Apps don't automatically get updated
      # when files in the storage account are updated.
      env {
        name  = "TRIGGER_NEW_REVISION_CONFIG_CHANGE"
        value = filesha256("${path.module}/etc/otel-config.yaml")
      }
      env {
        name  = "TRIGGER_NEW_REVISION_TLS_EXPIRY"
        value = sha256(acme_certificate.otel_collector.certificate_not_after)
      }
      env {
        name  = "TRIGGER_NEW_REVISION_CA_CHAIN"
        value = sha256(local.cert_with_intermediate_ca)
      }
    }

    volume {
      name         = local.otel_config_volume_name
      storage_type = "AzureFile"
      storage_name = azurerm_container_app_environment_storage.otel_config.name
    }

    volume {
      name         = "tls"
      storage_type = "Secret"
    }
  }

  secret {
    name                = "tls-cert-pem"
    key_vault_secret_id = azurerm_key_vault_secret.otel_collector_tls_cert.versionless_id
    identity            = azurerm_user_assigned_identity.otel_collector.id
  }
  secret {
    name                = "tls-key-pem"
    key_vault_secret_id = azurerm_key_vault_secret.otel_collector_private_key.versionless_id
    identity            = azurerm_user_assigned_identity.otel_collector.id
  }

  ingress {
    external_enabled           = true
    allow_insecure_connections = false
    target_port                = 54526
    exposed_port               = 54526
    transport                  = "tcp"

    traffic_weight {
      percentage      = 100
      latest_revision = true
    }

    dynamic "ip_security_restriction" {
      for_each = local.otel_collector_allowed_ranges
      content {
        action           = "Allow"
        description      = ip_security_restriction.value.description
        ip_address_range = ip_security_restriction.value.ip
        name             = ip_security_restriction.value.name
      }
    }
  }

  identity {
    type = "UserAssigned"
    identity_ids = [
      azurerm_user_assigned_identity.otel_collector.id
    ]
  }
}
```

You might have a bad feeling about the `env` values acting as a deployment trigger for the Container App, and you're totally right. I couldn't find a way to trigger a deployment expliticly otherwise than doing this, because Container Apps simply don't care if Storage Account files change or Key Vault secrets get updated. Instead, I rely on calculating SHA256 hashes on values which should trigger a deployment when changing. This includes the otel-config itself, the expiry date changing (i.e. a certificate being rotated) or the certificate with its intermediate CA changing.

The other settings are quite basic; we define `tcp` as a protocol and set multiple `ip_security_restriction` blobs which I won't go into further detail here for security reasons. Technically spoken, only we and the SaaS software should be able to access the otel-collector Container App.

### Custom Domain

Next up is defining the custom domain. This is done quite fast, actually! We can *not* use a `CNAME` record since this would require us to use HTTP (which we don't), but we can still use a `A` record to finally get our custom domain working. We will use the same one that we defined in our ACME certificate request, and since we are left to terminate TLS ourselves (which otel-collector takes care of when we configure it to do so), we are done with minimal effort:

```hcl
resource "azurerm_dns_a_record" "otel_collector" {
  name                = "otel-collector"
  zone_name           = azurerm_dns_zone.XXXXXXXXXXXXXXx.name
  resource_group_name = azurerm_resource_group.this.name
  ttl                 = 60
  records             = [azurerm_container_app_environment.this.static_ip_address]
}
```

Poof, we're done!

### otel-config

We will be done with the next part very fast. In a minimal setup, we only want the otel-collector to receive `syslog` logs via TCP+TLS and post them to `stdout`. Forwarding or further processing of the logs is not in scope of this blog post and might be a thing later, but for now this is sufficient:

```yaml
receivers:
  syslog:
    location: "UTC"
    tcp:
      listen_address: "0.0.0.0:54526" # 0.0.0.0 needed for serverless environments
      preserve_leading_whitespaces: false
      preserve_trailing_whitespaces: false
      encoding: utf-8
      tls:
        cert_file: "/mnt/tls/tls-cert-pem"
        key_file: "/mnt/tls/tls-key-pem"
    protocol: rfc3164
processors:
  batch:
exporters:
  debug:
    verbosity: normal
    sampling_initial: 5
    sampling_thereafter: 200
service:
  telemetry:
    logs:
      level: debug
      encoding: json
  pipelines:
    logs:
      receivers:
        - syslog
      processors:
        - batch
      exporters:
        - debug
```

Since we don't have any control over internal IP addresses inside Azure which call our Container App, we will rely on listening to all network interfaces with `0.0.0.0`. Also, `/mnt/tls/` is the place where we mounted our TLS certificate chain and private key when defining the Container App. RFC3164 is needed because our SaaS uses it. The rest is pretty basic and is needed to get some debug output to `stdout` for now.

## Conclusion

The above setup works like a charm for me and will be extended with other telemetry data like metrics and traces later.

As you have seen, Container Apps are **very** flexible and are highly integrated into the Azure ecosystem. They come with some weird tendencies - eventually consistency with Azure Blob Storage files and Key Vault secret versions, for example - but when one understands what Azure wants a user to do when working with Container Apps, it's really a charm. In my opinion, the integration works at least as good as f.e. Google Cloud Run would work with Google Secret Manager.

A big plus is that we can switch to private network traffic later on by peering the VNet which our application is running in, we're able to receive raw TCP traffic and handle it ourselves, attaching network storage to our Container App really is a charm and you can attach to a) live console log stream as well as system log stream and b) even open up a terminal session on your Container App for debugging, which is completely crazy and makes debugging feel like vacation.

All in all, I have learned so much about Azure Container Apps, got deeper knowledge in ACME Lets Encrypt and also got some better insights in `otel-collector` (which I will write more about later on, when I have learned even more).
