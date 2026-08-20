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

If you have room, raise it in `environment-values/my-rke2-capm3-virt/values.yaml` (the
**management** values — `libvirt_metal` creates the VM from there):

```yaml
libvirt_metal:
  nodes:
    management-cp-0:
      memGB: 48
    workload-cp-0:      # add this block
      memGB: 32
      numCPUs: 8
```

then push the change to the management cluster:

```bash
./apply.sh environment-values/my-rke2-capm3-virt
```

Resizing later means destroying and re-provisioning the node, so decide now.

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

1. **Without it the render hard-fails.** The default is `env_type: prod`
   (`charts/sylva-units/values.yaml:12320`), which disables the `sandbox-privileged-namespace`
   unit (`values.yaml:8957`). The telco units declare `depends_on` it, and Sylva raises
   `unit 'telco-demo-node-prereqs' is declared with a dependency on disabled unit
   'sandbox-privileged-namespace'` (`templates/units.yaml:305`) rather than silently skipping.
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

On this single-node server the API endpoint should be directly reachable — no SSH tunnel needed.
If it is not, the tunnel recipe is in `flux-demo-open5gs.md`, Appendix A.

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
