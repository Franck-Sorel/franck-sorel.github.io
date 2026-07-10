---
layout: default
title: K3s + Keycloak OIDC (TLS) Setup Guide
---

This guide configures **standalone k3s** (not k3d) with Keycloak OIDC over HTTPS using a self-signed certificate.

## Prerequisites

- Linux host with:
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

# Required for Keycloak container to read mounted key material
chmod 644 cert.pem key.pem
```

## Start Keycloak with HTTPS on port 8443

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

Verify:

```bash
docker ps --filter name=keycloak
docker logs --tail 50 keycloak
curl -kI https://localhost:8443/realms/master
```

You should see `0.0.0.0:8443->8443/tcp` and `HTTP/1.1 200` on the realms endpoint.

## Reinstall standalone k3s with OIDC flags

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

## Configure Keycloak realm/client/user

Open: `https://localhost:8443`

### Admin credentials

- Username: `admin`
- Password: `admin`

### Create realm

- Realm name: `k8s`

### Create client

- Client ID: `kubernetes-cli`
- Client type: OpenID Connect
- Client authentication: OFF
- Authorization: OFF
- Standard flow: ON
- Redirect URIs:
  - `http://localhost:8000`
  - `http://localhost:18000`
  - `http://127.0.0.1:8000`
  - `http://127.0.0.1:18000`
- Web origins:
  - `*`

### Client token setting

- Access token lifespan: `5m`

### Mapper

Create mapper under dedicated client scope:

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
- Password: `k8s-admin` (temporary OFF)
- Role mapping: assign `k8s-admins`

## Configure OIDC user/context in kubeconfig

Important: **Do not request `groups` as OAuth scope**. It is a claim mapper, not a scope.

```bash
kubectl config set-credentials oidc \
  --exec-api-version=client.authentication.k8s.io/v1beta1 \
  --exec-command=kubectl \
  --exec-arg=oidc-login \
  --exec-arg=get-token \
  --exec-arg=--oidc-issuer-url=https://localhost:8443/realms/k8s \
  --exec-arg=--oidc-client-id=kubernetes-cli

kubectl config set-context oidc-k3s \
  --cluster=default \
  --user=oidc \
  --namespace=default
```

## First login and RBAC

```bash
kubectl config use-context oidc-k3s
kubectl get ns
```

A browser window opens. Login with:

- Username: `k8s-admin`
- Password: `k8s-admin`

If you get `Invalid parameter: redirect_uri`, re-check redirect URIs above (localhost and 127.0.0.1).

If you get `Forbidden` (authenticated but unauthorized), authentication already works; grant RBAC from admin context:

```bash
kubectl config use-context default

kubectl create clusterrolebinding oidc-k8s-admins-group \
  --clusterrole=cluster-admin \
  --group='k8s-admins'
```

If rerun, this is expected:

- `clusterrolebindings.rbac.authorization.k8s.io "oidc-k8s-admins-group" already exists`

Retest:

```bash
kubectl config use-context oidc-k3s
kubectl get ns
```

Expected output includes:

- `default`
- `kube-system`
- `kube-public`
- `kube-node-lease`

## Notes and common confusion

- Redirect to `http://localhost:8000` is **expected**.
  - This is kubelogin local callback listener.
  - It does **not** mean Keycloak is on port 8000.
- Keycloak stays on `https://localhost:8443`.
- Error `invalid_scope ... groups` means your kubeconfig still has:
  - `--exec-arg=--oidc-extra-scope=groups`
  - Remove it.
- Keycloak may ask first/last name on first login.
  - This is profile completion, not an auth failure.

## If you still have old/bad OIDC args

Delete and recreate just the `oidc` user:

```bash
kubectl config delete-user oidc

kubectl config set-credentials oidc \
  --exec-api-version=client.authentication.k8s.io/v1beta1 \
  --exec-command=kubectl \
  --exec-arg=oidc-login \
  --exec-arg=get-token \
  --exec-arg=--oidc-issuer-url=https://localhost:8443/realms/k8s \
  --exec-arg=--oidc-client-id=kubernetes-cli
```
