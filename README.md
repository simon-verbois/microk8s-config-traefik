<p align="center">
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/graphs/traffic"><img src="https://api.visitorbadge.io/api/visitors?path=https%3A%2F%2Fgithub.com%2Fsimon-verbois%2FKomga-Meta-Manager&label=Visitors&countColor=26A65B&style=flat" alt="Visitor Count" height="28"/></a>
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/commits/main"><img src="https://img.shields.io/github/last-commit/simon-verbois/Komga-Meta-Manager?style=flat" alt="GitHub Last Commit" height="28"/></a>
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/stargazers"><img src="https://img.shields.io/github/stars/simon-verbois/Komga-Meta-Manager?style=flat&color=yellow" alt="GitHub Stars" height="28"/></a>
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/issues"><img src="https://img.shields.io/github/issues/simon-verbois/Komga-Meta-Manager?style=flat&color=red" alt="GitHub Issues" height="28"/></a>
  <a href="https://github.com/simon-verbois/microk8s-config-traefik/pulls"><img src="https://img.shields.io/github/issues-pr/simon-verbois/Komga-Meta-Manager?style=flat&color=blue" alt="GitHub Pull Requests" height="28"/></a>
</p>

# Note
- This implementation has been tested on RHEL 9
- There is an alias setup on my profile: `alias kubectl='microk8s kubectl'`
- All this implementation has been done with a non root user (in microk8s group)
- An ingress example is available, but there is not real testing in the repository for Traefik as you need a full app to test

<br>

# Install Traefik OVH TLS Resolver
```bash
# Enable the modules
microk8s enable helm3 metallb
## The LoadBalancer will ask for an IP Range
## Add a range that's in your cluster subnet

# Check all pods status
kubectl create namespace traefik

# Create a secret in the traefik namespace we just created
# Create your API credentials here: https://eu.api.ovh.com/createToken/
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
microk8s helm3 repo add traefik https://traefik.github.io/charts && microk8s helm3 repo update

## Copy template, and customize it
mv values.yaml.template values.yaml

## Install Traefik
microk8s helm3 install traefik traefik/traefik \
  --namespace=traefik \
  --create-namespace \
  -f values.yaml

## Add RBAC access for Traefik ingress
kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/master/docs/content/reference/dynamic-configuration/kubernetes-crd-rbac.yml
```

<br>

# Updating traefik and configuration 
```bash
microk8s helm3 upgrade traefik traefik/traefik -f values.yaml -n traefik
```

<br>

# Check if everything is OK
```bash
# Check if the pod in running correctly
kubectl get pods -n traefik

# Get Traefik service (listening IP is here)
kubectl get svc -n traefik

# Check Traefik logs
kubectl -n traefik logs -f -l app.kubernetes.io/name=traefik
```
