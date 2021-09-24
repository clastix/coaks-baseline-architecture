# AKS with AAD integration

Administrator of **Acme Corp's Caas** will have to provide a Kubernetes cluster using Azure [AKS](https://docs.microsoft.com/azure/aks/) and integrates with [AAD](https://azure.microsoft.com/services/active-directory/).

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

The second step is to create an AAD group for the solar business unit (solar tenant).

```bash
$ az ad group create --display-name myCoAKSSolarGroup --mail-nickname myCoAKSSolarGroup
```

### CoAKS Eolic Group

The third step is to create an AAD group for the eolic business unit (eolic tenant).

```bash
$ az ad group create --display-name myCoAKSEolicGroup --mail-nickname myCoAKSEolicGroup
```

### CoAKS Container Group

All the AAD groups that will act as Kubernetes user will be subgroups of a `Container Group`, in this case: `myCoAKSSolarGroup` and `myCoAKSEolicGroup`. This step is to facilitate the future configuration of Capsule.

```bash
# Group Creation 
$ az ad group create --display-name myCoAKSContainerGroup --mail-nickname myCoAKSContainerGroup
# Solar Group Assignation
$ az ad group member add -g myCoAKSContainerGroup  --member-id <Solar Group Object ID>
# Eolic Group Assignation
$ az ad group member add -g myCoAKSContainerGroup  --member-id <Eolic Group Object ID>
```

## AKS

### Create Resource Group

Every infrastructure resource has to be under a resource group which belongs to a chosen location.

```bash
$ az group create --name myCoAKSResourceGroup --location <location>
```

### Create AKS Cluster

Having a resource group and the AAD groups ready, the administrator just needs to create the cluster. Currently, the integration with Active Directory is very simple applying some options during the creation of the cluster where it is important to remember the `Object ID` of `myCoAKSAdminGroup` because this one will be considered the administrator of the cluster:

```bash
$ az aks create -g myCoAKSResourceGroup -n myCoAKSCluster --enable-aad --zones 1 2 3 --aad-admin-group-object-ids <myCoAKSAdminGroup Object ID>
```

[az aks create options](https://docs.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest#az_aks_create)

### Assign Kubernetes User Role to Container Group

`myCoAKSContainerGroup` will be cataloged as `Azure Kubernetes Service Cluster User Role` against our aks cluster. Every member of the group will have access to the Kubernetes API of the cluster.

```bash
$ az role assignment create \
  --assignee <myCoAKSContainerGroup Object ID> \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope $(az aks show -g myCoAKSResourceGroup -n myCoAKSCluster --query id -o tsv)
```

### Access Kubernetes Cluster using kubectl

Kubernetes cluster is ready. Administrator only needs to request their credentials to start to work with `kubectl` and going on with the installation but, it is important to rememeber that Administrator's user has to be under the AAD group `myCoAKSAdminGroup`.

```bash
$ az aks get-credentials --resource-group myCoAKSResourceGroup --name myCoAKSCluster --overwrite-existing
Merged "myCoAKSCluster" as current context in ~/.kube/config
```

> The first request to the API will trigger the AAD authentication

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