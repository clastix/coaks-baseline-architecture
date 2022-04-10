# Capsule Installation

Having our AKS cluster up and running and our `kubectl` has access to the API as **Energy Corp's PaaS** cluster administrator, i.e. `coaks-admin@energycorp.com`, we can go on with the **Capsule Operator** installation. 

## 1. Namespace creation

A dedicated namespaced will be created for Capsule's objects and the installation will be done using Helm although other methods are also available.

```bash
coaks-admin$ kubectl create namespace capsule-system
namespace/capsule-system created
```

## 2. Add Helm Chart Repository

```bash
coaks-admin$ helm repo add clastix https://clastix.github.io/charts
"clastix" has been added to your repositories
```

## 3. Installing Capsule Helm Chart

Capsule needs to know the allowed groups it will work with, therefore, we need to register the `Object ID` of the Azure AD group `myCoAKSCapsuleGroup` as Capsule User Group under the `CapsuleConfiguration`:

```bash
coaks-admin$ helm install capsule clastix/capsule \
   -n capsule-system \
   --set manager.options.force_tenant_prefix=true \
   --set manager.options.capsuleUserGroups=$CoAKS_CAPSULE_GROUP_OBJECTID
```

## 4. Installing Capsule-Proxy Helm Chart

Optionally, install the [Capsule Proxy](https://github.com/clastix/capsule-proxy), an add-on for the Capsule Operator. It allows to overcome the limitations of Kubernetes API Server on listing owned cluster-scoped resources, like _Namespaces_, _Ingress Classes_, _Storage Classes_, _Nodes_, and others covered by Capsule.

```bash
coaks-admin$ helm install capsule-proxy clastix/capsule-proxy -n capsule-system
```

## What’s next

**Energy Corp's PaaS** cluster administrator can start to set up the [multitenance environment](multitenance-environment.md).
