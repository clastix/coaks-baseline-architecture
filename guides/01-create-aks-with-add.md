# AKS with MS Entra ID integration

Microsoft offers one of the best OIDC providers out there, i.e. **Microsoft Entra ID**, formerly known as **Azure Active Directory**, therefore we would like to use it to provide secure access to **Energy Corp's PaaS** based on [Capsule Operator](https://projectcapsule.dev/).

The administrator will have to provide a Kubernetes cluster using Azure [AKS](https://docs.microsoft.comhttps://docs.microsoft.com/en-us/azure/aks/) and integrates with Entra ID.

## Azure Log In

The first step is to log into the Azure ecosystem using the `az` CLI. It will redirect to [Azure](https://portal.azure.com) where you will log in.

```bash
az login
```

This log in the Azure admin user to create Azure resources in one of the assigned subscriptions.  

## Active Directory

The Azure admin will create different groups according to the multitenant environment of the **Energy Corp's PaaS**.

### CoAKS Admin Group

The first group `myCoAKSAdminGroup` is the admin one. It will only contain users with the capability of managing all the resources in the cluster with cluster-admin permissions:

```bash
CoAKS_ADMIN_GROUP_OBJECTID=$(az ad group create \
  --display-name myCoAKSAdminGroup \
  --mail-nickname myCoAKSAdminGroup \
  --query id \
  --output tsv)
```

Assign a user to the group `myCoAKSAdminGroup` that will be the CoAKS cluster admin:

```bash
CoAKS_ADMIN_USER_NAME="admin@energycorp.com"
CoAKS_ADMIN_USER_PASSWORD="ChangeMe123#"

CoAKS_ADMIN_USER_OBJECTID=$(az ad user create \
  --display-name ${CoAKS_ADMIN_USER_NAME} \
  --user-principal-name ${CoAKS_ADMIN_USER_NAME} \
  --password ${CoAKS_ADMIN_USER_PASSWORD} \
  --query id -o tsv)

az ad group member add \
  --group myCoAKSAdminGroup \
  --member-id $CoAKS_ADMIN_USER_OBJECTID
```

### CoAKS Solar Group

The second step is to create a group for the `Solar` business unit:

```bash
CoAKS_SOLAR_GROUP_OBJECTID=$(az ad group create \
  --display-name myCoAKSSolarGroup \
  --mail-nickname myCoAKSSolarGroup \
  --query id \
  --output tsv)
```

> This group will be used later during the `Solar` tenant creation.

Assign a user to the group `myCoAKSSolarGroup` that will act as `Solar` tenant owner:

```bash
CoAKS_SOLAR_USER_NAME="alice@energycorp.com"
CoAKS_SOLAR_USER_PASSWORD="ChangeMe123#"

CoAKS_SOLAR_USER_OBJECTID=$(az ad user create \
  --display-name ${CoAKS_SOLAR_USER_NAME} \
  --user-principal-name ${CoAKS_SOLAR_USER_NAME} \
  --password ${CoAKS_SOLAR_USER_PASSWORD} \
  --query id -o tsv)

az ad group member add \
  --group myCoAKSSolarGroup \
  --member-id $CoAKS_SOLAR_USER_OBJECTID
```

> This user will act as CoAKS `Solar` tenant owner once he/she logged into Azure.

### CoAKS Eolic Group

The third step is to create an Azure AD group for the `Eolic` business unit:

```bash
CoAKS_EOLIC_GROUP_OBJECTID=$(az ad group create \
  --display-name myCoAKSEolicGroup \
  --mail-nickname myCoAKSEolicGroup \
  --query id \
  --output tsv)
```

> This group will be used later during the `Eolic` tenant creation.

Assign a user to the group `myCoAKSEolicGroup` that will act as `Eolic` tenant owner:

```bash
CoAKS_EOLIC_USER_NAME="bob@energycorp.com"
CoAKS_EOLIC_USER_PASSWORD="ChangeMe123#"

CoAKS_EOLIC_USER_OBJECTID=$(az ad user create \
  --display-name ${CoAKS_EOLIC_USER_NAME} \
  --user-principal-name ${CoAKS_EOLIC_USER_NAME} \
  --password ${CoAKS_EOLIC_USER_PASSWORD} \
  --query id -o tsv)

az ad group member add \
  --group myCoAKSEolicGroup \
  --member-id $CoAKS_EOLIC_USER_OBJECTID
```

> This user will act as CoAKS `Eolic` tenant owner once he/she logged into Azure.

### CoAKS Capsule Group

All the users and groups operating as tenant owners have to be subgroups of a `Capsule Group` called `myCoAKSCapsuleGroup`:

```bash
# Group Creation 
az ad group create \
  --display-name myCoAKSCapsuleGroup \
  --mail-nickname myCoAKSCapsuleGroup

# Solar Group Assignation
az ad group member add \
  --group myCoAKSCapsuleGroup \
  --member-id $CoAKS_SOLAR_GROUP_OBJECTID

# Eolic Group Assignation
az ad group member add \
  --group myCoAKSCapsuleGroup \
  --member-id $CoAKS_EOLIC_GROUP_OBJECTID
```

## AKS

### Create Resource Group

Every infrastructure resource needs to be provisioned under a resource group in a chosen location.

```bash
az group create --name myCoAKSResourceGroup --location <region>
```

### Create AKS Cluster

Having a resource group and the MS Entra ID groups ready, the administrator creates the cluster:

```bash
CoAKS_ADMIN_GROUP_OBJECTID=$(az ad group show \
  --group myCoAKSAdminGroup \
  --query id \
  --output tsv)
```

The following Azure CLI command creates an AKS cluster with:

- Public access to the kubernetes API server
- System node pool spread across 3 [availability zones](https://docs.microsoft.com/en-ushttps://docs.microsoft.com/en-us/azure/availability-zones/az-overview) for best intra-region resiliency
- [Entra ID Integration](https://docs.microsoft.com/en-us/azure/aks/managed-aad)
- [Azure RBAC for Kubernetes Authorization](https://docs.microsoft.com/en-us/azure/aks/manage-azure-rbac)

```bash

KUBERNETES_VERSION=1.28.12

az aks create \
  --resource-group myCoAKSResourceGroup \
  --name myCoAKSCluster \
  --kubernetes-version $KUBERNETES_VERSION \
  --enable-aad \
  --enable-azure-rbac \
  --zones 1 2 3 \
  --aad-admin-group-object-ids $CoAKS_ADMIN_GROUP_OBJECTID
```

Not all Azure regions support availability zones. For more information, see [Azure locations](https://docs.microsoft.com/en-ushttps://docs.microsoft.com/en-us/azure/availability-zones/az-overview#azure-regions-with-availability-zones). For more information on the Azure CLI command and provisioning options, see [az aks create](https://docs.microsoft.com/en-us/clihttps://docs.microsoft.com/en-us/azure/aks?view=azure-cli-latest#az_aks_create).

If you want to improve cluster security and minimize attacks, create a private AKS cluster or grant access to the API server to a limited set of IP address ranges. For more information, see [Create a private Azure Kubernetes Service cluster](https://docs.microsoft.com/en-ushttps://docs.microsoft.com/en-us/azure/aks/private-clusters) and [Secure access to the API server using authorized IP address ranges in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-ushttps://docs.microsoft.com/en-us/azure/aks/api-server-authorized-ip-ranges). For other best practices, see the [here](#references) section below.


### Get Cluster Credentials

When deploying an AKS cluster, the local account is by default a cluster admin. Once the AKS cluster gets created, you can grab the credentials:

```bash
az aks get-credentials \
  --resource-group myCoAKSResourceGroup \
  --name myCoAKSCluster \
  --overwrite-existing --admin
```

And check it out:

```bash
$ kubectl cluster-info

Kubernetes control plane is running at https://mycoaksclu-mycoaksresourceg-b7175e-d0nq20bj.hcp.<region>.azmk8s.io:443

CoreDNS is running at https://mycoaksclu-mycoaksresourceg-b7175e-d0nq20bj.hcp.<region>.azmk8s.io:443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

Metrics-server is running at https://mycoaksclu-mycoaksresourceg-b7175e-d0nq20bj.hcp.<region>.azmk8s.io:443/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy

```

### Assign Kubernetes Roles

All the users belonging to `myCoAKSAdminGroup` will be cataloged as users against CoAKS cluster with admin permissions:

```bash
CoAKS_CLUSTER_ID=$(az aks show \
  --resource-group myCoAKSResourceGroup \
  --name myCoAKSCluster \
  --query id -o tsv)

CoAKS_ADMIN_GROUP_OBJECTID=$(az ad group show \
  --group myCoAKSAdminGroup \
  --query id \
  --output tsv)

az role assignment create \
  --assignee $CoAKS_ADMIN_GROUP_OBJECTID \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope $CoAKS_CLUSTER_ID

az role assignment create \
  --assignee $CoAKS_ADMIN_GROUP_OBJECTID \
  --role "Azure Kubernetes Service Cluster Admin Role" \
  --scope $CoAKS_CLUSTER_ID
```

All the users belonging to `myCoAKSCapsuleGroup` will be cataloged as users against the CoAKS cluster without any specific permissions:

```bash
CoAKS_CAPSULE_GROUP_OBJECTID=$(az ad group show \
  --group myCoAKSCapsuleGroup \
  --query id \
  --output tsv)

az role assignment create \
  --assignee $CoAKS_CAPSULE_GROUP_OBJECTID \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope $CoAKS_CLUSTER_ID
```


### Disable local account (optional)

Optionally, Azure offers the ability to disable the local account to get cluster-admin permissions:

```bash
az aks update  \
  --resource-group myCoAKSResourceGroup \
  --name myCoAKSCluster \
  --disable-local-accounts
```

See [Manage local accounts with AKS-managed Microsoft Entra integration](https://learn.microsoft.com/en-us/azure/aks/manage-local-accounts-managed-azure-ad) for more detailed procedures. 


### Check Admin Permissions

Admin users belonging to `myCoAKSAdminGroup` should have full access to resources in the Cluster:

```bash
kubelogin remove-tokens

az login -u admin@energycorp.com
az aks get-credentials  --resource-group myCoAKSResourceGroup --name myCoAKSCluster --overwrite
```

Grab Cluster info:

```bash
kubectl cluster-info

Kubernetes control plane is running at https://mycoaksclu-mycoaksresourceg-b7175e-d0nq20bj.hcp.<region>.azmk8s.io:443

CoreDNS is running at https://mycoaksclu-mycoaksresourceg-b7175e-d0nq20bj.hcp.<region>.azmk8s.io:443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

Metrics-server is running at https://mycoaksclu-mycoaksresourceg-b7175e-d0nq20bj.hcp.<region>.azmk8s.io:443/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
```

### Check Tenant Permissions

Tenant users belonging to `myCoAKSCapsuleGroup` have no permissions to access resources in the Cluster:

```bash
kubelogin remove-tokens

az login -u alice@energycorp.com
az aks get-credentials  --resource-group myCoAKSResourceGroup --name myCoAKSCluster --overwrite
```

Grab Cluster info:

```bash
kubectl cluster-info
Error from server (Forbidden): services is forbidden: User "alice@energycorp.com" cannot list resource "services" in API group "" in the namespace "kube-system": User does not have access to the resource in Azure. Update role assignment to allow access.
```

We need to give them specific permissions on Cluster's slices, aka Capsule Tenants. 

## References

### Azure Kubernetes Service

- [Best practices for multitenancy and cluster isolation](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-isolation)
- [Best practices for basic scheduler features in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-scheduler)
- [Best practices for advanced scheduler features](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-advanced-scheduler)
- [Best practices for authentication and authorization](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-advanced-scheduler)
- [Best practices for cluster security and upgrades in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-security)
- [Best practices for container image management and security in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-container-image-management)
- [Best practices for network connectivity and security in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-network)
- [Best practices for storage and backups in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-storage)
- [Best practices for business continuity and disaster recovery in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-multi-region)
- [Azure Kubernetes Services (AKS) day-2 operations guide](https://docs.microsoft.com/en-us/azure/architecture/operator-guides/aks/day-2-operations-guide)

### Architectural guidance

- [Azure Kubernetes Service (AKS) solution journey](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks-start-here)
- [AKS cluster best practices](https://docs.microsoft.com/en-us/azure/aks/best-practices?toc=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fazure%2Farchitecture%2Ftoc.json&bc=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fazure%2Farchitecture%2Fbread%2Ftoc.json)
- [Azure Kubernetes Services (AKS) day-2 operations guide](https://docs.microsoft.com/en-us/azure/architecture/operator-guides/aks/day-2-operations-guide)
- [Choosing a Kubernetes at the edge compute option](https://docs.microsoft.com/en-us/azure/architecture/operator-guides/aks/choose-kubernetes-edge-compute-option)
- [Create a private Azure Kubernetes Service cluster](https://github.com/paolosalvatori/private-aks-cluster)
- [Create a private Azure Kubernetes Service cluster using Terraform and Azure DevOps](https://github.com/paolosalvatori/private-aks-cluster-terraform-devops)
- [Create an Azure Kubernetes Service cluster with the Application Gateway Ingress Controller](https://github.com/paolosalvatori/aks-agic)

### Reference architectures

- [Baseline architecture for an Azure Kubernetes Service (AKS) cluster](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks/secure-baseline-aks)
- [Microservices architecture on Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks-microservices/aks-microservices)
- [Advanced Azure Kubernetes Service (AKS) microservices architecture](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks-microservices/aks-microservices-advanced)
- [CI/CD pipeline for container-based workloads](https://docs.microsoft.com/en-us/azure/architecture/example-scenario/apps/devops-with-aks)
- [Building a telehealth system on Azure](https://docs.microsoft.com/en-us/azure/architecture/example-scenario/apps/telehealth-system)


### Role Based Access Control
- [Control access to cluster resources using Kubernetes role-based access control and Azure Active Directory identities in Azure Kubernetes Service](https://docs.microsoft.com/en-us/azure/aks/azure-ad-rbac)
- [Use Azure RBAC for Kubernetes Authorization](https://docs.microsoft.com/en-us/azure/aks/manage-azure-rbac).
- [Use Azure role-based access control to define access to the Kubernetes configuration file in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/control-kubeconfig-access)


## What’s next

We are ready to [Install Capsule](02-capsule-installation.md).