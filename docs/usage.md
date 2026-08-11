---
type: Usage
title: krateo-oxide-operator-kog — usage
description: How to install the KOG blueprint — mint the Oxide token Secret, install the operator layer (chart or Composition), wire the Configuration CRs, declare a project/VPC/disk/instance, verify and clean up.
resource: oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog
tags: [kog, oxide, install, helm, composition, usage]
timestamp: 2026-08-11T00:00:00Z
---

# Usage

Two ways to install the operator layer: the raw Helm chart, or a Krateo `Composition`. Either way the
result is the same 19 `RestDefinition`s and the CRD pairs `oasgen-provider` generates from them. The
end-to-end walkthrough with screenshots lives in [`docs/quickstart.md`](./quickstart.md); this page is the
condensed reference.

## Prerequisites

- A Kubernetes cluster running **Krateo ≥ 2.5.1** (so `oasgen-provider` and `rest-dynamic-controller` are
  present). For the `int64` byte fields to work end-to-end, use `oasgen-provider ≥ 0.10.4` and
  `rest-dynamic-controller ≥ 0.9.1` (see [overview](./overview.md)).
- `kubectl`, `helm`, and the [`oxide` CLI](https://docs.oxide.computer/cli).
- Network reachability from the cluster to your **Oxide silo API** (e.g. `https://oxide.example.com`).

## 1. Mint the Oxide API token into a Secret

Oxide authenticates with a long-lived device access token (`Authorization: Bearer <token>`). Log in once,
then capture the token into a Secret in your install namespace (`krateo-system` throughout):

```bash
oxide auth login --host https://oxide.example.com

kubectl create secret generic oxide-api-token \
  --from-literal=token="$(oxide auth status --token)" -n krateo-system
```

> The token's scope decides what you can manage. A silo-collaborator token covers projects and everything
> inside them; the fleet-scoped resources (`OxideSilo`, `OxideIpPool`, `OxideSubnetPool`) need a
> fleet-admin token ([overview](./overview.md)).

## 2. Install the operator layer

### Option A — the Helm chart

```bash
helm upgrade --install oxide-kog \
  oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog \
  -n krateo-system --set oxide.apiUrl=https://oxide.example.com
```

Disable resources you don't need or lack token scope for:

```bash
helm upgrade --install oxide-kog \
  oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog \
  -n krateo-system --set oxide.apiUrl=https://oxide.example.com \
  --set restDefinitions.silo.enabled=false \
  --set restDefinitions.ippool.enabled=false \
  --set restDefinitions.subnetpool.enabled=false
```

Turn on the OxideInstance convergence plugin with `--set instancePlugin.enabled=true` so an instance
attaches its boot disk and starts on reconcile ([overview](./overview.md)).

### Option B — a Krateo Composition

Register the blueprint (once) with the sibling `CompositionDefinition`, then apply a `Composition` CR:

```bash
kubectl apply -f compositiondefinition.yaml
kubectl apply -f examples/composition.yaml
```

`examples/composition.yaml` sets `spec.oxide.apiUrl` and the `restDefinitions` toggles as a
`KrateoOxideOperatorKog` (apiVersion `composition.krateo.io/v0-1-0`, derived from the chart). See
[examples](./examples.md) and [api](./api.md).

Wait for the RestDefinitions to register and the CRDs to appear:

```bash
kubectl get restdefinitions -n krateo-system     # all READY=True
kubectl get crds | grep oxide.ogen.krateo.io
```

## 3. Wire the token into the Configuration CRs

`oasgen-provider` generates a `<Kind>Configuration` CRD for every RestDefinition; it carries the
authentication block. Apply the ready-made set (all 19 point at the same `oxide-api-token` Secret):

```bash
kubectl apply -f chart/samples/00-configurations.yaml
```

Each is an `oxide-api` Configuration referencing the token Secret through
`spec.authentication.bearer.tokenRef` — for example `OxideProjectConfiguration`:

```yaml
apiVersion: oxide.ogen.krateo.io/v1alpha1
kind: OxideProjectConfiguration
metadata:
  name: oxide-api
  namespace: krateo-system
spec:
  authentication:
    bearer:
      tokenRef:
        name: oxide-api-token
        namespace: krateo-system
        key: token
```

## 4. Declare Oxide resources

Now declare actual cloud resources as plain Kubernetes CRs. Every resource references the `oxide-api`
Configuration and carries a `spec.name` (the natural key). `chart/samples/10-project-and-instance.yaml`
is a complete project + VPC + disk + instance example:

```yaml
apiVersion: oxide.ogen.krateo.io/v1alpha1
kind: OxideProject
metadata:
  name: demo
  namespace: krateo-system
spec:
  configurationRef:
    name: oxide-api
    namespace: krateo-system
  name: demo
  description: "Krateo-managed demo project"
```

Conventions to know:

- `spec.name` is the user-supplied natural key Oxide addresses the resource by; the server UUID lands in
  `status.id`.
- Project-scoped resources take `spec.project` (the project name → `?project=`); VPC children take
  `spec.vpc`; router routes take `spec.router`; network interfaces take `spec.instance`.
- Byte fields (`size`, `memory`) are `int64` in bytes (20 GiB = `21474836480`).

```bash
kubectl apply -f chart/samples/10-project-and-instance.yaml
```

## 5. Verify

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

Cross-check in Oxide (`oxide project list`, `oxide instance list --project demo`) or the web console.

## 6. Clean up

Deleting the CRs deletes the Oxide resources:

```bash
kubectl delete -f chart/samples/10-project-and-instance.yaml
helm uninstall oxide-kog -n krateo-system
```

## Troubleshooting

The full troubleshooting matrix (int32/int64 rejection, spurious numeric drift, `boot_disk` shape,
401/403, stale-schema regeneration) is in [`docs/quickstart.md`](./quickstart.md). The two most common:

- **Disk/instance rejected with `must be of type integer with format int32`** — the CRDs were generated
  by an `oasgen-provider` older than 0.10.4. Reinstall the provider with the pinned image.
- **401/403 from Oxide** — the token is missing, expired, or lacks scope. Recreate the Secret (step 1).
