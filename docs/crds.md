# authn CRDs (chart repo)

The CRDs authn owns ship from the **`crds-subchart/`** chart (`krateo-authn-crd`,
`crds-subchart/Chart.yaml`), deployed as its own Composition independently of the app (see
[overview.md](overview.md)). This page is a **curated** field map — the authoritative, exhaustive
schema is the CRD YAML itself, and the Go source of truth is the code repo's `apis/`.

## What this repo ships

This chart owns **four** CRDs — one per authentication method, each in its own API group, all
`v1alpha1`:

| Kind | Group/Version | Plural | File |
|------|---------------|--------|------|
| `User` | `basic.authn.krateo.io/v1alpha1` | `users` | `crds-subchart/templates/basic.authn.krateo.io_users.yaml` |
| `LDAPConfig` | `ldap.authn.krateo.io/v1alpha1` | `ldapconfigs` | `crds-subchart/templates/ldap.authn.krateo.io_ldapconfigs.yaml` |
| `OAuthConfig` | `oauth.authn.krateo.io/v1alpha1` | `oauthconfigs` | `crds-subchart/templates/oauth.authn.krateo.io_oauthconfigs.yaml` |
| `OIDCConfig` | `oidc.authn.krateo.io/v1alpha1` | `oidcconfigs` | `crds-subchart/templates/oidc.authn.krateo.io_oidcconfigs.yaml` |

CRD subchart version: **`0.22.4`** (`crds-subchart/Chart.yaml:20`), pinned independently of the app
chart (the README table still says `0.22.2` — the chart wins).

Each Config CR pairs with a **Secret** holding the relevant credential. The three non-basic methods
share an optional **`graphics`** object (`icon`, `displayName`, `backgroundColor`, `textColor`) that
styles the login button on the portal.

## `User` (basic auth) — `basic.authn.krateo.io/v1alpha1`

A portal user authenticated by username (the CR name) + password. Required:
`displayName`, `avatarURL`, `passwordRef`.

| Field | Type | Purpose |
|-------|------|---------|
| `displayName` | string (req) | The user's full name. |
| `avatarURL` | string (req) | The user's avatar image URL. |
| `groups` | []string | The Krateo groups the user belongs to (`admins`, `devs` are predefined). |
| `passwordRef` | object `{namespace, name, key}` (req) | Reference to the Secret key holding the password (a `kubernetes.io/basic-auth` Secret). |

> **Gotcha:** neither the basic-auth username nor password may contain a colon (`:`) — it is the
> HTTP Basic header separator.

## `LDAPConfig` — `ldap.authn.krateo.io/v1alpha1`

Authenticate against an LDAP directory. Required: `dialURL`, `baseDN`.

| Field | Type | Purpose |
|-------|------|---------|
| `dialURL` | string (req) | LDAP server address. |
| `baseDN` | string (req) | Base of the subtree the search is constrained to. |
| `bindDN` | string | Bind user DN (omit if the server supports anonymous bind). |
| `bindSecret` | object | Secret ref for the bind user's password. |
| `tls` | boolean | Use TLS to the LDAP server. |
| `graphics` | object | Optional login-button styling (see above). |

## `OAuthConfig` — `oauth.authn.krateo.io/v1alpha1`

OAuth 2.0 login (e.g. GitHub). Required: `authURL`, `tokenURL`, `clientID`, `clientSecretRef`,
`redirectURL`, `scopes`.

| Field | Type | Purpose |
|-------|------|---------|
| `authURL` | string (req) | Provider authorization URL. |
| `tokenURL` | string (req) | Provider token-exchange URL. |
| `clientID` | string (req) | The application's client ID. |
| `clientSecretRef` | object (req) | Secret ref for the application's client secret. |
| `redirectURL` | string (req) | Where the provider redirects after auth (the Krateo frontend's `/auth?kind=oauth`). |
| `scopes` | []string (req) | Requested permission scopes. |
| `authStyle` | integer | How the client ID/secret are sent to the endpoint. |
| `restActionRef` | object | **Mandatory in practice** — a `RESTAction` that resolves the user's groups. |
| `graphics` | object | Optional login-button styling (see above). |

## `OIDCConfig` — `oidc.authn.krateo.io/v1alpha1`

OpenID Connect login (e.g. Azure Entra). Required: `clientID`, `clientSecret`, `redirectURI`.

| Field | Type | Purpose |
|-------|------|---------|
| `clientID` | string (req) | The application's client ID. |
| `clientSecret` | object (req) | Secret-key selector for the client secret. |
| `redirectURI` | string (req) | Where the provider redirects after auth (the Krateo frontend's `/auth?kind=oidc`). |
| `discoveryURL` | string | OIDC discovery (`.well-known`) URL; preferred over the explicit URLs below. |
| `authorizationURL` / `tokenURL` / `userInfoURL` | string | Explicit endpoints when not using discovery. |
| `additionalScopes` | string | Extra requested scopes. |
| `restActionRef` | object | Optional `RESTAction` for group resolution. |
| `graphics` | object | Optional login-button styling (see above). |

## Exhaustive detail

- **The CRDs themselves:** `crds-subchart/templates/*.yaml` — the authoritative schemas with every
  field and constraint.
- **The Go source of truth:** the code repo `braghettos/krateo-authn` `apis/` — the Go types from
  which these CRDs are generated.
- **Provider setup** (Azure app registration, GitHub OAuth app, LDAP server, tenant/client IDs,
  scopes): read the CRD definitions and the code repo, not a memorized click-through.

## See also

- [overview.md](overview.md) — chart layout, CompositionDefinition, what gets deployed.
- [wiring.md](wiring.md) — the `values.yaml` surface, installer wiring, RBAC, gotchas.
