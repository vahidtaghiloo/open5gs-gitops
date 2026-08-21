# Bringing up the workload cluster + Open5GS on a fresh server

Starting point: the **management cluster is up**, the workload cluster does **not exist yet**,
single physical node.

End state: a single-node RKE2 workload cluster running Open5GS + UERANSIM, installed by the
management cluster's Flux from this repo, redeployed on every `git push`.

Steps 1–4 are edits. Step 6 is the one command that creates everything.
**Do steps 1 and 2 before step 6** — both are painful to change after the node is provisioned.

---

## Step 0 — check the management cluster is healthy

```bash
cd ~/sylva-core
export KUBECONFIG=$PWD/management-cluster-kubeconfig

kubectl get kustomization -n sylva-system management-flag     # must return one row
kubectl get cm -n sylva-system sylva-units-status             # must exist
flux get kustomizations -n sylva-system | grep -v " True "    # should print only the header
```

If `sylva-units-status` is missing, the management cluster never finished converging;
`apply-workload-cluster.sh` refuses to run and fixing that comes first.

---

## Step 1 — size the workload node

`workload-cp-0` defaults to **16 GB / 4 vCPU**
(`environment-values/base-capm3-virt/base/capm3-virt-values.yaml:144`). That node has to carry
RKE2 + Calico + Longhorn + the monitoring stack (Prometheus, Thanos, Grafana) **and** ~16 Open5GS
pods + MongoDB + UERANSIM. It is tight.

Check what the host can spare:

```bash
free -g; nproc
```

