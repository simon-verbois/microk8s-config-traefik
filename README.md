<p align="center">
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/graphs/traffic"><img src="https://api.visitorbadge.io/api/visitors?path=https%3A%2F%2Fgithub.com%2Fsimon-verbois%2Fmicrok8s-config-traefik&label=Visitors&countColor=26A65B&style=flat" alt="Visitor Count" height="28"/></a>
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/commits/main"><img src="https://img.shields.io/github/last-commit/simon-verbois/microk8s-config-traefik?style=flat" alt="GitHub Last Commit" height="28"/></a>
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/stargazers"><img src="https://img.shields.io/github/stars/simon-verbois/microk8s-config-traefik?style=flat&color=yellow" alt="GitHub Stars" height="28"/></a>
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/issues"><img src="https://img.shields.io/github/issues/simon-verbois/microk8s-config-traefik?style=flat&color=red" alt="GitHub Issues" height="28"/></a>
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/pulls"><img src="https://img.shields.io/github/issues-pr/simon-verbois/microk8s-config-traefik?style=flat&color=blue" alt="GitHub Pull Requests" height="28"/></a>
</p>

# Traefik on MicroK8s

Production-ready Traefik setup on MicroK8s featuring Crowdsec, Geoblocking, Security Headers, Rate limiting, and Let's Encrypt automation via OVH.

**Features:**

  * **Crowdsec:** Blocking IP based on suspicious content.
  * **Access Control:** Geo-blocking and IP whitelisting.
  * **Hardening:** OWASP security headers and modern TLS profile.
  * **Automation:** ACME DNS-01 challenge (OVH) and HTTP-to-HTTPS redirect.

## Prerequisites

  * **Cluster:** MicroK8s installed and running.
  * **Network:** MetalLB enabled (LoadBalancer).
  * **Provider:** OVH domain and API credentials (`GET/POST/PUT/DELETE` on `/domain/zone/*`).
  * **System:** Non-root user in `microk8s` group, or other authentification method.

### Cluster Preparation

Enable required addons.

```bash
microk8s enable helm3 metallb
```

## Installation

### 1. Credentials & Plugins

Create the OVH secret (replace placeholders) and download the required WAF/GeoBlock plugins locally.

```bash
# Create Secret for OVH resolver
microk8s kubectl create secret generic ovh-credentials \
  --namespace=traefik \
  --from-literal=OVH_ENDPOINT=ovh-eu \
  --from-literal=OVH_APPLICATION_KEY=CHANGE_ME \
  --from-literal=OVH_APPLICATION_SECRET=CHANGE_ME \
  --from-literal=OVH_CONSUMER_KEY=CHANGE_ME

# Install plugins from local source
mkdir -p ./src/github.com/maxlerebourg
mkdir -p ./src/github.com/PascalMinder

cd ./src/github.com/maxlerebourg
wget https://github.com/maxlerebourg/crowdsec-bouncer-traefik-plugin/archive/refs/tags/v1.4.6.zip # Add a more recent version if there is one
unzip v1.4.6.zip && rm -f v1.4.6.zip && mv crowdsec-bouncer-traefik-plugin-* crowdsec-bouncer-traefik-plugin

cd ../../PascalMinder
wget https://github.com/PascalMinder/geoblock/archive/refs/tags/v0.3.3.zip # Add a more recent version if there is one
unzip v0.3.3.zip && rm -f v0.3.3.zip && mv geoblock-* geoblock

cd ../../..
sudo chown -R 65532:65532 ./src
```

### 2. Deploy crowdsec

```bash
# Add repository and update cache
microk8s helm3 repo add crowdsec https://crowdsecurity.github.io/helm-charts
microk8s helm3 repo update

# Create Crowdsec namespace
microk8s kubectl create ns crowdsec

# Edit the values, then install it
microk8s helm3 install crowdsec crowdsec/crowdsec \
  -f crowdsec-values.yaml \
  -n crowdsec

# Wait for the pods to be running
microk8s kubectl get pods -n crowdsec -w

# Generate a LAPI key
microk8s kubectl exec -it -n crowdsec <lapi-pod-name> -- /bin/bash
> cscli bouncers add traefik-bouncer
```

### 3. Apply Security Middlewares

Deploy the middlewares (Crowdsec, GeoBlock, Headers, Whitelist, Rate limit, etc.) and TLS options.
*Edit `security-middleware.yaml` to customize allowed countries, LAPI key and IPs before applying.*

```bash
microk8s kubectl apply -f security-middleware.yaml
```

### 4. Deploy Traefik

Install Traefik using the provided Helm values.

```bash
# Add repository and update cache
microk8s helm3 repo add traefik https://traefik.github.io/charts
microk8s helm3 repo update

# Create Crowdsec namespace
microk8s kubectl create ns traefik

# Edit the values, then install it
microk8s helm3 install traefik traefik/traefik -n traefik --create-namespace -f traefik-values.yaml
```


## Management

**Update/Upgrade Configuration (and helm deployment):**

```bash
# Update helm, then ensure the pod is restarted
microk8s helm3 upgrade traefik traefik/traefik -f traefik-values.yaml -n traefik
kubectl -n traefik rollout restart deployment traefik

# Follow the restart
kubectl -n traefik get pods -w
```

**Debug Logs:**

```bash
microk8s kubectl -n traefik logs -f deployment/traefik
```
