# AMD GPU Complete Guide

> **Complete end-to-end guide for deploying AI models on AMD GPUs in OpenShift**

## 📋 Table of Contents

- [Overview](#overview)
- [Supported Hardware](#supported-hardware)
- [Prerequisites](#prerequisites)
- [Configuration](#configuration)
- [Deployment](#deployment)
- [Testing & Validation](#testing--validation)
- [Performance Benchmarks](#performance-benchmarks)
- [Troubleshooting](#troubleshooting)
- [Advanced Configurations](#advanced-configurations)
- [Best Practices](#best-practices)

---

## Overview

This guide covers everything you need to deploy AI models on AMD GPUs using ROCm and the OpenShift AI Observability Summarizer.

**What you'll learn:**
- How to configure AMD GPUs with ROCm for AI workloads
- Deploy models using make commands
- Test and validate deployments
- Optimize for AMD-specific features
- Troubleshoot common AMD GPU issues

**Time to complete:** 30-60 minutes

---

## Supported Hardware

### AMD GPU Models (Radeon Instinct MI Series)

| Model | Memory | Arch | Peak FP16 | Best For |
|-------|--------|------|-----------|----------|
| **MI300X** | 192GB HBM3 | CDNA 3 | 1.3 PFLOPS | Large models, training |
| **MI250X** | 128GB HBM2e | CDNA 2 | 383 TFLOPS | Training & inference |
| **MI250** | 128GB HBM2e | CDNA 2 | 383 TFLOPS | Training & inference |
| **MI210** | 64GB HBM2e | CDNA 2 | 181 TFLOPS | Inference workloads |
| **MI100** | 32GB HBM2 | CDNA | 184 TFLOPS | General purpose |

### Resource Information

- **Kubernetes Resource**: `amd.com/gpu`
- **Node Label**: `accelerator=amd`
- **Device Plugin**: AMD GPU k8s-device-plugin
- **Drivers**: ROCm (Radeon Open Compute)
- **Verification Command**: `rocm-smi`

---

## Prerequisites

### 1. ROCm Installation

AMD GPUs require ROCm (Radeon Open Compute) drivers:

```bash
# On each GPU node (run as root):

# Ubuntu 22.04
wget https://repo.radeon.com/amdgpu-install/latest/ubuntu/jammy/amdgpu-install_5.7.50700-1_all.deb
sudo apt install ./amdgpu-install_5.7.50700-1_all.deb
sudo amdgpu-install --usecase=rocm

# Verify installation
rocm-smi

# Expected output: Shows AMD GPU information
```

### 2. AMD GPU Device Plugin

```bash
# Deploy AMD GPU device plugin
kubectl apply -f https://raw.githubusercontent.com/RadeonOpenCompute/k8s-device-plugin/master/k8s-ds-amdgpu-dp.yaml

# Verify device plugin is running
kubectl get pods -n kube-system -l name=amd-gpu-device-plugin

# Check GPU resources
kubectl get nodes -o json | jq '.items[].status.capacity | select(.["amd.com/gpu"] != null)'
```

**Expected output:**
```json
{
  "amd.com/gpu": "8"
}
```

### 3. Node Setup

```bash
# Label AMD GPU nodes
kubectl label node <node-name> accelerator=amd

# Verify labels
kubectl get nodes -L accelerator

# Check node has AMD GPUs
kubectl describe node <node-name> | grep amd.com/gpu
```

### 4. ROCm Version Compatibility

```bash
# Check ROCm version
rocm-smi --showproductname
rocm-smi --showvbios

# Verify ROCm version matches container requirements
# ROCm 5.7.x recommended for vLLM support
```

### 5. Required Tools

- `kubectl` CLI with cluster access
- `helm` v3.x
- `make` (for using Makefile)
- OpenShift cluster with AMD GPU nodes
- Hugging Face token (for downloading models)

---

## Configuration

### Device Configuration Structure

```yaml
deviceConfig:
  amd-gpu:
    enabled: false
    resources:
      limits:
        amd.com/gpu: 1
        memory: 16Gi
        cpu: 8
      requests:
        amd.com/gpu: 1
        memory: 8Gi
        cpu: 4
    
    nodeSelector:
      accelerator: amd
    
    tolerations:
      - key: "amd.com/gpu"
        operator: "Exists"
        effect: "NoSchedule"
      - key: "accelerator"
        operator: "Equal"
        value: "amd"
        effect: "NoSchedule"
    
    env:
      - name: ROCR_VISIBLE_DEVICES
        value: "0"
      - name: HSA_OVERRIDE_GFX_VERSION
        value: "9.0.0"  # Adjust for your GPU
      - name: HIP_VISIBLE_DEVICES
        value: "0"
      - name: VLLM_USE_ROCM
        value: "1"
    
    volumeMounts:
      - name: amd-gpu-device
        mountPath: /dev/kfd
      - name: amd-gpu-render
        mountPath: /dev/dri
    
    volumes:
      - name: amd-gpu-device
        hostPath:
          path: /dev/kfd
          type: CharDevice
      - name: amd-gpu-render
        hostPath:
          path: /dev/dri
          type: Directory
```

### Multi-GPU Configuration (4 GPUs)

```yaml
deviceConfig:
  amd-gpu-multi:
    enabled: false
    resources:
      limits:
        amd.com/gpu: 4
        memory: 64Gi
        cpu: 32
      requests:
        amd.com/gpu: 4
        memory: 32Gi
        cpu: 16
    
    nodeSelector:
      accelerator: amd
      amd.com/gpu.count: "4"
    
    env:
      - name: ROCR_VISIBLE_DEVICES
        value: "0,1,2,3"
      - name: HIP_VISIBLE_DEVICES
        value: "0,1,2,3"
      - name: VLLM_USE_ROCM
        value: "1"
      - name: VLLM_TENSOR_PARALLEL_SIZE
        value: "4"
```

---

## Deployment

### Method 1: Using Makefile (Recommended)

#### Basic Deployment

```bash
# Deploy with default settings
make install \
  NAMESPACE=my-namespace \
  LLM=llama-3-2-3b-instruct \
  LLM_TOLERATION="amd.com/gpu"
```

#### With Safety Models

```bash
# Deploy with Llama Guard safety model
make install \
  NAMESPACE=my-namespace \
  LLM=llama-3-2-3b-instruct \
  LLM_TOLERATION="amd.com/gpu" \
  SAFETY=llama-guard-3-8b \
  SAFETY_TOLERATION="amd.com/gpu"
```

#### With Alerting

```bash
# Deploy with alerting enabled
make install \
  NAMESPACE=my-namespace \
  LLM=llama-3-2-3b-instruct \
  LLM_TOLERATION="amd.com/gpu" \
  ALERTS=TRUE
```

#### Large Model (70B) on 4 GPUs

```bash
# Deploy large model with tensor parallelism
make install \
  NAMESPACE=my-namespace \
  LLM=llama-3-3-70b-instruct \
  LLM_TOLERATION="amd.com/gpu"
```

### Method 2: Using Helm Directly

```bash
# Add Helm repository
helm repo add ai-architecture-charts https://rh-ai-quickstart.github.io/ai-architecture-charts
helm repo update

# Deploy with Helm
helm upgrade --install llm-service ai-architecture-charts/llm-service \
  --namespace my-namespace \
  --create-namespace \
  --set global.models.llama-3-2-3b-instruct.enabled=true \
  --set global.models.llama-3-2-3b-instruct.deviceConfig=amd-gpu \
  --set-json 'global.models.llama-3-2-3b-instruct.tolerations=[{"key":"amd.com/gpu","effect":"NoSchedule","operator":"Exists"}]' \
  --set llm-service.secret.hf_token="$HF_TOKEN"
```

### Available Models

```bash
# List all available models
make list-models

# Recommended models for AMD GPUs:
# - llama-3-2-1b-instruct (small, fast)
# - llama-3-2-3b-instruct (medium)
# - llama-3-1-8b-instruct (large)
# - llama-3-3-70b-instruct (requires 4+ GPUs)
```

---

## Testing & Validation

### Quick Test (5 minutes)

```bash
# 1. Check pod status
kubectl get pods -n my-namespace

# Expected: STATUS = Running
```

```bash
# 2. Verify GPU access
POD=$(kubectl get pod -n my-namespace -l app=llm-service -o name | head -1)
kubectl exec -n my-namespace $POD -- rocm-smi

# Expected output:
# ======================= ROCm System Management Interface =======================
# ================================= Concise Info =================================
# GPU  Temp   AvgPwr  SCLK    MCLK     Fan     Perf  PwrCap  VRAM%  GPU%
# 0    35.0c  20.0W   800Mhz  1200Mhz  0.0%    auto  300.0W    0%   0%
# ================================================================================
```

```bash
# 3. Test inference
kubectl exec -n my-namespace $POD -- \
  curl -X POST http://localhost:8080/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-3-2-3b-instruct",
    "prompt": "Write a haiku about technology",
    "max_tokens": 50,
    "temperature": 0.7
  }'

# Expected: JSON response with generated text
```

### Full Validation (30 minutes)

#### Step 1: Validate Pod Resources

```bash
kubectl describe pod -l app=llm-service -n my-namespace | grep -A 10 "Limits\|Requests"

# Expected:
# Limits:
#   amd.com/gpu: 1
#   memory: 16Gi
#   cpu: 8
# Requests:
#   amd.com/gpu: 1
#   memory: 8Gi
#   cpu: 4
```

#### Step 2: Verify ROCm Environment

```bash
# Check ROCm environment variables
kubectl exec -n my-namespace $POD -- env | grep -E "ROC|HIP|HSA"

# Expected:
# ROCR_VISIBLE_DEVICES=0
# HIP_VISIBLE_DEVICES=0
# HSA_OVERRIDE_GFX_VERSION=9.0.0
```

#### Step 3: Check Device Access

```bash
# Verify /dev/kfd access
kubectl exec -n my-namespace $POD -- ls -l /dev/kfd

# Expected: Device file exists and is accessible
# crw-rw-rw- 1 root root 511, 0 Oct 13 12:00 /dev/kfd

# Verify /dev/dri access
kubectl exec -n my-namespace $POD -- ls -l /dev/dri/

# Expected: renderD* devices
# total 0
# drwxr-xr-x 2 root root        80 Oct 13 12:00 by-path
# crw-rw---- 1 root video  226,   0 Oct 13 12:00 card0
# crw-rw-rw- 1 root render 226, 128 Oct 13 12:00 renderD128
```

#### Step 4: Monitor GPU Utilization

```bash
# Watch GPU in real-time
watch kubectl exec -n my-namespace $POD -- rocm-smi

# Run inference in another terminal
kubectl exec -n my-namespace $POD -- \
  curl -X POST http://localhost:8080/v1/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"llama-3-2-3b-instruct","prompt":"Generate a detailed story","max_tokens":500}'

# Observe GPU utilization increase during inference
```

#### Step 5: Performance Benchmark

```bash
# Create AMD-specific benchmark script
cat > benchmark-amd.sh << 'EOF'
#!/bin/bash
NAMESPACE=$1
MODEL=$2
REQUESTS=${3:-20}

POD=$(kubectl get pod -n $NAMESPACE -l app=llm-service -o name | head -1)

echo "Benchmarking AMD GPU with $MODEL ($REQUESTS requests)"
echo "==========================================================="

# Warm-up
echo "Warming up..."
for i in {1..5}; do
  kubectl exec -n $NAMESPACE $POD -- \
    curl -s -X POST http://localhost:8080/v1/completions \
    -H "Content-Type: application/json" \
    -d "{\"model\":\"$MODEL\",\"prompt\":\"Test\",\"max_tokens\":10}" > /dev/null
done

# Benchmark
echo "Running benchmark..."
total_time=0

for i in $(seq 1 $REQUESTS); do
  start=$(date +%s.%N)
  kubectl exec -n $NAMESPACE $POD -- \
    curl -s -X POST http://localhost:8080/v1/completions \
    -H "Content-Type: application/json" \
    -d "{\"model\":\"$MODEL\",\"prompt\":\"Write a story\",\"max_tokens\":100}" > /dev/null
  end=$(date +%s.%N)
  duration=$(echo "$end - $start" | bc)
  total_time=$(echo "$total_time + $duration" | bc)
  echo "Request $i: ${duration}s"
done

avg_time=$(echo "scale=3; $total_time / $REQUESTS" | bc)
throughput=$(echo "scale=2; $REQUESTS / $total_time" | bc)

echo ""
echo "Results:"
echo "--------"
echo "Average time: ${avg_time}s"
echo "Throughput: ${throughput} req/s"

# Show final GPU stats
echo ""
echo "Final GPU Stats:"
kubectl exec -n $NAMESPACE $POD -- rocm-smi
EOF

chmod +x benchmark-amd.sh

# Run benchmark
./benchmark-amd.sh my-namespace llama-3-2-3b-instruct 20
```

---

## Performance Benchmarks

### Single GPU Performance

| Model | GPU | Memory | Tokens/sec | Notes |
|-------|-----|--------|-----------|-------|
| Llama-3.2-1B | MI300X | 192GB | ~200 | Excellent performance |
| Llama-3.2-1B | MI250X | 128GB | ~170 | Very good |
| Llama-3.2-3B | MI300X | 192GB | ~150 | Great for production |
| Llama-3.2-3B | MI250X | 128GB | ~120 | Good performance |
| Llama-3.2-3B | MI210 | 64GB | ~90 | Cost-effective |
| Llama-3.1-8B | MI300X | 192GB | ~90 | Large model support |
| Llama-3.1-8B | MI250X | 128GB | ~75 | Solid performance |

### Multi-GPU Performance (4 GPUs)

| Model | GPUs | Total Memory | Tokens/sec | Tensor Parallel |
|-------|------|-------------|-----------|-----------------|
| Llama-3.3-70B | 4x MI300X | 768GB | ~85 | Yes |
| Llama-3.3-70B | 4x MI250X | 512GB | ~70 | Yes |
| Llama-3.1-8B | 4x MI250X | 512GB | ~250 | Yes (overkill) |

### ROCm vs CUDA Comparison

| Metric | AMD MI250X (ROCm) | NVIDIA A100 (CUDA) | Ratio |
|--------|------------------|-------------------|-------|
| Llama-3.2-3B | ~120 tok/s | ~180 tok/s | 0.67x |
| Llama-3.1-8B | ~75 tok/s | ~110 tok/s | 0.68x |
| Cost per Hour | ~$1.80 | ~$2.50 | 0.72x |
| Cost per Token | Lower | Higher | Better ROI |

**Note:** AMD GPUs offer excellent cost-performance ratio, especially for inference workloads.

---

## Troubleshooting

### Issue 1: Pod Stuck in Pending

**Symptoms:**
```bash
kubectl get pods -n my-namespace
# NAME                 READY   STATUS    RESTARTS   AGE
# llm-service-xxx      0/1     Pending   0          5m
```

**Diagnosis:**
```bash
kubectl describe pod -l app=llm-service -n my-namespace

# Look for:
# Events: 0/3 nodes are available: 3 Insufficient amd.com/gpu
```

**Solutions:**

1. Check AMD GPU availability:
```bash
kubectl describe nodes | grep -A 5 "amd.com/gpu"
```

2. Verify device plugin is running:
```bash
kubectl get pods -n kube-system -l name=amd-gpu-device-plugin
```

3. Check node labels:
```bash
kubectl get nodes -L accelerator
```

### Issue 2: HSA_STATUS_ERROR_OUT_OF_RESOURCES

**Symptoms:**
```bash
kubectl logs -l app=llm-service -n my-namespace
# Error: HSA_STATUS_ERROR_OUT_OF_RESOURCES
```

**Solutions:**

1. Reduce model size:
```bash
make install NAMESPACE=my-namespace LLM=llama-3-2-1b-instruct LLM_TOLERATION="amd.com/gpu"
```

2. Increase memory limits:
```yaml
resources:
  limits:
    memory: 32Gi  # Increase from 16Gi
```

3. Set HSA environment variables:
```yaml
env:
  - name: HSA_XNACK
    value: "1"
  - name: HSA_FORCE_FINE_GRAIN_PCIE
    value: "1"
```

### Issue 3: GFX Version Mismatch

**Symptoms:**
```bash
kubectl logs -l app=llm-service -n my-namespace
# Error: Unsupported gfx version
```

**Solutions:**

1. Check your GPU's GFX version:
```bash
rocminfo | grep gfx

# Output example: gfx90a (MI200 series)
```

2. Set correct HSA_OVERRIDE_GFX_VERSION:
```yaml
env:
  - name: HSA_OVERRIDE_GFX_VERSION
    value: "9.0.0"  # For gfx90a (MI200)
    # value: "9.4.0"  # For gfx940/942 (MI300)
    # value: "9.0.8"  # For gfx908 (MI100)
```

### Issue 4: /dev/kfd Permission Denied

**Symptoms:**
```bash
kubectl logs -l app=llm-service -n my-namespace
# Error: Permission denied: /dev/kfd
```

**Solutions:**

1. Check device permissions on node:
```bash
# On GPU node
ls -l /dev/kfd
# Should be: crw-rw-rw- (666 permissions)

# Fix if needed
sudo chmod 666 /dev/kfd
```

2. Add security context to pod:
```yaml
securityContext:
  privileged: true
  capabilities:
    add:
      - SYS_ADMIN
```

3. Verify volume mounts in pod:
```bash
kubectl exec -n my-namespace $POD -- ls -l /dev/kfd
```

### Issue 5: ROCm Library Not Found

**Symptoms:**
```bash
kubectl logs -l app=llm-service -n my-namespace
# Error: libamdhip64.so.5: cannot open shared object file
```

**Solutions:**

1. Use ROCm-compatible container image:
```yaml
image: rocm/pytorch:latest
# or
image: rocm/tensorflow:latest
```

2. Verify ROCm libraries in container:
```bash
kubectl exec -n my-namespace $POD -- ls -l /opt/rocm/lib/
```

3. Set LD_LIBRARY_PATH:
```yaml
env:
  - name: LD_LIBRARY_PATH
    value: "/opt/rocm/lib:$LD_LIBRARY_PATH"
```

### Issue 6: Poor Performance

**Symptoms:**
- Slow inference times
- Low GPU utilization

**Solutions:**

1. Check GPU utilization:
```bash
kubectl exec -n my-namespace $POD -- rocm-smi dmon -c 10

# GPU should be near 100% during inference
```

2. Enable optimizations:
```yaml
env:
  - name: PYTORCH_ROCM_ARCH
    value: "gfx90a"  # Match your GPU
  - name: HIP_FORCE_DEV_KERNARG
    value: "1"
```

3. Use FP16/BF16:
```yaml
env:
  - name: VLLM_USE_FP16
    value: "1"
```

---

## Advanced Configurations

### Multi-GPU with RCCL

For multi-GPU training/inference with RCCL (ROCm Collective Communications Library):

```yaml
env:
  - name: NCCL_DEBUG
    value: "INFO"
  - name: RCCL_KERNEL_COLL_TRACE_ENABLE
    value: "1"
  - name: VLLM_TENSOR_PARALLEL_SIZE
    value: "4"
```

### Memory Optimization

```yaml
env:
  - name: HIP_VISIBLE_DEVICES
    value: "0,1,2,3"
  - name: HSA_FORCE_FINE_GRAIN_PCIE
    value: "1"  # Better memory management
  - name: GPU_MAX_HEAP_SIZE
    value: "100"
  - name: GPU_MAX_ALLOC_PERCENT
    value: "100"
```

### Infinity Fabric Optimization (MI200/MI300)

```yaml
env:
  - name: AMD_SERIALIZE_KERNEL
    value: "3"
  - name: AMD_SERIALIZE_COPY
    value: "3"
  - name: HSA_ENABLE_SDMA
    value: "0"  # Disable SDMA for better latency
```

---

## Best Practices

### 1. Choose Right GPU for Workload

- **MI300X**: Large models, training
- **MI250X**: Production inference, medium/large models
- **MI210**: Cost-effective inference, small/medium models
- **MI100**: Development, testing

### 2. Optimize Container Image

```dockerfile
FROM rocm/pytorch:latest

# Install dependencies
RUN pip install vllm transformers

# Set environment
ENV ROCR_VISIBLE_DEVICES=0
ENV HIP_VISIBLE_DEVICES=0
ENV VLLM_USE_ROCM=1
```

### 3. Resource Allocation

```yaml
# MI300X (192GB memory)
resources:
  limits:
    amd.com/gpu: 1
    memory: 32Gi  # Large headroom

# MI210 (64GB memory)
resources:
  limits:
    amd.com/gpu: 1
    memory: 16Gi  # Moderate headroom
```

### 4. Monitor GPU Health

```bash
# Deploy monitoring
cat > amd-gpu-monitor.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: amd-gpu-monitor
spec:
  containers:
  - name: monitor
    image: rocm/pytorch:latest
    command: ["watch", "-n", "1", "rocm-smi"]
    resources:
      limits:
        amd.com/gpu: 1
EOF

kubectl apply -f amd-gpu-monitor.yaml
kubectl logs -f amd-gpu-monitor
```

---

## Quick Reference

### Commands Cheat Sheet

```bash
# Deploy
make install NAMESPACE=<ns> LLM=<model> LLM_TOLERATION="amd.com/gpu"

# Check status
kubectl get pods -n <ns>
kubectl describe pod -l app=llm-service -n <ns>

# Verify GPU (AMD-specific)
kubectl exec -n <ns> <pod> -- rocm-smi
kubectl exec -n <ns> <pod> -- rocminfo | grep gfx

# Test inference
kubectl exec -n <ns> <pod> -- curl -X POST http://localhost:8080/v1/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"<model>","prompt":"Test","max_tokens":10}'

# Check logs
kubectl logs -f -l app=llm-service -n <ns>

# Debug device access
kubectl exec -n <ns> <pod> -- ls -l /dev/kfd
kubectl exec -n <ns> <pod> -- ls -l /dev/dri/

# Delete
make uninstall NAMESPACE=<ns>
```

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `ROCR_VISIBLE_DEVICES` | `0` | Which GPUs are visible to ROCm |
| `HIP_VISIBLE_DEVICES` | `0` | Which GPUs are visible to HIP |
| `HSA_OVERRIDE_GFX_VERSION` | - | Override GPU architecture version |
| `VLLM_USE_ROCM` | `1` | Enable ROCm for vLLM |
| `VLLM_TENSOR_PARALLEL_SIZE` | `1` | Number of GPUs for tensor parallelism |

### GFX Versions

| GPU | GFX Version | HSA_OVERRIDE_GFX_VERSION |
|-----|-------------|--------------------------|
| MI300X | gfx942 | 9.4.2 |
| MI300A | gfx940 | 9.4.0 |
| MI250X | gfx90a | 9.0.0 |
| MI250 | gfx90a | 9.0.0 |
| MI210 | gfx90a | 9.0.0 |
| MI100 | gfx908 | 9.0.8 |

---

## Next Steps

1. **Production Deployment**: Add monitoring, alerting, load balancing
2. **Performance Tuning**: Optimize for your specific MI-series GPU
3. **Cost Optimization**: Use MI210 for inference, MI300X for training
4. **Multi-Model**: Deploy multiple models with different GPU allocations

## Resources

- [ROCm Documentation](https://rocm.docs.amd.com/)
- [AMD GPU Device Plugin](https://github.com/RadeonOpenCompute/k8s-device-plugin)
- [vLLM on ROCm](https://docs.vllm.ai/en/latest/getting_started/amd-installation.html)
- [AMD Instinct Documentation](https://www.amd.com/en/products/accelerators/instinct.html)

---

**Questions?** Open an issue: https://github.com/rh-ai-quickstart/openshift-ai-observability-summarizer/issues

**License:** MIT

