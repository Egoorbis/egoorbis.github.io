---
title: Deploy AKS with Bring your own CNI - "Cilium" 
date: 2024-05-19 08:30:SS +0200 # Updater
categories: [AKS, Networking]
tags: [terraform, aks, networking]     # TAG names should always be lowercase
description: In this tutorial, I will show you how to deploy and use an AKS environment with the Cilium CNI plugin and Cilium Network Policies via Infrastructure as Code. As we're quite lazy we will reuse Azure Verified Modules as much as possible. We will not use the AKS pattern module as it doesn't currently support Brining-your-own-CNI. 
---

## Prerequisites

To complete this tutorial, you will need the following
* An Azure subscription
* Basic knowledge of Terraform<sup>[1](#sources)</sup>

## Introduction

Azure Kubernetes Services (AKS) supports various networking option through different Container Networking Interface (CNI) plugins. Each of these solutions has advantages and disadvantages, choosing the right one depends on your specific needs and requirements https://learn.microsoft.com/en-us/azure/aks/concepts-network-cni-overview.

Besides using one of the provided options you can bring your own (BYO) CNI which provide advancaed functionalities. One of the most powerful CNI available is Cilium. Microsoft uses it to provide Azure CNI Powered by Cilium in AKS https://learn.microsoft.com/en-us/azure/aks/azure-cni-powered-by-cilium combining the Azure CNI with features from cilium.

However there are few limitations to it, if you want to use the full capacity, the BYO CNI option is your way to go and in this blog I'll show you how to setup an AKS with Cilium using Terraform and Helm.

# Note - Cilium is available in an OSS and an Enterprise Version. See for a feature comparision.

## Installing Clium CLI

Now first thing we aim to do is to install the Cilium CLI on our client which we will later on use to check the status of our Cilium installation.  https://github.com/cilium/cilium-cli/releases

Another easy way to install Cilium is to use a package manager such as Chocolately or winget.

## Perpare terraform 

We start with setting up the necessary configuration for our terraform deployment. Our data file contains references to our xx and our AKS cluster.

```terraform
data "azurerm_client_config" "current" {}

data "azurerm_kubernetes_cluster" "aks" {
  name                = azurerm_kubernetes_cluster.aks.name
  resource_group_name = azurerm_resource_group.aks-rg.name
}
```
Next up we configure our providers. We use the data resource on the helm provider in order to login to the cluster during our deployment and install cilium.

```terraform
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.71"
    }

    helm = {
      source  = "hashicorp/helm"
      version = "2.15.0"
    }

    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "2.4.1"
    }
  }
}

provider "azurerm" {
  features {}
}

provider "helm" {
  kubernetes {
    host                   = data.azurerm_kubernetes_cluster.aks.kube_config.0.host
    client_key             = base64decode(data.azurerm_kubernetes_cluster.aks.kube_config.0.client_key)
    client_certificate     = base64decode(data.azurerm_kubernetes_cluster.aks.kube_config.0.client_certificate)
    cluster_ca_certificate = base64decode(data.azurerm_kubernetes_cluster.aks.kube_config.0.cluster_ca_certificate)
  }
}
```


# Setup AKS Clusters

We start with deploying an AKS Cluster to our Azure Subscription. Wherever possible I'm reusing Azure Verified Moduls (AVM) for this job. However I'm not using the AVM AKS module, as this doesn't allow to specify the network profile.

Let's start with creating Entra ID group and assign our user to this group. We will later use it in as the AKS administrators group so we can enable Microsoft Entra ID Integration and disable local user accounts.

```terraform
resource "azuread_group" "aks-admins-group" {
  display_name     = "aks-admins"
  security_enabled = true
}

resource "azuread_group_member" "aks-admin-user" {
  group_object_id  = azuread_group.aks-admins-group.id
  member_object_id = "67c78feb-65e8-49ea-b4eb-b51053ec2205"
}
```

Now we load the Azure naming module. This is a handy module we can use to dynamically create our resource namings and deploying a resource  group


```terraform
module "naming" {
  source  = "Azure/naming/azurerm"
  version = "~> 0.4.1"
}

resource "azurerm_resource_group" "aks-rg" {
  name     = module.naming.resource_group.name_unique
  location = var.location
}
```

Next up is to deploy the required networking components. We first start with creating two network security groups followed by a virtual network with two subnets

```terraform
module "avm-res-network-networksecuritygroup" {
  count               = 2
  source              = "Azure/avm-res-network-networksecuritygroup/azurerm"
  version             = "0.2.0"
  resource_group_name = azurerm_resource_group.aks-rg.name
  name                = "${module.naming.network_security_group.name}-${count.index}"
  location            = azurerm_resource_group.aks-rg.location
}

module "avm-res-network-virtualnetwork" {
  source              = "Azure/avm-res-network-virtualnetwork/azurerm"
  version             = "0.4.0"
  location            = azurerm_resource_group.aks-rg.location
  resource_group_name = azurerm_resource_group.aks-rg.name
  name                = module.naming.virtual_network.name_unique

  address_space = ["10.11.0.0/16"]
  subnets = {
    subnet0 = {
      name             = "${module.naming.subnet.name_unique}0"
      address_prefixes = ["10.11.0.0/24"]
      network_security_group = {
        id = module.avm-res-network-networksecuritygroup[0].resource_id
      }
    }
    subnet1 = {
      name             = "${module.naming.subnet.name_unique}1"
      address_prefixes = ["10.11.1.0/24"]
      network_security_group = {
        id = module.avm-res-network-networksecuritygroup[1].resource_id
      }
    }
  }
}
```

The last step before deploying the cluster is to create a User Assigned Managed Identity to which we assign the Network Contributor role.


```terraform
module "avm-res-managedidentity-userassignedidentity" {
  source              = "Azure/avm-res-managedidentity-userassignedidentity/azurerm"
  version             = "0.3.3"
  name                = module.naming.user_assigned_identity.name_unique
  location            = azurerm_resource_group.aks-rg.location
  resource_group_name = azurerm_resource_group.aks-rg.name
}

module "avm-res-authorization-roleassignment" {
  source  = "Azure/avm-res-authorization-roleassignment/azurerm"
  version = "0.1.0"
  role_assignments_azure_resource_manager = {
    "aks" = {
      principal_id         = module.avm-res-managedidentity-userassignedidentity.principal_id
      role_definition_name = "Network Contributor"
      scope                = azurerm_resource_group.aks-rg.id
    }
  }
} 
```

Finally we deploy our basic cluster using the following code. Most importantly is to not configure any network profile. Besides that we assign the user assigned managed identity to the cluster. 

```terraform
resource "azurerm_kubernetes_cluster" "aks" {
  name                              = module.naming.kubernetes_cluster.name_unique
  location                          = azurerm_resource_group.aks-rg.location
  resource_group_name               = azurerm_resource_group.aks-rg.name
  dns_prefix                        = "cilium-aks"
  sku_tier                          = "Free"
  azure_policy_enabled              = true
  oidc_issuer_enabled               = true
  workload_identity_enabled         = true
  local_account_disabled            = true
  role_based_access_control_enabled = true

  default_node_pool {
    name                   = "systempool"
    enable_auto_scaling    = true
    enable_host_encryption = true
    node_count             = 1
    min_count              = 1
    max_count              = 3
    vm_size                = "Standard_DS2_v2"
    vnet_subnet_id         = module.avm-res-network-virtualnetwork.subnets["subnet0"].resource_id
    upgrade_settings {
      max_surge                     = "10%"
      drain_timeout_in_minutes      = 0
      node_soak_duration_in_minutes = 0
    }
  }

  identity {
    type         = "UserAssigned"
    identity_ids = [module.avm-res-managedidentity-userassignedidentity.resource_id]
  }

  network_profile {
    network_plugin = "none"
  }

  lifecycle {
    ignore_changes = [default_node_pool[0].node_count]
  }

  depends_on = [
    module.avm-res-authorization-roleassignment
  ]

  azure_active_directory_role_based_access_control {
    azure_rbac_enabled     = true
    admin_group_object_ids = [azuread_group.aks-admins-group.id]
    tenant_id              = data.azurerm_client_config.current.tenant_id
  }
}
```

Now in order to use helm


https://learn.microsoft.com/en-us/azure/aks/use-byo-cni?tabs=azure-
https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/
https://github.com/hashicorp/terraform-provider-kubernetes/blob/main/_examples/aks/kubernetes-config/main.tf
https://docs.cilium.io/en/stable/installation/k8s-install-helm/
https://github.com/ishuar/terraform-azure-aks/blob/main/main.tf



