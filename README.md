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
