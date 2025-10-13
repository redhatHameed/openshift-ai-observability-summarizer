# NVIDIA GPU Complete Guide

> **Complete end-to-end guide for deploying AI models on NVIDIA GPUs in OpenShift**

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

This guide covers everything you need to deploy AI models on NVIDIA GPUs using the OpenShift AI Observability Summarizer and ai-architecture-charts.

**What you'll learn:**
- How to configure NVIDIA GPUs for AI workloads
- Deploy models using make commands
- Test and validate deployments
- Optimize for performance
- Troubleshoot common issues

**Time to complete:** 30-60 minutes

---

## Supported Hardware

### NVIDIA GPU Models

| Model | Memory | Architecture | Best For |
|-------|--------|--------------|----------|
| **H100** | 80GB HBM3 | Hopper | Large models, training |
| **A100** | 40GB/80GB HBM2e | Ampere | Training & inference |
| **A10** | 24GB GDDR6 | Ampere | Inference workloads |
| **V100** | 16GB/32GB HBM2 | Volta | General purpose |
| **T4** | 16GB GDDR6 | Turing | Cost-effective inference |

### Resource Information

- **Kubernetes Resource**: `nvidia.com/gpu`
- **Node Label**: `accelerator=nvidia`
- **Device Plugin**: NVIDIA GPU Operator or k8s-device-plugin
- **Drivers**: NVIDIA GPU drivers + CUDA
- **Verification Command**: `nvidia-smi`

---

## Prerequisites

### 1. Cluster Requirements

```bash
# Check if NVIDIA GPU Operator is installed
kubectl get pods -n gpu-operator-resources

# Or check for NVIDIA device plugin
kubectl get pods -n kube-system -l name=nvidia-device-plugin-daemonset

# Verify GPU resources are available
kubectl get nodes -o json | jq '.items[].status.capacity | select(.["nvidia.com/gpu"] != null)'
```

**Expected output:**
```json
{
  "nvidia.com/gpu": "8"
}
```

### 2. Node Setup

```bash
# Label GPU nodes (if not already labeled)
kubectl label node <node-name> accelerator=nvidia

# Verify labels
kubectl get nodes -L accelerator

# Check node has GPUs
kubectl describe node <node-name> | grep nvidia.com/gpu
```

### 3. Driver Verification

```bash
# SSH to GPU node or run in a test pod
kubectl run nvidia-test --image=nvidia/cuda:12.2.0-base-ubuntu22.04 --rm -it -- nvidia-smi

# Expected output: GPU list with memory, utilization, etc.
```

### 4. Required Tools

- `kubectl` CLI with cluster access
- `helm` v3.x
- `make` (for using Makefile)
- OpenShift cluster with GPU nodes
- Hugging Face token (for downloading models)

---

## Configuration

### Device Configuration Structure

```yaml
deviceConfig:
  nvidia-gpu:
    enabled: false
    resources:
      limits:
        nvidia.com/gpu: 1
        memory: 16Gi
        cpu: 8
      requests:
        nvidia.com/gpu: 1
        memory: 8Gi
        cpu: 4
    
    nodeSelector:
      accelerator: nvidia
    
    tolerations:
      - key: "nvidia.com/gpu"
        operator: "Exists"
        effect: "NoSchedule"
    
    env:
      - name: NVIDIA_VISIBLE_DEVICES
        value: "all"
      - name: NVIDIA_DRIVER_CAPABILITIES
        value: "compute,utility"
      - name: CUDA_VISIBLE_DEVICES
        value: "0"
      - name: VLLM_USE_CUDA
        value: "1"
```

### Multi-GPU Configuration (8 GPUs)

```yaml
deviceConfig:
  nvidia-gpu-multi:
    enabled: false
    resources:
      limits:
        nvidia.com/gpu: 8
        memory: 128Gi
        cpu: 64
      requests:
        nvidia.com/gpu: 8
        memory: 64Gi
        cpu: 32
    
    env:
      - name: NVIDIA_VISIBLE_DEVICES
        value: "all"
      - name: CUDA_VISIBLE_DEVICES
        value: "0,1,2,3,4,5,6,7"
      - name: VLLM_TENSOR_PARALLEL_SIZE
        value: "8"
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
  LLM_TOLERATION="nvidia.com/gpu"
```

#### With Safety Models

```bash
# Deploy with Llama Guard safety model
make install \
  NAMESPACE=my-namespace \
  LLM=llama-3-2-3b-instruct \
  LLM_TOLERATION="nvidia.com/gpu" \
  SAFETY=llama-guard-3-8b \
  SAFETY_TOLERATION="nvidia.com/gpu"
```

