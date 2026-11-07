---
type: Runbook
title: krateo-oxide-operator-kog — release
description: How the chart ships — one bare-semver git tag stamps the version, pushes the chart to GHCR and publishes the same archive to the Marketplace's blueprints index, per the organisation's publishing standard.
resource: oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog
tags: [release, oci, ghcr, helm, runbook]
timestamp: 2026-11-07T00:00:00Z
---

# Release

The chart is published by `.github/workflows/release-tag.yaml`, twice and with the same bytes, as the
organisation's publishing standard requires
([krateo-blueprints/charts PUBLISHING.md](https://github.com/krateo-blueprints/charts/blob/main/PUBLISHING.md)):
to `oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog`, and to the Marketplace's
`blueprints` index at `https://krateo-blueprints.github.io/charts/blueprints`, which is what the portal
Marketplace installs from.

## What a tag ships

A push of a **SemVer git tag** matching `[0-9]+.[0-9]+.[0-9]+` (e.g. `0.1.4`, **no** `v` prefix) runs the
workflow. The committed `chart/Chart.yaml` and `compositiondefinition.yaml` carry the `CHART_VERSION`
placeholder, and the tag is the only place a version is written.

1. **Refuses to start** unless the tagged commit has a successful `lint` run and the org secret
   `CHARTS_PUBLISH_TOKEN` is set, so nothing reaches GHCR that cannot also reach the Marketplace.
2. **Stamps** `CHART_VERSION` in `compositiondefinition.yaml` and `chart/Chart.yaml` from the tag.
3. **Packages and checks** the chart. It is a blueprint (it has a `values.schema.json` and a
   `CompositionDefinition`), so the stamped `compositiondefinition.yaml` is copied into the chart root
   before `helm package`, and the archive itself is checked: it must hold `compositiondefinition.yaml`
   (installing `oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog` at the tag's version)
   and `values.schema.json`, and `Chart.yaml` must have an https `icon` and `keywords`.
4. **Pushes** it to `oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog:<tag>`.
5. **Publishes** the same archive to the `blueprints` index through the organisation's publisher
   (`krateo-blueprints/charts/.github/workflows/publish-chart.yaml@1`), which succeeds only once the
   served index lists the version and `helm pull --repo` installs it anonymously.
6. **Attaches** the stamped `compositiondefinition.yaml` to the GitHub release for the tag.

## Steps

```console
$ git tag 0.1.4 && git push origin 0.1.4
```

Verify the workflow went green and both copies exist:

```console
$ helm show chart oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog --version 0.1.4 | head -3
$ helm pull krateo-oxide-operator-kog --version 0.1.4 --repo https://krateo-blueprints.github.io/charts/blueprints
```

## After publishing

Nothing is bumped by hand. Apply the stamped `compositiondefinition.yaml` attached to the release (not the
branch copy, which carries the placeholder) to register this version ([api](./api.md)).

## The Oxide API version

`chart/Chart.yaml`'s `appVersion` records the **Oxide Region API version** the OAS subsets were curated
against (not the chart's own version). To track a newer Oxide API, regenerate the assets (see
`docs/ARCHITECTURE.md`), bump `appVersion`, and cut a new chart `version`.

## Other workflows

- `security.yml` — the shared Krateo security scan (`krateo-platformops/.github/.github/workflows/security.yml@main`),
  on push and PR to `main`.
- `lint.yaml` — runs the shared docs-standard linter (`lint-docs`) on every push and PR to `main`, keeping
  this doc bundle conformant.
