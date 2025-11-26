<p align="center">
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/graphs/traffic"><img src="https://api.visitorbadge.io/api/visitors?path=https%3A%2F%2Fgithub.com%2Fsimon-verbois%2Fmicrok8s-config-traefik&label=Visitors&countColor=26A65B&style=flat" alt="Visitor Count" height="28"/></a>
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/commits/main"><img src="https://img.shields.io/github/last-commit/simon-verbois/microk8s-config-traefik?style=flat" alt="GitHub Last Commit" height="28"/></a>
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/stargazers"><img src="https://img.shields.io/github/stars/simon-verbois/microk8s-config-traefik?style=flat&color=yellow" alt="GitHub Stars" height="28"/></a>
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/issues"><img src="https://img.shields.io/github/issues/simon-verbois/microk8s-config-traefik?style=flat&color=red" alt="GitHub Issues" height="28"/></a>
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/pulls"><img src="https://img.shields.io/github/issues-pr/simon-verbois/microk8s-config-traefik?style=flat&color=blue" alt="GitHub Pull Requests" height="28"/></a>
</p>

# Traefik with Coraza WAF on MicroK8s

Production-ready Traefik setup on MicroK8s featuring Coraza WAF, Geoblocking, Security Headers, and Let's Encrypt automation via OVH.

**Features:**

  * **WAF:** Coraza with custom ruleset (SQLi, XSS, Scanners).
  * **Access Control:** Geo-blocking and IP whitelisting.
  * **Hardening:** OWASP security headers and modern TLS profile.
  * **Automation:** ACME DNS-01 challenge (OVH) and HTTP-to-HTTPS redirect.

## Prerequisites

  * **Cluster:** MicroK8s installed and running.
  * **Network:** MetalLB enabled (LoadBalancer).
  * **Provider:** OVH domain and API credentials (`GET/POST/PUT/DELETE` on `/domain/zone/*`).
  * **System:** Non-root user in `microk8s` group.

## Installation

### 1\. Cluster Preparation

Enable required addons and create the namespace.

```bash
microk8s enable helm3 metallb
microk8s kubectl create namespace traefik
```

### 2\. Credentials & Plugins

Create the OVH secret (replace placeholders) and download the required WAF/GeoBlock plugins locally.

```bash
# 1. Create Secret
microk8s kubectl create secret generic ovh-credentials \
  --namespace=traefik \
  --from-literal=OVH_ENDPOINT=ovh-eu \
  --from-literal=OVH_APPLICATION_KEY=CHANGE_ME \
  --from-literal=OVH_APPLICATION_SECRET=CHANGE_ME \
  --from-literal=OVH_CONSUMER_KEY=CHANGE_ME

# 2. Download Plugins
mkdir -p ./src/github.com/jcchavezs/coraza-http-wasm-traefik
mkdir -p ./src/github.com/PascalMinder/geoblock

cd ./src/github.com/jcchavezs/coraza-http-wasm-traefik
wget https://github.com/jcchavezs/coraza-http-wasm-traefik/releases/download/v0.3.0/coraza-http-wasm-v0.3.0.zip
unzip coraza-http-wasm-v0.3.0.zip && rm coraza-http-wasm-v0.3.0.zip

cd ../../PascalMinder/geoblock
wget https://github.com/PascalMinder/geoblock/archive/refs/tags/v0.3.3.zip
unzip v0.3.3.zip && rm v0.3.3.zip && mv geoblock-*/* . && rmdir geoblock-*

cd ../../../..
sudo chown -R 65532:65532 ./src
```

### 3\. Deploy Traefik

Install Traefik using the provided Helm values.

```bash
microk8s helm3 repo add traefik https://traefik.github.io/charts
microk8s helm3 repo update
microk8s helm3 install traefik traefik/traefik -n traefik --create-namespace -f values.yaml
```

### 4\. Apply Security Policies

Deploy the middlewares (WAF, GeoBlock, Headers, Whitelist) and TLS options.
*Edit `security-middleware.yaml` to customize allowed countries and IPs before applying.*

```bash
microk8s kubectl apply -f security-middleware.yaml
```

### 5\. Validation

Deploy the test application (`whoami`) secured by the WAF.
*Edit `whoami-waf-test.yaml` with your domain before applying.*

```bash
microk8s kubectl apply -f whoami-waf-test.yaml
```

## Testing & Verification

Wait for the certificate generation, then verify the protection.

```bash
# 1. Check Security Headers (Should return 200 OK)
curl -I https://waf-test.example.org

# 2. Test WAF - SQL Injection (Should return 403 Forbidden)
curl -I "https://waf-test.example.org/?id=1%27%20OR%20%271%27=%271"

# 3. Test WAF - XSS (Should return 403 Forbidden)
curl -I "https://waf-test.example.org/?input=<script>alert(1)</script>"

# 4. Test WAF - User Agent (Should return 403 Forbidden)
curl -I -A "sqlmap/1.0" https://waf-test.example.org/
```

## Management

**Upgrade Configuration:**

```bash
microk8s helm3 upgrade traefik traefik/traefik -f values.yaml -n traefik
```

**Debug Logs:**

```bash
microk8s kubectl -n traefik logs -f deployment/traefik
```
