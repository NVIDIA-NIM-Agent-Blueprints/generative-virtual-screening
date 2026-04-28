# Deploying Generative Virtual Screening on Red Hat OpenShift AI

## What We're Deploying

The Generative Virtual Screening Blueprint runs four NVIDIA BioNeMo NIM microservices
that form an end-to-end drug discovery pipeline:

| Component | Image | GPU | Purpose |
|-----------|-------|-----|---------|
| **MSA-Search** (MMSeqs2) | `nvcr.io/nim/colabfold/msa-search:1.0` | 1 | Multiple sequence alignment from protein databases (~1.3 TB) |
| **OpenFold2** | `nvcr.io/nim/openfold/openfold2:1.0` | 1 | Predicts 3D protein structure from sequence + MSA |
| **GenMol** | `nvcr.io/nim/nvidia/genmol:1.0` | 1 | Generates optimized drug-like molecules |
| **DiffDock** | `nvcr.io/nim/mit/diffdock:2.2` | 1 | Predicts protein-ligand binding poses |

**Data flow:** Protein sequence → MSA-Search → OpenFold2 (3D structure) → GenMol (molecule generation) → DiffDock (docking)

**Total:** 4 pods, 4 GPUs minimum.

On OpenShift the NIMs are deployed via the **NIM Operator** as NIMCache + NIMService
custom resource pairs, rather than raw Deployments. The Operator handles model
downloading, caching on PVCs, GPU scheduling, and lifecycle management.

---

## Tested Hardware

| Resource | Specification |
|----------|---------------|
| GPU | 4 × NVIDIA L40s / A100 / H100 (1 per NIM) |
| VRAM | 48 GB per GPU minimum |
| Storage | 1.5 TB (MSA databases) + 500 GB (OpenFold2) + 100 GB (GenMol) + 100 GB (DiffDock) |
| CPU | 24+ cores |
| RAM | 64 GB minimum |
| API Keys | NGC API key with access to BioNeMo NIM containers |

---

## What's Different from Upstream

| Area | Upstream Default | OpenShift Overlay | Impact |
|------|------------------|-------------------|--------|
| NIM lifecycle | Raw Deployments | NIM Operator (NIMCache + NIMService) | Automatic model caching, GPU scheduling |
| Storage | hostPath `/data/nim` | PVC with dynamic provisioning | No node-local dependency |
| External access | kubectl port-forward | OpenShift Routes | Browser-accessible URLs |
| Security | Default pod security | Custom SCC + RoleBinding | MSA init container needs root |
| Volume permissions | Init container with `runAsUser: 0` | Handled by custom SCC | SCC allows root for init step |

---

## Prerequisites

### CLI Tools

- `oc` (OpenShift CLI) logged into your cluster
- `helm` v3.12+

### Cluster Requirements

- Red Hat OpenShift 4.14+
- NVIDIA GPU Operator installed and configured
- NIM Operator installed (provides `apps.nvidia.com/v1alpha1` API)
- At least 4 GPU nodes available

### Verify GPU Availability

```bash
oc get nodes -l nvidia.com/gpu.present=true
oc describe node <gpu-node> | grep -A 5 "Allocatable"
```

### NGC Secrets

Create the image pull secret and API secret in your target namespace:

```bash
export NAMESPACE=gvs
oc new-project $NAMESPACE || oc project $NAMESPACE

# Image pull secret (for pulling NIM containers from nvcr.io)
oc create secret docker-registry ngc-secret-docker-registry \
  --docker-server=nvcr.io \
  --docker-username='$oauthtoken' \
  --docker-password="$NGC_API_KEY" \
  -n $NAMESPACE

# Label for Helm adoption
oc label secret ngc-secret-docker-registry \
  app.kubernetes.io/managed-by=Helm -n $NAMESPACE
oc annotate secret ngc-secret-docker-registry \
  meta.helm.sh/release-name=gvs \
  meta.helm.sh/release-namespace=$NAMESPACE -n $NAMESPACE

# API secret (for NIM Operator model downloads)
oc create secret generic ngc-api \
  --from-literal=NGC_API_KEY="$NGC_API_KEY" \
  -n $NAMESPACE

oc label secret ngc-api \
  app.kubernetes.io/managed-by=Helm -n $NAMESPACE
oc annotate secret ngc-api \
  meta.helm.sh/release-name=gvs \
  meta.helm.sh/release-namespace=$NAMESPACE -n $NAMESPACE

# Link pull secret to nim-cache-sa (used by NIM Operator for model downloads)
oc create sa nim-cache-sa -n $NAMESPACE || true
oc secrets link nim-cache-sa ngc-secret-docker-registry --for=pull -n $NAMESPACE
```

