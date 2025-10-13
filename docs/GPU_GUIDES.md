# GPU & Accelerator Deployment Guides

> **Choose your hardware vendor and follow the complete end-to-end guide**

## 🚀 Quick Navigation

### I have NVIDIA GPUs
**→ [NVIDIA GPU Complete Guide](NVIDIA_GPU_GUIDE.md)**

Covers: H100, A100, V100, T4, A10  
Command: `make install NAMESPACE=<ns> LLM=<model> LLM_TOLERATION="nvidia.com/gpu"`

---

### I have AMD GPUs
**→ [AMD GPU Complete Guide](AMD_GPU_GUIDE.md)**

Covers: MI300X, MI250X, MI210, MI100  
Command: `make install NAMESPACE=<ns> LLM=<model> LLM_TOLERATION="amd.com/gpu"`

---

### I have Intel GPUs or Habana Gaudi
**→ [Intel Accelerators Complete Guide](INTEL_ACCELERATORS_GUIDE.md)**

Covers: Intel Data Center GPU Max/Flex, Habana Gaudi 1/2/3  
Commands:
- Intel GPU: `make install NAMESPACE=<ns> LLM=<model> LLM_TOLERATION="gpu.intel.com/i915"`
- Intel HPU: `make install NAMESPACE=<ns> LLM=<model> LLM_TOLERATION="habana.ai/gaudi"`

---

## 📚 What Each Guide Includes

Each vendor-specific guide provides a complete end-to-end workflow:

✅ **Hardware Overview** - Supported models and specifications  
✅ **Prerequisites** - Driver installation and device plugin setup  
✅ **Configuration** - Single and multi-accelerator configurations  
✅ **Deployment** - Make commands and Helm examples  
✅ **Testing & Validation** - Quick and full validation procedures  
✅ **Performance Benchmarks** - Real-world performance data  
✅ **Troubleshooting** - Vendor-specific issues and solutions  
✅ **Advanced Configurations** - Multi-GPU, optimization tips  
✅ **Quick Reference** - Command cheat sheets

---

## 🎯 Quick Start (30 seconds)

```bash
# 1. Choose your hardware and label your node
kubectl label node <node-name> accelerator=<nvidia|amd|intel-gpu|habana-gaudi>

# 2. Deploy a model
make install \
  NAMESPACE=my-namespace \
  LLM=llama-3-2-3b-instruct \
  LLM_TOLERATION="<gpu-resource-name>"

# 3. Verify deployment
kubectl get pods -n my-namespace
kubectl exec -n my-namespace <pod> -- <verification-command>
```

| Hardware | Toleration | Verify Command |
|----------|-----------|----------------|
| NVIDIA | `nvidia.com/gpu` | `nvidia-smi` |
| AMD | `amd.com/gpu` | `rocm-smi` |
| Intel GPU | `gpu.intel.com/i915` | `clinfo` |
| Intel HPU | `habana.ai/gaudi` | `hl-smi` |

---

## 📖 Additional Resources

- **[device-configs-values-template.yaml](device-configs-values-template.yaml)** - Helm values template for ai-architecture-charts integration
- **[Main README](../README.md)** - Project overview and getting started
- **[Helm Charts Documentation](HELM_CHARTS.md)** - Helm chart deployment details

---

## 🆘 Need Help?

1. **Check the vendor-specific guide** for your hardware
2. **Review the troubleshooting section** in your guide
3. **Open an issue**: https://github.com/rh-ai-quickstart/openshift-ai-observability-summarizer/issues

---

**Time to complete:** 30-90 minutes per guide  
**Difficulty:** Intermediate  
**Prerequisites:** OpenShift cluster with GPU nodes, kubectl, helm

---

**License:** MIT

