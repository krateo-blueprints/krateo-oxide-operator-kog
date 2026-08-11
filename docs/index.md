---
type: Component
title: krateo-oxide-operator-kog — index
description: The map of the krateo-oxide-operator-kog doc bundle — a Krateo Operator Generator (KOG) blueprint that turns Oxide Computer Cloud resources into native Kubernetes custom resources via oasgen-provider and rest-dynamic-controller.
resource: oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog
tags: [kog, oasgen-provider, rest-dynamic-controller, oxide, blueprint, helm-chart, compositiondefinition]
timestamp: 2026-08-11T00:00:00Z
---

# krateo-oxide-operator-kog

A **Krateo Operator Generator (KOG)** blueprint that turns [Oxide Computer](https://oxide.computer/)
Cloud resources — projects, instances, disks, snapshots, images, VPCs and their subnets/routers/routes,
floating IPs, network interfaces, affinity and anti-affinity groups, internet gateways, SSH keys,
certificates, silos, IP pools and subnet pools — into **native Kubernetes custom resources**. There is
**no hand-written controller**: each resource is a hand-curated OpenAPI subset of the Oxide Region API,
turned into a CRD pair by [`oasgen-provider`](https://github.com/krateoplatformops/oasgen-provider) and
reconciled by [`rest-dynamic-controller`](https://github.com/krateoplatformops/rest-dynamic-controller).

The repo is a single Helm chart (`chart/`) plus a sibling `CompositionDefinition` that registers it with
Krateo. Installing the chart emits **19 `RestDefinition`s** (one per `chart/assets/<key>.yaml`); each
becomes a `<Kind>` CR and a `<Kind>Configuration` CR that carries the Oxide API-token reference.

## The bundle (start here)

- [overview](./overview.md) — the KOG pipeline (asset → ConfigMap → RestDefinition → generated CRDs →
  HTTP), why each RestDefinition looks the way it does, the token-scope tiers, and the OxideInstance
  convergence plugin.
- [usage](./usage.md) — mint the token Secret, install the operator layer, wire the Configuration CRs,
  declare a project/VPC/disk/instance, verify and clean up.
- [configuration](./configuration.md) — the whole `values.yaml` surface: the Oxide target, the token
  contract, the 19 per-resource toggles, the resource group, and the instance plugin.
- [api](./api.md) — the CRD contract: the `<Kind>` and `<Kind>Configuration` CRDs the chart's
  RestDefinitions generate, plus the `CompositionDefinition` this blueprint registers as.
- [examples](./examples.md) — the runnable example under `examples/`.
- [release](./release.md) — how the chart ships to GHCR on a SemVer tag.
- [log](./log.md) — curated history.
- [llms.txt](./llms.txt) — the file index of this bundle.

## Layout

- `chart/` — the operator chart.
  - `Chart.yaml` / `values.yaml` / `values.schema.json` — the `oxide.apiUrl`, the token Secret contract,
    and the 19 per-resource toggles.
  - `assets/<key>.yaml` — one hand-curated OAS 3.0 subset per Oxide resource (19 files).
  - `templates/configmaps.yaml` — embeds each asset into a ConfigMap (`tpl`-resolves `oxide.apiUrl`).
  - `templates/rd-<key>.yaml` — one `RestDefinition` per resource (19 files).
  - `templates/instance-plugin.yaml` — the optional OxideInstance convergence plugin Deployment + Service.
  - `samples/` — the `<Kind>Configuration` CRs (token wiring) and an end-to-end project + instance sample.
- `compositiondefinition.yaml` — this blueprint's own registration, pointing at the published OCI chart.
- `examples/composition.yaml` — the Composition CR that installs the operator layer through Krateo.
- `docs/ARCHITECTURE.md`, `docs/quickstart.md` — the long-form architecture note and the end-to-end
  walkthrough.

The chart carries **no controller code**: `oasgen-provider` and `rest-dynamic-controller` (shipped with
Krateo ≥ 2.5.1) do all reconciliation. See [overview](./overview.md).