---

## Configuration Reference

### `openshift` Block

| Key | Default | Description |
|-----|---------|-------------|
| `openshift.enabled` | `false` | Master toggle for all OpenShift resources |
| `openshift.routes.<nim>.enabled` | `false` | Create an OpenShift Route for the NIM service |
| `openshift.routes.<nim>.host` | `""` | Custom hostname (auto-generated if empty) |
| `openshift.routes.<nim>.tls.termination` | `edge` | TLS termination strategy |
| `openshift.scc.create` | `false` | Create a custom SCC for NIM pods |
| `openshift.scc.priority` | `10` | SCC priority |

### `nimOperator` Block

Each NIM (`msa-search`, `openfold2`, `genmol`, `diffdock`) has the same schema:

| Key | Description |
|-----|-------------|
| `nimOperator.<nim>.enabled` | Enable NIMCache + NIMService for this NIM |
| `nimOperator.<nim>.service.name` | Kubernetes service name (used for DNS) |
| `nimOperator.<nim>.image.repository` | Container image |
| `nimOperator.<nim>.image.tag` | Image tag |
| `nimOperator.<nim>.resources` | GPU resource requests/limits |
| `nimOperator.<nim>.storage.pvc.*` | PVC settings for model cache |
| `nimOperator.<nim>.tolerations` | Node tolerations for GPU taints |
| `nimOperator.<nim>.env` | Environment variables |

---

## Deployment

```bash
cd generative-virtual-screening-chart/

helm install gvs . \
  -f values.yaml \
  -f values-openshift.yaml \
  --set imagePullSecret.secretName=ngc-registry-secret \
  -n gvs
```

To simulate NIM Operator presence when testing templates locally:

```bash
helm template gvs . \
  -f values.yaml \
  -f values-openshift.yaml \
  --api-versions apps.nvidia.com/v1alpha1
```

### What the Chart Creates

With the OpenShift overlay, `helm install` creates:

- 4 × NIMCache (model download and caching)
- 4 × NIMService (inference servers)
- 4 × OpenShift Routes (external access)
- 1 × Custom SCC (`gvs-nim`)
- 1 × RoleBinding (binds SCC to service accounts)

Standard Deployments are disabled (`replicaCount: 0`).

---

## Verification

### Check NIMCache Status

Model downloads can take significant time (MSA databases: ~4 hours, others: 10–60 min).

```bash
oc get nimcache -n gvs
```

Expected output when downloads are complete:

```
NAME              STATUS   AGE
msa-search-cache  Ready    4h
openfold2-cache   Ready    1h
genmol-cache      Ready    30m
diffdock-cache    Ready    45m
```

### Check NIMService Status

```bash
oc get nimservice -n gvs
```

### Check Pods

```bash
oc get pods -n gvs
```

All NIM pods should show `Running` with `1/1` ready containers.

### Health Endpoints

```bash
for svc in msa-search openfold2 genmol diffdock; do
  echo -n "$svc: "
  oc exec -n gvs deploy/$svc -- curl -s http://localhost:8000/v1/health/ready
  echo
done
```

---

## Running the Notebook on OpenShift AI

### Create an RHOAI Workbench

1. Open the OpenShift AI dashboard
2. Create a new workbench in the same namespace as the NIM deployment (e.g. `gvs`)
3. **Image:** Select **Jupyter | Minimal | CPU | Python 3.12** — the notebook only makes HTTP
   calls to the NIMs; no GPU, PyTorch, or CUDA is needed in the workbench itself