#### With Alerting

```bash
# Deploy with alerting enabled
make install \
  NAMESPACE=my-namespace \
  LLM=llama-3-2-3b-instruct \
  LLM_TOLERATION="nvidia.com/gpu" \
  ALERTS=TRUE
```

#### Large Model (70B)

```bash
# Deploy large model on 8 GPUs with tensor parallelism
make install \
  NAMESPACE=my-namespace \
  LLM=llama-3-3-70b-instruct \
  LLM_TOLERATION="nvidia.com/gpu"
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
  --set-json 'global.models.llama-3-2-3b-instruct.tolerations=[{"key":"nvidia.com/gpu","effect":"NoSchedule","operator":"Exists"}]' \
  --set llm-service.secret.hf_token="$HF_TOKEN"
```

### Available Models

```bash
# List all available models
make list-models

# Output:
# model: llama-3-1-8b-instruct
# model: llama-3-2-1b-instruct
# model: llama-3-2-1b-instruct-quantized
# model: llama-3-2-3b-instruct
# model: llama-3-3-70b-instruct
# model: llama-guard-3-1b
# model: llama-guard-3-8b
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
kubectl exec -n my-namespace $POD -- nvidia-smi

# Expected output:
# +-----------------------------------------------------------------------------+
# | NVIDIA-SMI 525.xx.xx    Driver Version: 525.xx.xx    CUDA Version: 12.0    |
# |-------------------------------+----------------------+----------------------+
# | GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
# | Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
# ...
```

```bash
# 3. Test inference
kubectl exec -n my-namespace $POD -- \
  curl -X POST http://localhost:8080/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-3-2-3b-instruct",
    "prompt": "Write a haiku about AI",
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
#   nvidia.com/gpu: 1
#   memory: 16Gi
#   cpu: 8
# Requests:
#   nvidia.com/gpu: 1
#   memory: 8Gi
#   cpu: 4
```

#### Step 2: Monitor GPU Utilization

```bash
# Watch GPU in real-time
watch kubectl exec -n my-namespace $POD -- nvidia-smi

# Run inference in another terminal
kubectl exec -n my-namespace $POD -- \
  curl -X POST http://localhost:8080/v1/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"llama-3-2-3b-instruct","prompt":"Generate a long story about space exploration","max_tokens":500}'

# Observe GPU utilization increase during inference
```

#### Step 3: Check Logs

```bash
# View application logs
kubectl logs -f -l app=llm-service -n my-namespace

# Look for:
# ✓ Model loaded successfully
# ✓ GPU detected
# ✓ No CUDA errors
# ✓ Inference requests completing
```

#### Step 4: Performance Test

```bash
# Create benchmark script
cat > benchmark-nvidia.sh << 'EOF'
#!/bin/bash
NAMESPACE=$1
MODEL=$2
REQUESTS=${3:-20}

POD=$(kubectl get pod -n $NAMESPACE -l app=llm-service -o name | head -1)

echo "Running $REQUESTS inference requests..."
total_time=0

for i in $(seq 1 $REQUESTS); do
  start=$(date +%s.%N)
  kubectl exec -n $NAMESPACE $POD -- \
    curl -s -X POST http://localhost:8080/v1/completions \
    -H "Content-Type: application/json" \
    -d "{\"model\":\"$MODEL\",\"prompt\":\"Test\",\"max_tokens\":100}" > /dev/null
  end=$(date +%s.%N)
  duration=$(echo "$end - $start" | bc)
  total_time=$(echo "$total_time + $duration" | bc)
  echo "Request $i: ${duration}s"
done

avg_time=$(echo "scale=3; $total_time / $REQUESTS" | bc)
throughput=$(echo "scale=2; $REQUESTS / $total_time" | bc)

echo ""
echo "Average time: ${avg_time}s"
echo "Throughput: ${throughput} req/s"
EOF

chmod +x benchmark-nvidia.sh

# Run benchmark
./benchmark-nvidia.sh my-namespace llama-3-2-3b-instruct 20
```

---

## Performance Benchmarks

### Single GPU Performance

