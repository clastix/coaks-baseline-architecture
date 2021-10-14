# Capsule over AKS (CoAKS) Baseline Architecture - DRAFT
This reference implementation introduces the recommended starting (baseline) infrastructure architecture for implementing a multi-tenancy [Azure AKS](https://azure.microsoft.com/services/kubernetes-service) cluster using the [Clastix Capsule Operator](https://github.com/clastix/capsule). 

This implementation and document is meant to guide Azure users through the process of getting this secure baseline infrastructure deployed and understanding the components of it.

This architecture is infrastructure focused, more so than on applications. It concentrates on the AKS cluster itself, including concerns with user identity, cluster governance, policies management, data persistence and network topologies.

The implementation presented here is the minimum recommended baseline for most AKS clusters. This implementation integrates with other Azure services that will provide a network topology that will support multi-regional scenarios, and keep the in-cluster traffic secure as well. This architecture should be considered your starting point for pre-production and production stages.

The material here is relatively dense. We strongly encourage you to dedicate time to walk through these instructions, with a mind to learning. We do NOT provide any "one click" deployment here. However, once you've understood the components involved it is encouraged that you build suitable, auditable GitOps deployment processes around your final infrastructure.


## Use Case: Container as a Service

Every organization with several business units have the same dilemma:
> Reduncing cost infrastructure but keeping isolation between the units.

This dilemma points us to a multitenancy environment and this is the purpose of [Capsule](https://github.com/clastix/capsule).

For our use case, we are going to work through an energy company called **Acme Corp** which is trying to build their own CaaS (Container as a Service) based on Kubernetes to serve multiple lines of business. Each line of business has its team of engineers that are responsible for the development, deployment, and operating of their digital products. Currently they have two business units: Solar and Eolic.

[Azure AKS](https://docs.microsoft.com/azure/aks/) will be used to provide Kubernetes cluster and it will be integrated with [Azure Active Directory (AAD)](https://azure.microsoft.com/services/active-directory/) to ensure a robust authentication and authorization mechanism.

## Cloud Architecture

![cloud architecture](./diagrams/cloud-architecture.drawio.png)


## Prerequisites

We are going to mainly use the following three tools:

* The [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli) version 2.11.0 or later.
 
* [Kubectl](https://kubernetes.io/docs/tasks/tools/) with a minimum version of 1.18.1 or kubelogin using azure-cli:
  
  ```bash
    $ sudo az aks install-cli
    $ kubectl version --client
    $ kubelogin --version
  ```

* [Helm](https://helm.sh/docs/intro/install/) with minimum version of helm 3.3.


## Steps

1. [AKS Creation with AAD integration](guides/create-aks-with-add.md)
2. [Capsule Installation](guides/capsule-installation.md)
3. [Multitenance Environment](guides/multitenance-environment.md)

## What’s next

[Let's start building the infrastructure](guides/create-aks-with-add.md)
