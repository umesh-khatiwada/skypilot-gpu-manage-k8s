# Nvidia GPU Setup for SkyPilot on Kubernetes

Follow these steps to configure your Kubernetes cluster with Nvidia GPUs for use with SkyPilot.

## 1. Prerequisites: Nvidia GPU Operator

If you are not using a managed service (like GKE/EKS) that pre-configures GPUs, you need to install the Nvidia GPU Operator to manage drivers and runtimes.

### Install via Helm
```bash
helm install --wait --generate-name \
     -n gpu-operator --create-namespace \
     nvidia/gpu-operator
```

### Verify Installation
Apply a test pod to ensure the Nvidia runtime is working correctly:

```bash
# Apply a test pod requesting a GPU
kubectl apply -f https://raw.githubusercontent.com/skypilot-org/skypilot/master/tests/kubernetes/gpu_test_pod.yaml

# Wait for the status to become 'Completed'
kubectl get pods -w
```

## 2. Label Nodes for SkyPilot

SkyPilot needs to identify which GPU models (e.g., A100, T4) are available on your nodes.

### Automatic Labeling (Recommended)
SkyPilot provides a utility to automatically inspect and label your nodes:

```bash
sky gpus label
```
*This runs a Kubernetes job to label your nodes. It may take a few minutes.*

### Manual Labeling (Alternative)
If you prefer to label manually, use the `skypilot.co/accelerator` label. Replace `<gpu_name>` with the lowercase simplified name (e.g., `h100`, `a10g`, `t4`).

```bash
kubectl label nodes <node-name> skypilot.co/accelerator=<gpu_name>
```

## 3. Verify SkyPilot Configuration

Finally, check that SkyPilot successfully detects your Kubernetes cluster and GPUs.

### Check Status
```bash
sky check
```
Look for `Kubernetes: Enabled`.

### List Available GPUs
Confirm that SkyPilot sees the GPU resources:

```bash
sky gpus list --infra k8s
```