## Multitenant Environment

Although the company wants to decrease the total cost of infrastructure, every business unit has their own needs and they have to be isolated between them.

We will define the Azure AD groups `myCoAKSSolarGroup` and `myCoAKSEolicGroup` as tenant owners of `solar` and `eolic` tenants. As we have already did before for the `myCoAKSCapsuleGroup`, we have to register them using their `Object ID`s instead of their names.

### Solar Tenant

```bash
coaks-admin$ kubectl apply -f - << EOF
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
coaks-admin$ kubectl apply -f - << EOF
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
coaks-admin$ kubectl get tenants.capsule.clastix.io 
NAME   STATE    NAMESPACE QUOTA   NAMESPACE COUNT   NODE SELECTOR   AGE
eolic    Active                     0                                 1m
solar    Active                     0                                 1m
```

## Tenant's Namespace Creation

A tenant owner will be able to create Kubernetes namespaces only within its tenant.

In order to start, we have to use an user that belongs to a Tenant Group, `myCoAKSSolarGroup` or `myCoAKSEolicGroup`, and request credentials:

```bash
$ az aks get-credentials --resource-group myCoAKSResourceGroup --name myCoAKSCluster --overwrite-existing
Merged "myCoAKSCluster" as current context in /home/alice/.kube/config
```

and tenant owners can now create their own namespaces: 

```bash
solar-user$ kubectl create ns solar-production
namespace/solar-production created

solar-user$ kubectl create ns solar-staging
namespace/solar-staging created

solar-user$ kubectl create ns solar-testing
namespace/solar-testing created
```

If we check the tenant using an admin account, we will see the 3 namespaces:

```bash
coaks-admin$ kubectl get tenants.capsule.clastix.io solar -o custom-columns=NAME:.metadata.name,STATE:.status.state,NAMESPACES:.status.namespaces
NAME   STATE    NAMESPACES
solar    Active   [solar-production solar-staging solar-testing]
```
