# Boltz2 Helm Chart

Helm chart to deploy Boltz2 NIM on Kubernetes cluster.

## Prerequisites

- Configured Kubernetes cluster
- NVIDIA GPU nodes (if using GPU)
- NGC Registry Key from NVIDIA

## Set up Dependencies

- [Docker](https://docs.docker.com/engine/install/ubuntu/)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
- [Minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Farm64%2Fstable%2Fbinary+download) / Kubernetes cluster created using [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Helm](https://helm.sh/docs/intro/install/)

## Minikube Specific Configuration

Start Minikube:
```bash
minikube start --driver docker --container-runtime docker --gpus all --cpus 8
minikube addons enable nvidia-device-plugin
```

Minikube has a limitation when dealing with symbolic links - symbolic links inside a minikube pod can not be created in a mounted path from the host using `minikube mount <host_folder>:<minikube_target_path>`.

Instead, you can copy over the data using `minikube cp <Host models path> /data/nim` command from your host SSD to minikube host.
In [values.yaml](values.yaml), we define the minikube folder path that the PV is created under.

Note, it is important to save the copied files under a [specific locations](https://minikube.sigs.k8s.io/docs/handbook/persistent_volumes/) on the minikube container to prevent data loss between reboots.

## Quick Start

### 1. Configure NGC Registry Secret

Create Kubernetes secret with NGC registry key:
```bash
kubectl create secret generic ngc-registry-secret --from-literal=NGC_REGISTRY_KEY=<YOUR_NGC_REGISTRY_KEY>
```

### 2. Deploy Helm Chart

Set chart name:
```bash
export CHART_NAME=boltz2-nim
```
Install Helm Chart:
```bash
cd boltz2-nim-chart/
helm install "${CHART_NAME}" . --debug
```

### 3. Check Deployment

View pod status:
```bash
kubectl get pods
```

View logs:
```bash
kubectl logs <pod-name> -f
```

### 4. Access Service

Port forward to access from local machine:
```bash
kubectl port-forward service/"${CHART_NAME}-boltz2-nim-chart" 8080:8081
```

Service will be available at: http://localhost:8080

### 5. Test API endpoint
```bash
curl http://localhost:8080/v1/health/live
```

## Configuration

Edit `values.yaml` file to modify configuration:

### Boltz2 Container Configuration
- `boltz2.repository`: Docker image repository
- `boltz2.tag`: Docker image tag

## Uninstall

To remove deployment:
```bash
helm uninstall "${CHART_NAME}" --wait
```

## Troubleshooting

### Check pod status
```bash
kubectl describe pod <pod-name>
```

### View logs
```bash
kubectl logs <pod-name> -f
```

### Test GPU access (if available)
```bash
kubectl run gpu-test --image=nvidia/cuda:12.6.2-base-ubuntu22.04 --restart=Never --command -- nvidia-smi
```

## Notes

- Due to large model files, pods may take several minutes to start
- Ensure cluster has sufficient resources (CPU, Memory, Storage)
- For GPU workloads, NVIDIA device plugin must be installed on the cluster
