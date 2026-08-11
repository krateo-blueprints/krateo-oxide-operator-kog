---
type: Configuration
title: krateo-oxide-operator-kog — configuration
description: The whole values.yaml surface — the Oxide target, the API-token contract, the 19 per-resource RestDefinition toggles, the resource group and verbose flag, and the OxideInstance convergence plugin.
resource: oci://ghcr.io/krateo-blueprints/charts/krateo-oxide-operator-kog
tags: [kog, oxide, values, configuration, instance-plugin]
timestamp: 2026-08-11T00:00:00Z
---

# Configuration

Everything is `chart/values.yaml`, typed by `chart/values.schema.json`. Under a Krateo `Composition` the
same keys are set under `spec` of the `KrateoOxideOperatorKog` CR ([api](./api.md)); `oxide.apiUrl` is the
one required input.

## Oxide target (`oxide.*`)

| key | default | effect |
|---|---|---|
| `oxide.apiUrl` | `https://oxide.example.com` | **The key input.** Base URL of the Oxide silo API the generated controllers call — the same host the `oxide` CLI and web console talk to. **No trailing slash.** Injected into every OAS asset's `servers[0].url` via Helm `tpl` at install time. |

## API token (`auth.*`)

`rest-dynamic-controller` reads a bearer token from a Secret referenced by each CR's Configuration
(`spec.authentication.bearer.tokenRef`). Oxide device tokens are long-lived, so — unlike short-lived
Keycloak admin tokens — no rotation machinery is required: create the Secret once (see [usage](./usage.md))
and reference it. These values describe the shared Secret the sample Configuration CRs point at.

| key | default | effect |
|---|---|---|
| `auth.secretName` | `oxide-api-token` | Name of the Secret holding the Oxide API token. |
| `auth.secretKey` | `token` | Key within that Secret that holds the token value. |

## RestDefinitions (`restDefinitions.*`)

Which RestDefinitions to emit. Each maps 1:1 to an `assets/<key>.yaml` OAS. Every entry has an `enabled`
flag (all `true` by default) and a `kind` (the Kubernetes kind for the generated CRD, prefixed `Oxide` to
avoid crdgen collisions — [overview](./overview.md)). Resources are grouped by the token scope they need.

**Project scope (silo token):**

| key | kind | resource |
|---|---|---|
| `restDefinitions.project` | `OxideProject` | projects |
| `restDefinitions.instance` | `OxideInstance` | instances |
| `restDefinitions.disk` | `OxideDisk` | block-storage disks |
| `restDefinitions.snapshot` | `OxideSnapshot` | disk snapshots |
| `restDefinitions.image` | `OxideImage` | images |
| `restDefinitions.vpc` | `OxideVpc` | VPCs |
| `restDefinitions.vpcsubnet` | `OxideVpcSubnet` | VPC subnets |
| `restDefinitions.vpcrouter` | `OxideVpcRouter` | VPC routers |
| `restDefinitions.routerroute` | `OxideRouterRoute` | VPC router routes |
| `restDefinitions.floatingip` | `OxideFloatingIp` | floating IPs |
| `restDefinitions.networkinterface` | `OxideNetworkInterface` | instance network interfaces |
| `restDefinitions.affinitygroup` | `OxideAffinityGroup` | affinity groups |
| `restDefinitions.antiaffinitygroup` | `OxideAntiAffinityGroup` | anti-affinity groups |
| `restDefinitions.internetgateway` | `OxideInternetGateway` | internet gateways |

**Silo-user scope:**

| key | kind | resource |
|---|---|---|
| `restDefinitions.sshkey` | `OxideSshKey` | the authenticated user's SSH keys |

**Silo scope (certificates):**

| key | kind | resource |
|---|---|---|
| `restDefinitions.certificate` | `OxideCertificate` | TLS certificates |

**Fleet/system scope (fleet-admin token):**

| key | kind | resource |
|---|---|---|
| `restDefinitions.silo` | `OxideSilo` | silos |
| `restDefinitions.ippool` | `OxideIpPool` | system IP pools |
| `restDefinitions.subnetpool` | `OxideSubnetPool` | system subnet pools |

Disable any resource you don't need — or lack token scope for — by setting its `.enabled` to `false`.
The typical trim for a silo-collaborator token:

```bash
--set restDefinitions.silo.enabled=false \
--set restDefinitions.ippool.enabled=false \
--set restDefinitions.subnetpool.enabled=false
```

Overriding a `.kind` also renames the generated CRD; the default `Oxide`-prefixed names are chosen to
avoid crdgen collisions, so change them only with care.

## CRD group and logging

| key | default | effect |
|---|---|---|
| `resourceGroup` | `oxide.ogen.krateo.io` | Kubernetes API group the generated CRDs register under. GVK = `<group>/v1alpha1`. |
| `verbose` | `false` | Verbose logging on the generated connectors (stamps the `krateo.io/connector-verbose: "true"` annotation on each RestDefinition). |

## OxideInstance convergence plugin (`instancePlugin.*`)

The optional wrapper web service that makes an instance converge to running regardless of apply ordering —
it attaches the boot disk and starts the instance on reconcile ([overview](./overview.md)). When enabled,
the OxideInstance OAS `servers[0].url` points at the plugin instead of the raw silo API.

| key | default | effect |
|---|---|---|
| `instancePlugin.enabled` | `false` | Deploy the plugin (Deployment + Service, `templates/instance-plugin.yaml`) and route the OxideInstance controller through it. Off = plain create/get/update/delete against the silo API. |
| `instancePlugin.image.repository` | `ghcr.io/krateo-blueprints/oxide-rest-dynamic-controller-plugin` | The plugin image. |
| `instancePlugin.image.tag` | `0.1.0` | The plugin image tag. |
| `instancePlugin.image.pullPolicy` | `IfNotPresent` | Image pull policy. |
| `instancePlugin.replicaCount` | `1` | Plugin Deployment replicas. |

The plugin reads `oxide.apiUrl` from an `OXIDE_API_URL` env var and forwards the bearer token to it; it
exposes `/healthz` on port 8080 for its liveness/readiness probes.
