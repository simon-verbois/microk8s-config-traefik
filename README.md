<p align="center">
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/graphs/traffic"><img src="https://api.visitorbadge.io/api/visitors?path=https%3A%2F%2Fgithub.com%2Fsimon-verbois%2Fmicrok8s-config-traefik&label=Visitors&countColor=26A65B&style=flat" alt="Visitor Count" height="28"/></a>
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/commits/main"><img src="https://img.shields.io/github/last-commit/simon-verbois/microk8s-config-traefik?style=flat" alt="GitHub Last Commit" height="28"/></a>
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/stargazers"><img src="https://img.shields.io/github/stars/simon-verbois/microk8s-config-traefik?style=flat&color=yellow" alt="GitHub Stars" height="28"/></a>
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/issues"><img src="https://img.shields.io/github/issues/simon-verbois/microk8s-config-traefik?style=flat&color=red" alt="GitHub Issues" height="28"/></a>
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/pulls"><img src="https://img.shields.io/github/issues-pr/simon-verbois/microk8s-config-traefik?style=flat&color=blue" alt="GitHub Pull Requests" height="28"/></a>
</p>

# Note
- This implementation has been tested on RHEL 9
- All this implementation has been done with a non root user (in microk8s group)
- An ingress example is available, but there is not real testing in the repository for Traefik as you need a full app to test
- You need a load balancer setup (MetalLB in my case)

<br>

# Install Traefik OVH TLS Resolver
```bash
# Enable the modules
microk8s enable helm3 metallb
## The LoadBalancer will ask for an IP Range
## Add a range that's in your cluster subnet

# Check all pods status
microk8s kubectl create namespace traefik

# Create a secret in the traefik namespace we just created
# Create your API credentials here: [https://eu.api.ovh.com/createToken/](https://eu.api.ovh.com/createToken/)
# GET /domain/zone/*
# POST /domain/zone/*
# PUT /domain/zone/*
# DELETE /domain/zone/*
microk8s kubectl create secret generic ovh-credentials \
  --namespace=traefik \
  --from-literal=OVH_ENDPOINT=ovh-eu \
  --from-literal=OVH_APPLICATION_KEY=xxxxxxxxxxxxxxxxx \
  --from-literal=OVH_APPLICATION_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx \
  --from-literal=OVH_CONSUMER_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxx

## Add Traefik repository and update the cache
microk8s helm3 repo add traefik [https://traefik.github.io/charts](https://traefik.github.io/charts) && microk8s helm3 repo update

## Copy template, and customize it
mv values.yaml.template values.yaml

## Install Traefik
microk8s helm3 install traefik traefik/traefik \
  --namespace=traefik \
  --create-namespace \
  -f values.yaml

## Add traefik CRD
microk8s kubectl apply -f https://raw.githubusercontent.com/traefik/traefik-helm-chart/master/traefik/crds/traefik.io_ingressroutes.yaml
microk8s kubectl apply -f https://raw.githubusercontent.com/traefik/traefik-helm-chart/master/traefik/crds/traefik.io_middlewares.yaml
microk8s kubectl apply -f https://raw.githubusercontent.com/traefik/traefik-helm-chart/master/traefik/crds/traefik.io_serverstransports.yaml
microk8s kubectl apply -f https://raw.githubusercontent.com/traefik/traefik-helm-chart/master/traefik/crds/traefik.io_tlsoptions.yaml
microk8s kubectl apply -f https://raw.githubusercontent.com/traefik/traefik-helm-chart/master/traefik/crds/traefik.io_tlsstores.yaml
microk8s kubectl apply -f https://raw.githubusercontent.com/traefik/traefik-helm-chart/master/traefik/crds/traefik.io_traefikservices.yaml
````

<br>

## Configuration Overview (values.yaml)

The `values.yaml` file configures the Traefik deployment. Here is a breakdown of the key sections:

  * `env` / `envFrom`: Sets the container's timezone to `Europe/Brussels` and injects the OVH API credentials from the `ovh-credentials` Kubernetes secret.
  * `certificatesResolvers`: Defines the Let's Encrypt resolver named `ovhresolver`. It's configured to:
      * Use `admin@verbois.ovh` as the ACME contact email.
      * Store certificates in `/data/acme.json` (inside the persistent volume).
      * Use the OVH DNS-01 challenge provider.
  * `providers`: Enables both the modern `kubernetesCRD` (for `IngressRoute` objects) and the legacy `kubernetesIngress` (for standard `Ingress` objects).
  * `additionalArguments`: Passes static configuration flags to the Traefik pod on startup.
      * `--entrypoints.web.address=:8000`: The pod listens for HTTP on port 8000.
      * `--entrypoints.websecure.address=:8443`: The pod listens for HTTPS on port 8443.
      * `--entrypoints.web.http.redirections...`: Sets up a global redirect from HTTP (web) to HTTPS (websecure) on port 443.
      * `--api.dashboard=false`: Disables the insecure Traefik dashboard.
  * `entrypoints`: Configures properties on the entrypoints defined in `additionalArguments`.
      * `websecure.http.compress: {}`: Enables default (Gzip) compression on all HTTPS traffic.
      * `websecure.tls.sniStrict: true`: Enables Strict SNI, enhancing security by rejecting requests that do not specify a server name.
  * `persistence`: Ensures certificate data (`acme.json`) is persistent.
      * Creates a 1Gi Persistent Volume Claim using the `hostpath-sc` storage class, mounted at `/data`.
  * `service`: Defines the Kubernetes `LoadBalancer` service that exposes Traefik.
      * Maps the host's port **80** to the pod's port **8000** (`web`).
      * Maps the host's port **443** to the pod's port **8443** (`websecure`).
  * `deployment`: Sets the deployment strategy to `Recreate` (the old pod is killed before the new one is created).

<br>

# Updating traefik and configuration

```bash
microk8s helm3 upgrade traefik traefik/traefik -f values.yaml -n traefik
```

<br>

# Check if everything is OK

```bash
# Check if the pod in running correctly
microk8s kubectl get pods -n traefik

