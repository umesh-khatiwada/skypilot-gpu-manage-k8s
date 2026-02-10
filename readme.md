# SkyPilot GPU Management on Kubernetes

This repository contains configurations and examples for running SkyPilot workloads on Kubernetes clusters with both NVIDIA and AMD GPUs.

## Overview

SkyPilot is a framework for running AI/ML workloads seamlessly across different cloud providers and on-premise clusters. This project demonstrates:

- **Multi-GPU Support**: Both NVIDIA (RTX 4060) and AMD (MI300X) GPUs
- **Kubernetes Integration**: Native K8s deployment using k3s
- **Advanced Features**: Managed jobs, model serving, distributed training, and data management

## Hardware Configuration

### NVIDIA Setup
- **GPU**: RTX 4060 (Shared)
- **Driver**: NVIDIA GPU Operator
- **Runtime**: CUDA/cuDNN

### AMD Setup  
- **GPU**: AMD Instinct MI300X VF
- **VRAM**: ~191GB per GPU
- **Driver**: AMD GPU Operator v1.2.2
- **Runtime**: ROCm with PyTorch
- **GFX Version**: gfx942
- **VBIOS**: 113-M3000100-103

## Prerequisites

- Kubernetes cluster (k3s recommended)
- Python 3.8+
- kubectl configured with cluster access
- GPU operators installed (NVIDIA or AMD)

## Installation

### 1. Install SkyPilot

```bash
# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install SkyPilot with Kubernetes support
pip install --upgrade "skypilot[kubernetes]"
```

### 2. Configure Kubernetes

```bash
# Set kubeconfig
export KUBECONFIG=/path/to/your/kubeconfig.yaml

# Verify GPU detection
sky show-gpus --infra k8s
```

## Project Structure

```
.
├── skypilot_tasks/
│   ├── train.yaml                  # NVIDIA basic training
│   ├── train_amd.yaml             # AMD basic diagnostics ✅
│   ├── managed_train_amd.yaml     # AMD managed jobs ✅
│   ├── serve_amd.yaml             # AMD model serving ✅
│   └── distributed_amd.yaml       # AMD distributed training ✅
├── amd-setup/
│   ├── readme.md                  # Detailed AMD setup guide
│   └── diag-pod.yaml             # Direct ROCm diagnostics
└── readme.md                      # This file
```

## Usage Examples

### Basic GPU Diagnostics (AMD)

```bash
sky launch skypilot_tasks/train_amd.yaml --infra k8s -y --down
```

**Features:**
- ROCm SMI information
- PyTorch GPU detection
- Memory verification
- ~191GB VRAM available

### Managed Jobs (Fault Tolerance)

```bash
sky jobs launch skypilot_tasks/managed_train_amd.yaml --infra k8s -y
```

**Features:**
- Automatic retries on failure
- Checkpoint support
- Local code sync via `workdir`
- Background execution

### Model Serving

```bash
sky serve up skypilot_tasks/serve_amd.yaml --infra k8s -y
```

**Features:**
- Multi-replica serving
- Auto-scaling (1-3 replicas)
- Health checks and readiness probes
- Port 8000 exposed

### Distributed Training

```bash
sky launch skypilot_tasks/distributed_amd.yaml --infra k8s -y
```

**Features:**
- Multi-node support (scale `num_nodes`)
- RCCL for AMD GPUs
- Torchrun integration
- Automatic rank/world size assignment

## Validated Configurations

All AMD manifest files have been validated:

| File | Resources | Status | Purpose |
|------|-----------|--------|---------|
| `train_amd.yaml` | 4 vCPUs, 16GB, MI300X:1 | ✅ | Basic diagnostics |
| `managed_train_amd.yaml` | 8 vCPUs, 32GB, MI300X:1 | ✅ | Managed jobs |
| `serve_amd.yaml` | 8 vCPUs, 64GB, MI300X:1 | ✅ | Model serving |
| `distributed_amd.yaml` | 16 vCPUs, 128GB, MI300X:1 | ✅ | Distributed training |

## Advanced Features

### Data Management

**Local Code Sync:**
```yaml
workdir: .
```

**External Data Mounts:**
```yaml
file_mounts:
  /data: s3://my-bucket/dataset
  /models: gs://my-models
```

### Environment Variables

```yaml
envs:
  NCCL_DEBUG: INFO      # NVIDIA debugging
  RCCL_DEBUG: INFO      # AMD debugging
  HF_HOME: /tmp/hf      # Hugging Face cache
```

### Custom Images

```yaml
resources:
  image_id: docker:rocm/pytorch:latest  # AMD
  # or
  image_id: docker:nvidia/cuda:12.1.0-runtime-ubuntu22.04  # NVIDIA
```

## Troubleshooting

### GPU Not Detected

```bash
# Check GPU operator status
kubectl get pods -n gpu-operator

# Verify node labels
kubectl get nodes --show-labels | grep gpu
```

### SkyPilot Not Finding GPUs

```bash
# Check SkyPilot GPU detection
sky show-gpus --infra k8s

# Verify cluster access
sky check kubernetes
```

### AMD ROCm Issues

```bash
# Run direct diagnostics pod
kubectl apply -f amd-setup/diag-pod.yaml
kubectl logs amd-gpu-diag
```

## Key Differences: NVIDIA vs AMD

| Feature | NVIDIA | AMD |
|---------|--------|-----|
| **Runtime** | CUDA | ROCm |
| **Collective Ops** | NCCL | RCCL |
| **Debug Flag** | `NCCL_DEBUG` | `RCCL_DEBUG` |
| **Docker Images** | `nvidia/cuda:*` | `rocm/pytorch:latest` |
| **GPU Detection** | `torch.cuda.*` | `torch.cuda.*` (same API) |

## Performance Notes

- **AMD MI300X**: 191GB VRAM, excellent for large models
- **NVIDIA RTX 4060**: Shared resources, good for development
- Both support PyTorch with identical API
- ROCm provides full PyTorch compatibility

## Documentation

- **AMD Setup Guide**: See [amd-setup/readme.md](amd-setup/readme.md)
- **SkyPilot Docs**: https://skypilot.readthedocs.io/
- **ROCm Docs**: https://rocm.docs.amd.com/

## Contributing

Contributions welcome! Please ensure:
1. All manifests pass `sky launch --dryrun` validation
2. Documentation is updated
3. Examples are tested on actual hardware

## License

This project is licensed under the MIT License.

## Acknowledgments

- SkyPilot team for the excellent framework
- AMD for ROCm and MI300X support
- Kubernetes GPU Operator maintainers