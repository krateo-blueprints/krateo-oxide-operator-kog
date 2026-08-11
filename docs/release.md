---
type: Runbook
title: krateo-oxide-operator-kog — release
description: How the chart ships — one SemVer git tag matching Chart.yaml drives helm package and push to GHCR, then bump the CompositionDefinition to the published version.
resource: oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog
tags: [release, oci, ghcr, helm, runbook]
timestamp: 2026-08-11T00:00:00Z
---

# Release

The chart is published to `oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog` by
`.github/workflows/release-chart.yaml`.

## What a tag ships

A push of a **SemVer git tag** matching `[0-9]+.[0-9]+.[0-9]+` (e.g. `0.1.0`, **no** `v` prefix), or a
manual `workflow_dispatch`, runs the `release-chart` workflow:

1. `helm lint chart/` — validates the chart and its `values.schema.json`.
2. **Tag-vs-version guard** — on a tag push, the tag must equal `chart/Chart.yaml`'s `version`, or the
   job fails. So the tag you push and the chart version are always the same.
3. `helm package chart/` — `helm package` does not render templates, so a chart requiring runtime input
   (`oxide.apiUrl`) still packages cleanly.
4. `helm registry login ghcr.io` with `GITHUB_TOKEN`, then `helm push` to
   `oci://ghcr.io/<owner>/charts` — the owner namespace is derived from the repository owner
   (`GITHUB_TOKEN` can only write its own namespace), with a 5-attempt retry for GHCR first-push
   flakiness.

## Steps

Bump `chart/Chart.yaml`'s `version` on `main`, then tag it to match:

```console
$ git tag 0.1.0 && git push origin 0.1.0
```

Verify the workflow went green and the artifact exists:

```console
$ helm show chart oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog --version 0.1.0 | head -3
```

## After publishing

Bump `compositiondefinition.yaml`'s `spec.chart.version` to the published version on `main` — it is this
blueprint's own registration and must point at a chart version that exists ([api](./api.md)). The url
`oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog` already matches the release namespace.

## The Oxide API version

`chart/Chart.yaml`'s `appVersion` records the **Oxide Region API version** the OAS subsets were curated
against (not the chart's own version). To track a newer Oxide API, regenerate the assets (see
`docs/ARCHITECTURE.md`), bump `appVersion`, and cut a new chart `version`.

## Other workflows

- `security.yml` — the shared Krateo security scan (`krateo-platformops/.github/.github/workflows/security.yml@main`),
  on push and PR to `main`.
- `lint.yaml` — runs the shared docs-standard linter (`lint-docs`) on every push and PR to `main`, keeping
  this doc bundle conformant.
