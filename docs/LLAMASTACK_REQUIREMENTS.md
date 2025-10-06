# LLM Provider Integration Guide

**Document Purpose:** Explain how our observability system integrates with multiple LLM providers (LlamaStack, OpenAI, Google Gemini) for AI-powered metric analysis.

**Last Updated:** 2025-10-06

---

## How LLM Integration Works

### Architecture Overview

```
User Request → MCP/UI/Alerting → LLM Client (src/core/llm_client.py)
                                       ↓
                    ┌──────────────────┼──────────────────┐
                    ↓                  ↓                  ↓
            LlamaStack(Local)   OpenAI API(Ext)   Google Gemini(Ext)
                    ↓                  ↓                  ↓
                    └──────────────────┴──────────────────┘
                                       ↓
                    "Model health is GOOD. 99% success rate..."
                                       ↓
               Display / Store / Send Alert
```

### Integration Flow

#### **Step 1: Gather Metrics**
```python
# Query Prometheus/Thanos
metrics = thanos_service.query_range(...)

# Calculate statistics
stats = {
    "total_requests": 1523,
    "success_rate": 99.0,
    "p95_latency": 0.45,
    "avg_gpu_utilization": 87.3
}
```

#### **Step 2: Build Analysis Prompt**
```python
prompt = f"""
You are a senior SRE analyzing vLLM model performance.
Metrics: Total Requests: {stats['total_requests']}, Success Rate: {stats['success_rate']}%
Analyze: health status, performance, recommendations.
"""
```

#### **Step 3: Call LLM Provider**
```python
from core.llm_client import summarize_with_llm

summary = summarize_with_llm(
    prompt=prompt,
    summarize_model_id="local/llama-3.2-3b",  # or openai/gpt-4o-mini, google/gemini-2.5-flash
    response_type=ResponseType.VLLM,
    api_key=None  # Required for external providers
)
```

**Provider Routing (src/core/llm_client.py):**
```python
def summarize_with_llm(prompt, summarize_model_id, response_type, api_key):
    model_info = MODEL_CONFIG.get(summarize_model_id)
    is_external = model_info.get("external", False)
    
    if is_external:
        provider = model_info.get("provider")
        if provider == "google":
            # Google Gemini: contents array, x-goog-api-key header
            response = requests.post(api_url, 
                json={"contents": [{"parts": [{"text": prompt}]}]},
                headers={"x-goog-api-key": api_key})
        else:
            # OpenAI: messages array, Bearer token
            response = requests.post(api_url,
                json={"model": model_name, "messages": [{"role": "user", "content": prompt}]},
                headers={"Authorization": f"Bearer {api_key}"})
    else:
        # Local LlamaStack: prompt field, no auth required
        response = requests.post(f"{LLAMA_STACK_URL}/completions",
            json={"model": model_info["serviceName"], "prompt": prompt})
    
    return extract_text(response.json(), provider)
```

#### **Step 4: Display Results**

**LLM Response Example:**
```
✅ HEALTH STATUS: GOOD

The llama-3.2-3b model demonstrates excellent operational health. 
With 1,523 requests and a 99% success rate, the system is handling 
load effectively. P95 latency of 450ms meets SLO targets.

Recommendations:
1. Continue monitoring - no immediate action needed
2. Review failed requests (15) for patterns
```

**Display Locations:**

**UI (Streamlit):**
```python
st.markdown("### 🧠 AI Analysis")
st.markdown(summary)

with st.expander("📊 Raw Metrics"):
    st.json(stats)
```

**MCP Tool:**
```python
@mcp.tool()
def analyze_vllm(model_name: str, summarize_model_id: str):
    # Returns data + analysis to Claude/Cursor
    return {"llm_summary": summary, "statistics": stats}
```

**Alerts:**
```python
def generate_description(alert_data: dict) -> str:
    prompt = f"Alert: {alert_data['name']}, Severity: {alert_data['severity']}"
    return summarize_with_llm(prompt, "llama-3.2-3b", ResponseType.OPENSHIFT)
```

---

## Use Cases

| Component | Purpose | File Location |
|-----------|---------|---------------|
| **MCP Server** | Summarize vLLM/OpenShift metrics | `src/mcp_server/tools/observability_*_tools.py` |
| **UI** | Generate analysis reports | `src/ui/ui.py` |
| **Alerting** | Generate alert descriptions | `src/alerting/alert_receiver.py` |

---

## Why LlamaStack?

**LlamaStack** is one of three LLM provider options in our system.

---

### ⚠️ Important: When is LlamaStack Required?

| Scenario | LlamaStack Required? | Explanation |
|----------|---------------------|-------------|
| **Using Local Models** (Llama 3.2) | ✅ **YES - Required** | LlamaStack runs the model in your cluster |
| **Using OpenAI** (GPT-4, etc.) | ❌ **NO - Not needed** | Calls OpenAI API directly via internet |
| **Using Google Gemini** | ❌ **NO - Not needed** | Calls Google API directly via internet |

**Key Point:** LlamaStack is **only needed for local models**. If you only use external APIs (OpenAI/Gemini), you can skip LlamaStack deployment entirely and save infrastructure resources (4 CPU, 16GB RAM).

---

**Code Behavior:**
```python
# Local model (needs LlamaStack)
if is_external == False:
    response = requests.post(
        f"{LLAMA_STACK_URL}/completions",  # ← Requires LlamaStack service
        json={"model": "llama-3.2-3b", "prompt": prompt}
    )

# External model (no LlamaStack needed)
if is_external == True:
    response = requests.post(
        "https://api.openai.com/v1/chat/completions",  # ← Direct to OpenAI
        json={"model": "gpt-4o-mini", "messages": [...]},
        headers={"Authorization": f"Bearer {api_key}"}
    )
```

