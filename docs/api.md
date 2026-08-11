---
type: API
title: krateo-oxide-operator-kog — API
description: The CRD contract this blueprint generates — the RestDefinition inputs, the per-resource <Kind> and <Kind>Configuration CRDs, their spec/status shape, and the CompositionDefinition the blueprint registers as.
resource: oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog
tags: [kog, crd, restdefinition, compositiondefinition, api]
timestamp: 2026-08-11T00:00:00Z
---

# API

This blueprint does not expose an HTTP API of its own. Its contract is the set of **Custom Resource
Definitions** it causes to exist: the `RestDefinition`s the chart emits, the `<Kind>` and
`<Kind>Configuration` CRDs `oasgen-provider` generates from them, and the `CompositionDefinition` the
blueprint is registered as.

## 1. The `RestDefinition` (input to oasgen-provider)

Each `chart/templates/rd-<key>.yaml` renders one `RestDefinition` (`ogen.krateo.io/v1alpha1`). It is the
chart's output and `oasgen-provider`'s input. Shape (from `rd-project.yaml`):

```yaml
apiVersion: ogen.krateo.io/v1alpha1
kind: RestDefinition
metadata:
  name: <release>-project
spec:
  oasPath: configmap://<ns>/<release>-project/project.yaml   # the ConfigMap holding the OAS subset
  resourceGroup: oxide.ogen.krateo.io                        # from values.resourceGroup
  resource:
    kind: OxideProject                                        # from values.restDefinitions.project.kind
    identifiers:
      - name                                                  # Oxide's natural key
    additionalStatusFields:
      - id                                                    # server UUID → status.id
    excludedSpecFields:
      - project                                               # the {project} path placeholder
    verbsDescription:
      - action: create
        method: POST
        path: /v1/projects
      - action: get
        method: GET
        path: /v1/projects/{project}
        requestFieldMapping:
          - inPath: project
            inCustomResource: spec.name
      # update, delete: same {project} ← spec.name mapping
```

Fleet resources use `/v1/system/...` paths; the SSH-key resource uses `/v1/me/ssh-keys/{ssh_key}`.
Immutable resources omit the `update` verb ([overview](./overview.md)).

## 2. The generated resource CRDs — `<Kind>`

From each RestDefinition, `oasgen-provider` generates a namespaced CRD under
`oxide.ogen.krateo.io/v1alpha1`, one per enabled resource:

`OxideProject`, `OxideInstance`, `OxideDisk`, `OxideSnapshot`, `OxideImage`, `OxideVpc`, `OxideVpcSubnet`,
`OxideVpcRouter`, `OxideRouterRoute`, `OxideFloatingIp`, `OxideNetworkInterface`, `OxideAffinityGroup`,
`OxideAntiAffinityGroup`, `OxideInternetGateway`, `OxideSshKey`, `OxideCertificate`, `OxideSilo`,
`OxideIpPool`, `OxideSubnetPool`.

**`spec`** is derived from the OAS request-body schema plus the operation's path/query parameters:

| field | source | notes |
|---|---|---|
| `configurationRef` | injected by oasgen-provider | `{name, namespace}` → the `<Kind>Configuration` that carries the token |
| `name` | OAS request body / natural key | the user-supplied name Oxide addresses the resource by |
| `description`, other body fields | OAS request body | e.g. `size`, `memory`, `ncpus`, `hostname`, `dns_name`, `disk_backend`, `boot_disk`, `external_ips` |
| `project` / `vpc` / `router` / `instance` | operation `in: query` params | scope fields sent as `?project=…` etc. (present only on scoped resources) |

**`status`** carries `id` (the server-generated UUID, read-only from `additionalStatusFields`) plus the
standard Krateo condition set; `Ready=True` when the resource is reconciled in Oxide.

Example `OxideDisk` (from `chart/samples/10-project-and-instance.yaml`) — note the flattened tagged-union
`disk_backend` and the `int64` byte `size`:

```yaml
apiVersion: oxide.ogen.krateo.io/v1alpha1
kind: OxideDisk
metadata:
  name: demo-boot
  namespace: krateo-system
spec:
  configurationRef:
    name: oxide-api
    namespace: krateo-system
  project: demo
  name: demo-boot
  description: "Boot disk created from an image"
  size: 21474836480             # 20 GiB, in bytes (int64)
  disk_backend:
    type: distributed
    disk_source:
      type: image
      image_id: "00000000-0000-0000-0000-000000000000"
```

## 3. The generated Configuration CRDs — `<Kind>Configuration`

Because every asset declares an `http`/`bearer` security scheme, `oasgen-provider` also generates a
`<Kind>Configuration` CRD per resource (`OxideProjectConfiguration`, `OxideInstanceConfiguration`, … one
for each of the 19 kinds). It carries the authentication block; a resource CR references it through
`spec.configurationRef`.

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
        name: oxide-api-token       # the Secret holding the Oxide token
        namespace: krateo-system
        key: token
```

`chart/samples/00-configurations.yaml` supplies all 19, each named `oxide-api` and pointing at the same
`oxide-api-token` Secret. Splitting them per kind is what lets you scope different tokens per resource if
you ever need to (e.g. a fleet-admin token only for `OxideSiloConfiguration`).

## 4. The `CompositionDefinition` (blueprint registration)

`compositiondefinition.yaml` registers the chart with Krateo:

```yaml
apiVersion: core.krateo.io/v1alpha1
kind: CompositionDefinition
metadata:
  name: krateo-oxide-operator-kog
  namespace: krateo-system
spec:
  chart:
    url: oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog
    version: "0.1.0"
```

From `chart/Chart.yaml`, core-provider derives the **Composition** API this blueprint installs as:

- **Group** = `composition.krateo.io`.
- **Kind** = PascalCase of the CD `metadata.name`, hyphens dropped: `krateo-oxide-operator-kog` →
  `KrateoOxideOperatorKog`.
- **Version** = the chart version with dots turned to hyphens and prefixed `v`: `0.1.0` → `v0-1-0`.

So a `Composition` CR is `composition.krateo.io/v0-1-0`, kind `KrateoOxideOperatorKog`; its `spec` mirrors
`values.yaml` ([configuration](./configuration.md)). `examples/composition.yaml` is a ready instance —
see [examples](./examples.md).

`spec.chart.version` must be a published chart version; bump it to the released tag after publishing
([release](./release.md)).
