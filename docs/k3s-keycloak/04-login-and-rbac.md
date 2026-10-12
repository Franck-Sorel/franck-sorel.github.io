---
layout: default
title: First Login, RBAC, and Troubleshooting
parent: K3s + Keycloak OIDC Guide
nav_order: 4
description: "Wire the OIDC user into kubeconfig, complete the first browser login, grant RBAC, and fix the common confusions."
---

# First Login, RBAC, and Troubleshooting

**Goal of this page:** a working `kubectl` login through Keycloak, admin rights mapped from the realm role, and fixes for the errors you're most likely to hit.

## Configure the OIDC user/context in kubeconfig

{: .warning }
Do **not** request `groups` as an OAuth scope. It is a claim mapper, not a scope — requesting it produces `invalid_scope` errors.

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

## First login

```bash
kubectl config use-context oidc-k3s
kubectl get ns
```

A browser window opens. Log in with:

- Username: `k8s-admin`
- Password: `k8s-admin`

Expected namespaces: `default`, `kube-system`, `kube-public`, `kube-node-lease`.

## Grant RBAC

If you get `Forbidden` (authenticated but unauthorized), authentication already works — grant RBAC from the admin context:

```bash
kubectl config use-context default

kubectl create clusterrolebinding oidc-k8s-admins-group \
  --clusterrole=cluster-admin \
  --group='k8s-admins'
```

If rerun, this is expected: `clusterrolebindings.rbac.authorization.k8s.io "oidc-k8s-admins-group" already exists`.

Retest:

```bash
kubectl config use-context oidc-k3s
kubectl get ns
```

## Notes and common confusion

- Redirect to `http://localhost:8000` is **expected**. That is kubelogin's local callback listener — it does **not** mean Keycloak is on port 8000. Keycloak stays on `https://localhost:8443`.
- Error `invalid_scope ... groups` means your kubeconfig still has `--exec-arg=--oidc-extra-scope=groups` — remove it.
- Keycloak may ask for first/last name on first login. That is profile completion, not an auth failure.

## If you still have old or bad OIDC args

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
