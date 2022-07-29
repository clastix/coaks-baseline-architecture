## Multitenant Environment

In this section, we're going to create a minimal multi-tenant environment according to our example company **Energy Corp** needs. The two lines of business **Solar** and **Eolic** have a team of engineers that are responsible for the development and the operations of their digital products.

We will define the Azure AD groups `myCoAKSSolarGroup` and `myCoAKSEolicGroup` as tenant owners of `solar` and `eolic` tenants, respectively. As we have already done before for the `myCoAKSCapsuleGroup`, we have to register them using their `Object ID`s instead of their names.

Login as cluster-admin:

```bash
$ az login
$ az aks get-credentials --resource-group myCoAKSResourceGroup --name myCoAKSCluster --overwrite-existing
```

### Solar Tenant

```bash
$ kubectl apply -f - << EOF
apiVersion: capsule.clastix.io/v1beta1
kind: Tenant
metadata:
  name: solar
spec:
  owners:
  - name: ${CoAKS_SOLAR_USER_OBJECTID}
    kind: Group
EOF
tenant.capsule.clastix.io/solar created
```

### Eolic Tenant

```bash
$ kubectl apply -f - << EOF
apiVersion: capsule.clastix.io/v1beta1
kind: Tenant
metadata:
  name: eolic
spec:
  owners:
  - name: ${CoAKS_EOLIC_USER_OBJECTID}
    kind: Group
EOF
tenant.capsule.clastix.io/eolic created
```

### Check Tenants

```bash
$ kubectl get tenants.capsule.clastix.io 
NAME   STATE    NAMESPACE QUOTA   NAMESPACE COUNT   NODE SELECTOR   AGE
eolic  Active                     0                                 1m
solar  Active                     0                                 1m
```

## Setting Tenant's environment

The tenant owner will be able to operate in the **Energy Corp's PaaS** autonomously. Let's to set up the access to CoAKS platform for one of the tenants, for example, **Solar**. Login into Azure as a user that belongs to `myCoAKSSolarGroup` and request the credentials:

```bash
$ az login
$ az aks get-credentials --resource-group myCoAKSResourceGroup --name myCoAKSCluster
Merged "myCoAKSCluster" as current context in /home/alice/.kube/config
```

and tenant owners can now create their namespaces: 

```bash
$ kubectl create ns solar-production
namespace/solar-production created

$ kubectl create ns solar-staging
namespace/solar-staging created

$ kubectl create ns solar-testing
namespace/solar-testing created
```

Now let's to list all the namespaces:

```bash
$ kubectl get namespaces
Error from server (Forbidden): namespaces are forbidden:
User "alice@energycorp.com" cannot list resource "namespaces" in API group "" at the cluster scope:
The user does not have access to the resource in Azure. Update role assignment to allow access.
```

That's expected since Kubernetes lacks ACL-filtered APIs. To overcome this limitation, use the Capsule Proxy acting as a gatekeeper in front of the CoAKS APIs server.

Setup a dedicated context for accessing APIs server through the Capsule Proxy:

```bash
$ kubectl config set-cluster coaks \
    --server=https://coaks.<region>.cloudapp.azure.com:443

$ kubectl config set-context coaks \
    --cluster=coaks \
    --user=$(kubectl config get-users | tail -1)

$ kubectl config use-context coaks
```

and check the list of owned namespaces:

```bash
$ kubectl get namespaces 

NAME                STATUS   AGE
solar-production    Active   1h
solar-staging       Active   1h
solar-testing       Active   1h
```

## What’s next

[Isolate tenants](isoalte-tenants.md).