---
layout: default
title: Install K3s with OIDC and Configure the Realm
parent: K3s + Keycloak OIDC Guide
nav_order: 3
description: "Reinstall k3s with OIDC API-server flags, prepare kubeconfig, and create the Keycloak realm, client, mapper and test user."
---

# Install K3s with OIDC and Configure the Realm

**Goal of this page:** k3s's API server trusting Keycloak, a kubeconfig on the host, and the realm/client/user inside Keycloak.

## Reinstall standalone k3s with OIDC flags

{: .danger}
This uninstalls any existing k3s. It is destructive to the current cluster state.

```bash
sudo /usr/local/bin/k3s-uninstall.sh 2>/dev/null || true

curl -sfL https://get.k3s.io | \
  INSTALL_K3S_EXEC="server \
  --write-kubeconfig-mode 644 \
  --kube-apiserver-arg=oidc-issuer-url=https://localhost:8443/realms/k8s \
  --kube-apiserver-arg=oidc-client-id=kubernetes-cli \
  --kube-apiserver-arg=oidc-username-claim=preferred_username \
  --kube-apiserver-arg=oidc-groups-claim=groups \
  --kube-apiserver-arg=oidc-ca-file=/home/virtualhost/playground/code/kubectl-oauth/certs/cert.pem" \
  sh -
```

## Prepare kubeconfig for local kubectl

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown "$(id -u):$(id -g)" ~/.kube/config
chmod 600 ~/.kube/config
kubectl config set-cluster default --server=https://127.0.0.1:6443
kubectl get nodes
```

## Configure Keycloak realm, client, and user

Open `https://localhost:8443` and log in as `admin` / `admin`.

### Create realm

- Realm name: `k8s`

### Create client

- Client ID: `kubernetes-cli`
- Client type: OpenID Connect
- Client authentication: **OFF**
- Authorization: OFF
- Standard flow: ON
- Redirect URIs:
  - `http://localhost:8000`
  - `http://localhost:18000`
  - `http://127.0.0.1:8000`
  - `http://127.0.0.1:18000`
- Web origins: `*`

### Client token setting

- Access token lifespan: `5m`

### Mapper

Create a mapper under the dedicated client scope:

- Name: `groups`
- Mapper type: `User Realm Role`
- Token claim name: `groups`
- Add to ID token: ON
- Add to access token: ON
- Add to userinfo: ON

### Realm role

- Create role: `k8s-admins`

### Test user

- Username: `k8s-admin`
- Email: `k8s-admin@local`
- Password: `k8s-admin` (temporary **OFF**)
- Role mapping: assign `k8s-admins`