| Model | GPU | Batch Size | Tokens/sec | Latency (ms) |
|-------|-----|-----------|-----------|--------------|
| Llama-3.2-1B | H100 | 1 | ~250 | ~4 |
| Llama-3.2-1B | A100 | 1 | ~220 | ~4.5 |
| Llama-3.2-3B | H100 | 1 | ~200 | ~5 |
| Llama-3.2-3B | A100 | 1 | ~180 | ~5.5 |
| Llama-3.1-8B | H100 | 1 | ~130 | ~7.5 |
| Llama-3.1-8B | A100 | 1 | ~110 | ~9 |
| Llama-3.1-8B | V100 | 1 | ~80 | ~12.5 |
| Llama-3.1-8B | T4 | 1 | ~60 | ~16.5 |

### Multi-GPU Performance (8 GPUs)

| Model | GPUs | Tokens/sec | Memory/GPU |
|-------|------|-----------|------------|
| Llama-3.3-70B | 8x H100 | ~100 | ~12GB |
| Llama-3.3-70B | 8x A100-80GB | ~90 | ~12GB |
| Llama-3.3-70B | 8x A100-40GB | ~85 | ~12GB |

### Optimization Tips

1. **Quantization**: Use quantized models for 2-4x speedup
2. **Batch Size**: Increase for higher throughput (lower latency)
3. **Tensor Parallelism**: Split large models across GPUs
4. **FP16/BF16**: Use mixed precision for faster inference

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
# Events: 0/3 nodes are available: 3 Insufficient nvidia.com/gpu
```

**Solutions:**

1. Check GPU availability:
```bash
kubectl describe nodes | grep -A 5 "nvidia.com/gpu"
```

2. Check if another pod is using GPUs:
```bash
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.containers[].resources.limits."nvidia.com/gpu" != null) | .metadata.name'
```

3. Increase GPU nodes or reduce GPU request

### Issue 2: CUDA Out of Memory

**Symptoms:**
```bash
kubectl logs -l app=llm-service -n my-namespace
# Error: CUDA out of memory
```

**Solutions:**

1. Use a smaller model:
```bash
make install NAMESPACE=my-namespace LLM=llama-3-2-1b-instruct LLM_TOLERATION="nvidia.com/gpu"
```

2. Use quantized model:
```bash
make install NAMESPACE=my-namespace LLM=llama-3-2-1b-instruct-quantized LLM_TOLERATION="nvidia.com/gpu"
```

3. Reduce max_model_len:
```bash
# Add to env variables
- name: MAX_MODEL_LEN
  value: "2048"
```

### Issue 3: CUDA Driver/Runtime Mismatch

**Symptoms:**
```bash
kubectl logs -l app=llm-service -n my-namespace
# Error: CUDA driver version is insufficient for CUDA runtime version
```

**Solutions:**

1. Check NVIDIA driver version:
```bash
kubectl exec -n my-namespace $POD -- nvidia-smi

# Driver Version should be >= 525.xx for CUDA 12.x
```

2. Update container image to match driver version:
```yaml
# Use compatible CUDA version
image: nvidia/cuda:11.8.0-base-ubuntu22.04  # For older drivers
# or
image: nvidia/cuda:12.2.0-base-ubuntu22.04  # For newer drivers
```

### Issue 4: GPU Not Detected

**Symptoms:**
```bash
kubectl logs -l app=llm-service -n my-namespace
# Warning: No CUDA devices found
```

**Solutions:**

1. Verify GPU Operator is running:
```bash
kubectl get pods -n gpu-operator-resources
```

2. Check device plugin logs:
```bash
kubectl logs -n gpu-operator-resources -l app=nvidia-device-plugin-daemonset
```

3. Restart the pod:
```bash
kubectl delete pod -l app=llm-service -n my-namespace
```

### Issue 5: Poor Performance

**Symptoms:**
- Slow inference times
- Low GPU utilization

**Solutions:**

1. Check GPU utilization:
```bash
kubectl exec -n my-namespace $POD -- nvidia-smi dmon -s u

# GPU should be near 100% during inference
```

2. Enable FP16:
```bash
# Add to env variables
- name: VLLM_USE_FP16
  value: "1"
```

3. Increase batch size:
```bash
# Add to env variables
- name: VLLM_MAX_NUM_SEQS
  value: "256"
```

4. Use tensor parallelism (multi-GPU):
```bash
make install \
  NAMESPACE=my-namespace \
  LLM=llama-3-3-70b-instruct \
  LLM_TOLERATION="nvidia.com/gpu"
# This will automatically configure tensor parallelism
```

---

## Advanced Configurations

### Multi-GPU Tensor Parallelism

```yaml
deviceConfig:
  nvidia-gpu-multi:
    resources:
      limits:
        nvidia.com/gpu: 4
        memory: 64Gi
        cpu: 32
    env:
      - name: VLLM_TENSOR_PARALLEL_SIZE
        value: "4"
      - name: CUDA_VISIBLE_DEVICES
        value: "0,1,2,3"
