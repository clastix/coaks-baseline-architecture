# Capsule Installation

Having our AKS cluster up and running and our `kubectl` has access to the API as **Energy Corp's PaaS** cluster administrator, i.e. `admin@energycorp.com`, we can go on with the **Capsule Operator** installation. 

Login as cluster admin:

```bash
az login
az aks get-credentials --resource-group myCoAKSResourceGroup --name myCoAKSCluster
```

## Add Capsule Helm Chart Repository

```bash
helm repo add projectcapsule https://projectcapsule.github.io/charts
helm repo update
```

## Installing Capsule Helm Chart

Capsule needs to know the allowed groups it will work with, therefore, we need to register the `Object ID` of the Azure AD group `myCoAKSCapsuleGroup` as Capsule User Group under the `CapsuleConfiguration`:

```bash
helm upgrade --install capsule projectcapsule/capsule \
   --namespace capsule-system --create-namespace \
   --set manager.options.forceTenantPrefix=true \
   --set "manager.options.capsuleUserGroups[0]=$CoAKS_CAPSULE_GROUP_OBJECTID"
```
## References

### Capsule

- [Capsule](https://projectcapsule.dev/)
- [Access and identity options for Azure Kubernetes Service AKS](https://docs.microsoft.com/en-us/azure/aks/concepts-identity)
- [Kubernetes Authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)

## What’s next

We are ready to [install Capsule Proxy](03-capsule-proxy-installation.md).