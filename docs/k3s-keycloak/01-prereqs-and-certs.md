---
layout: default
title: Prerequisites and TLS Certificates
parent: K3s + Keycloak OIDC Guide
nav_order: 1
description: "Tools you need on the host, and generating the self-signed TLS certificate for Keycloak."
---

# Prerequisites and TLS Certificates

**Goal of this page:** every tool on the host, and the self-signed certificate Keycloak will serve HTTPS with.

## Prerequisites

Linux host with:

- `docker`
- `kubectl`
- `openssl`
- `curl`
- `kubelogin` plugin installed so `kubectl oidc-login get-token` works

## Generate self-signed TLS certs for Keycloak

```bash
mkdir -p ~/playground/code/kubectl-oauth/certs
cd ~/playground/code/kubectl-oauth/certs

openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
  -keyout key.pem \
  -out cert.pem \
  -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,DNS:host.k3d.internal,IP:127.0.0.1"
```

{: .note }
The `subjectAltName` must cover every name clients will use to reach Keycloak — here `localhost`, the k3d internal name, and `127.0.0.1`. A cert that doesn't match the requested host fails with opaque TLS errors later.

Required for the Keycloak container to read the mounted key material:

```bash
chmod 644 cert.pem key.pem
```
