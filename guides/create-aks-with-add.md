# AKS with Azure AD integration

Administrator of **Acme Corp's Caas** will have to provide a Kubernetes cluster using Azure [AKS](https://docs.microsoft.comhttps://docs.microsoft.com/en-us/azure/aks/) and integrates with [Azure AD](https://azure.microsoft.com/services/active-directory/).

## Azure Log In

The first step is to log into Azure ecosystem using `az cli`. It will redirect to [portal Azure](https://portal.azure.com) where you will actually log in.

```bash
$ az login
```

## Active Directory

Azure offers the best OIDC provider out there, Active Directory, therefore we would like to use it in order to provide a secure access to our AKS cluster. The administrator will create different ADD groups according to their needs.

### CoAKS Admin Group

The first group is the admin one. It will only contain users with the capability of managing the cluster.

```bash
$ az ad group create --display-name myCoAKSAdminGroup --mail-nickname myCoAKSAdminGroup
```

This group will be used later during the AKS cluster creation.

### CoAKS Solar Group

The second step is to create an Azure AD group for the solar business unit (solar tenant).

```bash
$ az ad group create --display-name myCoAKSSolarGroup --mail-nickname myCoAKSSolarGroup
```

### CoAKS Eolic Group

The third step is to create an Azure AD group for the eolic business unit (eolic tenant).

```bash
$ az ad group create --display-name myCoAKSEolicGroup --mail-nickname myCoAKSEolicGroup
```

### CoAKS Container Group

All the Azure AD users and groups that will act as a Kubernetes user will be subgroups of a `Container Group`, in this case: `myCoAKSSolarGroup` and `myCoAKSEolicGroup`. This step is to facilitate the future configuration of Capsule.

```bash
# Group Creation 
$ az ad group create --display-name myCoAKSContainerGroup --mail-nickname myCoAKSContainerGroup

# Solar Group Assignation
$ az ad group member add --resource-group myCoAKSContainerGroup  --member-id <Solar Group Object ID>

# Eolic Group Assignation
$ az ad group member add --resource-group myCoAKSContainerGroup  --member-id <Eolic Group Object ID>
```

## AKS

### Create Resource Group

Every infrastructure resource needs to be provisioned under a resource group in a chosen location.

```bash
$ az group create --name myCoAKSResourceGroup --location <location>
```

### Create AKS Cluster

Having a resource group and the Azure AD groups ready, the administrator just needs to create the cluster. Currently, the integration with Active Directory is very simple applying some options during the creation of the cluster where it is important to remember the `Object ID` of `myCoAKSAdminGroup` because this one will be configured as the Azure AD group containing the administrator users of the cluster. The following Azure CLI command creates an AKS cluster with public access to the API server, a single system node pool spread across 3 [availability zones](https://docs.microsoft.com/en-ushttps://docs.microsoft.com/en-us/azure/availability-zones/az-overview) for best intra-region resiliency. If you want to improve cluster security and minimize attacks, create a private AKS cluster or grant the access to the API server to a limited set of IP address ranges. For more information, see [Create a private Azure Kubernetes Service cluster](https://docs.microsoft.com/en-ushttps://docs.microsoft.com/en-us/azure/aks/private-clusters) and [Secure access to the API server using authorized IP address ranges in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-ushttps://docs.microsoft.com/en-us/azure/aks/api-server-authorized-ip-ranges). For other best practices, see the [here](#references) section below.

```bash
$ az aks create \
  --resource-group myCoAKSResourceGroup \
  --name myCoAKSCluster \
  --enable-Azure AD \
  --zones 1 2 3 \
  --Azure AD-admin-group-object-ids <myCoAKSAdminGroup Object ID>
```
For more information on the Azure CLI command and provisionig options, see [az aks create](https://docs.microsoft.com/en-us/clihttps://docs.microsoft.com/en-us/azure/aks?view=azure-cli-latest#az_aks_create).

> NOTE:
> Not all Azure regions support availability zones. For more information, see [Azure regions with availability zones](https://docs.microsoft.com/en-ushttps://docs.microsoft.com/en-us/azure/availability-zones/az-overview#azure-regions-with-availability-zones).

### Assign Kubernetes Cluster Admin Role to the Admins Group
The following code shows how to:
- create an admin user in Azure AD
- create an group in Azure AD for the AKS cluster administrators
- add the admin users to the AKS cluster administrators group

```bash
# Variables
AKS_NAME="myCoAKSCluster"
AKS_RESOURCE_GROUP_NAME="myCoAKSResourceGroup"
AKS_ADMIN_NAME="aksadminuser"
AKS_ENDUSER_NAME="aksuser"
AKS_ENDUSER_PASSWORD="ChangeMebu0001a0008AdminChangeMe"
K8S_RBAC_AAD_PROFILE_ADMIN_GROUP_NAME="aksclusteradmins"

# Retrieve the AKS resource id
AKS_ID=$(az aks show \
  --resource-group $AKS_RESOURCE_GROUP_NAME \ 
  --name $AKS_NAME \
  --query id \
  --output tsv)

# Create an Admin user 
AKS_ADMIN_OBJECTID=$(az ad user create \
  --display-name $AKS_ADMIN_NAME \
  --user-principal-name $AKS_ADMIN_NAME \
  --force-change-password-next-login  \
  --password $AKS_ENDUSER_PASSWORD \
  --query objectId -o tsv)

# Create the Azure AD group for AKS cluster administrators
K8S_RBAC_AAD_PROFILE_ADMIN_GROUP_OBJECTID=$(az ad group create \
  --display-name ${K8S_RBAC_AAD_PROFILE_ADMIN_GROUP_NAME} \
  --mail-nickname ${K8S_RBAC_AAD_PROFILE_ADMIN_GROUP_NAME} \
  --query objectId -o tsv)

# Add the admin user to the AKS administrators group
az ad group member add \
  --group $K8S_RBAC_AAD_PROFILE_ADMIN_GROUP_NAME \
  --member-id $AKS_ADMIN_OBJECTID

# Assign Azure Kubernetes Service Cluster Admin Role to the AKS administrators group
az role assignment create \
  --role "Azure Kubernetes Service Cluster Admin Role" \
  --assignee $K8S_RBAC_AAD_PROFILE_ADMIN_GROUP_OBJECTID \
  --scope $AKS_ID

# Assign Azure Kubernetes Service RBAC Admin Role to the AKS administrators group
az role assignment create \
  --role "Azure Kubernetes Service RBAC Admin" \
  --assignee $K8S_RBAC_AAD_PROFILE_ADMIN_GROUP_OBJECTID \
  --scope $AKS_ID
```


### Assign Kubernetes User Role to Container Group

`myCoAKSContainerGroup` will be cataloged as `Azure Kubernetes Service Cluster User Role` against our AKS cluster. Every member of the Azure AD group will have access to the Kubernetes API of the cluster with user permissions. For more information about AKS cluster access control and RBAC, see: 
-[Control access to cluster resources using Kubernetes role-based access control and Azure Active Directory identities in Azure Kubernetes Service](https://docs.microsoft.com/en-us/azure/aks/azure-ad-rbac)
- [Use Azure RBAC for Kubernetes Authorization](https://docs.microsoft.com/en-us/azure/aks/manage-azure-rbac).
- [Use Azure role-based access control to define access to the Kubernetes configuration file in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/control-kubeconfig-access)

```bash
$ az role assignment create \
  --assignee <myCoAKSContainerGroup Object ID> \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope $(az aks show -g myCoAKSResourceGroup -n myCoAKSCluster --query id --output tsv)
```

### Access Kubernetes Cluster using kubectl

Kubernetes cluster is ready. Administrator only needs to request their credentials to start to work with `kubectl` and going on with the installation but, it is important to rememeber that Administrator's user has to be under the Azure AD group `myCoAKSAdminGroup`.

```bash
$ az aks get-credentials --resource-group myCoAKSResourceGroup --name myCoAKSCluster --overwrite-existing
Merged "myCoAKSCluster" as current context in ~/.kube/config
```

> The first request to the API will trigger the Azure AD authentication

```bash
$ kubectl get nodes
To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code DJ8AP22W9 to authenticate.
NAME                                STATUS   ROLES   AGE    VERSION
aks-nodepool1-12329560-vmss000000   Ready    agent   5m     v1.20.9
aks-nodepool1-12329560-vmss000001   Ready    agent   5m     v1.20.9
aks-nodepool1-12329560-vmss000002   Ready    agent   5m     v1.20.9
```

## What’s next

We are ready to [install Capsule](capsule-installation.md).

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