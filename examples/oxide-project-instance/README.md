---
type: Example
title: oxide-project-instance — install the KOG and declare a project, VPC, disk and instance
description: A runnable end-to-end example — install the Oxide KOG operator layer through a Krateo Composition, wire the token Configuration CRs, then declare a project, VPC, disk and instance as native Kubernetes resources.
resource: oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog
tags: [example, kog, oxide, composition, instance]
timestamp: 2026-08-11T00:00:00Z
---

# oxide-project-instance

An end-to-end walk of the blueprint: install the KOG **operator layer**, wire the API token, then declare
real Oxide cloud resources — a **project, a VPC, a disk and an instance that boots from it** — as plain
Kubernetes custom resources that reconcile to `READY=True`.

Three manifests in this directory:

- `composition.yaml` — the `KrateoOxideOperatorKog` Composition CR that installs the operator layer (the
  19 `RestDefinition`s). Sets `spec.oxide.apiUrl` and disables the fleet-scoped resources by default.
- `00-configurations.yaml` — the `<Kind>Configuration` CRs (token wiring). All 19 point at the same
  `oxide-api-token` Secret through `spec.authentication.bearer.tokenRef`.
- `10-project-and-instance.yaml` — the actual cloud resources: `OxideProject` → `OxideVpc` → `OxideDisk`
  → `OxideInstance`, each referencing the `oxide-api` Configuration.

Applying `composition.yaml` installs the operator **only**; it does not create Oxide resources itself.
Once the generated CRDs exist you declare resources with the CRs in `10-project-and-instance.yaml`.

## Prerequisites

- A Kubernetes cluster running Krateo ≥ 2.5.1 with `oasgen-provider` and `rest-dynamic-controller`
  (`oasgen-provider ≥ 0.10.4` and `rest-dynamic-controller ≥ 0.9.1` for the `int64` byte fields).
- The Oxide API-token Secret in the install namespace:

  ```bash
  kubectl create secret generic oxide-api-token \
    --from-literal=token="$(oxide auth status --token)" -n krateo-system
  ```

- The blueprint registered (once): `kubectl apply -f ../../compositiondefinition.yaml`.

## Run it

```bash
# 1. Install the operator layer (edit spec.oxide.apiUrl to your silo first).
kubectl apply -f composition.yaml

# Wait for the RestDefinitions and generated CRDs.
kubectl get restdefinitions -n krateo-system          # all READY=True
kubectl get crds | grep oxide.ogen.krateo.io

# 2. Wire the token into the Configuration CRs.
kubectl apply -f 00-configurations.yaml

# 3. Declare the project, VPC, disk and instance.
#    Replace the placeholder image_id in the OxideDisk with a real image UUID first
#    (oxide image list --project <p>), or switch disk_source to type: blank.
kubectl apply -f 10-project-and-instance.yaml
```

## Verify

```bash
kubectl get oxideproject,oxidevpc,oxidedisk,oxideinstance -n krateo-system
```

```
NAME                                         READY
oxideproject.oxide.ogen.krateo.io/demo       True
oxidevpc.oxide.ogen.krateo.io/demo-vpc       True
oxidedisk.oxide.ogen.krateo.io/demo-boot     True
oxideinstance.oxide.ogen.krateo.io/demo-vm   True
```

Cross-check in Oxide: `oxide project list`, `oxide disk list --project demo`,
`oxide instance list --project demo`, or open the web console.

## Notes

- `spec.name` is the natural key Oxide addresses each resource by; the server UUID lands in `status.id`.
- Project-scoped resources take `spec.project`; VPC children take `spec.vpc`. Byte fields (`size`,
  `memory`) are `int64` in bytes (20 GiB = `21474836480`, 4 GiB = `4294967296`).
- The boot disk in `10-project-and-instance.yaml` uses an `image` source with a **placeholder UUID** —
  replace it, or set `disk_source.type: blank` with a `block_size` to boot a blank disk. If you enabled
  `instancePlugin`, the instance attaches its boot disk and starts on reconcile regardless of ordering;
  otherwise apply the disk first and wait for `READY`.

## Clean up

```bash
kubectl delete -f 10-project-and-instance.yaml
kubectl delete -f composition.yaml
```

The step-by-step version with screenshots is in [`docs/quickstart.md`](../../docs/quickstart.md). The raw
manifests also live at `chart/samples/` and `examples/composition.yaml` in the repo root.