```

**Deploy:**
```bash
helm upgrade --install llm-service ai-architecture-charts/llm-service \
  --namespace my-namespace \
  --set global.models.llama-3-3-70b-instruct.enabled=true \
  --set global.models.llama-3-3-70b-instruct.resources.limits."nvidia\.com/gpu"=4 \
  --set-json 'global.models.llama-3-3-70b-instruct.env=[{"name":"VLLM_TENSOR_PARALLEL_SIZE","value":"4"}]'
```

### MIG (Multi-Instance GPU) Support

For A100/H100 GPUs with MIG enabled:

```yaml
resources:
  limits:
    nvidia.com/mig-1g.10gb: 1  # 1/7 of A100
```

```bash
make install \
  NAMESPACE=my-namespace \
  LLM=llama-3-2-1b-instruct \
  LLM_TOLERATION="nvidia.com/mig-1g.10gb"
```

### GPU Time-Slicing

Share one GPU among multiple pods:

```yaml
# Configure in NVIDIA GPU Operator
# Each pod gets a time slice of the GPU
resources:
  limits:
    nvidia.com/gpu: 1  # Shared GPU
```

### Custom CUDA Configuration

```yaml
env:
  - name: CUDA_VISIBLE_DEVICES
    value: "0,1"  # Use specific GPUs
  - name: CUDA_LAUNCH_BLOCKING
    value: "1"  # For debugging
  - name: NVIDIA_TF32_OVERRIDE
    value: "0"  # Disable TF32 for precision
```

---

## Best Practices

### 1. Resource Management

```yaml
# Always set both requests and limits
resources:
  requests:
    nvidia.com/gpu: 1
    memory: 8Gi
    cpu: 4
  limits:
    nvidia.com/gpu: 1
    memory: 16Gi
    cpu: 8
```

### 2. Node Affinity

```yaml
# Prefer specific GPU types
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
            - key: nvidia.com/gpu.product
              operator: In
              values:
                - NVIDIA-A100-SXM4-80GB
```

### 3. Pod Disruption Budget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: llm-service-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: llm-service
```

### 4. Monitoring

```bash
# Deploy Prometheus GPU exporter
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/dcgm-exporter/main/deployment/gpu-metrics-exporter.yaml

# Query GPU metrics
curl http://prometheus:9090/api/v1/query?query=DCGM_FI_DEV_GPU_UTIL
```

### 5. Auto-Scaling

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: llm-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: llm-service
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: nvidia.com/gpu
        target:
          type: Utilization
          averageUtilization: 80
```

---

## Quick Reference

### Commands Cheat Sheet

```bash
# Deploy
make install NAMESPACE=<ns> LLM=<model> LLM_TOLERATION="nvidia.com/gpu"

# Check status
kubectl get pods -n <ns>
kubectl describe pod -l app=llm-service -n <ns>

# Verify GPU
kubectl exec -n <ns> <pod> -- nvidia-smi

# Test inference
kubectl exec -n <ns> <pod> -- curl -X POST http://localhost:8080/v1/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"<model>","prompt":"Test","max_tokens":10}'

# Check logs
kubectl logs -f -l app=llm-service -n <ns>

# Delete
make uninstall NAMESPACE=<ns>
```

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `NVIDIA_VISIBLE_DEVICES` | `all` | Which GPUs are visible |
| `CUDA_VISIBLE_DEVICES` | `0` | CUDA device index |
| `VLLM_TENSOR_PARALLEL_SIZE` | `1` | Number of GPUs for tensor parallelism |
| `VLLM_USE_FP16` | `0` | Enable FP16 precision |
| `MAX_MODEL_LEN` | `4096` | Maximum sequence length |

---

## Next Steps

1. **Production Deployment**: Add monitoring, alerting, and auto-scaling
2. **Performance Tuning**: Optimize batch size, quantization, and parallelism
3. **Cost Optimization**: Use smaller GPUs (T4, A10) for inference workloads
4. **Multi-Model**: Deploy multiple models with different GPU allocations

## Resources

- [NVIDIA GPU Operator Documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [CUDA Documentation](https://docs.nvidia.com/cuda/)
- [vLLM Documentation](https://docs.vllm.ai/)
- [OpenShift AI Documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed)

---

**Questions?** Open an issue: https://github.com/rh-ai-quickstart/openshift-ai-observability-summarizer/issues

**License:** MIT