If you have room, resize it — **see [Appendix A](#appendix-a--resizing-workload-cp-0)**, because
this setting does *not* live in the workload cluster values and is not applied by any of the three
`*.sh` scripts. The VM is created by the `libvirt-metal` release on the **kind bootstrap cluster**.

Do it now: once the workload cluster is running on that node, resizing means destroying and
re-provisioning it.

If the host is too small, skip the resize and instead disable monitoring on the workload
cluster in step 4's file:

```yaml
units:
  monitoring:
    enabled: false
```

---

## Step 2 — set `env_type: dev`

Add to `environment-values/workload-clusters/my-workload/values.yaml`:

```yaml
env_type: dev
```

Two reasons, both load-bearing:

1. **It is half of what `sandbox-privileged-namespace` needs.** That unit creates the
   PSA-privileged `sandbox` namespace Open5GS runs in, and enabling it takes **two** things:
   `enabled: true` (already set for you in `telco-demo.yaml`) *and* `env_type` being `dev` or
   `ci`, which its `enabled_conditions` require (`values.yaml:8957`). The default is
   `env_type: prod` (`values.yaml:12320`). Miss either one and the render aborts with
   `unit 'telco-demo-node-prereqs' is declared with a dependency on disabled unit
   'sandbox-privileged-namespace'` (`templates/units.yaml:306`).

   Why `enabled: true` is needed at all: the unit definition has no `enabled` key, so it falls
   back to `units_enabled_default` — which is `false` for workload clusters (`values.yaml:306`;
   only `management.values.yaml:4` flips it to `true`). `enabled_conditions` are then ANDed on
   top (`_helpers.tpl:179-202`), so on a workload cluster the unit is off no matter what
   `env_type` says.
   No `sandbox` namespace also means nowhere for Open5GS to land.
2. **It right-sizes the stack.** Under `dev`, Prometheus/Thanos memory requests drop from 2Gi to
   512Mi, PVCs from 20Gi to 10Gi, and Prometheus retention from 50 GB to 8 GB. On a single small
   node this is the difference between scheduling and not.

Side effect: `enable_test_units` becomes true, so a few test units appear. Harmless here.

---

## Step 3 — push this repo and set its URL

Flux pulls over the network; a local directory will not do.

```bash
cd ~/thesis/open5gs-gitops
git remote add origin https://gitlab.com/<you>/open5gs-gitops.git
git push -u origin main
```

Then set that URL in `sylva/telco-demo-units.yaml`:

```yaml
source_templates:
  telco-demo:
    spec:
      url: https://gitlab.com/<you>/open5gs-gitops.git
```

Private repo? Uncomment the `auth:` block below it and put a read-only deploy token there;
Sylva generates the `Secret` and the `secretRef` for you.

---

## Step 4 — wire the units into the my-workload overlay

**Use a separate file, not an append to `values.yaml`.** This is Sylva's own pattern — see
`environment-values/workload-clusters/base-capm3-virt/base/kustomization.yaml`, which adds its
`capm3-virt-values` key exactly this way. Platform config and app config stay in separate files
with separate diffs.

**4a.** Copy the unit definitions in:

```bash
cp ~/thesis/open5gs-gitops/sylva/telco-demo-units.yaml \
   ~/sylva-core/environment-values/workload-clusters/my-workload/telco-demo.yaml
```

**4b.** Edit `environment-values/workload-clusters/my-workload/kustomization.yaml` so it reads:

```yaml
configMapGenerator:
  - name: sylva-units-values
    behavior: merge
    files:
      - values=values.yaml
      - telco-demo=telco-demo.yaml        # <-- added

patches:                                   # <-- added block
  - target:
      kind: SylvaUnitsRelease
      name: sylva-units
    patch: |
      - op: add
        path: /spec/valuesFrom/-
        value:
          type: ConfigMap
          name: sylva-units-values
          valuesKey: telco-demo
```

Two things this does. The `files:` line adds a second **key** to the same ConfigMap — you cannot
add a second file under the existing `values=` key, kustomize rejects the duplicate. The patch
appends a `valuesFrom` entry so `sylva-units-operator` actually reads that key; without it the
ConfigMap gains a key nobody looks at, and nothing happens.

`path: /spec/valuesFrom/-` appends, so `telco-demo` merges **last** and wins over `values.yaml`
on any overlapping key. That is what you want for an add-on.

Verify the wiring locally before touching the cluster:

```bash
cd ~/sylva-core
kustomize build environment-values/workload-clusters/my-workload | grep -A3 "telco-demo"
```

Expect the ConfigMap to carry three keys — `capm3-virt-values`, `values`, `telco-demo` — and a
`valuesFrom` entry naming `valuesKey: telco-demo` with the same hash-suffixed ConfigMap name.

---

## Step 5 — dry run

`preview.sh` renders every unit through a real Flux engine in a throwaway
`sylva-units-preview` namespace, with all Kustomizations suspended, so nothing deploys:

```bash
cd ~/sylva-core
./preview.sh environment-values/workload-clusters/my-workload
```

This is where a bad Go template, a typo'd unit name, or a dependency on a disabled unit surfaces
— in seconds, instead of thirty minutes into a cluster build.

---

## Step 6 — create the workload cluster

```bash
cd ~/sylva-core
export KUBECONFIG=$PWD/management-cluster-kubeconfig
./apply-workload-cluster.sh environment-values/workload-clusters/my-workload
```

**This is the only command you run by hand, and the only one you ever repeat** — and only when
`telco-demo.yaml` or `values.yaml` changes. Changes inside the `open5gs-gitops` repo never need it.

Expect 30–60 minutes: a libvirt VM is created, PXE-booted, inspected by Ironic, provisioned with
the OS image, RKE2 installed, then ~20 platform units, then the telco units.

---

## Step 7 — watch it converge

```bash
export WC=my-workload-rke2-capm3-virt

watch -n10 "kubectl get cluster,machine -n $WC"        # CAPI provisioning
flux get kustomizations -n $WC                          # the unit DAG
flux get helmreleases -n $WC                            # includes open5gs, ueransim-gnb
```

Order to expect, and why: `cluster` → `cluster-machines-ready` → `cluster-reachable` →
`calico-ready` → `sandbox-privileged-namespace` → `telco-demo-node-prereqs` (SCTP module) →
`telco-demo` (the two HelmReleases). Each arrow is a real `dependsOn` edge — Flux is the executor
of that graph, nothing here is timing luck.

Not-Ready units are normal for a long while. Stuck ones:

```bash
kubectl describe kustomization <name> -n $WC | tail -30
flux logs -f --namespace $WC
```

---

## Step 8 — verify Open5GS

Everything so far ran against the management cluster. To look **inside** the workload cluster,
extract its kubeconfig:

```bash
kubectl get secret ${WC}-kubeconfig -n $WC -o jsonpath='{.data.value}' | base64 -d > ~/sylva-core/wc-local
export WCK=~/sylva-core/wc-local
```

**You will need the socat tunnel for this.** The CAPI kubeconfig points at
`https://192.168.100.3:6443`, a libvirt network that exists only inside the `libvirt-metal` pods'
network namespace — it has no route on the host. Sylva rewrites the endpoint for the *management*
cluster only; there is no equivalent for workload clusters. Recipe: `flux-demo-open5gs.md`,
"The tunnel — reaching the workload cluster".

This affects inspection only. Flux never needs it: its controllers run on the management node at
`192.168.100.20`, already on that network.

```bash
kubectl --kubeconfig=$WCK get pods -n sandbox

# the node prerequisite actually took effect
kubectl --kubeconfig=$WCK -n sandbox logs -l app=sctp-module-loader --tail=5
#   expect: RESULT: sctp protocol module LOADED

# AMF is the canary: it is the one that dies without SCTP
kubectl --kubeconfig=$WCK -n sandbox logs deploy/open5gs-amf --tail=20
```

Expect ~16 Open5GS pods plus MongoDB, and `ueransim-gnb` with 2 UEs. To prove the 5G data path
end to end (`ping -I uesimtun0`), follow Appendix A of `flux-demo-open5gs.md`.

---

## Step 9 — day 2: this is the part that is GitOps

```bash
cd ~/thesis/open5gs-gitops
vi mgmt/values/open5gs-values.yaml
git commit -am "open5gs: ..." && git push
```

Nothing else. `GitRepository/telco-demo` polls every 60s, kustomize-controller reapplies,
helm-controller upgrades the release in the workload cluster.

Impatient:

```bash
flux reconcile source git telco-demo -n $WC
```

Confirm what is actually deployed:

```bash
kubectl get gitrepository telco-demo -n $WC -o jsonpath='{.status.artifact.revision}{"\n"}'
```

Re-run `apply-workload-cluster.sh` **only** for changes to `telco-demo.yaml` / `values.yaml`
(new unit, changed dependency, changed repo URL).

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `dependency on disabled unit 'sandbox-privileged-namespace'` | `env_type` is still `prod` | Step 2 |
| `units.telco-demo.repo refers to 'telco-demo' that isn't defined in .source_templates` | `telco-demo.yaml` not reaching the operator | Step 4b — the `patches` block is missing |
| ConfigMap has a `telco-demo` key but no unit appears | same — key present, `valuesFrom` entry absent | Step 4b |
| `GitRepository/telco-demo` not Ready, auth error | private repo without credentials | add `auth:` in `telco-demo.yaml` |
| `Kustomization/telco-demo` Ready, but no pods in `sandbox` | HelmReleases applied but not reconciled | `flux get helmreleases -n $WC`; check `kubeConfig` resolved — `kubectl get hr open5gs -n $WC -o jsonpath='{.spec.kubeConfig}'` |
| AMF crashloops, `socket create(2:1:132) failed (93:Protocol not supported)` | SCTP module not loaded | check `telco-demo-node-prereqs` is Ready before `telco-demo` |
| MongoDB `ImagePullBackOff` | Bitnami retired the tag | already pinned to `bitnamilegacy`; re-check if you bumped the chart |
| Pods `Pending`, insufficient cpu/memory | node too small | Step 1 — resize, or disable `monitoring` |

Full state dump for anything else:

```bash
cd ~/sylva-core && ./sylva-dump.sh
```

---

## Appendix A — resizing `workload-cp-0`

`libvirt_metal.nodes.workload-cp-0` is **not** a workload-cluster value, and none of `bootstrap.sh`
/ `apply.sh` / `apply-workload-cluster.sh` will apply a change to it:

- the `libvirt-metal` unit is defined only in `charts/sylva-units/bootstrap.values.yaml:487`, so it
  exists in the **bootstrap** context — the VMs live in pods on the kind cluster
  (`libvirt-metal-workload-cp-0-0`), not on the management cluster
- `apply.sh` acts on the management cluster, which has no such unit
- `bootstrap.sh` refuses to run once pivot has completed (`tools/shell-lib/common.sh:184`)
- the pivot job suspended the `sylva-units` HelmRelease on the kind cluster
  (`charts/sylva-units/scripts/pivot.sh:31-32`), so nothing re-renders it there

So: edit the values file to keep it the source of truth, then patch the live HelmRelease on kind.

**A.1 — record the intent** in `environment-values/my-rke2-capm3-virt/values.yaml`:

```yaml
libvirt_metal:
  nodes:
    management-cp-0:
      memGB: 48
    workload-cp-0:      # add this block
      memGB: 32
      numCPUs: 8
```

This does not apply anything by itself. It matters so a future rebuild from scratch reproduces the
same sizing.

**A.2 — check the host can take it.** `management-cp-0` already holds 48 GB:

```bash
free -g; nproc
```

**A.3 — patch the kind cluster.** `KUBECONFIG` must be **unset** — the kind cluster is the default
context:

```bash
unset KUBECONFIG
kubectl config current-context                     # a kind-* context
kubectl -n sylva-system get hr libvirt-metal
kubectl -n sylva-system get hr libvirt-metal -o jsonpath='{.spec.values.nodes}' | yq -P

kubectl -n sylva-system patch helmrelease libvirt-metal --type=merge \
  -p '{"spec":{"values":{"nodes":{"workload-cp-0":{"memGB":32,"numCPUs":8}}}}}'

flux -n sylva-system reconcile helmrelease libvirt-metal
```

A JSON merge patch merges nested maps, so `management-cp-0` is left untouched — confirm with the
`jsonpath` command above before and after.

**A.4 — the pod restarts and redefines the VM:**

```bash
kubectl -n sylva-system get pod libvirt-metal-workload-cp-0-0 -w
kubectl -n sylva-system exec libvirt-metal-workload-cp-0-0 -c vbmh -- \
  sh -c 'virsh dominfo $(virsh list --all --name | head -1)'
```

Expect the new `Max memory` and `CPU(s)`.

**Why doing this before step 6 is safe:** the BareMetalHost for `workload-cp-0` is created by the
*workload cluster* values (`environment-values/workload-clusters/base-capm3-virt/base/capm3-virt-values.yaml:98`,
via the `cluster-bmh` unit), so it does not exist yet. Nothing is running on the VM, and Ironic will
inspect the resized hardware fresh when step 6 registers it.

Because the patch lives only on the cluster, it is lost if the kind cluster is ever rebuilt — which
is exactly why A.1 is not optional.
