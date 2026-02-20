# Manifests

This directory is organized into numbered manifests so they can be applied in order and kept readable.
Argo CD sync-wave annotations enforce the same ordering during GitOps syncs.

Apply order

kubectl apply -f manifests/01-namespace-rbac.yaml
kubectl apply -f manifests/02-keycloak-configmaps.yaml
kubectl apply -f manifests/03-vault-bootstrap-jobs.yaml
kubectl apply -f manifests/04-keycloak-app.yaml
kubectl apply -f manifests/05-admin-redirect.yaml
kubectl apply -f manifests/06-cloudflare-tunnel.yaml

Notes
- The `vault` namespace (and Vault itself) must already exist before applying the vault bootstrap jobs.
- The Yugabyte services referenced by the vault bootstrap job must already be running.

What lives where
- 01-namespace-rbac.yaml: keycloak namespace, service account, and vault job status RBAC.
- 02-keycloak-configmaps.yaml: keycloak client, role, and token exchange configuration.
- 03-vault-bootstrap-jobs.yaml: vault/Yugabyte bootstrap jobs for Keycloak.
- 04-keycloak-app.yaml: keycloak services and deployment.
- 05-admin-redirect.yaml: admin redirect config, services, and deployment.
- 06-cloudflare-tunnel.yaml: Cloudflare Tunnel deployment and External Secrets wiring for Keycloak.