---

**Why We Chose LlamaStack (Default for Local Models):**
1. **Data Privacy:** Metrics/alerts stay in cluster
2. **Cost Predictability:** Fixed infrastructure cost
3. **No External Dependencies:** Works without internet
4. **OpenShift Native:** Standard Kubernetes workload
5. **Open Source:** Full control over models

**When to Use Alternatives:**
- **OpenAI/Gemini:** Better quality, don't need data privacy guarantees, no LlamaStack deployment needed
- **Templates:** No AI needed, just basic status reporting

---

## Multi-Provider Support

### Provider Comparison

| Provider | Auth | Privacy | Cost | Speed | Quality |
|----------|------|---------|------|-------|---------|
| **LlamaStack** | Optional | ✅ In-cluster | Fixed ($500/mo) | 5-30s | Good |
| **OpenAI** | Bearer token | ⚠️ External | Per-request ($0.002) | 2-10s | Excellent |
| **Google Gemini** | x-goog-api-key | ⚠️ External | Per-request ($0.0003) | 2-10s | Excellent |

### Provider Configuration

**File:** `deploy/helm/model-config.json`

```json
{
  "local/llama-3.2-3b": {
    "external": false,
    "requiresApiKey": false,
    "serviceName": "llama-3.2-3b-instruct",
    "provider": "local"
  },
  "openai/gpt-4o-mini": {
    "external": true,
    "requiresApiKey": true,
    "provider": "openai",
    "apiUrl": "https://api.openai.com/v1/chat/completions",
    "modelName": "gpt-4o-mini",
    "cost": {"prompt_rate": 0.00000015, "output_rate": 0.0000006}
  },
  "google/gemini-2.5-flash": {
    "external": true,
    "requiresApiKey": true,
    "provider": "google",
    "apiUrl": "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent",
    "modelName": "gemini-2.5-flash",
    "cost": {"prompt_rate": 0.0000003, "output_rate": 0.000001}
  }
}
```

### API Format Differences

#### Local LlamaStack
```
Endpoint: /completions
Request:  {"model": "llama-3.2-3b-instruct", "prompt": "..."}
Response: {"choices": [{"text": "..."}]}
Auth:     None (optional)
```

#### OpenAI
```
Endpoint: /chat/completions
Request:  {"model": "gpt-4o-mini", "messages": [{"role": "user", "content": "..."}]}
Response: {"choices": [{"message": {"content": "..."}}]}
Auth:     Authorization: Bearer <api_key>
```

#### Google Gemini
```
Endpoint: /v1beta/models/gemini-2.5-flash:generateContent
Request:  {"contents": [{"parts": [{"text": "..."}]}]}
Response: {"candidates": [{"content": {"parts": [{"text": "..."}]}}]}
Auth:     x-goog-api-key: <api_key>
```

### Usage Example

```python
# User selects provider
selected_model = st.selectbox("LLM Model", [
    "local/llama-3.2-3b",
    "openai/gpt-4o-mini",
    "google/gemini-2.5-flash"
])

# Get API key if needed
model_info = MODEL_CONFIG.get(selected_model, {})
api_key = st.text_input("API Key", type="password") if model_info.get("requiresApiKey") else None

# Call LLM (same function works for all providers!)
summary = summarize_with_llm(
    prompt=prompt,
    summarize_model_id=selected_model,
    response_type=ResponseType.VLLM,
    api_key=api_key
)
```

---

## Future Architecture

> **Note:** See [PROPOSAL_MCP_LLM_SEPARATION.md](./PROPOSAL_MCP_LLM_SEPARATION.md) for details

**Current:** `UI → MCP Server → LlamaStack → Response`

**Proposed:** `UI → LlamaStack → Response` + `MCP Server (data only, no LLM)`

**Benefits:**
- MCP Server simpler (no LLM dependency)
- UI has full control over LLM experience
- Claude/Cursor use their own LLMs (better quality)
- Easier testing and deployment

---

## Alternatives

### External LLM API (OpenAI, Gemini)

**Pros:** Better quality, faster, no infrastructure overhead  
**Cons:** Data leaves cluster, per-request costs, needs internet  
**Use When:** Data privacy not critical, need best AI quality

### No LLM (Templates)

**Pros:** Instant (<1s), no cost, no dependencies  
**Cons:** Static responses, limited quality  
**Use When:** Budget tight, simple status reporting only

### Larger Self-Hosted Models

**Pros:** Better quality than 3B, still self-hosted  
**Cons:** More resources (32GB RAM, GPU), slower, higher cost  
**Use When:** Need better quality but must stay in-cluster

---

## References

### Documentation
- LlamaStack: https://github.com/meta-llama/llama-stack
- OpenAI API: https://platform.openai.com/docs/api-reference
- Model Card: https://huggingface.co/meta-llama/Llama-3.2-3B

### Internal Docs
- [MCP Architecture](./MCP_ARCHITECTURE.md)
- [Proposal: LLM Separation](./PROPOSAL_MCP_LLM_SEPARATION.md)
- [Development Guide](./DEV_GUIDE.md)

### Related Code
- `src/core/llm_client.py` - LLM provider integration
- `src/mcp_server/tools/observability_vllm_tools.py` - MCP usage
- `src/alerting/alert_receiver.py` - Alerting usage
- `deploy/helm/rag/` - LlamaStack deployment

---

**Document Status:** Current  
**Next Review:** After LLM separation implementation  
**Maintainer:** Architecture Team