# k8s-keycloak
Suncoast Systems Keycloak Auth server

## Profile Field Standard (IdP -> Keycloak -> Apps)

`manifests/04-keycloak-configurator.yaml` configures IdPs and imports as much profile data as is available from
external identity providers into Keycloak user attributes. Apps then receive these fields via an OIDC client
scope named `user-profile` (added as a **default** client scope for the main app clients in both realms).

### Imported User Attributes (canonical)
- `picture` (avatar URL)
- `profile` (profile URL)
- `website`
- `locale`
- `name`
- `given_name`
- `family_name`
- `preferred_username`
- `email_verified`

### Imported User Attributes (provider-specific)
- Google: `hd`, `google_sub`
- GitHub: `company`, `location`, `bio`, `twitter_username`, `github_id`, `github_node_id`

### How Apps Consume It
- Tokens/userinfo include the above attributes as claims when the client has the `user-profile` scope attached.
- The configurator also ensures the standard `email` scope is present for app clients that expect `email`.

## Cloudflare Tunnel
This repo includes a Cloudflare Tunnel deployment at `manifests/06-cloudflare-tunnel.yaml`.
It runs `cloudflared` in the `keycloak` namespace and reads `TUNNEL_TOKEN` from a Kubernetes Secret named `cloudflare-tunnel-token`.
That Secret is created by External Secrets using Vault.

Vault policy + role for External Secrets (`externalsecrets-keycloak`) are created by:
- `manifests/03-vault-bootstrap-jobs.yaml`

### One-time setup
1. Create a named tunnel in Cloudflare Zero Trust (or with the CLI) and copy the tunnel token.

```sh
cloudflared tunnel login
cloudflared tunnel create keycloak
cloudflared tunnel route dns keycloak auth.suncoast.systems
cloudflared tunnel token keycloak
```

2. Write the token into Vault (KVv2) at `secret/data/keycloak-cloudflare-tunnel-token` with key `value`.

```sh
vault kv put secret/keycloak-cloudflare-tunnel-token value='<PASTE_TUNNEL_TOKEN>'
```

3. Sync ArgoCD so these manifests are applied:
- `manifests/03-vault-bootstrap-jobs.yaml`
- `manifests/06-cloudflare-tunnel.yaml`

4. In Cloudflare Zero Trust, set the tunnel public hostname and origin service:
- Hostname: `auth.suncoast.systems`
- Service URL: `http://keycloak.keycloak.svc.cluster.local:8080`

### Verify
```sh
kubectl -n keycloak get deploy,pod -l app=cloudflared
kubectl -n keycloak logs deploy/cloudflared --tail=100 -f
kubectl -n keycloak get externalsecret cloudflare-tunnel-token
```
