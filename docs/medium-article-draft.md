# Declaring Oxide Cloud VMs as Kubernetes YAML with Krateo — and the four `int64` gremlins I met along the way

*A hands-on story of turning the [Oxide](https://oxide.computer/) Cloud API into native Kubernetes
custom resources with [Krateo](https://krateo.io)'s Operator Generator (KOG) — and the four bugs that
stood between "it renders" and "the VM is actually running."*

---

## The one-paragraph pitch

I wanted to write this:

```yaml
apiVersion: oxide.ogen.krateo.io/v1alpha1
kind: OxideInstance
metadata:
  name: demo-vm
  namespace: krateo-system
spec:
  configurationRef:
    name: oxide-api
    namespace: krateo-system
  project: demo
  name: demo-vm
  memory: 4294967296   # 4 GiB
  ncpus: 2
  start: true
  boot_disk:
    type: attach
    name: demo-boot
  external_ips:
    - type: ephemeral
```

…run `kubectl apply`, and watch a real virtual machine boot in my Oxide silo. No hand-written controller.
Just Kubernetes as the API.

That's exactly what Krateo's **KOG** promises: point it at a curated subset of an OpenAPI spec, and it
generates the CRDs and a generic controller that reconciles them against the real cloud. It worked
beautifully for `OxideProject` and `OxideVpc` on the first try. The disk and the instance — the whole
point — did not. Getting them to run turned into a four-layer debugging session across three separate
Krateo components. Here's the trail.

## How the KOG fits together

Three moving parts:

- **`oasgen-provider`** reads a `RestDefinition` (a reference to a curated OpenAPI subset) and generates
  a CRD pair per resource — a `<Kind>Configuration` (holding the API token) and the resource `<Kind>`.
- **`rest-dynamic-controller`** (RDC) is the generic reconciler: for each CR it does create / get /
  update / delete against the REST API, and compares the live resource to your spec to detect drift.
- The **KOG package** (this repo) is just the curated OpenAPI subsets and the `RestDefinition`s — 19 of
  them, one per Oxide resource.

Byte-sized fields matter here. A disk `size` or an instance `memory` is measured in bytes, so a 20 GiB
disk is `21474836480` and 4 GiB of RAM is `4294967296` — both comfortably beyond the ~2.1 billion ceiling
of a signed 32-bit integer. Remember that number.

## Gremlin #1 — the 20 GiB disk that couldn't be 20 GiB (`int64` → `int32`)

The very first `kubectl apply` of a disk failed at *admission*:

```
OxideDisk ... is invalid: spec.size: Invalid value: "":
  must be of type integer with format int32
```

The curated schema clearly says `format: int64`. But the generated CRD validated `size` as **`int32`** —
so any value above ~2 GiB was rejected before it ever reached Oxide. A 20 GiB disk simply could not be
expressed.

Tracing it: `oasgen-provider` builds the JSON schema from the OpenAPI, then hands it to a CRD generator.
The bug was in the first step — the OpenAPI→JSON-schema converter **dropped the `format` keyword**,
appending it only to the field's *description* ("(format: int64)"). With no format on the schema, the CRD
generator defaulted integers to `int32`. The description literally told you the truth the schema had
thrown away.

**Fix:** preserve numeric `format` (`int32`/`int64`/`float`/`double`) through the conversion, so the CRD
keeps `format: int64`. Shipped as `oasgen-provider 0.10.4`. Now `size` and `memory` accept real values.

## Gremlin #2 — the resource that existed but was never `Ready` (`int64` vs `float64`)

With the disk now applying, it got *created in Oxide* — but the Kubernetes CR sat forever at
`READY=False` while `Synced=True`. The resource existed; the controller just refused to call it done.
Project and VPC (string-only fields) were `Ready=True`. Only the resources with big integers weren't.

The RDC decides "up to date" by comparing your spec against the API's GET response. Turning on debug
logging gave it away:

```
values differ, FirstValue=21474836480, SecondValue=2.147483648e+10
```

Same number. The CR stores `size` as an `int64` (`21474836480`); the JSON response decodes it as a
`float64`, which Go's `fmt` prints in exponential form (`2.147...e+10`). The comparator normalized numbers
by round-tripping through a string, but its number parser rejected the exponential form and fell back to
`float64` — so `int64(21474836480) != float64(21474836480)`, forever. **Every** `int64` field looked like
perpetual drift, so the resource never reached `Ready`.

**Fix:** coerce whole-number floats back to integers during comparison. Shipped as
`rest-dynamic-controller 0.9.1`. Ironic twin of #1: one component *dropped* `int64`, the other
*mishandled* it.

## Gremlin #3 — a boot disk modeled as the wrong shape

The instance create returned a flat `400`. A direct API call with a hand-written body worked, which
narrowed it to the request shape. The culprit was in the curated schema: `boot_disk` was modeled as a
**string** (a disk name), but Oxide's `InstanceCreate.boot_disk` is an **object** —
`{ type: attach|create, name, ... }`. A one-line-of-intent asset fix, but a real one: the KOG's schemas
have to match the API's actual shapes.

## The interesting one — apply-once, and the state machine that wasn't there

Now everything reconciled. But applying the disk and the instance **in the same manifest** left the
instance *stopped*, with its boot disk *detached* — and it never recovered.

The cause is a race with no self-healing: the instance controller creates the instance before the disk
has finished provisioning, and Oxide's create-time `boot_disk` attach **silently no-ops** on a not-ready
disk (it records the intended boot disk but doesn't attach the volume). The generic RDC compares spec to
the GET response; disk *attachment* isn't in that response, so it never notices anything is wrong.

The tempting fix is "wrap it in a Krateo Composition and let that sequence things." I checked the source:
the composition controller runs `helm install` with `Wait: false`, and Helm doesn't readiness-gate
arbitrary custom resources anyway. A Composition is a state machine for the *release* lifecycle — not a
sequencer of independent child-resource readiness. It would apply disk and instance together and hit the
same race.

So I put the state machine where it belongs: **in front of the resource.** Krateo supports
*plugins* — small wrapper web services the RDC calls instead of the raw API. I wrote one for
`OxideInstance` whose **`GET` is an idempotent reconcile**:

1. read the instance's recorded `boot_disk_id`;
2. if that disk isn't attached, attach it (stopping first if needed);
3. once attached and stopped, start the instance.

Because the controller GETs on every resync, the instance **converges to running regardless of apply
ordering** — a tiny finite-state machine that repairs the attach-and-start dance the generic CRUD model
can't express. It ships as a container image, published by CI, and the KOG chart wires it in with a
single value: `instancePlugin.enabled=true`.

## The payoff

Fresh kind cluster, the fixed providers, the plugin enabled, and one `kubectl apply` of project + VPC +
disk + instance — together, no ordering:

```
[2]  disk_ready=False attached=[]           instance=stopped     ← the race
[8]  disk_ready=True  attached=[]           instance=stopped
[12] disk_ready=True  attached=[demo-boot]  instance=starting    ← plugin attached it
[13] disk_ready=True  attached=[demo-boot]  instance=running     ← plugin started it
```

All four CRs report `READY=True`, and in the Oxide console:

![demo-vm running in the Oxide console with its 20 GiB boot disk attached](screenshots/oxide-instance.png)

A 2-vCPU / 4-GiB VM, booting an imported Alpine image, with a 20 GiB `int64` boot disk attached and an
external IP — declared entirely as Kubernetes YAML, self-healed to running.

![demo-boot 20 GiB attached to demo-vm](screenshots/oxide-disks.png)

## Takeaways

- **`int64` is a recurring failure mode in generated infrastructure APIs.** Byte-sized fields overflow
  `int32`, and `int64`/`float64` JSON round-trips break naive comparisons. It bit me in *two* different
  components, in opposite directions.
- **Generated CRUD gets you 80% for free — and the last 20% is stateful.** Attach, start, "must be
  stopped to resize": these aren't create/get/update/delete, and no amount of schema curation expresses
  them. That's what plugins are for.
- **Put the state machine at the right altitude.** A Composition orchestrates a release; it doesn't
  reconcile a resource's internal readiness. When "apply once and converge" is the requirement, the
  convergence has to live next to the resource.

*The fixes are open: `oasgen-provider 0.10.4`, `rest-dynamic-controller 0.9.1`, the
`oxide-rest-dynamic-controller-plugin`, and the KOG package with the `boot_disk` fix and the
`instancePlugin` toggle. Full reproducible walkthrough in the [quickstart](quickstart.md).*
