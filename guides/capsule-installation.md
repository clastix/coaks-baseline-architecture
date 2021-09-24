# Capsule Installation

Having our AKS cluster up and running and our kubectl has access to the API, we can go on with the capsule installation. 

A dedicated namespaced will be created for capsule's objects and the installation will be done using Helm although other methods are also available.

## 1. Namespace creation

```bash
$ kubectl create namespace capsule-system
namespace/capsule-system created
```

## 2. Add Helm Chart Repository

```bash
$ helm repo add clastix https://clastix.github.io/charts
"clastix" has been added to your repositories
```

## 3. Installing Capsule Helm Chart

```bash
$ helm install capsule clastix/capsule -n capsule-system
NAME: capsule
LAST DEPLOYED: Fri Oct  1 14:52:59 2021
NAMESPACE: capsule-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
- Capsule Operator Helm Chart deployed:

   # Check the capsule logs
   $ kubectl logs -f deployment/capsule-controller-manager -c manager -n capsule-system


   # Check the capsule logs
   $ kubectl logs -f deployment/capsule-controller-manager -c manager -n capsule-system

- Manage this chart:

   # Upgrade Capsule
   $ helm upgrade capsule -f <values.yaml> capsule -n capsule-system

   # Show this status again
   $ helm status capsule -n capsule-system

   # Uninstall Capsule
   $ helm uninstall capsule -n capsule-system
```

[capsule helm chart options](https://github.com/clastix/capsule/tree/master/charts/capsule)

## 4. Installing Capsule-Proxy Helm Chart

```bash
$ helm install capsule-proxy clastix/capsule-proxy -n capsule-system
NAME: capsule-proxy
LAST DEPLOYED: Fri Oct  1 14:54:25 2021
NAMESPACE: capsule-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
- Capsule-proxy Helm Chart deployed:

   # Check the capsule-proxy logs
   $ kubectl logs -f deployment/capsule-proxy -n capsule-system

- Manage this chart:

   # Upgrade capsule-proxy
   $ helm upgrade capsule-proxy -f <values.yaml> capsule-proxy -n capsule-system

   # Show this status again
   $ helm status capsule-proxy -n capsule-system

   # Uninstall capsule-proxy
   $ helm uninstall capsule-proxy -n capsule-system
```

[capsule-proxy helm chart options](https://github.com/clastix/capsule-proxy/blob/master/charts/capsule-proxy)

## 5. Add the Object IDs of the groups to CapsuleConfiguration

Capsule needs to know the allowed groups it will work with, therefore, we need to register the `Object ID` of the group `myCoAKSContainerGroup` under `CapsuleConfiguration`.

```bash
$ kubectl patch capsuleconfiguration -n capsule-system default --type=json -p '[{"op": "add", "path": "/spec/userGroups/1", "value": "<myCoAKSContainerGroup Object ID>"}]'
capsuleconfiguration.capsule.clastix.io/default patched
```

## What’s next

Administrator of Acme Corp's Caas can start to set up the [multitenance environment](multitenance-environment.md).
