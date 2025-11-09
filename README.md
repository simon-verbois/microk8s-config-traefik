<p align="center">
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/graphs/traffic"><img src="https://api.visitorbadge.io/api/visitors?path=https%3A%2F%2Fgithub.com%2Fsimon-verbois%2Fmicrok8s-config-traefik&label=Visitors&countColor=26A65B&style=flat" alt="Visitor Count" height="28"/></a>
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/commits/main"><img src="https://img.shields.io/github/last-commit/simon-verbois/microk8s-config-traefik?style=flat" alt="GitHub Last Commit" height="28"/></a>
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/stargazers"><img src="https://img.shields.io/github/stars/simon-verbois/microk8s-config-traefik?style=flat&color=yellow" alt="GitHub Stars" height="28"/></a>
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/issues"><img src="https://img.shields.io/github/issues/simon-verbois/microk8s-config-traefik?style=flat&color=red" alt="GitHub Issues" height="28"/></a>
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/pulls"><img src="https://img.shields.io/github/issues-pr/simon-verbois/microk8s-config-traefik?style=flat&color=blue" alt="GitHub Pull Requests" height="28"/></a>
</p>

# Traefik with Coraza WAF on MicroK8s

This repository provides a comprehensive guide and configuration files for deploying **Traefik** on a **MicroK8s** cluster with advanced security features enabled.

This setup is designed to be robust and production-ready, focusing on:

  * **Coraza WAF:** A powerful Web Application Firewall (WAF) integrated via the official Traefik WASM plugin, complete with a custom ruleset.
  * **Security Headers:** A hardened middleware to apply OWASP-recommended security headers.
  * **Rate Limiting:** A middleware to protect services from simple brute-force attacks.
  * **ACME DNS-01 Challenge:** Automatic SSL/TLS certificate generation and renewal from Let's Encrypt using the OVH DNS-01 challenge.
  * **HTTP to HTTPS:** Global redirection for all traffic.

## 📋 Prerequisites

Before you begin, ensure you have the following:

  * A running MicroK8s cluster.
  * The `microk8s` CLI tool installed and configured.
  * An OVH domain name.
  * OVH API credentials with the following permissions:
      * `GET /domain/zone/*`
      * `POST /domain/zone/*`
      * `PUT /domain/zone/*`
      * `DELETE /domain/zone/*`
  * A Load Balancer solution for MicroK8s, such as **MetalLB**.

> **Note:** This implementation has been tested on **RHEL 9** with a **non-root user** (who is part of the `microk8s` group).

-----

## 🚀 Installation Guide

### Step 1: Prepare MicroK8s Cluster

First, enable the required MicroK8s addons and create a namespace for Traefik.

```bash
# Enable Helm 3 (for installing Traefik) and MetalLB
microk8s enable helm3 metallb

# When prompted, provide an IP range for MetalLB
# This range should be on your cluster's subnet.

# Create the traefik namespace
microk8s kubectl create namespace traefik
```

### Step 2: Create OVH API Secret

Create a Kubernetes secret in the `traefik` namespace to securely store your OVH API credentials. Traefik will use this secret to solve the DNS-01 challenge.

Replace the `xxxxxxxx` placeholders with your actual credentials.

```bash
microk8s kubectl create secret generic ovh-credentials \
  --namespace=traefik \
  --from-literal=OVH_ENDPOINT=ovh-eu \
  --from-literal=OVH_APPLICATION_KEY=xxxxxxxxxxxxxxxxx \
  --from-literal=OVH_APPLICATION_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx \
  --from-literal=OVH_CONSUMER_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### Step 3: Install Traefik with Coraza WAF Plugin

This repository's `values.yaml` is pre-configured to enable the Coraza WASM plugin, set up the OVH certificate resolver, and configure persistence.

```bash
# Add the Traefik Helm repository and update
microk8s helm3 repo add traefik https://traefik.github.io/charts
microk8s helm3 repo update

# Install Traefik using the provided values.yaml
# This file automatically enables the Coraza plugin and CRDs
microk8s helm3 install traefik traefik/traefik \
  --namespace=traefik \
  --create-namespace \
  -f values.yaml