# Get Traefik service (listening IP is here)
microk8s kubectl get svc -n traefik

# Check Traefik logs
microk8s kubectl -n traefik logs -f -l app.kubernetes.io/name=traefik
```

<br>

# IngressRoute example

This example shows how to configure an `IngressRoute` with OWASP-recommended security headers, a rate limit, and a modern TLS profile.

```yaml
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: example-security-headers
  namespace: example # <-- Change to your app's namespace
spec:
  headers:
    frameDeny: true
    contentTypeNosniff: true
    stsSeconds: 31536000
    stsIncludeSubdomains: true
    stsPreload: true
    customResponseHeaders:
      Content-Security-Policy: "script-src 'self'"
      Referrer-Policy: "strict-origin-when-cross-origin"
      Permissions-Policy: "camera=(), microphone=(), geolocation=(), payment=(), usb=()"
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: example-ratelimit
  namespace: example # <-- Change to your app's namespace
spec:
  rateLimit:
    average: 100
    period: "1m"
    burst: 50
---
apiVersion: traefik.io/v1alpha1
kind: TLSOption
metadata:
  name: example-tls-profile
  namespace: example # <-- Change to your app's namespace
spec:
  minVersion: VersionTLS12
  cipherSuites:
    - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
    - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
    - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
    - TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
    - TLS_CHACHA20_POLY1305_SHA256
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: example-ingressroute
  namespace: example # <-- Change to your app's namespace
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`example.your-domain.com`) # <-- Change to your domain
      kind: Rule
      services:
        - name: example-service # <-- Change to your app's service name
          port: http # <-- Change to your app's service port name
      middlewares:
        - name: example-security-headers
        - name: example-ratelimit
  tls:
    certResolver: ovhresolver # <-- Uses the resolver from values.yaml
    domains:
      - main: "example.your-domain.com" # <-- Change to your domain
    options:
      name: example-tls-profile # <-- Links to the TLSOption created above
```
