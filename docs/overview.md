---
type: Architecture
title: krateo-oxide-operator-kog — overview
description: How the KOG blueprint is built — the asset-to-CRD pipeline, why each RestDefinition looks the way it does, the token-scope tiers, the modelling choices, and the OxideInstance convergence plugin.
resource: oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog
tags: [kog, oasgen-provider, rest-dynamic-controller, oxide, restdefinition, architecture]
timestamp: 2026-08-11T00:00:00Z
---

# Overview

This chart is a thin, declarative packaging layer. It contains **no controller code**: all reconciliation
is done by Krateo's generic [`oasgen-provider`](https://github.com/krateoplatformops/oasgen-provider) and
[`rest-dynamic-controller`](https://github.com/krateoplatformops/rest-dynamic-controller), which must
already be installed in the cluster (they ship with Krateo ≥ 2.5.1). What the chart ships is a curated
OpenAPI subset per Oxide resource and the `RestDefinition` that binds it.

## The pipeline

One `helm install` walks each enabled resource through this chain (`chart/templates/`):

```
chart/assets/<key>.yaml          (hand-curated OAS 3.0 subset of one Oxide resource)
        │  embedded verbatim into a ConfigMap (templates/configmaps.yaml, via tpl)
        ▼
ConfigMap <release>-<key>         (data.<key>.yaml = the OAS, with oxide.apiUrl substituted)
        ▲  referenced by oasPath: configmap://<ns>/<release>-<key>/<key>.yaml
        │
RestDefinition <release>-<key>    (templates/rd-<key>.yaml — verbs, identifiers, field mappings)
        │  reconciled by oasgen-provider
        ▼
generated CRDs:  <Kind>            (the resource — spec fields from the OAS request bodies + path/query
                 <Kind>Configuration   params; status carries id)
        │  reconciled by rest-dynamic-controller
        ▼
HTTP calls to the Oxide Region API  (Authorization: Bearer <token>)
```

The asset is embedded through Helm `tpl`, so `{{ .Values.oxide.apiUrl }}` in each OAS `servers[0].url` is
resolved at install time. The ConfigMap name matches the `oasPath` of the matching `RestDefinition`, and
`oasgen-provider` reads the OAS from that ConfigMap to generate the CRD pair.

## The resources

19 `RestDefinition`s, one per `chart/assets/<key>.yaml`. Every kind is prefixed **`Oxide`** to avoid
crdgen collisions with same-named lowercase body/path/query identifiers (e.g. kind `Vpc` vs the `vpc`
query param, kind `Instance` vs the `{instance}` path param).

| Kind | Oxide API | Verbs | Scope params | Token scope |
|------|-----------|-------|--------------|-------------|
| `OxideProject` | `/v1/projects` | create / get / update / delete | — | silo |
| `OxideInstance` | `/v1/instances` | create / get / update / delete | project | silo |
| `OxideDisk` | `/v1/disks` | create / get / delete | project | silo |
| `OxideSnapshot` | `/v1/snapshots` | create / get / delete | project | silo |
| `OxideImage` | `/v1/images` | create / get / delete | project | silo |
| `OxideVpc` | `/v1/vpcs` | create / get / update / delete | project | silo |
| `OxideVpcSubnet` | `/v1/vpc-subnets` | create / get / update / delete | project, vpc | silo |
| `OxideVpcRouter` | `/v1/vpc-routers` | create / get / update / delete | project, vpc | silo |
| `OxideRouterRoute` | `/v1/vpc-router-routes` | create / get / update / delete | project, vpc, router | silo |
| `OxideFloatingIp` | `/v1/floating-ips` | create / get / update / delete | project | silo |
| `OxideNetworkInterface` | `/v1/network-interfaces` | create / get / update / delete | instance, project | silo |
| `OxideAffinityGroup` | `/v1/affinity-groups` | create / get / update / delete | project | silo |
| `OxideAntiAffinityGroup` | `/v1/anti-affinity-groups` | create / get / update / delete | project | silo |
| `OxideInternetGateway` | `/v1/internet-gateways` | create / get / delete | project, vpc | silo |
| `OxideSshKey` | `/v1/me/ssh-keys` | create / get / delete | — | silo-user |
| `OxideCertificate` | `/v1/certificates` | create / get / delete | — | silo |
| `OxideSilo` | `/v1/system/silos` | create / get / delete | — | fleet |
| `OxideIpPool` | `/v1/system/ip-pools` | create / get / update / delete | — | fleet |
| `OxideSubnetPool` | `/v1/system/subnet-pools` | create / get / update / delete | — | fleet |

## Why each RestDefinition looks the way it does

Every `templates/rd-<key>.yaml` follows the same shape (see `rd-project.yaml`, `rd-instance.yaml`):

- **`identifiers: [name]`** — Oxide's natural key is the user-supplied `name`, unique within the parent
  collection. It is known at creation time, so it can address the resource directly. No `findby` verb is
  needed because the key is not server-generated.
- **`additionalStatusFields: [id]`** — the server-generated UUID is read-only; it is surfaced into
  `status.id` (it lives only in the response/view schema, never in a request body).
- **`excludedSpecFields: [<pathParam>]`** — the item path placeholder (`{project}`, `{instance}`,
  `{disk}`, …) would otherwise surface as a redundant spec field. It is excluded and its value is sourced
  from `spec.name`.
- **`requestFieldMapping: inPath:<pathParam> → spec.name`** on get/update/delete — wires the natural key
  into the path (`/v1/instances/{instance}` gets `{instance}` from `spec.name`).
- **query params as spec fields** — `project`, `vpc`, `router`, `instance` are declared as
  operation-level `in: query` parameters. `oasgen-provider` exposes them as spec fields and
  `rest-dynamic-controller` sends them on every call that declares them, scoping the request (`?project=…`).
- **`http`/`bearer` security scheme** — present in every asset; this is what makes `oasgen-provider` emit
  the `<Kind>Configuration` CRD with `spec.authentication.bearer.tokenRef`.

## Verbs and immutability

Resources Oxide treats as immutable — disks, snapshots, images, internet gateways, SSH keys, certificates,
silos — carry **no `update` verb**; change them by delete + recreate. Power/attach actions
(`start`/`stop`/`reboot`, disk/IP attach-detach, IP-pool ranges, silo links) are single-fire RPCs outside
KOG's declarative create/get/update/delete model and are intentionally **not** modelled here (the
OxideInstance plugin below is the one exception that automates attach+start).

