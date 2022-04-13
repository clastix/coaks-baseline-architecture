# Capsule Installation

Having our AKS cluster up and running and our `kubectl` has access to the API as **Energy Corp's PaaS** cluster administrator, i.e. `coaks-admin@energycorp.com`, we can go on with the **Capsule Operator** installation. 

Login as cluster admin:

```bash
$ az login
$ az aks get-credentials --resource-group myCoAKSResourceGroup --name myCoAKSCluster
```

## Namespace creation

A dedicated namespaced will be created for Capsule's objects and the installation will be done using Helm although other methods are also available.

```bash
$ kubectl create namespace capsule-system
namespace/capsule-system created
```

## Add Helm Chart Repository

```bash
$ helm repo add clastix https://clastix.github.io/charts
"clastix" has been added to your repositories
```

## Installing Capsule Helm Chart

Capsule needs to know the allowed groups it will work with, therefore, we need to register the `Object ID` of the Azure AD group `myCoAKSCapsuleGroup` as Capsule User Group under the `CapsuleConfiguration`:

```bash
$ helm install capsule clastix/capsule \
   -n capsule-system \
   --set manager.options.forceTenantPrefix=true \
   --set manager.options.capsuleUserGroups[0]=$CoAKS_CAPSULE_GROUP_OBJECTID
```

## Installing Capsule-Proxy Helm Chart

Install the [Capsule Proxy](https://github.com/clastix/capsule-proxy), an add-on for the Capsule Operator. It allows to overcome the limitations of Kubernetes API Server on listing owned cluster-scoped resources, like _Namespaces_, _Ingress Classes_, _Storage Classes_, _Nodes_, and others covered by Capsule.

The Capsule Proxy acts as a _gatekeeper_ for tenant users in order to list owned cluster-scoped resources. The tenant users access the APIs server through the Capsule Proxy. Behind the scene, it implements a simple reverse proxy that intercepts only specific requests to the APIs server. All the other requests are proxied transparently to the APIs server for regular RBAC evaluation.

```bash
$ helm install capsule-proxy clastix/capsule-proxy \
   -n capsule-system \
   --set service.type=LoadBalancer \
   --set service.port=443 \
   --set options.oidcUsernameClaim=unique_name
```

The Capsule Proxy will be exposed to tenant users with a LoadBalancer service type and it will be reached as `https://coaks.<region>.cloudapp.azure.com:443`. To achieve this, annotate the service:

```bash
$ kubectl -n capsule-system annotate \
   service capsule-proxy service.beta.kubernetes.io/azure-dns-label-name=coaks
```

Actually, the Capsule Proxy generates a self signed TLS certificate using a fake CA. If you have your own certificate, create a TLS secrets in the same namespace:

```bash
$ kubectl -n capsule-system create secrets tls capsule-proxy \
   --cert=/path/to/certificate/file/tls.crt \
   --key=/path/to/key/file/tls.key
```

and let's Capsule Proxy to use it:

```bash
$ helm upgrade capsule-proxy clastix/capsule-proxy \
   -n capsule-system \
   --set service.type=LoadBalancer \
   --set service.port=443 \
   --set options.oidcUsernameClaim=unique_name \
   --set options.generateCertificates=false
```

## References

### Capsule

- [Capsule](https://capsule.clastix.io)
- [Capsule Proxy](https://capsule.clastix.io/docs/general/proxy)
- [Access and identity options for Azure Kubernetes Service AKS](https://docs.microsoft.com/en-us/azure/aks/concepts-identity)
- [Kubernetes Authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)

## What’s next

**Energy Corp's PaaS** cluster administrator can start to set up the [multitenance environment](multitenance-environment.md).
