# authn — composition wiring & operations (chart repo)

The `chart/values.yaml` surface, how the installer pins and wires authn, its dependencies, and the
real operational gotchas. Everything is traced to the chart; where a stale note disagrees with the
rendered chart, the chart wins.

## The `values.yaml` surface (`chart/values.yaml`)

### Exposure

- `service.type: ClusterIP`, `service.port: 8082` (`values.yaml:40-42`). The Service maps `port` →
  `targetPort: http` (`service.yaml:14-17`); `nodePort` is only rendered when
  `service.type == NodePort` (`service.yaml:18-20`). authn is **cluster-internal by default** —
  the portal frontend reaches it in-cluster. Expose it (if at all) through the **installer CR**, not
  by hand-patching the Service.
- `ingress.enabled: false` by default (`values.yaml:44-45`); HPA off
  (`autoscaling.enabled: false`, `values.yaml:111-115`).

### The single http/8082 port + probes

The container exposes one `http` port = `service.port` = 8082 (`deployment.yaml:44-47`). Both probes
target it: `livenessProbe` and `readinessProbe` are `GET /health` on `port: http`
(`values.yaml:72-79`).

### Image

- `image.repository: ghcr.io/braghettos/krateo-authn`, `pullPolicy: IfNotPresent`, `tag: ""` —
  defaults to the chart `appVersion` (`values.yaml:7-11`, `deployment.yaml:40`). `appVersion` is the
  `APP_VERSION` placeholder, stamped from the code repo's latest semver tag, so the deployed image
  tag is the **image version** an agent reads off the Deployment.

### Config delivery (env, ConfigMap, envFrom)

- The container has **no direct `env:` array**. Every var goes under `.Values.env`, rendered into the
  chart's `<fullname>` ConfigMap and consumed via `envFrom` (`configmap.yaml:8-13`,
  `deployment.yaml:34-39`). The ConfigMap also hardcodes `AUTHN_PORT` (= `service.port`),
  `AUTHN_NAMESPACE` and `POD_NAMESPACE` (= the release namespace) (`configmap.yaml:8-10`).
- Defaults under `env` (`values.yaml:139-145`):
  - `AUTHN_DEBUG: "true"` — verbose logging.
  - `AUTHN_KUBECONFIG_CLUSTER_NAME: "krateo"` — the cluster name stamped into minted kubeconfigs.
  - `AUTHN_KUBECONFIG_CRT_EXPIRES_IN: "24h"` — TTL of the minted client cert.
  - `AUTHN_KUBECONFIG_SERVER_URL: "https://kube-apiserver:6443"` — the apiserver URL written into
    the user kubeconfig; **set this to the cluster's reachable apiserver endpoint**.
  - `AUTHN_CORS: "true"` — enable CORS (the browser SPA calls authn cross-origin).
  - `AUTHN_DUMP_ENV: "true"` — dump env at startup (turn off in production).
- `jwtSignKeySecretName: jwt-sign-key` (`values.yaml:147`) — mounted as a Secret `envFrom`
  (`deployment.yaml:37-39`); the named Secret must exist in the namespace.

### Resources

`resources: {}` by default (`values.yaml:62`) — no requests/limits are set unless you provide them.

## Dependencies (what must exist around authn)

- **The `authn-crd` Composition** (the four `*.authn.krateo.io` CRDs from `crds-subchart/`, version
  `0.22.4`) — deployed as its own Composition. The app chart deliberately does NOT bundle them; see
  [overview.md](overview.md).
- **The `jwt-sign-key` Secret** in the release namespace, holding `JWT_SIGN_KEY` — a non-optional
  `envFrom` secretRef (`deployment.yaml:37-39`, `README.md:46-58`); the pod won't start without it.
- **Cluster RBAC for CSR mint/approve** — the chart ships a ClusterRole granting
  create/approve/update on `certificatesigningrequests` (+ `signers`/`approval`) against the
  `kubernetes.io/kube-apiserver-client` signer (`clusterrole.yaml:7-37`); required because authn
  mints + approves a short-lived client cert per login to build the user's kubeconfig. A second
  ClusterRole grants `get` on `templates.krateo.io/restactions`. A namespaced `Role` grants
  read/write on `secrets`/`configmaps`/`restactions` and read on the four auth CRDs (`role.yaml`).

## How the installer wires it

The [krateo-installer](https://github.com/braghettos/krateo-installer) umbrella owns authn's
`CompositionDefinition` in a full platform (a standalone copy also lives at this repo's
`compositiondefinition.yaml`, `README.md:24-42`). The umbrella pins `spec.chart.version` to the
released chart tag and points `core-provider` at `oci://ghcr.io/braghettos/krateo/authn`;
`core-provider` reads `chart/values.schema.json`, generates the typed `Authn.composition.krateo.io`
CRD, and reconciles one Composition per instance. The deployed chart version is readable from
`CompositionDefinition.spec.chart.version` (the tag at which to fetch THIS repo's docs). The
`crds-subchart` (`authn-crd`) is pinned separately by the installer.

## Gotchas

- **`jwt-sign-key` Secret is required.** It is a non-optional `envFrom` secretRef
  (`deployment.yaml:37-39`); if absent the pod stalls in `ContainerCreating` /
  `CreateContainerConfigError` on the missing Secret.
- **authn is ClusterIP by default.** It is not externally exposed by the chart
  (`values.yaml:40-42`); the portal reaches it in-cluster. Don't expect a LoadBalancer IP — expose
  via the installer CR if external access is genuinely needed.
- **`AUTHN_KUBECONFIG_SERVER_URL` must point at the real apiserver.** The default
  (`https://kube-apiserver:6443`, `values.yaml:142`) is a placeholder; minted user kubeconfigs embed
  it, so a wrong value yields kubeconfigs the user can't actually use.
- **CSR RBAC is load-bearing.** Without the CSR create/approve ClusterRole
  (`clusterrole.yaml:7-37`) authn cannot mint user kubeconfigs and login fails after credential
  verification.
- **Colon ban on basic auth.** Basic-auth username and password cannot contain `:` — it's the Basic
  header separator (see [crds.md](crds.md)).
- **Don't bundle the CRDs.** Adding `crds-subchart` as a dependency of `chart/` makes the app
  release try to own the CRDs and collide with the `authn-crd` release; keep them separate (see
  [overview.md](overview.md)).

## See also

- [overview.md](overview.md) — chart layout, CompositionDefinition, what gets deployed.
- [crds.md](crds.md) — the four auth CRD field maps.
- Code repo runtime view: `braghettos/krateo-authn`
  [`docs/llms.txt`](https://github.com/braghettos/krateo-authn/blob/main/docs/llms.txt) (login
  handlers, JWT signing, kubeconfig minting). Versioned at the **image** tag.
