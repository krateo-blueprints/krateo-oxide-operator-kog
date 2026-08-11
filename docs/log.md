---
type: Log
title: krateo-oxide-operator-kog — log
description: Curated chronological history of krateo-oxide-operator-kog — notable changes and decisions, not a generated changelog.
resource: oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog
tags: [log, history]
timestamp: 2026-08-11T00:00:00Z
---

# Log

Curated history; release notes live in GitHub Releases.

## 2026-08-11 — Documentation Standard adoption

The repo adopts the Krateo Documentation Standard (OKF): the invariant docs bundle
(`docs/{index,overview,usage,configuration,api,examples,release,log}.md` + `docs/llms.txt`), a runnable
`examples/oxide-project-instance`, and the shared `lint-docs` check wired into a new `lint.yaml`. The
long-form architecture note (`docs/ARCHITECTURE.md`) and the end-to-end walkthrough (`docs/quickstart.md`)
are kept as-is and cross-linked from the bundle; the README is rewritten into the six standard sections.

## 0.1.0 — first release

The blueprint ships whole: the operator chart (19 curated OAS assets, 19 `RestDefinition` templates, the
ConfigMap embedding, the sample `<Kind>Configuration` and end-to-end resource CRs) plus the sibling
`CompositionDefinition`. `oasgen-provider` + `rest-dynamic-controller` do all reconciliation; the chart
carries no controller code. Decisions worth keeping:

- **`Oxide`-prefixed kinds.** Every generated kind is prefixed `Oxide` to avoid crdgen collisions with the
  same-named lowercase body/path/query identifiers (kind `Vpc` vs the `vpc` query param, kind `Instance`
  vs the `{instance}` path param).
- **Natural-key addressing, no `findby`.** Oxide addresses resources by the user-supplied `name`, known at
  creation time, so each RestDefinition maps the item path placeholder back to `spec.name` via
  `requestFieldMapping` and surfaces the server UUID read-only in `status.id`.
- **Long-lived token, no rotation machinery.** Oxide device tokens are long-lived, so the auth model is a
  single Secret each `<Kind>Configuration` references — no short-lived-token rotation like the Keycloak
  KOG.
- **Immutable resources omit `update`.** Disks, snapshots, images, internet gateways, SSH keys,
  certificates and silos are delete-and-recreate; power/attach RPCs are intentionally out of scope.
- **The OxideInstance convergence plugin.** An optional wrapper service (`instancePlugin.enabled`) makes
  an instance attach its boot disk and start on reconcile, so applying a disk and instance together
  self-heals to a running VM regardless of ordering — working around Oxide's create-time boot-disk attach
  no-op on a not-yet-ready disk.
- **`int64` byte fields.** Disk `size` and instance `memory` are `int64` in bytes; end-to-end support
  needs `oasgen-provider ≥ 0.10.4` and `rest-dynamic-controller ≥ 0.9.1` (see `docs/quickstart.md`).
