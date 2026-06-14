# krateo-authn-chart

Krateo PlatformOps **AuthN** blueprint — a fork of
[`krateoplatformops/authn`](https://github.com/krateoplatformops/authn) packaged as a Krateo
blueprint: the upstream Helm chart plus a `values.schema.json` (so `core-provider` can generate
a typed CRD) and a `CompositionDefinition`.

Part of the [krateo-installer](https://github.com/braghettos/krateo-installer) ecosystem.

## What it ships

| Path | Chart | OCI artifact | Versioning |
|------|-------|--------------|-----------|
| `chart/` | `authn` (appVersion 0.22.2) | `oci://ghcr.io/braghettos/krateo/authn` | tracks the git tag |
| `crds-subchart/` | `authn-crd` | `oci://ghcr.io/braghettos/krateo/authn-crd` | pinned `0.22.2` (independent of the app tag) |

The authn CRDs (users/ldap/oauth/oidc) version independently of the authn app, so
`crds-subchart/Chart.yaml` carries a literal `version: 0.22.2` rather than the `CHART_VERSION`
placeholder — the release workflow leaves it untouched.

## How the installer consumes it

The installer umbrella emits a `CompositionDefinition` that points `core-provider` at the OCI
chart; `core-provider` generates `Authn.composition.krateo.io` and reconciles one Composition
per instance:

```yaml
apiVersion: core.krateo.io/v1alpha1
kind: CompositionDefinition
metadata:
  name: authn
  namespace: krateo-system
spec:
  chart:
    url: oci://ghcr.io/braghettos/krateo/authn
    version: "0.22.2"
```

## Requirements

A Secret named `jwt-sign-key` must exist in the namespace where `authn` is deployed, holding the
JWT signing key:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: jwt-sign-key
type: Opaque
stringData:
  JWT_SIGN_KEY: change-me
```

## Local validation

```sh
helm lint chart
helm template smoke chart
```

## Release

Push a semver tag (`X.Y.Z`) — CI packages `chart/` and `crds-subchart/` and publishes both to
`oci://ghcr.io/braghettos/krateo`.

## Links

- Installer umbrella: https://github.com/braghettos/krateo-installer
- Upstream: https://github.com/krateoplatformops/authn
