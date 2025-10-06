# MCP Architecture Design

## 📖 Table of Contents
1. [Overview](#overview)
2. [High-Level Architecture](#high-level-architecture)
3. [Component Architecture](#component-architecture)
4. [Data Flow Diagrams](#data-flow-diagrams)
5. [Integration Patterns](#integration-patterns)
6. [Deployment Architecture](#deployment-architecture)
7. [Communication Protocols](#communication-protocols)
8. [Security & RBAC](#security--rbac)
9. [Scalability & Performance](#scalability--performance)

---

## Overview

The **Model Context Protocol (MCP) Server** is the central intelligence hub of the OpenShift AI Observability Summarizer platform. It provides a unified API for AI-powered metrics analysis, report generation, and AI assistant integration (Claude Desktop, Cursor IDE).

### Key Capabilities
- **AI Assistant Integration**: Native MCP protocol support for Claude Desktop & Cursor IDE
- **Metrics Intelligence**: Automated analysis of vLLM, OpenShift, and GPU metrics
- **Report Generation**: Multi-format reports (HTML, PDF, Markdown)
- **Distributed Tracing**: Tempo/OpenTelemetry integration for trace analysis
- **Natural Language Interface**: Chat-based metrics exploration via LLM
- **Multi-Model Support**: Internal (LlamaStack) and external (OpenAI, Gemini) LLM backends

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      OpenShift AI Observability Stack                     │
└──────────────────────────────────────────────────────────────────────────┘

┌─────────────────────┐     ┌─────────────────────┐     ┌─────────────────┐
│   AI Assistants     │     │    Streamlit UI     │     │   Alerting      │
│                     │     │                     │     │   System        │
│  - Claude Desktop   │────▶│  - vLLM Dashboard   │     │                 │
│  - Cursor IDE       │     │  - OpenShift Dash   │     │  - Slack Notify │
│  - MCP Inspector    │     │  - Chat Interface   │     │  - CronJob      │
└─────────────────────┘     └─────────────────────┘     └─────────────────┘
         │                            │                           │
         │                            │                           │
         └────────────────────────────┼───────────────────────────┘
                                      ▼
                        ┌─────────────────────────────┐
                        │      MCP Server             │
                        │  (Port 8085)                │
                        │                             │
                        │  ┌───────────────────────┐  │
                        │  │  FastMCP Framework    │  │
                        │  │  - HTTP Transport     │  │
                        │  │  - SSE Transport      │  │
                        │  │  - STDIO Transport    │  │
                        │  └───────────────────────┘  │
                        │                             │
                        │  ┌───────────────────────┐  │
                        │  │   MCP Tools (21+)     │  │
                        │  │  - vLLM Tools (10)    │  │
                        │  │  - OpenShift (4)      │  │
                        │  │  - Prometheus (9)     │  │
                        │  │  - Tempo (3)          │  │
                        │  └───────────────────────┘  │
                        │                             │
                        │  ┌───────────────────────┐  │
                        │  │  Report Endpoints     │  │
                        │  │  - Generate Report    │  │
                        │  │  - Download Report    │  │
                        │  └───────────────────────┘  │
                        └─────────────────────────────┘
                                      │
                ┌─────────────────────┼─────────────────────┐
                ▼                     ▼                     ▼
    ┌───────────────────┐ ┌───────────────────┐ ┌──────────────────┐
    │   Prometheus/     │ │  LlamaStack/      │ │  Tempo           │
    │   Thanos          │ │  External LLMs    │ │  TempoStack      │
    │                   │ │                   │ │                  │
    │  - Metrics Query  │ │  - Summarization  │ │  - Trace Query   │
    │  - Time Series    │ │  - Analysis       │ │  - Span Details  │
    │  - PromQL API     │ │  - Chat Interface │ │  - TraceQL       │
    └───────────────────┘ └───────────────────┘ └──────────────────┘
            │                       │                      │
            ▼                       ▼                      ▼
    ┌───────────────────────────────────────────────────────────┐
    │              OpenShift Cluster Resources                   │
    │  - vLLM Models    - GPU Metrics (DCGM)    - Workloads    │
    │  - Namespaces     - Pods/Services         - Traces       │
    └───────────────────────────────────────────────────────────┘
```

---

## Component Architecture

### 1. MCP Server (`src/mcp_server/`)

The MCP Server is built on **FastMCP**, a Python framework for building Model Context Protocol servers.

#### Core Files:
- **`api.py`**: FastAPI application with MCP transport mounting and report endpoints
- **`observability_mcp.py`**: MCP server initialization and tool registration
- **`settings.py`**: Configuration management (Pydantic BaseSettings)
- **`main.py`**: HTTP/SSE server entrypoint
- **`stdio_server.py`**: STDIO server entrypoint (for Claude Desktop/Cursor IDE)
- **`cli.py`**: Command-line interface

#### Tools Structure (`tools/`):
```
tools/
├── observability_vllm_tools.py      # 10 vLLM-related tools
├── observability_openshift_tools.py # 4 OpenShift-related tools
├── prometheus_tools.py              # 9 Prometheus intelligence tools
└── tempo_query_tool.py              # 3 distributed tracing tools
```

### 2. MCP Tools Catalog

#### vLLM Tools (10)
| Tool | Purpose | Returns |
|------|---------|---------|
| `list_models()` | Discover vLLM models from Prometheus | `"namespace \| model_name"` format |
| `list_namespaces()` | List monitored K8s namespaces | Sorted namespace list |
| `get_model_config()` | Get available LLM models | Internal/external configs |
| `list_summarization_models()` | List summarization model IDs | Bullet list of model IDs |
| `get_gpu_info()` | GPU fleet information | JSON: gpus, vendors, temps |
| `get_deployment_info()` | Deployment status check | JSON: is_new, date, message |
| `get_vllm_metrics_tool()` | Fetch raw vLLM metrics | Time series data |
| `calculate_metrics()` | Compute statistics | Mean, p95, p99, etc. |
| `analyze_vllm()` | Full analysis with LLM | Summary + metrics + prompt |
| `chat_vllm()` | Conversational interface | Natural language analysis |

#### OpenShift Tools (4)
| Tool | Purpose | Categories |
|------|---------|------------|
| `analyze_openshift()` | Analyze by category/scope | Fleet, Workloads, GPU, etc. |
| `list_openshift_metric_groups()` | List cluster-wide categories | 8 categories |
| `list_openshift_namespace_metric_groups()` | List namespace categories | 3 categories |
| `chat_openshift()` | Conversational interface | Natural language analysis |

**Available Categories:**
- Fleet Overview (cluster-wide)
- Services & Networking (cluster-wide)
- Jobs & Workloads (cluster-wide)
- Storage & Config (cluster-wide)
- Workloads & Pods (namespace-scoped)
- GPU & Accelerators (cluster-wide)
- Storage & Networking (namespace-scoped)
- Application Services (namespace-scoped)

#### Prometheus Tools (9)
| Tool | Purpose | Intelligence Level |
|------|---------|-------------------|
| `search_metrics()` | Pattern-based search | Basic discovery |
| `get_metric_metadata()` | Metric type/help text | Metadata lookup |
| `get_label_values()` | Label value discovery | Label exploration |
| `execute_promql()` | Raw PromQL execution | Direct query |
| `explain_results()` | LLM result analysis | AI-powered insight |
| `suggest_queries()` | Related query ideas | AI-powered suggestions |
| `select_best_metric()` | Metric selection | AI-powered selection |
| `find_best_metric_with_metadata()` | Smart selection v1 | Metadata + AI |
| `find_best_metric_with_metadata_v2()` | Smart selection v2 | Enhanced AI logic |

#### Tempo Tools (3)
| Tool | Purpose | Returns |
|------|---------|---------|
| `query_tempo_tool()` | Search traces by service/time | Trace list with analysis |
| `get_trace_details_tool()` | Get trace spans by ID | Detailed trace data |
| `chat_tempo_tool()` | Conversational trace analysis | Natural language insights |

### 3. UI Integration (`src/ui/`)

#### MCP Client Helper (`mcp_client_helper.py`)
```python
class MCPClientHelper:
    """FastMCP HTTP client for UI integration"""
    
    def __init__(self, server_url: str = "http://localhost:8085"):
        self.server_url = server_url
        self.mcp_endpoint = f"{server_url}/mcp"
    
    async def call_tool(self, tool_name: str, params: dict) -> Any:
        """Async tool invocation via HTTP transport"""
        # Uses fastmcp.Client internally
        # Returns parsed MCP response
```

#### Streamlit UI (`ui.py`)
The UI provides three main dashboards:

**1. vLLM Metric Summarizer**
- Model selection via `list_models()`
- Time range picker (calendar + preset ranges)
- Analysis via `analyze_vllm()`
- Report generation (HTML/PDF/Markdown)
- GPU info display via `get_gpu_info()`
- Deployment status via `get_deployment_info()`

**2. OpenShift Dashboard**
- Category selection via `list_openshift_metric_groups()`
- Cluster-wide vs namespace-scoped toggle
- Analysis via `analyze_openshift()`
- Fleet overview with GPU metrics

**3. Chat Interface (Claude Desktop-style)**
- Natural language queries
- Tool use transparency (shows MCP calls)
- Multi-iteration analysis
- Modes: vLLM, OpenShift, Tempo
- Powered by `PrometheusChatBot` (Claude API)

### 4. Core Business Logic (`src/core/`)

Shared modules used by MCP tools:

| Module | Purpose | Used By |
|--------|---------|---------|
| `llm_client.py` | LLM interface (LlamaStack/OpenAI/Gemini) | analyze_*, chat_* tools |
| `llm_summary_service.py` | Summarization logic & prompt templates | analyze_* tools |
| `thanos_service.py` | Thanos/Prometheus query client | All metric tools |
| `promql_service.py` | PromQL generation from natural language | Prometheus tools |
| `metrics.py` | Metric processing & transformation | vLLM tools |
| `analysis.py` | Statistical analysis & anomaly detection | analyze_* tools |
| `reports.py` | Report generation (HTML/PDF/Markdown) | Report endpoints |
| `models.py` | Pydantic data models | All tools |
| `response_validator.py` | LLM output validation | analyze_* tools |

---

## Data Flow Diagrams

### 1. vLLM Analysis Flow

```
User Query: "Analyze model X for last 24h"
                │
                ▼
┌───────────────────────────────────┐
│  UI: vLLM Dashboard               │
│  - list_models() → Select model   │
│  - Choose time range              │
│  - Click "Analyze"                │
└───────────────┬───────────────────┘
                │
                ▼
┌───────────────────────────────────────────────┐
│  MCP Client: call_tool("analyze_vllm", {...}) │
│  POST /mcp                                    │
│  {                                            │
│    "model_name": "ns | model",                │
│    "time_range": "last 24h",                  │
│    "summarize_model_id": "llama-3.2-3b"       │
│  }                                            │
└───────────────┬───────────────────────────────┘
                │
                ▼
┌────────────────────────────────────────────────┐
│  MCP Server: analyze_vllm()                    │
│  ┌──────────────────────────────────────────┐  │
│  │ 1. thanos_service.query_range()          │  │
│  │    - vllm:request_success_total          │  │
│  │    - vllm:time_to_first_token_seconds    │  │
│  │    - vllm:time_per_output_token_seconds  │  │
│  │    - DCGM_FI_DEV_GPU_UTIL                │  │
│  └──────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────┐  │
│  │ 2. analysis.py                           │  │
│  │    - Calculate mean, p95, p99            │  │
│  │    - Detect anomalies                    │  │
│  │    - Generate insights                   │  │
│  └──────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────┐  │
│  │ 3. llm_summary_service.build_prompt()    │  │
│  │    - Format metrics as context           │  │
│  │    - Add operational guidelines          │  │
│  └──────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────┐  │
│  │ 4. llm_client.generate_summary()         │  │
│  │    - POST to LlamaStack/OpenAI           │  │
│  │    - Parse & validate response           │  │
│  └──────────────────────────────────────────┘  │
│  Return: {                                     │
│    "health_prompt": "...",                     │
│    "llm_summary": "...",                       │
│    "metrics": {...}                            │
│  }                                             │
└────────────────┬───────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────┐
│  UI: Display Results                           │
│  - Summary (expandable)                        │
│  - Metrics table                               │
│  - Charts (matplotlib)                         │
│  - Generate Report button                      │
└────────────────────────────────────────────────┘
                 │ (Optional)
                 ▼
┌────────────────────────────────────────────────┐
│  POST /generate_report                         │
│  {                                             │
│    "format": "pdf",                            │
│    "metrics_data": {...},                      │
│    "llm_summary": "...",                       │
│    "health_prompt": "..."                      │
│  }                                             │
│  → Save to /tmp/<report_id>.pdf                │
└────────────────┬───────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────┐
│  GET /download_report/{report_id}              │
│  → FileResponse(report.pdf)                    │
└────────────────────────────────────────────────┘
```

### 2. Chat Interface Flow (Claude Desktop-style)

```
User: "How many pods are running in the cluster?"
                │
                ▼
┌───────────────────────────────────────────────┐
│  PrometheusChatBot (claude_integration.py)    │
│  - Uses Anthropic Claude API                  │
│  - Tools: search_metrics, execute_promql, ... │
└───────────────┬───────────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────────┐
│  Claude API Decision: Use tools sequentially  │
│  Iteration 1: search_metrics("pod running")   │
└───────────────┬───────────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────────┐
│  MCP Tool: search_metrics()                   │
│  Query: /api/v1/label/__name__/values         │
│  Filter: matches ".*pod.*"                    │
│  Return: [kube_pod_status_phase, ...]         │
└───────────────┬───────────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────────┐
│  Claude API Decision: Execute query           │
│  Iteration 2: execute_promql(                 │
│    "sum(kube_pod_status_phase{phase='Running'})" │
│  )                                            │
└───────────────┬───────────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────────┐
│  MCP Tool: execute_promql()                   │
│  Query Prometheus: /api/v1/query             │
│  Return: {"value": 42}                        │
└───────────────┬───────────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────────┐
│  Claude Final Response:                       │
│  "There are 42 running pods in the cluster."  │
│                                               │
│  UI Display:                                  │
│  ✓ Tool: search_metrics                       │
│  ✓ Tool: execute_promql                       │
│  📊 Final Answer                              │
└───────────────────────────────────────────────┘
```

### 3. AI Assistant Integration (Cursor/Claude Desktop)

```
┌────────────────────────────────┐
│  User in Cursor IDE:           │
│  "List vLLM models in test1"   │
└────────────────┬───────────────┘
                 │ (STDIO)
                 ▼
┌────────────────────────────────────────┐
│  Cursor MCP Client                     │
│  - Spawns: obs-mcp-stdio               │
│  - Communication: stdin/stdout         │
│  - Protocol: JSON-RPC 2.0              │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│  obs-mcp-stdio (stdio_server.py)       │
│  - Reads stdin: tool call requests     │
│  - Writes stdout: tool responses       │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│  ObservabilityMCPServer                │
│  Tool Registry:                        │
│  - list_models()                       │
│  - list_namespaces()                   │
│  - analyze_vllm()                      │
│  - ... (21+ tools)                     │
└────────────────┬───────────────────────┘
                 │
                 ▼ (Multi-step)
┌────────────────────────────────────────┐
│  Claude/Cursor AI:                     │
│  1. Call list_namespaces()             │
│     → ["test1", "prod", "dev"]         │
│  2. Call list_models()                 │
│     → ["test1 | llama-3.2-3b", ...]    │
│  3. Filter by namespace                │
│  4. Format response                    │
│                                        │
│  Response:                             │
│  "The vLLM models in test1:            │
│   - llama-3.2-3b                       │
│   - mistral-7b"                        │
└────────────────────────────────────────┘
```

---

## Integration Patterns

### 1. UI ↔ MCP Server (HTTP)

**Protocol**: FastMCP HTTP Client

```python
# UI side (mcp_client_helper.py)
from fastmcp import FastMCP

client = FastMCP(server_url="http://mcp-server:8085/mcp")
result = await client.call_tool("analyze_vllm", {
    "model_name": "test1 | llama-3.2-3b",
    "time_range": "last 24h"
})
```

**Request Format** (JSON-RPC 2.0):
```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "analyze_vllm",
    "arguments": {
      "model_name": "test1 | llama-3.2-3b",
      "time_range": "last 24h"
    }
  },
  "id": "1"
}
```

**Response Format**:
```json
{
  "jsonrpc": "2.0",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"health_prompt\": \"...\", \"llm_summary\": \"...\"}"
      }
    ]
  },
  "id": "1"
}
```

### 2. MCP Server ↔ Prometheus/Thanos (HTTP + Bearer Auth)

```python
# MCP Server side (core/thanos_service.py)
import requests

response = requests.get(
    f"{prometheus_url}/api/v1/query_range",
    params={
        "query": "vllm:request_success_total",
        "start": start_ts,
        "end": end_ts,
        "step": "60"
    },
    headers={"Authorization": f"Bearer {thanos_token}"},
    verify="/etc/pki/ca-trust/extracted/pem/ca-bundle.crt"
)
```

### 3. MCP Server ↔ LlamaStack/External LLMs (OpenAI-compatible API)

```python
# MCP Server side (core/llm_client.py)
import requests

response = requests.post(
    f"{llama_stack_url}/chat/completions",
    headers={"Authorization": f"Bearer {api_token}"},
    json={
        "model": model_id,
        "messages": [
            {"role": "system", "content": "You are a metrics analyst..."},
            {"role": "user", "content": prompt}
        ],
        "temperature": 0.7,
        "max_tokens": 2000
    }
)
```

### 4. MCP Server ↔ Tempo (HTTP + Tenant ID header)

```python
# MCP Server side (tools/tempo_query_tool.py)
import requests

response = requests.get(
    f"{tempo_url}/api/search",
    params={
        "q": "service.name=ui",
        "start": start_ts,
        "end": end_ts
    },
    headers={
        "Authorization": f"Bearer {token}",
        "X-Scope-OrgID": tenant_id
    }
)
```

---

## Deployment Architecture

### OpenShift/Kubernetes Deployment

```
┌───────────────────────────────────────────────────────────┐
│          Application Namespace (e.g., test1)              │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  ┌────────────────────┐   ┌─────────────────────────┐    │
│  │  UI Deployment     │   │  MCP Server Deployment  │    │
│  │  ───────────────   │   │  ──────────────────────  │    │
│  │  Pod: ui           │   │  Pod: mcp-server-app    │    │
│  │  Image: aiobs-ui   │   │  Image: aiobs-mcp-server│    │
│  │  Port: 8501        │   │  Port: 8085             │    │
│  │  Replicas: 1       │   │  Replicas: 1            │    │
│  │                    │   │  ServiceAccount:        │    │
│  │  Env:              │   │    mcp-analyzer         │    │
│  │  - MCP_SERVER_URL  │◀──│                         │    │
│  │  - PROM_URL        │   │  Env:                   │    │
│  └────────────────────┘   │  - PROMETHEUS_URL       │    │
│           │                │  - THANOS_TOKEN         │    │
│           │                │  - TEMPO_URL            │    │
│           │                │  - LLAMA_STACK_URL      │    │
│           │                │  - MODEL_CONFIG         │    │
│           │                └─────────────────────────┘    │
│           │                           │                   │
│           ▼                           ▼                   │
│  ┌────────────────────┐   ┌─────────────────────────┐    │
│  │  UI Service        │   │  MCP Server Service     │    │
│  │  ClusterIP:8501    │   │  ClusterIP:8085         │    │
│  └────────────────────┘   └─────────────────────────┘    │
│           │                           │                   │
│           ▼                           ▼                   │
│  ┌────────────────────┐   ┌─────────────────────────┐    │
│  │  UI Route          │   │  MCP Server Route       │    │
│  │  TLS: edge         │   │  TLS: edge              │    │
│  │  ui.apps.domain    │   │  mcp.apps.domain        │    │
│  └────────────────────┘   └─────────────────────────┘    │
│                                                           │
│  ┌─────────────────────────────────────────────────┐     │
│  │  Alerting CronJob                               │     │
│  │  Schedule: "0 * * * *" (hourly)                 │     │
│  │  Image: aiobs-alerting                          │     │
│  │  Env: SLACK_WEBHOOK_URL                         │     │
│  └─────────────────────────────────────────────────┘     │
│                                                           │
│  ┌─────────────────────────────────────────────────┐     │
│  │  LlamaStack Deployment (optional)               │     │
│  │  Pod: llamastack                                │     │
│  │  Port: 8321                                     │     │
│  │  Serves: OpenAI-compatible API                  │     │
│  └─────────────────────────────────────────────────┘     │
│                                                           │
└───────────────────────────────────────────────────────────┘
                            │
                            │ (Cross-namespace)
                            ▼
┌───────────────────────────────────────────────────────────┐
│         openshift-monitoring namespace                    │
├───────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────┐     │
│  │  Thanos Querier                                 │     │
│  │  Service: thanos-querier.svc:9091               │     │
│  │  TLS: Yes (service-ca)                          │     │
│  │  Access: via ServiceAccount token               │     │
│  └─────────────────────────────────────────────────┘     │
└───────────────────────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────┐
│         observability-hub namespace                       │
├───────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │ TempoStack   │  │ OTel         │  │ MinIO        │    │
│  │ Gateway      │  │ Collector    │  │ Storage      │    │
│  │ :8080        │  │ :4318        │  │ :9000        │    │
│  └──────────────┘  └──────────────┘  └──────────────┘    │
└───────────────────────────────────────────────────────────┘
```

### RBAC Configuration

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mcp-analyzer
  namespace: test1
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: mcp-analyzer-monitoring-test1
subjects:
  - kind: ServiceAccount
    name: mcp-analyzer
    namespace: test1
roleRef:
  kind: ClusterRole
  name: cluster-monitoring-view
  apiGroup: rbac.authorization.k8s.io
---
# Trusted CA ConfigMap (auto-injected by OpenShift)
apiVersion: v1
kind: ConfigMap
metadata:
  name: mcp-server-trusted-ca-bundle
  namespace: test1
  labels:
    config.openshift.io/inject-trusted-cabundle: "true"
```

**Token Usage**:
```yaml
# In MCP Server deployment
env:
  - name: THANOS_TOKEN
    valueFrom:
      secretKeyRef:
        name: mcp-analyzer  # Auto-generated SA token
        key: token

volumeMounts:
  - name: thanos-token
    mountPath: /var/run/secrets/kubernetes.io/serviceaccount
    readOnly: true
  - name: trusted-ca
    mountPath: /etc/pki/ca-trust/extracted/pem
    readOnly: true
```

---

## Communication Protocols

### 1. Model Context Protocol (MCP)

**Specification**: MCP 1.0.0  
**Transports**: HTTP, SSE, STDIO

#### Tool Schema Example
```json
{
  "name": "analyze_vllm",
  "description": "Analyze vLLM model metrics and generate LLM-powered summary",
  "inputSchema": {
    "type": "object",
    "properties": {
      "model_name": {
        "type": "string",
        "description": "Model identifier (format: 'namespace | model')"
      },
      "time_range": {
        "type": "string",
        "description": "Time range (e.g., 'last 24h', 'last 1w', 'on 2025-01-15')"
      },
      "summarize_model_id": {
        "type": "string",
        "description": "LLM model ID for summarization (from MODEL_CONFIG)"
      },
      "api_key": {
        "type": "string",
        "description": "API key for external LLM providers (optional)"
      }
    },
    "required": ["model_name", "summarize_model_id"]
  }
}
```

### 2. Prometheus Query API

**Endpoints**:
- `/api/v1/query` - Instant query
- `/api/v1/query_range` - Range query
- `/api/v1/series` - Series metadata
- `/api/v1/label/<label>/values` - Label values

**Example**:
```http
GET /api/v1/query_range?query=vllm:request_success_total&start=1704067200&end=1704153600&step=60
Host: thanos-querier.openshift-monitoring.svc.cluster.local:9091
Authorization: Bearer eyJhbGciOi...
```

### 3. Tempo TraceQL API

**Endpoints**:
- `/api/search` - Search traces
- `/api/traces/<trace_id>` - Get trace details

**Example**:
```http
GET /api/search?q=service.name=ui&start=1704067200&end=1704153600
Host: tempo-tempostack-gateway.observability-hub.svc.cluster.local:8080
Authorization: Bearer <token>
X-Scope-OrgID: dev
```

---

## Security & RBAC

### Authentication Flow

```
MCP Server Pod Start
        │
        ▼
ServiceAccount: mcp-analyzer
- Token auto-mounted at /var/run/secrets/kubernetes.io/serviceaccount/token
        │
        ▼
Environment Variable
THANOS_TOKEN=$(cat /var/run/secrets/.../token)
        │
        ▼
Prometheus/Thanos Query
Authorization: Bearer $THANOS_TOKEN
```

### TLS/CA Trust

```yaml
# ConfigMap created with label
config.openshift.io/inject-trusted-cabundle: "true"
# OpenShift injects service-ca.crt automatically

# Volume mount in deployment
volumeMounts:
  - name: trusted-ca
    mountPath: /etc/pki/ca-trust/extracted/pem
    readOnly: true

# Python usage
requests.get(url, verify="/etc/pki/ca-trust/extracted/pem/ca-bundle.crt")
```

---

## Scalability & Performance

### Performance Optimizations

1. **Prometheus Query Optimization**
   - Auto-calculated step size: `ceil((end-start)/11000)` rounded to nice intervals
   - Maximum time range: 90 days (configurable via `MAX_TIME_RANGE_DAYS`)
   - Default time range: 7 days (configurable via `DEFAULT_TIME_RANGE_DAYS`)

2. **Async I/O**
   - FastAPI async endpoints
   - Non-blocking LLM calls
   - Concurrent Prometheus queries

3. **Resource Limits**
```yaml
resources:
  requests:
    cpu: "1"
    memory: "1Gi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

### Scaling Considerations

**Horizontal Scaling**:
- MCP Server is stateless (except /tmp reports)
- Can scale to N replicas
- Load balancing via Kubernetes Service

**Bottlenecks**:
1. LLM inference time (5-30s per request)
2. Prometheus query response time (1-10s)
3. Report generation (CPU-bound, 2-5s)

**Solutions**:
- Use external LLMs (OpenAI, Gemini) for faster inference
- Implement Redis caching for frequent queries
- Offload report generation to background jobs
- Scale MCP Server replicas for high traffic

---

## Summary

### Key Design Decisions
1. **FastMCP Framework**: Chosen for built-in MCP protocol support, multiple transports, and OpenAPI schema generation
2. **Tool-Based Architecture**: Each capability exposed as an MCP tool for maximum flexibility
3. **Shared Core Logic**: Business logic in `src/core/` reused across tools and UI
4. **Multi-Transport Support**: HTTP for UI, STDIO for AI assistants, SSE for streaming
5. **OpenShift-Native**: RBAC, ServiceAccounts, TLS via service-ca, ConfigMap injection

### Architecture Benefits
- **Unified API**: Single MCP server serves UI, AI assistants, and external clients
- **Extensibility**: Easy to add new tools without changing UI
- **Observability**: Full OpenTelemetry tracing, structured logging
- **Security**: Token-based auth, TLS verification, RBAC
- **Multi-Model**: Support for internal and external LLMs

### Component Count
- **MCP Tools**: 21+ (vLLM: 10, OpenShift: 4, Prometheus: 9, Tempo: 3)
- **Core Modules**: 10 (llm_client, thanos_service, reports, etc.)
- **Deployments**: 3-4 (UI, MCP Server, Alerting, optional LlamaStack)
- **Namespaces**: 3 (application, openshift-monitoring, observability-hub)

---

**Document Version**: 1.0  
**Last Updated**: 2025-10-06  
**Author**: OpenShift AI Observability Team

