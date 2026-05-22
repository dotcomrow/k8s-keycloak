# k8s-keycloak
Suncoast Systems Keycloak Auth server

## Public SPA Client (External Realm)

`manifests/02-keycloak-configmaps.yaml` defines a shared public OIDC client for browser apps:

- client id: `shell-spa-public`
- realm: `external`
- issuer URL: `https://auth.suncoast.systems/realms/external`
- discovery URL: `https://auth.suncoast.systems/realms/external/.well-known/openid-configuration`

This client is configured for Authorization Code + PKCE (`S256`) and includes redirect/web-origin patterns for:

- `http://localhost/*`
- `http://127.0.0.1/*`
- `https://*.suncoast.systems/*`
- `https://*.app.suncoast.systems/*`

## Auth Gateway Service (Shared Callback + GUI APIs)

The dedicated auth gateway source now lives in:
`https://github.com/suncoast-systems/keycloak-auth-gateway`

This repo keeps the deployment manifest at `manifests/07-auth-gateway.yaml`.
The Keycloak external realm client `auth-gateway-public` is defined in `manifests/02-keycloak-configmaps.yaml`.

Purpose:
- Centralize Keycloak redirect handling to a single callback URL (for example `https://login.suncoast.systems/callback`).
- Store allowed apps in a database so a GUI can manage them dynamically.
- Expose APIs for CRUD management of allowed apps.

Database tables created automatically at startup:
- `auth_gateway_allowed_apps`
- `auth_gateway_login_state`
- `auth_gateway_exchange_codes`

Runtime flow:
1. App sends users to `GET /start?app=<slug>&return_to=<url-or-path>`.
2. Gateway redirects to Keycloak and receives callback at `GET /callback`.
3. Gateway redirects back to app with one-time `gateway_code` query param.
4. App exchanges that code with `POST /v1/auth/exchange`.

Allowed app URL policy:
- Apps now support `base_urls` (list of allowed URLs) in addition to legacy `base_url`.
- This allows one app slug to support multiple trusted origins (for example localhost + preview + prod) without wildcard matching.

Management APIs (for GUI):
- `GET /v1/apps`
- `POST /v1/apps`
- `GET /v1/apps/{slug}`
- `PUT /v1/apps/{slug}`
- `DELETE /v1/apps/{slug}`

GitOps sync for app registrations:
- `manifests/10-auth-gateway-sync-apps.yaml`
- Bootstrap Job + CronJob reconcile `auth_gateway_allowed_apps` from the registry service API.

Management APIs require:
- `Authorization: Bearer <ADMIN_API_TOKEN>`
- `ADMIN_API_TOKEN` is sourced from Vault KVv2 path `secret/data/auth-gateway-admin-api`, key `value` (synced into Kubernetes Secret `auth-gateway-admin-api` by External Secrets).

### Build and Publish Auth Gateway Image

Build/publish is managed in the `keycloak-auth-gateway` repo via GitHub Actions.
Deployment in this repo expects:

- `ghcr.io/dotcomrow/keycloak-auth-gateway:latest`

### Deploy Auth Gateway

```sh
kubectl apply -f manifests/07-auth-gateway.yaml
kubectl apply -f manifests/10-auth-gateway-sync-apps.yaml
```

Required Vault secret (KVv2 path `secret/data/keycloak-client-secret-auth-gateway-exchange`):

```sh
vault kv put secret/keycloak-client-secret-auth-gateway-exchange value='<strong-client-secret>'
```

Required Vault secret:

```sh
vault kv put secret/auth-gateway-admin-api value='<strong-admin-token>'
```

Optional secret:

```sh

kubectl -n keycloak create secret generic auth-gateway-oidc-client \
  --from-literal=client-secret='<client-secret-if-using-confidential-client>'
```

APISIX example route:
- `manifests/08-auth-gateway-apisix-route.example.yaml`

## Profile Field Standard (IdP -> Keycloak -> Apps)

`manifests/04-keycloak-configurator.yaml` configures IdPs and imports as much profile data as is available from
external identity providers into Keycloak user attributes. Apps then receive these fields via an OIDC client
scope named `user-profile` (added as a **default** client scope for the main app clients in both realms).
The configurator also reconciles realm `users/profile` metadata so these attributes are visible in the Keycloak
Account Console Personal Info page. The account UI only renders attributes explicitly present in `users/profile`,
so custom IdP fields must be defined there.
IdP attribute mappers are reconciled on every configurator run and use `syncMode=FORCE` for profile fields.

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

Notification contact notes:
- Phone and device contact claims are in optional scope `notification-contact` (not default).
- `phone_number`/`phone_number_verified` are emitted only when this scope is requested and the upstream/user profile has values.
- `device_id` is emitted from the Keycloak user attribute `device_id` when present.
- Google IdP is configured with `openid email profile phone` so OIDC phone claims can be imported when available.
- GitHub's standard `/user` payload does not include a phone field, so internal users typically need phone populated from another upstream source or pre-set user attributes.
- Existing users may need to authenticate through their IdP again after mapper/scope changes for new attribute values to hydrate into Keycloak user attributes.

### How Apps Consume It
- Tokens/userinfo include the above attributes as claims when the client has the `user-profile` scope attached.
- To read phone/device contact claims, request optional scope `notification-contact` in the OIDC `scope` parameter.
- The configurator also ensures the standard `email` scope is present for app clients that expect `email`.

### Notification Capability Evaluation (recommended)
For a fast cross-platform decision, evaluate notification capability from token/userinfo claims as follows:

- `can_email`: `email` present AND (`email_verified` is `true` OR your policy allows unverified email)
- `can_sms`: `phone_number` present AND `phone_number_verified` is `true`
- `can_voice_call`: `phone_number` present (and optionally `phone_number_verified` by policy)
- `can_voicemail`: same as `can_voice_call`
- `can_push`: `device_id` present

Recommended implementation pattern:
- Request `notification-contact` only for apps/features that need contact channels.
- If `phone_number`/`device_id` claims are absent when scope was requested, treat the channel as unavailable.
- Return a normalized capability object from your auth/profile service so all apps use the same decision logic.

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