## Token scope tiers

The Oxide API token's scope determines which resources you can manage. `values.yaml` groups the
RestDefinitions by the scope of the token they need:

- **silo-user** — the authenticated user (SSH keys).
- **silo** — silo collaborator/admin: projects, certificates, and everything inside a project.
- **fleet** — fleet/system admin: silos, system IP pools, subnet pools.

A silo-collaborator token covers projects and everything inside them; the fleet-scoped resources
(`OxideSilo`, `OxideIpPool`, `OxideSubnetPool`) require a fleet-admin token. Disable resources you lack
scope for via the per-resource toggles ([configuration](./configuration.md)).

## Auth: the Oxide API token

Oxide authenticates with a long-lived **device access token** sent as `Authorization: Bearer <token>`.
Because the token is long-lived, **no rotation machinery** is required (unlike the short-lived Keycloak
admin tokens in other KOGs): supply it once via a Secret that each generated `<Kind>Configuration`
references through `spec.authentication.bearer.tokenRef` ([usage](./usage.md)).

## Modelling choices

- **Tagged unions flattened to objects.** Oxide encodes several inputs as adjacently-tagged enums
  (`{ "type": "...", ... }`): disk backends, route destinations/targets, external IPs, address
  allocators. These are modelled as plain objects with a `type` enum plus the superset of value fields
  (the relevant ones required per `type`), so crdgen emits a clean CRD while the JSON sent to Oxide stays
  valid.
- **Curated subsets, not the full spec.** Each asset covers the fields that make sense to set
  declaratively; power/attach actions and read-only telemetry are omitted.
- **`int64` byte fields.** Disk `size` and instance `memory` are `int64` in bytes; end-to-end support
  needs `oasgen-provider ≥ 0.10.4` (preserves `format: int64`) and `rest-dynamic-controller ≥ 0.9.1`
  (compares numeric fields by value). See [usage](./usage.md) and `docs/quickstart.md`.

## The OxideInstance convergence plugin

Oxide needs two actions the generic `rest-dynamic-controller` can't express to bring an instance up —
attach the boot disk and start — and Oxide's create-time `boot_disk` attach silently no-ops on a
not-yet-ready disk, so applying a disk and an instance together races into a stopped instance with a
detached boot disk.

When `instancePlugin.enabled=true`, the chart deploys a small **wrapper web service**
(`templates/instance-plugin.yaml`) and points the OxideInstance OAS `servers[0].url` at it instead of the
raw silo API. The plugin forwards the bearer token to `oxide.apiUrl` and makes GET an idempotent reconcile
(attach the boot disk, then start once attached), so the instance converges to running regardless of apply
ordering. The plugin image is `ghcr.io/krateo-blueprints/oxide-rest-dynamic-controller-plugin`. It is
**off by default**; leave it off if you only need create/get/delete semantics.

## Regenerating the assets

The 19 assets and RestDefinition templates were generated from the Oxide Region API OpenAPI document
(`openapi/nexus/nexus-<version>.json` in `oxidecomputer/omicron`) by a catalogue-driven script. To track a
newer Oxide API version, update the curated property sets and re-emit; `appVersion` in `Chart.yaml` records
the Oxide API version the subsets were curated against. See `docs/ARCHITECTURE.md` for the full note.
