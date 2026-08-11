<p align="center">
  <img src="docs/krateo-logo.svg" alt="Krateo" height="80"/>
  &nbsp;&nbsp;❤️&nbsp;&nbsp;
  <img src="docs/oxide-logo.svg" alt="Oxide Computer" height="80"/>
</p>

# krateo-oxide-operator-kog

A Krateo Operator Generator (KOG) blueprint that turns [Oxide Computer](https://oxide.computer/) Cloud
resources into native Kubernetes custom resources — no hand-written controller, just a curated OpenAPI
subset per resource and Krateo's generic `rest-dynamic-controller`.

## What is this

A single Helm chart (`chart/`) plus a sibling `CompositionDefinition` that registers it with Krateo.
Installing it emits **19 `RestDefinition`s** (one per `chart/assets/<key>.yaml`); Krateo's
[`oasgen-provider`](https://github.com/krateo-platformops/oasgen-provider) turns each into a CRD pair — a
`<Kind>` (the resource) and a `<Kind>Configuration` (carrying the API-token reference) — and
[`rest-dynamic-controller`](https://github.com/krateo-platformops/rest-dynamic-controller) reconciles each
CR against the Oxide Region API. There is no controller code in this repo; `oasgen-provider` and
`rest-dynamic-controller` ship with Krateo ≥ 2.5.1.

The 19 kinds (all prefixed `Oxide` to avoid crdgen collisions): `OxideProject`, `OxideInstance`,
`OxideDisk`, `OxideSnapshot`, `OxideImage`, `OxideVpc`, `OxideVpcSubnet`, `OxideVpcRouter`,
`OxideRouterRoute`, `OxideFloatingIp`, `OxideNetworkInterface`, `OxideAffinityGroup`,
`OxideAntiAffinityGroup`, `OxideInternetGateway`, `OxideSshKey`, `OxideCertificate`, `OxideSilo`,
`OxideIpPool`, `OxideSubnetPool`. See [docs/overview.md](docs/overview.md) for the full resource table and
the asset→CRD pipeline.

Oxide authenticates with a long-lived **device access token** (`Authorization: Bearer <token>`) supplied
via a Kubernetes Secret each generated `<Kind>Configuration` references. The token's scope decides what you
can manage: a silo-collaborator token covers projects and everything inside them; the fleet resources
(`OxideSilo`, `OxideIpPool`, `OxideSubnetPool`) need a fleet-admin token.

## Install

```bash
# 1. The token Secret.
kubectl create secret generic oxide-api-token \
  --from-literal=token="$(oxide auth status --token)" -n krateo-system

# 2. The operator layer (RestDefinitions + ConfigMaps).
helm upgrade --install oxide-kog \
  oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog \
  -n krateo-system --set oxide.apiUrl=https://oxide.example.com

# 3. The per-Kind Configuration CRs (token wiring).
kubectl apply -f chart/samples/00-configurations.yaml
```

Or install through a Krateo `Composition`: `kubectl apply -f compositiondefinition.yaml` then
`kubectl apply -f examples/composition.yaml`. Full walkthrough with screenshots:
[docs/quickstart.md](docs/quickstart.md). Condensed reference: [docs/usage.md](docs/usage.md).

## Configure

The key input is `oxide.apiUrl` (your silo endpoint, no trailing slash). Disable resources you don't need
or lack token scope for via the per-resource toggles:

```bash
helm upgrade --install oxide-kog \
  oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog \
  -n krateo-system --set oxide.apiUrl=https://oxide.example.com \
  --set restDefinitions.silo.enabled=false \
  --set restDefinitions.ippool.enabled=false \
  --set restDefinitions.subnetpool.enabled=false
```

Enable the OxideInstance convergence plugin (`--set instancePlugin.enabled=true`) to make an instance
attach its boot disk and start on reconcile. The whole `values.yaml` surface — the Oxide target, the token
contract, the 19 toggles, the resource group and the plugin — is documented in
[docs/configuration.md](docs/configuration.md); it is typed by `chart/values.schema.json`.

## Examples

- [examples/oxide-project-instance](examples/oxide-project-instance/README.md) — install the KOG, wire the
  token, then declare a project, VPC, disk and instance as native Kubernetes resources that reconcile to
  `READY=True`.
- [examples/composition.yaml](examples/composition.yaml) — the `KrateoOxideOperatorKog` Composition CR
  that installs the operator layer through Krateo.
- `chart/samples/` — the ready-made `<Kind>Configuration` CRs and an end-to-end resource set.

Index: [docs/examples.md](docs/examples.md).

## Docs

- [docs/index.md](docs/index.md) — the map of the whole bundle.
- [docs/overview.md](docs/overview.md) — the KOG pipeline, the resources, the RestDefinition rationale, the
  instance plugin.
- [docs/usage.md](docs/usage.md) — install, wire the token, declare resources, verify.
- [docs/configuration.md](docs/configuration.md) — the whole `values.yaml` surface.
- [docs/api.md](docs/api.md) — the generated `<Kind>` / `<Kind>Configuration` CRDs and the
  `CompositionDefinition`.
- [docs/examples.md](docs/examples.md) — the runnable examples.
- [docs/release.md](docs/release.md) — how the chart ships.
- [docs/log.md](docs/log.md) — curated history.
- [docs/quickstart.md](docs/quickstart.md) — the end-to-end walkthrough with screenshots.
- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) — the long-form architecture note.

## Develop & release

The chart is published to `oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog` by
`.github/workflows/release-chart.yaml` on a SemVer git tag matching `chart/Chart.yaml`'s `version` (e.g.
`0.1.0`, no `v` prefix); the workflow guards that the tag equals the chart version, `helm lint`s,
`helm package`s and pushes to GHCR. After publishing, bump `compositiondefinition.yaml`'s
`spec.chart.version` to the released version. `appVersion` records the Oxide Region API version the OAS
subsets were curated against — regenerate the assets to track a newer API (see
[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)). The `lint.yaml` workflow runs the shared docs-standard
linter on every push and PR. Full runbook: [docs/release.md](docs/release.md). Licensed Apache-2.0
([LICENSE](LICENSE)).
