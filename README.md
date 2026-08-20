# open5gs-gitops

Open5GS 5G SA core + UERANSIM RAN, deployed onto a **Sylva workload cluster** by the
**management cluster's Flux**, with this repository as the source of truth.

This is the declarative rewrite of `flux-demo-open5gs.md`, which reached the same end
state through `flux create ...` / `kubectl apply`. Nothing here is applied by hand.

## Layout

```
mgmt/       applied to the MANAGEMENT cluster, into the workload cluster's namespace
            HelmRepository + two HelmReleases; each HelmRelease carries
            kubeConfig.secretRef, so the Helm releases install into the WORKLOAD cluster
workload/   applied INTO the WORKLOAD cluster
            SCTP kernel-module DaemonSet (node prerequisite for Open5GS AMF)
sylva/      the unit definitions to paste into your Sylva workload-cluster values
```

The split is not cosmetic. `helm-controller` only exists on the management cluster, so
HelmRelease *objects* must live there; the DaemonSet is a plain workload and belongs in
the target cluster. Sylva expresses exactly this with the
`kustomization-deployed-on-mgmt-cluster` unit template.

## Why no Flux in the workload cluster

Sylva installs none, deliberately — none of its ~180 workload-cluster units include
one. Remote reconciliation via `kubeConfig.secretRef` keeps a single global dependency
DAG spanning "provision the cluster" and "deploy onto the cluster", one `flux get all -A`,
and a delete hook that can tear objects down (it selects them by
`spec.kubeConfig.secretRef.name`) before the cluster disappears. A second Flux would
split the DAG in two, and cross-cluster `dependsOn` does not exist.

## Install

Fresh server, workload cluster not yet created: follow [INSTALL.md](INSTALL.md), which covers
node sizing, `env_type`, and cluster creation as well.

Short version, if the workload cluster already exists:

1. Push this repo somewhere Flux can reach.
2. Set the URL in [`sylva/telco-demo-units.yaml`](sylva/telco-demo-units.yaml) and paste
   its contents into
   `sylva-core/environment-values/workload-clusters/my-workload/values.yaml`.
3. ```bash
   cd ~/sylva-core
   export KUBECONFIG=$PWD/management-cluster-kubeconfig
   ./apply-workload-cluster.sh environment-values/workload-clusters/my-workload
   ```

Then, from the management cluster:

```bash
export WC=my-workload-rke2-capm3-virt
flux get all -n $WC
```

## Day-2

```bash
vi mgmt/values/open5gs-values.yaml
git commit -am 'open5gs: ...' && git push
```

Flux picks it up within the `interval: 1m` poll. To not wait:

```bash
flux reconcile source git telco-demo -n $WC
```

`configMapGenerator` hashes the values file into the ConfigMap name, and
`mgmt/configuration.yaml` teaches kustomize to rewrite `HelmRelease.spec.valuesFrom` to
match — so a values edit is a real change to the HelmRelease, not a silently ignored one.

## What the demo did by hand, and where it went

| Demo step | Here |
| --- | --- |
| `flux create source git sylva-core-repo` + `flux create kustomization sandbox-ns` | dropped — `sandbox-privileged-namespace` is already a built-in Sylva unit; the units `depends_on` it |
| `flux create source helm gradiant` | `mgmt/helmrepository.yaml` |
| `flux create helmrelease open5gs --values=... ` + `kubectl apply` | `mgmt/open5gs-hr.yaml` + `mgmt/values/open5gs-values.yaml` |
| `yq -i '.spec.timeout = "30m"'` (the CLI's `--timeout` is its own wait, not Helm's) | written into the manifests |
| `--release-name=open5gs` | `releaseName:` in the manifests. Sylva-declared *units* get this for free (`templates/units.yaml` sets `releaseName` to the unit name), but these HelmReleases are hand-written, so it stays explicit |
| SCTP DaemonSet applied with `kubectl --kubeconfig=$WCK apply -f -` | `workload/sctp-module-loader.yaml`, ordered ahead of the core by `depends_on` |

## Known caveats

- `mongodb.image` is pinned to `bitnamilegacy/*` because Bitnami retired the public tag
  the chart defaults to. Revisit if you bump the open5gs chart version.
- The SCTP DaemonSet edits `/etc/modprobe.d` on the node, undoing one CIS hardening
  control on purpose. It is scoped to this demo cluster; do not carry it to a cluster
  you care about without deciding that trade explicitly.
- Adding or changing a *unit definition* still requires `apply-workload-cluster.sh`.
  Sylva's platform layer is push-based; only the contents of this repo are pull-based.
