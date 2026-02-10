# SkyPilot Setup for AMD MI300X GPUs on Kubernetes

Complete guide for setting up SkyPilot with AMD MI300X GPUs on Kubernetes clusters.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Initial System Setup](#initial-system-setup)
- [SkyPilot Installation](#skypilot-installation)
- [AMD GPU Operator Installation](#amd-gpu-operator-installation)
- [Kubernetes GPU Verification](#kubernetes-gpu-verification)
- [SkyPilot AMD Integration](#skypilot-amd-integration)
- [Running Workloads](#running-workloads)
- [Troubleshooting](#troubleshooting)

## Prerequisites

- Kubernetes cluster with AMD MI300X GPUs
- `kubectl` configured with cluster access (`$HOME/.kube/config`)
- Helm 3.x installed
- Docker Hub account (or alternative container registry)
- Root or sudo access on the system

## Initial System Setup

### 1. Update System and Install Python

```bash
sudo apt update
sudo apt install -y python3-venv python3-full
```

### 2. Create Python Virtual Environment

```bash
python3 -m venv /opt/skypilot-venv
source /opt/skypilot-venv/bin/activate
```

**Note:** Add the activation command to your shell profile for persistence:
```bash
echo "source /opt/skypilot-venv/bin/activate" >> ~/.bashrc
```

## SkyPilot Installation

### 1. Install Required APT Packages

```bash
sudo apt update
sudo apt install -y kubectl socat netcat-openbsd
```

### 2. Install SkyPilot with Kubernetes Support

```bash
pip install --upgrade "skypilot[kubernetes]"
```

### 3. Verify Kubernetes Connection

```bash
sky check kubernetes
```

Expected output should show your Kubernetes cluster is accessible.

## AMD GPU Operator Installation

### 1. Install cert-manager

Add the Jetstack Helm repository:
```bash
helm repo add jetstack https://charts.jetstack.io --force-update
helm repo update
```

Install cert-manager v1.15.1:
```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.15.1 \
  --set crds.enabled=true
```

### 2. Install AMD GPU Operator

Add the ROCm Helm repository:
```bash
helm repo add rocm https://rocm.github.io/gpu-operator
helm repo update
```

Install the AMD GPU Operator v1.2.2:
```bash
helm install amd-gpu-operator rocm/gpu-operator-charts \
  --namespace kube-amd-gpu \
  --create-namespace \
  --version v1.2.2
```

### 3. Create Docker Registry Secret

Replace placeholders with your Docker Hub credentials:
```bash
kubectl create secret docker-registry my-docker-secret \
  -n kube-amd-gpu \
  --docker-username=YOUR_DOCKER_USERNAME \
  --docker-email=YOUR_EMAIL \
  --docker-password=YOUR_PASSWORD
```

### 4. Deploy AMD DeviceConfig

Create a file named `amd-device-config.yaml`:

```yaml
apiVersion: amd.com/v1alpha1
kind: DeviceConfig
metadata:
  name: gpu-operator
  namespace: kube-amd-gpu
spec:
  driver:
    enable: false
  devicePlugin:
    devicePluginImage: 'rocm/k8s-device-plugin:latest'
    devicePluginImagePullPolicy: IfNotPresent
    nodeLabellerImage: 'rocm/k8s-device-plugin:labeller-latest'
    nodeLabellerImagePullPolicy: IfNotPresent
    enableNodeLabeller: true
  metricsExporter:
    enable: false
  selector:
    feature.node.kubernetes.io/amd-gpu: 'true'
  testRunner:
    enable: true
    logsLocation:
      mountPath: /var/log/amd-test-runner
      hostPath: /var/log/amd-test-runner
```

Apply the configuration:
```bash
kubectl apply -f amd-device-config.yaml
```

## Kubernetes GPU Verification

### Check GPU Availability

Verify that Kubernetes detects the AMD GPUs:

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,GPUs:.status.capacity.'amd\.com/gpu'
```

### Verified System Details (MI300X)

Based on diagnostic runs:
- **GPU Model**: AMD Instinct MI300X VF
- **VRAM**: ~191 GB (205,822,885,888 Bytes) per GPU
- **GFX Version**: gfx942
- **VBIOS**: 113-M3000100-103
- **ROCm Status**: Verified with PyTorch (ROCm/CUDA available: True)

## SkyPilot AMD Integration

### 1. Label GPU Nodes for SkyPilot

Replace `<NODE_NAME>` with your actual node name:

```bash
# Label for SkyPilot MI300X recognition
kubectl label nodes <NODE_NAME> skypilot.co/accelerator=mi300x --overwrite

# Label for AMD GPU feature detection
kubectl label nodes <NODE_NAME> feature.node.kubernetes.io/amd-gpu=true --overwrite
```

### 2. Verify SkyPilot GPU Detection

```bash
sky show-gpus --infra k8s
```

Expected output:
```
Kubernetes GPUs
Context: amdxskypilot
GPU     REQUESTABLE_QTY_PER_NODE  UTILIZATION  
MI300X  1, 2, 4, 8                8 of 8 free  

Kubernetes per-node GPU availability
CONTEXT       NODE                                       GPU     UTILIZATION  
amdxskypilot  np-bdd12851-1.us-east1-a.compute.internal  MI300X  8 of 8 free
```

## Running Workloads

### Example: PyTorch Distributed Training

Create a file named `amd-minGPT.yaml`:

```yaml
name: amd-rocm-minGPT-ddp

resources:
  cloud: kubernetes
  image_id: docker:rocm/pytorch-training:v25.6
  accelerators: MI300X:1
  cpus: 10
  memory: 32+

setup: |
  echo "minGPT example derived from https://github.com/pytorch/examples"

run: |
  # Clone PyTorch examples
  git clone https://github.com/pytorch/examples.git
  cd examples/distributed/minGPT-ddp
  
  # Setup virtual environment
  python3 -m venv .venv
  source .venv/bin/activate
  pip install --upgrade pip
  pip install -r requirements.txt
  
  # Run distributed training
  echo "Running PyTorch minGPT example..."
  torchrun --nproc_per_node=4 ./mingpt/main.py
  
  # Check GPU status
  rocm-smi
```

### Launch the Job

```bash
sky launch amd-minGPT.yaml
```

## High-Level SkyPilot Features on AMD

The MI300X supports all of SkyPilot's advanced orchestration features.

### 1. Managed Jobs (Fault-Tolerance)
Managed jobs automatically restart on failure and can be queued.

```bash
# Launch as a managed job
sky jobs launch skypilot_tasks/managed_train_amd.yaml
```

- **Resilience**: If a node fails, SkyPilot re-schedules the job.
- **Monitoring**: `sky jobs dashboard` or `sky jobs logs <job_id>`.

### 2. Sky Serve (Model Serving)
Scale your inference API across multiple MI300X nodes.

```bash
# Spin up a model serving endpoint
sky serve up skypilot_tasks/serve_amd.yaml
```

- **Auto-scaling**: Scale replicas based on traffic (QPS).
- **Endpoint**: Get your service URL with `sky serve status`.

### 3. Data Management
Sync local code and large datasets automatically.

```yaml
# In your task YAML:
workdir: ./my_project  # Syncs local folder to task
file_mounts:
  /data:
    source: s3://my-dataset # Syncs S3 bucket to MI300X local storage
```

### 4. Distributed Training
Scale to multi-node/multi-GPU clusters using RCCL.

```bash
# Launch on multi-node cluster
sky launch skypilot_tasks/distributed_amd.yaml
```

## Troubleshooting

### Common Issues

#### GPUs Not Detected
```bash
# Check AMD GPU operator pods
kubectl get pods -n kube-amd-gpu

# Check node labels
kubectl get nodes --show-labels | grep amd-gpu

# Verify device plugin logs
kubectl logs -n kube-amd-gpu -l app=amd-gpu-device-plugin
```

#### SkyPilot Can't Find GPUs
```bash
# Verify node labels are correct
kubectl describe node <NODE_NAME> | grep -A 10 Labels

# Ensure both labels are present:
# - skypilot.co/accelerator=mi300x
# - feature.node.kubernetes.io/amd-gpu=true
```

#### Docker Registry Authentication Issues
```bash
# Verify secret exists
kubectl get secret my-docker-secret -n kube-amd-gpu

# Recreate if needed
kubectl delete secret my-docker-secret -n kube-amd-gpu
kubectl create secret docker-registry my-docker-secret \
  -n kube-amd-gpu \
  --docker-username=YOUR_USERNAME \
  --docker-email=YOUR_EMAIL \
  --docker-password=YOUR_PASSWORD
```

#### Pod Scheduling Issues
```bash
# Check GPU resource allocation
kubectl describe node <NODE_NAME> | grep -A 5 "Allocated resources"

# View pod events
kubectl describe pod <POD_NAME> -n <NAMESPACE>
```

### Useful Commands

```bash
# Check all SkyPilot clusters
sky status

# SSH into a SkyPilot job
sky ssh <job-name>

# Stop a running job
sky stop <job-name>

# Delete a job
sky down <job-name>

# View GPU availability across all contexts
sky show-gpus

# Check ROCm version on GPU nodes
kubectl exec -it <POD_NAME> -- rocm-smi --showversion
```

## Additional Resources

- [SkyPilot Documentation](https://skypilot.readthedocs.io/)
- [AMD ROCm GPU Operator](https://rocm.github.io/gpu-operator/)
- [Crusoe AI Blog: Running AI Workloads on AMD GPUs](https://www.crusoe.ai/resources/blog/running-ai-workloads-on-amd-gpus-with-skypilot)
- [ROCm Documentation](https://rocm.docs.amd.com/)
