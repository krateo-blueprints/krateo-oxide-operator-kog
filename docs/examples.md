---
type: ExampleIndex
title: krateo-oxide-operator-kog — examples
description: Index of the runnable examples under examples/.
resource: oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog
tags: [examples, kog, oxide]
timestamp: 2026-08-11T00:00:00Z
---

# Examples

- [examples/oxide-project-instance](../examples/oxide-project-instance/README.md) — the full
  end-to-end walk: install the KOG operator layer through a `KrateoOxideOperatorKog` Composition, wire the
  token `<Kind>Configuration` CRs, then declare a project, VPC, disk and an instance that boots from it as
  native Kubernetes resources that reconcile to `READY=True`. Bundles the three manifests
  (`composition.yaml`, `00-configurations.yaml`, `10-project-and-instance.yaml`).

The step-by-step walkthrough with screenshots lives in [`docs/quickstart.md`](./quickstart.md). The raw
manifests are also in the repo at `examples/composition.yaml` and `chart/samples/`.