4. **Container size:** Small (2 CPU / 8 GB RAM) is sufficient
5. **Environment variable:** Add `DEPLOYMENT_MODE=openshift` as a ConfigMap variable
6. (Optional) Set per-NIM host overrides if you changed service names:
   - `MSA_HOST=http://<custom-msa-service>:8000`
   - `OPENFOLD2_HOST=http://<custom-openfold2-service>:8000`
   - `GENMOL_HOST=http://<custom-genmol-service>:8000`
   - `DIFFDOCK_HOST=http://<custom-diffdock-service>:8000`
7. Upload or clone the repository into the workbench
8. Open `src/generative-virtual-screening.ipynb` and run all cells

> **Scheduling note:** If the workbench pod stays in `Pending` with
> `FailedScheduling` / `Insufficient cpu` / `Insufficient memory`, CPU worker
> nodes may be full. Patch the Notebook CR to tolerate GPU node taints so the
> workbench can schedule on a GPU node (it won't consume a GPU):
>
> ```bash
> oc patch notebook <workbench-name> -n <namespace> --type=merge -p '{
>   "spec": {"template": {"spec": {"tolerations": [
>     {"key": "g6-gpu", "operator": "Equal", "value": "true", "effect": "NoSchedule"},
>     {"key": "nvidia.com/gpu", "operator": "Exists", "effect": "NoSchedule"}
>   ]}}}
> }'
> oc delete pod <workbench-name>-0 -n <namespace>  # trigger recreation
> ```
> Adjust the taint keys to match your cluster.

The notebook auto-detects `DEPLOYMENT_MODE` and switches from localhost URLs to
in-cluster Kubernetes DNS names (`http://<service-name>:8000`).

### Testing End-to-End

The notebook runs the full pipeline:

1. **MSA Search** — Submits the SARS-CoV-2 protease sequence, receives alignment data
2. **OpenFold2** — Predicts 3D protein structure from sequence + alignments
3. **GenMol** — Generates 5 optimized molecules from Nirmatrelvir seed
4. **DiffDock** — Docks generated molecules to the predicted structure

If all cells execute without error and produce visualizations, the deployment is validated.

---

## OpenShift-Specific Challenges and Solutions

### 1. MSA Init Container Requires Root

**What:** The MSA deployment includes an init container that runs `mkdir -p && chmod -R 777`
on the cache volume, requiring `runAsUser: 0`.

**Error:** `container has runAsNonRoot and image has non-numeric user` or
`unable to ensure permissions on /opt/nim/.cache`

**Fix:** The custom SCC (`gvs-nim`) sets `runAsUser: type: RunAsAny`, which allows the
init container to run as root. The SCC is created declaratively by the Helm chart when
`openshift.scc.create: true`.

### 2. Large PVC SELinux Relabeling

**What:** OpenShift's default restricted SCC triggers recursive SELinux relabeling on
mounted PVCs. For the MSA databases (1.3 TB, millions of files) this can add 30+ minutes
to every pod startup.

**Fix:** The custom SCC uses `seLinuxContext: type: RunAsAny`, which skips recursive
relabeling.

### 3. MSA Database Download Time

**What:** The MSA NIM downloads ~1.3 TB of sequence databases on first startup. This takes
approximately 4 hours depending on network and storage speed.

**Impact:** The NIMCache for MSA may show `InProgress` for several hours. The startup probe
has `failureThreshold: 1410` (235 minutes) to accommodate this.

**Mitigation:** Once downloaded, the NIMCache PVC is preserved across upgrades via
`helm.sh/resource-policy: keep`. Subsequent deployments skip the download.

### 4. GPU Node Tolerations

**What:** GPU nodes are often tainted to prevent non-GPU workloads from scheduling on them.

**Error:** Pods stuck in `Pending` state with `0/N nodes are available: N node(s) had
untolerated taint`.

**Fix:** The `values-openshift.yaml` overlay includes `tolerations` for
`nvidia.com/gpu: Exists`. Adjust the taint keys to match your cluster's actual GPU node
taints (check with `oc describe node <gpu-node> | grep Taints`).

### 5. TOKENIZERS_PARALLELISM Race Condition

**What:** The HuggingFace tokenizers library has a thread pool race condition that can
cause NIMs to crash or fail startup probes intermittently.

**Fix:** All NIM Operator env blocks include `TOKENIZERS_PARALLELISM=false` as a
preventive measure.

### 6. Helm Secret Adoption

**What:** If you create secrets with `oc create secret` before `helm install`, Helm
refuses to manage them on subsequent upgrades.

**Fix:** Pre-create secrets with Helm ownership labels:
```bash
oc label secret <name> app.kubernetes.io/managed-by=Helm
oc annotate secret <name> meta.helm.sh/release-name=gvs meta.helm.sh/release-namespace=gvs
```

### 7. hostPath Not Available on OpenShift

**What:** The upstream chart defaults to `persistence.volume.type: hostPath`, which
requires privileged access not available under standard OpenShift SCCs.

**Fix:** The overlay switches to `persistence.volume.type: pvc` with dynamic provisioning.

### 8. OpenFold2 PVC Too Small (200 GB → 500 GB)

**What:** OpenFold2 downloads model weights (~200 GB) but also extracts PDB template
databases at runtime into the same PVC. The default 200 GB PVC fills up completely,
causing `tar: Cannot write: No space left on device` errors during startup.

**Error:** Pod logs show `No space left on device` for `.cif` structure files.

**Fix:** Increase the OpenFold2 PVC to 500 GB in `values-openshift.yaml`. Note that
NIMCache PVCs are immutable — you must delete the NIMCache, NIMService, and PVC, then
re-run `helm upgrade` to apply the new size.

### 9. NIM_CACHE_PATH Conflicts with NIM Operator

**What:** The upstream `values.yaml` sets `NIM_CACHE_PATH=/opt/nim/.cache` for standard
Deployments. When the NIM Operator creates NIMService pods, it mounts the NIMCache PVC
at `/model-store` and expects NIM_CACHE_PATH to point there. If the env var from values
overrides the Operator's default, the NIM ignores its PVC and re-downloads the full
model to ephemeral container storage, which runs out of space.

**Error:** NIM logs show downloads to `/opt/nim/.cache/ngc/hub/...` instead of
`/model-store/ngc/hub/...`. Root filesystem fills up.

**Fix:** The `values-openshift.yaml` overlay omits `NIM_CACHE_PATH` from all NIM env
blocks. Since Helm arrays replace entirely (not merge), the overlay's env list takes
precedence and the NIM Operator correctly sets `NIM_CACHE_PATH=/model-store`.

### 10. DiffDock 2.0 Incompatible with NIM Operator

**What:** DiffDock 2.0 image does not contain the `download-to-cache` binary used by the
NIM Operator's NIMCache controller to download models.

**Error:** `CreateContainerError` on `diffdock-cache-job` with
`executable file not found in $PATH`.

**Fix:** Use DiffDock 2.2 (`nvcr.io/nim/mit/diffdock:2.2`), which includes NIM Operator
support. Version 2.2 also brings improved accuracy and BF16 support.

### 11. RHOAI Workbench FailedScheduling on Shared Clusters

**What:** When GPU nodes are tainted and CPU worker nodes are fully utilized, the RHOAI
workbench pod cannot schedule, even though it only needs a small CPU allocation.

**Error:** `FailedScheduling: Insufficient cpu, Insufficient memory, untolerated taint`

**Fix:** Patch the Notebook CR with tolerations for GPU node taints. The workbench will
schedule on a GPU node but won't consume a GPU (no `nvidia.com/gpu` resource request).
Delete the existing pod after patching to trigger recreation with the new tolerations.

---

## Cleanup

```bash
# Uninstall the Helm release
helm uninstall gvs -n gvs

# NIMCache PVCs persist by default (helm.sh/resource-policy: keep).
# Delete manually if you want to reclaim storage:
oc delete nimcache --all -n gvs
oc delete pvc -l app.kubernetes.io/instance=gvs -n gvs

# Delete the project
oc delete project gvs
```
