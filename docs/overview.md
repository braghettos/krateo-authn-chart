# authn — deployment overview (chart repo)

What authn is, and **how it deploys** as a Krateo composition. This is the deployment view; the
internals/runtime view lives in the code repo `braghettos/krateo-authn`
([`docs/llms.txt`](https://github.com/braghettos/krateo-authn/blob/main/docs/llms.txt)). Every claim
below is traced to a file in this repo — if a comment disagrees with what the chart actually renders,
the rendered chart wins.

## What authn is

The Krateo PlatformOps **authentication service**. It logs a portal user in via one of four
methods — **basic auth**, **LDAP**, **OAuth 2.0**, **OIDC** — each declared by its own CRD
(`README.md:1`, `chart/Chart.yaml:2`). On a successful login it mints a short-lived Kubernetes
client kubeconfig for the user (signed via a CertificateSigningRequest it creates and approves —
hence the CSR RBAC, `chart/templates/clusterrole.yaml:7-37`) and issues a JWT signed with the
`jwt-sign-key` Secret. A reachable authn is a hard prerequisite for logging into the portal.

This repo is the **braghettos fork packaged as a Krateo blueprint**: the upstream Helm chart plus a
`values.schema.json` so `core-provider` can generate a typed CompositionDefinition CRD
(`README.md:3-6`).

## Repo layout — three charts

| Path | Chart name | OCI artifact | Versioning |
|------|------------|--------------|------------|
| `chart/` | `krateo-authn` | `oci://ghcr.io/braghettos/krateo/authn` | tracks the git tag (`Chart.yaml` `version: CHART_VERSION`, `chart/Chart.yaml:18`) |
| `crds-subchart/` | `krateo-authn-crd` | `oci://ghcr.io/braghettos/krateo/authn-crd` | pinned `0.22.4`, independent of the app tag (`crds-subchart/Chart.yaml:20`) |
| `kagent/chart/` | `krateo-authn-agent` | `oci://ghcr.io/braghettos/krateo/krateo-authn-agent` | `0.1.x`, independent (`kagent/chart/Chart.yaml`) |

The three version **independently**:

- **The main app chart** (`chart/`) is the deployable authn workload. `version` is the
  `CHART_VERSION` placeholder, substituted to the git tag at release; `appVersion` is the
  `APP_VERSION` placeholder, stamped from the latest semver tag of the code repo
  (`chart/Chart.yaml:18,24`). So the chart always deploys the container image the app actually
  published (`chart/values.yaml:7-11`, image `ghcr.io/braghettos/krateo-authn`).
- **The CRD subchart** (`crds-subchart/`) is deliberately **NOT bundled** into the app chart. Per
  the golden rule, CRDs live in a dedicated chart deployed as its own Composition (`authn-crd`).
  The CRDs version independently of the app (a literal `version: 0.22.4`, NOT the placeholder), so
  the release workflow leaves it untouched (`crds-subchart/Chart.yaml:19-20`).
  > **Chart vs. README:** the `README.md` table still says the CRD subchart is pinned `0.22.2`; the
  > actual `crds-subchart/Chart.yaml:20` is `0.22.4`. The chart wins.
- **The agent chart** (`kagent/chart/`) is the federated specialist agent (`krateo-authn-agent`)
  registered on `krateo-autopilot`; it versions on its own `0.1.x` line and is **not** the authn
  workload. `kagent/compositiondefinition.yaml` ships its CompositionDefinition.

## The CompositionDefinition

A standalone `compositiondefinition.yaml` for authn lives at the repo root (`compositiondefinition.yaml`):
`core.krateo.io/v1alpha1`, name `authn`, namespace `krateo-system`, pointing `core-provider` at
`oci://ghcr.io/braghettos/krateo/authn`. In a full platform the
[krateo-installer](https://github.com/braghettos/krateo-installer) umbrella owns and emits it
(`README.md:24-42`), pinning `spec.chart.version` to the released chart tag. `core-provider` reads
the chart's `values.schema.json`, generates the typed `Authn.composition.krateo.io` CRD, and
reconciles one Composition per instance. The deployed chart version is cluster-observable from
`CompositionDefinition.spec.chart.version` (this is the tag at which an agent should fetch THIS
repo's docs — see [llms.txt](llms.txt)).

## What the app chart deploys (`chart/templates/`)

Rendering the main `chart/` produces:

- **Deployment** (`deployment.yaml`) — one replica by default (`values.yaml:5`), the authn
  container exposing a **single `http` port (containerPort = `service.port` = 8082**,
  `deployment.yaml:44-47`, `values.yaml:42`). Config is delivered entirely via `envFrom`
  (`deployment.yaml:34-39`): the chart-managed ConfigMap and the `jwt-sign-key` Secret. The
  container has **no direct `env:` array** — every var is set through the ConfigMap.
- **ConfigMap** (`configmap.yaml`) — the env ConfigMap. It hardcodes `AUTHN_PORT` (= `service.port`),
  `AUTHN_NAMESPACE` and `POD_NAMESPACE` (the release namespace), then renders every key under
  `.Values.env` (`configmap.yaml:8-13`). Defaults under `env` are `AUTHN_DEBUG`,
  `AUTHN_KUBECONFIG_*` (cluster name / cert TTL / apiserver URL), `AUTHN_CORS`, `AUTHN_DUMP_ENV`
  (`values.yaml:139-145`).
- **Probes** — liveness and readiness both `GET /health` on the single `http` port
  (`values.yaml:72-79`, `deployment.yaml:48-51`).
- **Service** (`service.yaml`) — type `ClusterIP` by default, port 8082 → `targetPort: http`
  (`service.yaml:11-19`, `values.yaml:41-42`); `nodePort` is only rendered when
  `service.type == NodePort`.
- **RBAC** — `ServiceAccount` (`serviceaccount.yaml`); two `ClusterRole`s — one for **CSR
  create/approve** against the `kubernetes.io/kube-apiserver-client` signer (to mint user
  kubeconfigs), one granting `get` on `templates.krateo.io/restactions`
  (`clusterrole.yaml:7-50`) — plus a namespaced `Role` granting read/write on
  `secrets`/`configmaps`/`restactions` and read on the four auth CRDs (`role.yaml`), and their
  bindings (`clusterrolebinding.yaml`, `rolebinding.yaml`).
- **Optional** — `Ingress` (`ingress.yaml`, disabled by default, `values.yaml:44-45`) and an HPA
  (`hpa.yaml`, `autoscaling.enabled: false`, `values.yaml:111-115`).

For the full `values.yaml` surface (exposure, the ConfigMap-only env model, resources) and the
operational gotchas, see [wiring.md](wiring.md). For the auth CRD fields, see [crds.md](crds.md).

## Cross-references

- **Code repo (internals & runtime):** `braghettos/krateo-authn` —
  [`docs/llms.txt`](https://github.com/braghettos/krateo-authn/blob/main/docs/llms.txt). That set is
  versioned at the **image** tag (= chart `appVersion`); this set is versioned at the **chart** tag.
- **Installer umbrella:** `braghettos/krateo-installer` (owns authn's CompositionDefinition in a
  full platform).
- **Upstream:** `krateoplatformops/authn`.
