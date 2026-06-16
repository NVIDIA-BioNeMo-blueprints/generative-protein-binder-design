# Deploying Protein Binder Design on Red Hat OpenShift AI

## What We're Deploying

The NVIDIA BioNeMo Blueprint for Protein Binder Design runs four NIM microservices that
together form an _in silico_ protein binder design pipeline:

| Component | Image | GPU | Port | Purpose |
|-----------|-------|-----|------|---------|
| AlphaFold2 | `nvcr.io/nim/deepmind/alphafold2:2.1` | 1 | 8081 | Predict monomer structure from amino acid sequence |
| RFDiffusion | `nvcr.io/nim/ipd/rfdiffusion:2.0` | 1 | 8082 | Generate protein binder backbones via diffusion |
| ProteinMPNN | `nvcr.io/nim/ipd/proteinmpnn:1.0` | 1 | 8083 | Design amino acid sequences for generated backbones |
| AlphaFold2-Multimer | `nvcr.io/nim/deepmind/alphafold2-multimer:2.1` | 1 | 8084 | Predict multimer structure of binder + target complex |

**Data flow:** Target sequence &rarr; AlphaFold2 (structure) &rarr; RFDiffusion (binder backbone)
&rarr; ProteinMPNN (binder sequence) &rarr; AlphaFold2-Multimer (complex structure) &rarr; PDB output

**Total resources:** 4 pods, 4 GPUs (minimum), ~1.3 TB model storage.

### NIM Operator Components

When deployed with the NIM Operator (recommended on OpenShift), each NIM is managed as a
pair of custom resources:

- **NIMCache** &mdash; downloads and caches the model on a PVC. Annotated with
  `helm.sh/resource-policy: keep` to survive upgrades and uninstalls.
- **NIMService** &mdash; runs the inference server, references its NIMCache for model storage,
  manages replicas, GPU allocation, and health probes.

## Tested Hardware

| Parameter | Value |
|-----------|-------|
| Platform | Red Hat OpenShift AI (RHOAI) 4.14+ |
| GPU nodes | 2+ nodes with NVIDIA A100 80 GB or H100 |
| GPUs per node | 2+ |
| Total GPUs | 4 (one per NIM) |
| CPU | 24+ cores across GPU nodes |
| RAM | 64 GB+ per GPU node |
| Storage | 1.5 Ti+ fast NVMe (dynamic provisioning recommended) |
| API keys | NVIDIA NGC API key |

**Minimum for reproduction:** 4 x NVIDIA L40s / A100 / H100 GPUs, 1.3 TB storage.

## What's Different from Upstream

| Area | Upstream Default | OpenShift Deployment | Impact |
|------|-----------------|---------------------|--------|
| Pod management | Raw Deployments | NIM Operator (NIMCache + NIMService) | Automated model download, cache lifecycle |
| External access | kubectl port-forward | OpenShift Routes with TLS | Production-grade ingress |
| Volume provisioning | hostPath PV + static PVC | Dynamic PVCs via NIM Operator | No manual PV creation required |
| Security context | Init container runs as root | anyuid SCC RoleBinding | Compatible with OpenShift SCC |
| Probe delays | 3600–21600 s initialDelay | startupProbe 120 s + failureThreshold | Faster failure detection |

## Prerequisites

### CLI Tools

- `oc` (OpenShift CLI) v4.14+
- `helm` v3.12+

### Cluster Requirements

- OpenShift 4.14+ with NVIDIA GPU Operator installed
- NIM Operator installed (provides `apps.nvidia.com/v1alpha1` API)
- At least 4 available GPUs

### Verify GPU Availability

```bash
oc get nodes -l nvidia.com/gpu.present=true -o custom-columns=\
NAME:.metadata.name,\
GPUS:.status.capacity.nvidia\.com/gpu,\
ALLOCATABLE:.status.allocatable.nvidia\.com/gpu
```

### Resource Requirements

| Resource | Per NIM | Total (4 NIMs) |
|----------|---------|----------------|
| GPU | 1 | 4 |
| Model storage | 100–500 Gi | ~1.5 Ti |

## Configuration Reference

### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `NGC_API_KEY` | Yes | — | NGC API key for image pulls and model downloads |

### OpenShift Block (`openshift:`)

| Key | Default | Description |
|-----|---------|-------------|
| `openshift.enabled` | `false` | Master toggle for all OpenShift resources |
| `openshift.routes.<nim>.enabled` | `false` | Create a Route for the named NIM service |
| `openshift.routes.<nim>.host` | `""` | Hostname (auto-generated if empty) |
| `openshift.routes.<nim>.tls.termination` | `edge` | TLS termination strategy |
| `openshift.scc.create` | `false` | Create anyuid SCC RoleBinding |

