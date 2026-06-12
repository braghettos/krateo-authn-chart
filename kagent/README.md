# `krateo-authn-agent` — the authn component's federated specialist agent

A [kagent](https://kagent.dev) Agent that manages Krateo authentication (LDAP, OIDC, OAuth 2.0,
basic auth) and is the **dedicated expert on authn**. Per the
[`/kagent` standard](https://github.com/braghettos/krateo-autopilot/blob/main/AGENTS-VERSIONING.md),
it knows its component from this chart's `Chart.yaml` `sources`:

- **codebase:** `github.com/braghettos/authn` (the braghettos fork of the authn service)
- **chart that packages it:** `github.com/braghettos/krateo-authn-chart` (this repo)

It reads both via the github MCP tools to ground answers in the real code/CRDs/schema — it does not
guess. Reachable only through the `krateo-autopilot` orchestrator's A2A routing (registered via the
`extraAgents` hook); not a standalone entry point.

## What it ships (`chart/`)

| Template | Purpose |
|----------|---------|
| `agent.yaml` | the `krateo-authn-agent` Agent — k8s tools + github tools (read its codebase/chart) |
| `prompts.yaml` + `files/prompts-{eng,ita}.yaml` | the `authn_agent` prompt ConfigMap (+ the component-knowledge section) |

Published to `oci://ghcr.io/braghettos/krateo/krateo-authn-agent` (pinned `0.1.0` — it versions
independently of the authn chart, which shares this repo).

## How it ships

An **installer component** (gated on `features.observabilityAgents`), registered on the
orchestrator via `componentValues.krateo-autopilot.extraAgents: [{ name: krateo-authn-agent }]`.

## Related

- The `krateo-autopilot` repo's `AGENTS-VERSIONING.md` — the `/kagent` agents standard
- `CHART-STANDARD.md` (krateo-installer) — the app/crds/agent `Chart.yaml` standard (the `sources` rule)
