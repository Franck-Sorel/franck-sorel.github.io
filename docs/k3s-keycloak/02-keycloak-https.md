---
layout: default
title: Run Keycloak with HTTPS
parent: K3s + Keycloak OIDC Guide
nav_order: 2
description: "Start the Keycloak container serving HTTPS on port 8443 and verify it."
---

# Run Keycloak with HTTPS

**Goal of this page:** Keycloak running in Docker, serving HTTPS on port 8443 with the generated certificate.

## Start the container

```bash
docker rm -f keycloak 2>/dev/null || true

docker run -d \
  --name keycloak \
  --restart unless-stopped \
  -p 8443:8443 \
  -e KC_BOOTSTRAP_ADMIN_USERNAME=admin \
  -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin \
  -v ~/playground/code/kubectl-oauth/certs:/opt/keycloak/certs \
  quay.io/keycloak/keycloak:latest \
  start-dev \
  --https-certificate-file=/opt/keycloak/certs/cert.pem \
  --https-certificate-key-file=/opt/keycloak/certs/key.pem \
  --https-port=8443 \
  --hostname=localhost \
  --hostname-strict=false
```

## Verify

```bash
docker ps --filter name=keycloak
docker logs --tail 50 keycloak
curl -kI https://localhost:8443/realms/master
```

Expected: `0.0.0.0:8443->8443/tcp` in the port listing and `HTTP/1.1 200` on the realms endpoint.

{: .note }
Admin credentials for the Keycloak console at `https://localhost:8443`: username `admin`, password `admin`.