### NIM Operator Block (`nimOperator:`)

Each NIM (alphafold2, rfdiffusion, proteinmpnn, alphafold2-multimer) has:

| Key | Default | Description |
|-----|---------|-------------|
| `enabled` | `false` | Deploy this NIM via NIM Operator |
| `replicas` | `1` | Number of inference replicas |
| `image.repository` | (per NIM) | NGC container image |
| `image.tag` | (per NIM) | Image version |
| `resources.limits.nvidia.com/gpu` | `1` | GPU allocation |
| `storage.pvc.size` | `100–500Gi` | Model cache PVC size |
| `storage.pvc.storageClass` | `""` | StorageClass (uses cluster default if empty) |
| `expose.service.port` | `8081–8084` | Service port |

## Deployment

### 1. Create Namespace

```bash
oc new-project protein-design
```

### 2. Create NGC Secrets

The NGC registry secret must carry Helm ownership labels so `helm install` can adopt it:

```bash
export NGC_API_KEY="<your-ngc-api-key>"

oc create secret docker-registry ngc-secret-protein-design \
  --docker-server=nvcr.io \
  --docker-username='$oauthtoken' \
  --docker-password="${NGC_API_KEY}" \
  -n protein-design

oc label secret ngc-secret-protein-design \
  app.kubernetes.io/managed-by=Helm -n protein-design
oc annotate secret ngc-secret-protein-design \
  meta.helm.sh/release-name=protein-design \
  meta.helm.sh/release-namespace=protein-design -n protein-design
```

Create the NGC API secret for the NIM Operator:

```bash
oc create secret generic ngc-api \
  --from-literal=NGC_API_KEY="${NGC_API_KEY}" \
  -n protein-design

oc label secret ngc-api \
  app.kubernetes.io/managed-by=Helm -n protein-design
oc annotate secret ngc-api \
  meta.helm.sh/release-name=protein-design \
  meta.helm.sh/release-namespace=protein-design -n protein-design
```

Create the NGC registry key secret referenced by Deployment env vars:

```bash
oc create secret generic ngc-registry-secret \
  --from-literal=NGC_REGISTRY_KEY="${NGC_API_KEY}" \
  -n protein-design

oc label secret ngc-registry-secret \
  app.kubernetes.io/managed-by=Helm -n protein-design
oc annotate secret ngc-registry-secret \
  meta.helm.sh/release-name=protein-design \
  meta.helm.sh/release-namespace=protein-design -n protein-design
```

### 3. Install the Chart

```bash
cd protein-design-chart/

helm install protein-design . \
  -f values.yaml \
  -f values-openshift.yaml \
  -n protein-design
```

This creates:
- 4 NIMCache resources (one per NIM — triggers model download)
- 4 NIMService resources (one per NIM — runs inference after cache is ready)
- 4 OpenShift Routes (edge TLS termination)
- 1 RoleBinding granting anyuid SCC to the default ServiceAccount
- Standard Deployments are scaled to 0 (`replicaCount: 0` in overlay)

### 4. Monitor Model Downloads

NIMCache resources download models from NGC. This can take 30–60+ minutes per NIM
depending on network speed and model size (100–500 GB each).

```bash
oc get nimcache -n protein-design -w
```

Wait until all caches show `Ready`:

```
NAME                       STATUS   AGE
alphafold2-cache           Ready    45m
rfdiffusion-cache          Ready    38m
proteinmpnn-cache          Ready    15m
alphafold2-multimer-cache  Ready    48m
```

## Verification

### Check All Pods

```bash
oc get pods -n protein-design
```

Expected pods (NIM Operator creates these after NIMCache is ready):

```
NAME                                    READY   STATUS    RESTARTS   AGE
alphafold2-0                            1/1     Running   0          10m
rfdiffusion-0                           1/1     Running   0          10m
proteinmpnn-0                           1/1     Running   0          10m
alphafold2-multimer-0                   1/1     Running   0          10m
```

### Check NIMService Status

```bash
oc get nimservice -n protein-design
```

### Health Checks

```bash
for svc in alphafold2 rfdiffusion proteinmpnn alphafold2-multimer; do
  echo "--- ${svc} ---"
  oc exec deploy/${svc} -n protein-design -- \
    curl -s http://localhost:8000/v1/health/ready
  echo
done
```

## Accessing the NIM APIs

