# Capsule Proxy Installation

Having Capsule deployed we can continue with the [Capsule Proxy](https://github.com/projectcapsule/capsule-proxy) installation, an add-on for the Capsule Operator. It allows to overcome the limitations of Kubernetes API Server on listing owned cluster-scoped resources, like _Namespaces_, _Ingress Classes_, _Storage Classes_, _Nodes_, and others covered by Capsule.

The Capsule Proxy acts as a _gatekeeper_ for tenant users to list owned cluster-scoped resources. The tenant users access the APIs server through the Capsule Proxy. Behind the scene, it implements a simple reverse proxy that intercepts only specific requests to the APIs server. All the other requests are proxied transparently to the APIs server for regular RBAC evaluation.


## HTTPS vs HTTP
Capsule proxy supports https and http. By default, capsule proxy works with HTTPS.

### HTTP
To work with http disable ssl configuration for capsule-proxy
```bash
helm upgrade --install capsule-proxy projectcapsule/capsule-proxy \
   --namespace capsule-system \
   --set crds.install=true \
   --set options.enableSSL=false \
   --set "options.ignoredUserGroups[0]=$CoAKS_ADMIN_GROUP_OBJECTID" 
```

> Note: 0.7.x introduces a new method to manage all capsule-proxy CRDs and their lifecycle. Use `--set crds.install=true` option.

### HTTPS
By default, capsule proxy generates a self-signed TLS certificate using a fake CA. For production environments it is recommended to bring you own valid certificate. To bring your own certificate, create a secret:

```bash
kubectl -n capsule-system create secrets tls capsule-proxy \
   --cert=/path/to/certificate/file/tls.crt \
   --key=/path/to/key/file/tls.key
```

and let's Capsule Proxy to use it.

```bash
helm upgrade capsule-proxy projectcapsule/capsule-proxy \
   --namespace capsule-system \
   --set crds.install=true \
   --set options.generateCertificates=false \
   --set "options.ignoredUserGroups[0]=$CoAKS_ADMIN_GROUP_OBJECTID" 
```
NOTE: Take into consideration that capsule proxy will use by default `"capsule-proxy.fullname" template` as secret name. So if you use another secret name you need to configure `--set options.certificateVolumeName` option to make it work.

## Exposing Capsule Proxy
Capsule Proxy support different ways of exposing the app:

* Ingress
* NodePort Service
* LoadBalance Service
* HostPort
* HostNetwork

### Ingress Controller

To enable the ingress use:

```bash
helm upgrade --install capsule-proxy projectcapsule/capsule-proxy \
   --namespace capsule-system \
   --set crds.install=true \
   --set ingress.enabled=true \
   --set ingress.hosts[0].host="coaks.energycorp.com" \
   --set ingress.hosts[0].paths[0]="/" \
   --set "options.ignoredUserGroups[0]=$CoAKS_ADMIN_GROUP_OBJECTID" 
```

If you want to use HTTPS to connect through your ingress object you can use:

```bash
helm upgrade --install capsule-proxy projectcapsule/capsule-proxy \
   --namespace capsule-system \
   --set crds.install=true \
   --set ingress.enabled=true \
   --set ingress.hosts[0].host="coaks.energycorp.com" \
   --set ingress.hosts[0].paths[0]="/" \
   --set ingress.tls[0].secretName="capsule-proxy-tls" \
   --set ingress.tls[0].hosts[0]="coaks.energycorp.com" \
   --set "options.ignoredUserGroups[0]=$CoAKS_ADMIN_GROUP_OBJECTID" 
```

if you are using SSL enabled in Capsule Proxy you need to redirect the traffic to the https listener:

```bash
helm upgrade --install capsule-proxy projectcapsule/capsule-proxy \
   --namespace capsule-system \
   --set crds.install=true \
   --set ingress.enabled=true \
   --set ingress.hosts[0].host="coaks.energycorp.com" \
   --set ingress.hosts[0].paths[0]="/" \
   --set ingress.tls[0].secretName="capsule-proxy-tls" \
   --set ingress.tls[0].hosts[0]="coaks.energycorp.com" \
   --set ingress.annotations."nginx\.ingress\.kubernetes\.io/backend-protocol"= "HTTPS" \
   --set ingress.annotations."nginx\.ingress\.kubernetes\.io/force-ssl-redirect"= "true" \
   --set options.ignoredUserGroups[0]=$CoAKS_ADMIN_GROUP_OBJECTID 
```

You can use the TLS secret created/generated for the ingress controller to be used by the Capsule Proxy backend by adding: 

```bash
helm upgrade --install capsule-proxy projectcapsule/capsule-proxy \
   --namespace capsule-system \
   --set crds.install=true \
   --set ingress.enabled=true \
   --set ingress.hosts[0].host="coaks.energycorp.com" \
   --set ingress.hosts[0].paths[0]="/" \
   --set ingress.tls[0].secretName="capsule-proxy-tls" \
   --set ingress.tls[0].hosts[0]="coaks.energycorp.com" \
   --set ingress.annotations."nginx\.ingress\.kubernetes\.io/backend-protocol"= "HTTPS" \
   --set ingress.annotations."nginx\.ingress\.kubernetes\.io/force-ssl-redirect"= "true" \
   --set options.certificateVolumeName="capsule-proxy-tls" \
   --set options.ignoredUserGroups[0]=$CoAKS_ADMIN_GROUP_OBJECTID
```

Then you can use this certificate CA to connect using kubectl.

### LoadBalancer

Use:

```bash
helm upgrade --install capsule-proxy projectcapsule/capsule-proxy \
   --namespace capsule-system \
   --set crds.install=true \
   --set service.type=LoadBalancer \
   --set service.port=443 \
   --set options.oidcUsernameClaim=unique_name \
   --set options.ignoredUserGroups[0]=$CoAKS_ADMIN_GROUP_OBJECTID \
   --set options.generateCertificates=true \
   --set options.additionalSANs[0]=coaks.<region>.cloudapp.azure.com
```

The Capsule Proxy can be exposed to tenant users with a LoadBalancer service type and it will be reached as `https://coaks.<region>.cloudapp.azure.com:443`. To achieve this, annotate the service:

```bash
kubectl -n capsule-system annotate \
   service capsule-proxy service.beta.kubernetes.io/azure-dns-label-name=coaks
```
## References

### Capsule

- [Capsule](https://projectcapsule.dev/)
- [Capsule Proxy](https://github.com/projectcapsule/capsule-proxy)
- [Access and identity options for Azure Kubernetes Service AKS](https://docs.microsoft.com/en-us/azure/aks/concepts-identity)
- [Kubernetes Authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)

## What’s next

**Energy Corp's PaaS** cluster administrator can start to set up the [multi-tenancy environment](04-multitenant-environment.md).
