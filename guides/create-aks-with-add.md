# AKS with Azure AD integration

Microsoft offers one of the best OIDC provider out there, i.e. [Azure Active Directory](https://azure.microsoft.com/services/active-directory/), therefore we would like to use it in order to provide a secure access to **Energy Corp's PaaS** based on [Clastix Capsule](https://capsule.clastix.io).

The administrator will have to provide a Kubernetes cluster using Azure [AKS](https://docs.microsoft.comhttps://docs.microsoft.com/en-us/azure/aks/) and integrates with Active Directory.

## Azure Log In

The first step is to log into Azure ecosystem using the `az` CLI. It will redirect to [portal Azure](https://portal.azure.com) where you will actually log in.

```bash
az login
```

This logins the Azure admin user to create Azure resources in one of the assigned subscriptions.  

## Active Directory

The Azure admin will create different groups according to the multitenant environment of the **Energy Corp's PaaS**.

### CoAKS Admin Group

The first group `myCoAKSAdminGroup` is the admin one. It will only contain users with the capability of managing the cluster with CoAKS cluster admin permissions:

```bash

```bash
CoAKS_ADMIN_GROUP_OBJECTID=$(az ad group create \
  --display-name myCoAKSAdminGroup \
  --mail-nickname myCoAKSAdminGroup \
  --query objectId \
  --output tsv)
```

> This group will be used later during the CoAKS cluster creation.

Assign a user to the group `myCoAKSAdminGroup` that will be the CoAKS cluster admin:

```bash
CoAKS_ADMIN_USER_NAME="coaks-admin@energycorp.com"
CoAKS_ADMIN_USER_PASSWORD="ChangeMe123#"

CoAKS_ADMIN_USER_OBJECTID=$(az ad user create \
  --display-name ${CoAKS_ADMIN_USER_NAME} \
  --user-principal-name ${CoAKS_ADMIN_USER_NAME} \
  --force-change-password-next-login  \
  --password ${CoAKS_ADMIN_USER_PASSWORD} \
  --query objectId -o tsv)

az ad group member add \
  --group myCoAKSAdminGroup \
  --member-id $CoAKS_ADMIN_USER_OBJECTID
```

> This user will act as CoAKS admin once he/she logged into Azure AD.

### CoAKS Solar Group

The second step is to create an Azure AD group for the `Solar` business unit:

```bash
CoAKS_SOLAR_GROUP_OBJECTID=$(az ad group create \
  --display-name myCoAKSSolarGroup \
  --mail-nickname myCoAKSSolarGroup \
  --query objectId \
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
  --force-change-password-next-login  \
  --password ${CoAKS_SOLAR_USER_PASSWORD} \
  --query objectId -o tsv)

az ad group member add \
  --group myCoAKSSolarGroup \
  --member-id $CoAKS_SOLAR_USER_OBJECTID
```

> This user will act as CoAKS `Solar` tenant owner once he/she logged into Azure AD.

### CoAKS Eolic Group

The third step is to create an Azure AD group for the `Eolic` business unit:

```bash
CoAKS_EOLIC_GROUP_OBJECTID=$(az ad group create \
  --display-name myCoAKSEolicGroup \
  --mail-nickname myCoAKSEolicGroup \
  --query objectId \
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
  --force-change-password-next-login  \
  --password ${CoAKS_EOLIC_USER_PASSWORD} \
  --query objectId -o tsv)

az ad group member add \
  --group myCoAKSEolicGroup \
  --member-id $CoAKS_EOLIC_USER_OBJECTID
```

> This user will act as CoAKS `Eolic` tenant owner once he/she logged into Azure AD.

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
az group create --name myCoAKSResourceGroup --location <location>
```

### Create AKS Cluster

Having a resource group and the Azure AD groups ready, the administrator just needs to create the cluster. Currently, the integration with Active Directory is very simple applying some options during the creation of the cluster where it is important to keep the `Object ID` of `myCoAKSAdminGroup` because this one will be configured as the Azure AD group containing the administrator users of the cluster.

```bash
CoAKS_ADMIN_GROUP_OBJECTID=$(az ad group show \
  --group myCoAKSAdminGroup \
  --query objectId \
  --output tsv)
```

The following Azure CLI command creates an AKS cluster with:

- Public access to the API server
- System node pool spread across 3 [availability zones](https://docs.microsoft.com/en-ushttps://docs.microsoft.com/en-us/azure/availability-zones/az-overview) for best intra-region resiliency
- [Azure AD Integration](https://docs.microsoft.com/en-us/azure/aks/managed-aad)
- [Azure RBAC for Kubernetes Authorization](https://docs.microsoft.com/en-us/azure/aks/manage-azure-rbac)

```bash
az aks create \
  --resource-group myCoAKSResourceGroup \
  --name myCoAKSCluster \
  --enable-aad \
  --enable-azure-rbac \
  --zones 1 2 3 \
  --aad-admin-group-object-ids $CoAKS_ADMIN_GROUP_OBJECTID
```

Not all Azure regions support availability zones. For more information, see [Azure regions with availability zones](https://docs.microsoft.com/en-ushttps://docs.microsoft.com/en-us/azure/availability-zones/az-overview#azure-regions-with-availability-zones). For more information on the Azure CLI command and provisionig options, see [az aks create](https://docs.microsoft.com/en-us/clihttps://docs.microsoft.com/en-us/azure/aks?view=azure-cli-latest#az_aks_create).

If you want to improve cluster security and minimize attacks, create a private AKS cluster or grant the access to the API server to a limited set of IP address ranges. For more information, see [Create a private Azure Kubernetes Service cluster](https://docs.microsoft.com/en-ushttps://docs.microsoft.com/en-us/azure/aks/private-clusters) and [Secure access to the API server using authorized IP address ranges in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-ushttps://docs.microsoft.com/en-us/azure/aks/api-server-authorized-ip-ranges). For other best practices, see the [here](#references) section below.


### Assign Kubernetes Roles

All the users belonging to the `mycoaks-adminGroup` act as cluster admin:

```bash
$ az aks get-credentials --resource-group myCoAKSResourceGroup --name myCoAKSCluster --overwrite-existing
Merged "myCoAKSCluster" as current context in ~/.kube/config
```

The first request to the CoAKS API server will trigger the Azure AD authentication. Login as CoAKS admin user `coaks-admin@energycorp.com` defined above

```bash
$ kubectl get nodes
To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code ******** to authenticate.
NAME                                STATUS   ROLES   AGE    VERSION
aks-nodepool1-12329560-vmss000000   Ready    agent   5m     v1.21.9
aks-nodepool1-12329560-vmss000001   Ready    agent   5m     v1.21.9
aks-nodepool1-12329560-vmss000002   Ready    agent   5m     v1.21.9
```

All the users belonging to `myCoAKSCapsuleGroup` will be cataloged as `Azure Kubernetes Service Cluster User Role` against the CoAKS cluster

```bash
CoAKS_CLUSTER_ID=$(az aks show \
  --resource-group myCoAKSResourceGroup \
  --name myCoAKSCluster \
  --query id -o tsv)

CoAKS_CAPSULE_GROUP_OBJECTID=$(az ad group show \
  --group myCoAKSCapsuleGroup \
  --query objectId \
  --output tsv)

az role assignment create \
  --assignee $CoAKS_CAPSULE_GROUP_OBJECTID \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope $CoAKS_CLUSTER_ID
```

For more information about AKS cluster access control and RBAC, see:

- [Control access to cluster resources using Kubernetes role-based access control and Azure Active Directory identities in Azure Kubernetes Service](https://docs.microsoft.com/en-us/azure/aks/azure-ad-rbac)
- [Use Azure RBAC for Kubernetes Authorization](https://docs.microsoft.com/en-us/azure/aks/manage-azure-rbac).
- [Use Azure role-based access control to define access to the Kubernetes configuration file in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/control-kubeconfig-access)

### Disable local accounts

When deploying an AKS cluster, local accounts are enabled by default as cluster admins. Once the AKS cluster gets created, as local user, you can grab the credentials and check it out:

```bash
az aks get-credentials --resource-group myCoAKSResourceGroup --name myCoAKSCluster --overwrite-existing --admin
Merged "myCoAKSCluster" as current context in ~/.kube/config
```

The `--admin` option, essentially acts as a non-auditable backdoor in the AKS cluster: 

```yaml
kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: REDACTED
  name: myCoAKSCluster
contexts:
- context:
    cluster: myCoAKSCluster
    user: clusterAdmin_myCoAKSResourceGroup_myCoAKSCluster
  name: myCoAKSCluster-admin
current-context: myCoAKSCluster-admin
kind: Config
preferences: {}
users:
- name: clusterAdmin_myCoAKSResourceGroup_myCoAKSCluster
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
    token: REDACTED
```

Azure offers the ability to disable local accounts via a flag, see [Disable local accounts](https://docs.microsoft.com/en-us/azure/aks/managed-aad#disable-local-accounts). 


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