Since this blueprint has no frontend UI, the NIM APIs are accessed directly via the
Jupyter notebook. There are two ways to run the notebook against the OpenShift deployment.

### Option A: RHOAI Workbench (recommended)

Run the notebook inside an RHOAI workbench in the **same namespace** as the NIM services.
The workbench connects to NIMs via cluster-internal Kubernetes DNS names — no Routes or
port-forwarding needed.

#### 1. Create the Workbench

In the RHOAI dashboard, create a new workbench in the `protein-design` namespace:

- **Image:** Jupyter minimal or Jupyter data science
- **Environment variables** (set during workbench creation):

| Variable | Value | Purpose |
|----------|-------|---------|
| `DEPLOYMENT_MODE` | `openshift` | Switches notebook to cluster-internal URLs |
| `NVIDIA_API_KEY` | `<your-ngc-api-key>` | NGC authentication for NIM API calls |

Optional overrides (only needed if your service names differ from the NIM Operator defaults):

| Variable | Default | Description |
|----------|---------|-------------|
| `ALPHAFOLD2_HOST` | `http://alphafold2` | AlphaFold2 NIMService URL |
| `RFDIFFUSION_HOST` | `http://rfdiffusion` | RFDiffusion NIMService URL |
| `PROTEINMPNN_HOST` | `http://proteinmpnn` | ProteinMPNN NIMService URL |
| `AF2_MULTIMER_HOST` | `http://alphafold2-multimer` | AlphaFold2-Multimer NIMService URL |

If you are using the standard Helm chart Deployments (not NIM Operator), set the host
variables to match your release name, e.g.:

```
ALPHAFOLD2_HOST=http://<release>-protein-design-chart-alphafold2
```

#### 2. Upload and Run the Notebook

Upload `src/protein-binder-design.ipynb` to the workbench and run all cells. The notebook
automatically detects `DEPLOYMENT_MODE=openshift` and uses cluster-internal URLs:

```
OpenShift/RHOAI Mode - Using Helm-deployed NIM services

Service URLs:
  ALPHAFOLD2: http://alphafold2:8081
  RFDIFFUSION: http://rfdiffusion:8082
  PROTEINMPNN: http://proteinmpnn:8083
  AF2_MULTIMER: http://alphafold2-multimer:8084
```

The comprehensive health check cell verifies all four NIMs are reachable before the
pipeline starts.

#### 3. Fallback: Set DEPLOYMENT_MODE in the Notebook

If you forgot to set `DEPLOYMENT_MODE` when creating the workbench, uncomment this line
in the NIM configuration cell:

```python
os.environ['DEPLOYMENT_MODE'] = 'openshift'
```

### Option B: External Access via Routes or Port-Forwarding

For running the notebook on your local machine against the OpenShift cluster:

**Using Routes:**

```bash
oc get routes -n protein-design
```

Update the NIM host environment variables to use Route hostnames (with `https://` prefix
and port `443`).

**Using port-forward:**

```bash
oc port-forward svc/protein-design-protein-design-chart-alphafold2 8081:8081 -n protein-design &
oc port-forward svc/protein-design-protein-design-chart-rfdiffusion 8082:8082 -n protein-design &
oc port-forward svc/protein-design-protein-design-chart-proteinmpnn 8083:8083 -n protein-design &
oc port-forward svc/protein-design-protein-design-chart-alphafold2multimer 8084:8084 -n protein-design &
```

Then run the notebook in local mode (default, no `DEPLOYMENT_MODE` needed):

```bash
cd src/
jupyter notebook
```

## Testing the Pipeline End-to-End

Pods being `Running` does not mean the pipeline works. Validate with an actual protein
sequence through all four steps.

### 1. Verify Each NIM Responds

```bash
ALPHAFOLD2_URL="http://localhost:8081"
RFDIFFUSION_URL="http://localhost:8082"
PROTEINMPNN_URL="http://localhost:8083"
MULTIMER_URL="http://localhost:8084"

for url in $ALPHAFOLD2_URL $RFDIFFUSION_URL $PROTEINMPNN_URL $MULTIMER_URL; do
  echo "--- ${url} ---"
  curl -s "${url}/v1/health/ready"
  echo
done
```

### 2. Run the Notebook

Open `src/protein-binder-design.ipynb` and execute all cells. The notebook:

1. Sends a target protein sequence to AlphaFold2 for structure prediction
2. Passes the structure to RFDiffusion for binder backbone generation
3. Sends the backbone to ProteinMPNN for sequence design
4. Submits the binder + target to AlphaFold2-Multimer for complex structure prediction
5. Outputs PDB files of the predicted multimer structures

