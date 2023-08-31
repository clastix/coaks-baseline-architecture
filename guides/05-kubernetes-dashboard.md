## Kubernetes dashboard

In the previous section, we saw how to setup **Capsule** to get full advantage of the multi-tenancy in Kubernetes by setting up the **Proxy** addon to get CLI access.

The same multi-tenant access can be achieved from a user-interface such as the [Kubernetes dashboard](https://github.com/kubernetes/dashboard).

### Prerequisites

- An Ingress Controller, for the sake of simplicity the [Ingress-NGINX Controller](https://github.com/kubernetes/ingress-nginx) has been implemented, YMMV using a different one due to different annotations for redirects, etc.
- [oauth2-proxy](https://github.com/oauth2-proxy/oauth2-proxy) to authenticate the user from the browser and pass the authentication token to the Dashboard
- [cert-manager](https://github.com/cert-manager/cert-manager) for the x.509 TLS certificates generation

### Installing the Ingress Controller

To simplify the lifecycle and customization of the installation, a Helm Chart release has been used, ensure to have it in your local cache.

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Install the Ingress Controller according to your needs.

```
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --create-namespace --namespace ingress-nginx
```

By default, the Ingress Controller will be installed using a `LoadBalancer` Service type which is checked by the Azure Load Balancer health-check: to get it reachable on its IP, a successful default backend must be provided.

You can override the default backend using the Helm flag `--set` as follows:

```
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --create-namespace --namespace ingress-nginx \
  --set controller.extraArgs.default-backend-service=default/nginx
```

### Installing cert-manager

`cert-manager` simplifies the generation of TLS certificate, required to secure the inbound connections thanks to encryption.

As with the Ingress Controller, its lifecycle is managed via Helm and the repository must be installed locally.

```
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

It's time to install the `cert-manager`.

```
helm upgrade --install cert-manager bitnami/cert-manager \
    --namespace certmanager-system --create-namespace \
    --set "installCRDs=true"
```

Generation of certificates will be offloaded to Let's Encrypt, thus, a `ClusterIssuer` must be created: your mileage may vary, for our use-case the email address is bound to the administrator of the AKS cluster, and the selected Ingress Class is the `nginx` one, since using the `Ingress-NGINX Controller`.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: coaks-admin@energycorp.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```

### Creating Azure web application for AAD 

The `Kubernetes Dashboard` supports authorization delegation by reading the `Authorization` HTTP header.
Thanks to this, end users can access it using the same credential used by the AKS cluster with AAD enabled.

This is possible by creating an application for your Azure Active Directory: this requires the required Azure permissions and we're giving for granted you have no limitations.

1. Access the **Azure Active Directory** section
2. Access the section **App registrations** and click on **New registration** upper menu link
3. Set the application name (`kubernetes-dashboard`, although YMMV) and the `Redirect URI` to the expected URL of the dashboard (`https://dashboard.energycorp.com/oauth2/callback`, YMMV)
4. Register it by clicking the **Register** button
5. Take note of the resulting **Application (client) ID**, this will be requested when setting up the `oauth2-proxy` application

Once registered, a client secret must be created: access the section **Certificates & secrets**

1. In **Client secrets**, create a new one by clicking **New client secret** by providing a description (e.g.: `oauth2-proxy`) and, optionally, an expiration period
2. Take note of the generated **Secret ID** and **Value**

In managed _Azure AD_, a pair of cross-tenant AAD applications are provided.
The generated `kubeconfig` is a perfect example: `kubectl` uses a public client to let log you into the Azure AD to access an Azure AD resource with ID `6dae42f8-4368-4678-94ff-3960e28e3630`, a managed Azure resource that identifies the **Azure Kubernetes Service AAD Server**.
Once you complete the device code login, `kubectl` sends to the Kubernetes API Server the access token issued to access resource `6dae42f8-4368-4678-94ff-3960e28e3630`: this is akin to how Azure CLI allows accessing Azure, such as using a public client application to request access token to Azure (https://management.azure.com/).

This is made possible by creating an **API permission**:

1. Access the section **API permissions** > **Add a permission**
2. In the side menu, select **APIs my organization uses** and type the ID `6dae42f8-4368-4678-94ff-3960e28e3630` which refers to the **Azure Kubernetes Service AAD Server**
3. Grant and **Add permission** by clicking on the `user.read` one

### Configuring oauth2-proxy

[oauth2-proxy](https://github.com/oauth2-proxy/oauth2-proxy) is a simple reverse proxy that provides authentication using Providers to validate accounts by email, domain or group.

The lifecycle of the required installation is made possible via Helm.

```
helm repo add oauth2-proxy https://oauth2-proxy.github.io/manifests
helm repo update
```

A release can be installed as follows, please, pay attention to the values.

```
helm upgrade --install \
    oauth2-proxy oauth2-proxy/oauth2-proxy \
    -n capsule-system \
    --set image.tag=v7.1.2 \
    --set ingress.enabled=true \
    --set ingress.className=nginx \
    --set ingress.path=/oauth2 \
    --set "ingress.hosts[0]=dashboard.energycorp.com" \
    --set ingress.annotations."cert-manager\.io/cluster-issuer"=letsencrypt-prod \
    --set ingress.annotations."acme\.cert-manager\.io/http01-ingress-class=nginx" \
    --set ingress.annotations."nginx\.ingress\.kubernetes\.io/proxy-buffer-size=16k" \
    --set "ingress.tls[0].hosts[0]=dashboard.energycorp.com" \
    --set "ingress.tls[0].secretName=dashboard-energy-io-tls" \
    --set "extraArgs[0]=--provider=azure" \
    --set "extraArgs[1]=--oidc-issuer-url=https://sts.windows.net/$TENANT_ID/" \
    --set "extraArgs[2]=--email-domain=*" \
    --set "extraArgs[3]=--client-id=$CLIENT_ID" \
    --set "extraArgs[4]=--client-secret=$CLIENT_SECRET" \
    --set "extraArgs[5]=--azure-tenant=$TENANT_ID" \
    --set "extraArgs[6]=--oidc-email-claim=sub" \
    --set "extraArgs[7]=--pass-access-token=true" \
    --set "extraArgs[8]=--resource=6dae42f8-4368-4678-94ff-3960e28e3630" \
    --set "extraArgs[9]=--set-xauthrequest=true"
```

- `$CLIENT_ID` refers to the **Application (client) ID** provided when generating a web application (please, refer to the **Creating Azure web application for AAD** section)
- `$CLIENT_SECRET` refers to the web application client secret value generated in the section **Creating Azure web application for AAD**
- `$TENANT_ID` refers to your Azure tenant ID, it can be retrieved from the **Overview** page of your registered **Azure web application for AAD**

> We're giving for granted that the DNS values for your FQDN `dashboard.energycorp.com` (YMMV) have been correctly configured, otherwise your application will be unaccessible, as well as the authentication callback.

### Configuring kubernetes-dashboard

Finally, we can install the Kubernetes Dashboard, as usual using the provided Helm Chart.

```
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm repo update
```

The dashboard must be hacked a bit in order to point to the Capsule Proxy instance, rather than the default `kubernetes.default.svc` endpoint.

```
export NGINX_CONFIG_SNIPPET=$(cat <<EOF
auth_request_set \$token \$upstream_http_x_auth_request_access_token;
proxy_set_header Authorization "Bearer \$token";
EOF
)


helm upgrade --install \
  kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard \
  --create-namespace -n capsule-system \
  --set "extraVolumes[0].name=token-ca" \
  --set "extraVolumes[0].projected.sources[0].serviceAccountToken.expirationSeconds=86400" \
  --set "extraVolumes[0].projected.sources[0].serviceAccountToken.path=token" \
  --set "extraVolumes[0].projected.sources[1].secret.name=capsule-proxy" \
  --set "extraVolumes[0].projected.sources[1].secret.items[0].key=ca" \
  --set "extraVolumes[0].projected.sources[1].secret.items[0].path=ca.crt" \
  --set "extraVolumeMounts[0].mountPath=/var/run/secrets/kubernetes.io/serviceaccount" \
  --set "extraVolumeMounts[0].name=token-ca" \
  --set "ingress.enabled=true" \
  --set "ingress.className=nginx" \
  --set "ingress.annotations.cert-manager\.io/cluster-issuer=letsencrypt-prod" \
  --set ingress.annotations."acme\.cert-manager\.io/http01-ingress-class=nginx" \
  --set "ingress.annotations.nginx\.ingress\.kubernetes\.io/auth-signin=https://dashboard.energycorp.com/oauth2/start?rd=$escaped_request_uri" \
  --set "ingress.annotations.nginx\.ingress\.kubernetes\.io/auth-url=https://dashboard.energycorp.com/oauth2/auth" \
  --set "ingress.annotations.nginx\.ingress\.kubernetes\.io/auth-response-headers=X-Auth-Request-Access-Token" \
  --set "ingress.annotations.nginx\.ingress\.kubernetes\.io/configuration-snippet=$NGINX_CONFIG_SNIPPET" \
  --set "ingress.hosts[0]=dashboard.energycorp.com" \
  --set "ingress.tls[0].hosts[0]=dashboard.energycorp.com" \
  --set "ingress.tls[0].secretName=dashboard-energy-io-tls" \
  --set "extraEnv[0].name=KUBERNETES_SERVICE_HOST" \
  --set "extraEnv[0].value=capsule-proxy.capsule-system.svc"
```

### Try the whole workflow

You can connect to the URL `https://dashboard.energycorp.com` which will redirect you to the Azure AD.

Users can prompt their credentials and, after confirming the required grants, upon successful login will be redirect to the Kubernetes Dashboard backed by the Capsule Proxy providing the required filtered resources.

### Additional resources

A complete guide in regards to customizing the Kubernetes Dashboard can be found on the [Capsule documentation website](https://capsule.clastix.io/docs/guides/kubernetes-dashboard).