```

### Step 4: Apply Security Middlewares

The `security-middleware.yaml` file defines our reusable security policies. Apply it to the cluster.

⚠️ <b>Edit the geoblock middleware to add you own country in the allow list.</b>

```bash
microk8s kubectl apply -f security-middleware.yaml
```

This file creates five crucial objects in the `traefik` namespace:

1.  **`coraza-waf` (Middleware):** Configures the WAF engine with a custom ruleset to block common attacks like SQLi, XSS, Path Traversal, and more.
2.  **`security-headers` (Middleware):** Adds security headers like `Content-Security-Policy`, `Strict-Transport-Security`, and `X-Content-Type-Options`.
3.  **`rate-limit` (Middleware):** Applies a rate limit of 100 requests per minute.
4.  **`geoblock` (Middleware):** Applies a geo blocking filter to the request.
5.  **`tls-profile` (TLSOption):** Enforces modern TLS 1.2+ with strong cipher suites.

### Step 5: Deploy & Expose a Test Application

The `whoami-waf-test.yaml` file provides a complete example of deploying an application (`whoami`) and securing it with an `IngressRoute` that uses all the middlewares we just created.

**Before applying, edit `whoami-waf-test.yaml`** and change `waf-test.example.org` to your own domain.

```bash
# Edit the file first!
# nano whoami-waf-test.yaml

# Deploy the test application and its secure IngressRoute
microk8s kubectl apply -f whoami-waf-test.yaml
```

The `IngressRoute` in this file is the key:

  * It listens on the `websecure` (HTTPS) entrypoint.
  * It applies the three middlewares: `coraza-waf`, `security-headers`, and `rate-limit`.
  * It uses the `ovhresolver` to automatically get a Let's Encrypt certificate.

-----

## 🛡️ Verify the Configuration & Test the WAF

Wait a minute or two for the certificate to be issued.

### 1. Check the Application & Headers

First, check if the site is accessible and if the security headers are applied.

```bash
# Replace with your domain
curl -I https://waf-test.example.org
```

You should see a `HTTP/1.1 200 OK` status and the headers we defined, such as:

  * `content-security-policy: script-src 'self'`
  * `strict-transport-security: max-age=31536000; includeSubDomains; preload`
  * `x-content-type-options: nosniff`

### 2. Test the WAF (Block Attacks)

Now, try to send malicious payloads. The WAF should block them with a **`403 Forbidden`** response. The `curl -i` command will show you the HTTP response headers, so you can confirm the `403` status.

```bash
# --- TEST 1: Check for a normal 200 OK response (Baseline)
# (This should NOT be blocked)
curl -i https://waf-test.example.org/

# --- TEST 2: SQL Injection (Rule 1001)
# (This SHOULD be blocked with 403 Forbidden)
curl -i "https://waf-test.example.org/?id=1%27%20OR%20%271%27=%271"

# --- TEST 3: Cross-Site Scripting (XSS) (Rule 1002)
# (This SHOULD be blocked with 403 Forbidden)
curl -i "https://waf-test.example.org/?input=<script>alert(1)</script>"

# --- TEST 4: Blocked User-Agent (Rule 1004)
# (This SHOULD be blocked with 403 Forbidden)
curl -i -A "sqlmap/1.0" https://waf-test.example.org/
```

### 3. Check Logs

To see the WAF in action, you can check the Traefik pod logs. Blocked requests will be logged.

```bash
# Follow the logs from the Traefik pod
microk8s kubectl -n traefik logs -f deployment/traefik
```

-----

## 🔧 Management

### Update Traefik Configuration

If you make changes to `values.yaml`, upgrade your Traefik release:

```bash
microk8s helm3 upgrade traefik traefik/traefik -f values.yaml -n traefik
```

### Check Pods & Services

```bash
# Check if the Traefik pod is running
microk8s kubectl get pods -n traefik

# Get Traefik service (the LoadBalancer IP is here)
microk8s kubectl get svc -n traefik
```

## 📁 File Overview

  * **`values.yaml`**: The main Helm configuration for Traefik. It enables the Coraza plugin, configures the OVH ACME resolver, and sets up persistence.
  * **`security-middleware.yaml`**: A collection of Kubernetes CRDs defining our reusable security policies (WAF, Headers, Rate Limit, GeoBlock, TLS Profile).
  * **`whoami-waf-test.yaml`**: A complete, self-contained example of a test application secured by an `IngressRoute` that uses all the security middlewares.