A successful run produces PDB output in the final cells.

## OpenShift-Specific Challenges and Solutions

### 1. Init Container Requires Root (anyuid SCC)

**What happened:** The AlphaFold2 Deployment includes an init container
(`volume-permissions`) that runs with `runAsUser: 0` to `mkdir` and `chmod 777` the model
cache directory. OpenShift's default `restricted` SCC blocks this.

**Error:** `Error: container has runAsNonRoot and image has non-numeric user`

**Services affected:** AlphaFold2 (Deployment path only; NIM Operator path does not use
the init container).

**Fix:** The `openshift.yaml` template creates a RoleBinding granting `anyuid` SCC to the
`default` ServiceAccount when `openshift.scc.create: true`. This is set in the overlay.

### 2. hostPath PV Not Permitted on OpenShift

**What happened:** The upstream chart creates a `hostPath`-based PersistentVolume
(`pv.yaml`). OpenShift clusters typically restrict hostPath to privileged pods.

**Error:** PVC stays `Pending`; PV cannot bind.

**Services affected:** All four NIMs (they share a single PVC).

**Fix:** When using the NIM Operator path, each NIMCache creates its own dynamically
provisioned PVC. Set `persistence.storageClass` to your cluster's StorageClass (e.g.
`gp3-csi`, `ocs-storagecluster-cephfs`). The static hostPath PV/PVC are unused when
`replicaCount: 0`.

### 3. Large Model Downloads and Probe Timeouts

**What happened:** AlphaFold2 and AlphaFold2-Multimer models are hundreds of GB. The
upstream chart uses `initialDelaySeconds: 21600` (6 hours) on liveness/readiness probes
to accommodate this.

**Services affected:** AlphaFold2, AlphaFold2-Multimer.

**Fix:** The NIM Operator separates model downloading (NIMCache) from serving (NIMService).
The NIMService only starts after the cache is ready, so startup probes use reasonable
values (120 s initial delay, 30 s period, high failureThreshold). This improves failure
detection without risking premature termination.

### 4. GPU Scheduling and Tolerations

**What happened:** GPU nodes in OpenShift clusters often carry taints
(`nvidia.com/gpu=present:NoSchedule`) to prevent non-GPU workloads from landing on them.

**Services affected:** All four NIMs.

**Fix:** Add tolerations in the values overlay or per-NIM config:

```yaml
nimOperator:
  alphafold2:
    tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
```

### 5. Service Type Override for Routes

**What happened:** The upstream chart defaults to `ClusterIP`, which is correct for
OpenShift Routes. If it had used `NodePort` or `LoadBalancer`, Routes would conflict.

**Services affected:** All four NIMs.

**Fix:** The overlay explicitly sets `service.type: ClusterIP` and creates Routes for
external access. No action needed if the upstream default was already `ClusterIP`, but
the overlay makes the intent explicit.

### 6. NGC Secret Ownership by Helm

**What happened:** If NGC secrets are created with `oc create secret` before `helm install`,
Helm refuses to adopt them during install or upgrade.

**Error:** `Error: rendered manifests contain a resource that already exists`

**Services affected:** All (secret is shared).

**Fix:** Pre-label secrets with Helm ownership metadata before install (see
[Deployment > Create NGC Secrets](#2-create-ngc-secrets)):

```bash
oc label secret <name> app.kubernetes.io/managed-by=Helm
oc annotate secret <name> \
  meta.helm.sh/release-name=protein-design \
  meta.helm.sh/release-namespace=protein-design
```

### 7. TOKENIZERS_PARALLELISM Race Condition

**What happened:** NIM containers that use HuggingFace tokenizers may hit a thread pool
race condition, causing startup failures or intermittent crashes.

**Services affected:** Potentially all NIMs.

**Fix:** Set `TOKENIZERS_PARALLELISM=false` in each NIM's env block. The
`values-openshift.yaml` overlay includes this as a preventive measure.

## Cleanup

Remove the Helm release:

```bash
helm uninstall protein-design -n protein-design
```

**Important:** NIMCache PVCs persist after uninstall due to
`helm.sh/resource-policy: keep`. This is intentional — model downloads are expensive
(100–500 GB each). To remove them manually:

```bash
oc delete nimcache --all -n protein-design
oc delete pvc -l app.kubernetes.io/managed-by=nim-operator -n protein-design
```

Delete the namespace (removes all remaining resources):

```bash
oc delete project protein-design
```
