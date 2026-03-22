# Drone Demo - Let us fly a drone to analyze audience engagement

A GitOps-based Kubernetes AI Agent demo that combines AI agents and MCP (Model Context Protocol) servers to power an autonomous AI agent capable of diagnosing, remediating, and managing Kubernetes clusters through ArgoCD-driven GitOps workflows.

## Repository Structure

```
drone-demo/
├── agents/          # AI agent definition (AIRE Agent)
├── argo/            # Argo MCP server configuration (deprecated)
├── mcp/             # MCP server deployments
├── mcp-config/      # Agent Gateway and MCP backend configuration
├── skills/          # kagent Agent Skills (OCI-packaged)
└── yamls/           # Application deployment manifests
```

## Folders

### `yamls/` — Application Deployment Manifests

Kubernetes manifests for deploying the demo application workloads into the cluster:

| File | Description |
|------|-------------|
| `frontend.yaml` | React frontend (`tello-frontend`) — Deployment, Service, and ServiceAccount |
| `backend.yaml` | Backend service deployment |
| `gateway.yaml` | Gateway resource for routing external traffic |
| `frontend-httproute.yaml` | HTTPRoute for the frontend service |
| `se-ollama.yaml` | ServiceEntry for an Ollama LLM endpoint |

The source code of the drone-apps (frontend and backend) are located at https://github.com/linsun/drone-apps.

---

### `skills/` — kagent Agent Skills

Contains OCI-packaged [Agent Skills](https://www.kagent.dev/docs/kagent/examples/skills) that extend the AI agent's capabilities. Skills are built as minimal container images (using `FROM scratch`) and referenced by the agent at runtime.

| Skill | Description |
|-------|-------------|
| `httproute/` | Skill providing HTTPRoute resource definitions and knowledge to the agent |

Skills are published to a container registry (e.g. `docker.io/linsun/httproute:v2`) and loaded by the agent via the `skills.refs` field in the Agent spec.

---

### `mcp-config/` — Agent Gateway & MCP Backend Configuration

Configuration for the [Agent Gateway](https://agentgateway.dev) that acts as the MCP proxy, routing requests from the AI agent to the various MCP backend servers.

| File | Description |
|------|-------------|
| `mcp-gateway.yaml` | Deploys the Agent Gateway (`GatewayClass`, `Gateway`, and `AgentgatewayParameters`) listening on ports 3000 and 3001 |
| `mcp-backends.yaml` | Defines the `AgentgatewayBackend` MCP targets (ArgoCD, GitHub, kagent) and the `HTTPRoute` rules to connect them |
| `mcp-backend2.yaml` | Alternate/updated backend configuration |

The backend targets currently configured are:

- **argocd** — ArgoCD MCP server (`argocd-mcp-server.mcp.svc.cluster.local:3000`)
- **github** — GitHub MCP server (`github-mcp-server.mcp.svc.cluster.local:3000`)
- **kagent** — kagent built-in tool server (`kagent-controller.kagent.svc.cluster.local:8083`)

---

### `mcp/` — MCP Server Deployments

Kubernetes manifests for deploying the individual MCP servers that back the Agent Gateway. Each server runs as a Deployment alongside the agentgateway binary (injected via init container) to bridge stdio-based MCP tools over HTTP.

| File | Description |
|------|-------------|
| `argocd-mcp-server.yaml` | Deploys the ArgoCD MCP server — runs `argocd-mcp@latest` via `npx`, proxied by agentgateway. Includes ServiceAccount, Service, Deployment, ConfigMap, and Secret |
| `github-mcp-server.yaml` | Deploys the GitHub MCP server — runs the official `ghcr.io/github/github-mcp-server` image, proxied by agentgateway. Includes ServiceAccount, Service, Deployment, and ConfigMap |

---

### `argo/` — Argo MCP Server Configuration (Deprecated)

> **Deprecated.** This folder contains an earlier approach to integrating with ArgoCD using kagent's native `MCPServer` CRD directly (without Agent Gateway). It has been superseded by the `mcp/` + `mcp-config/` approach.

| File | Description |
|------|-------------|
| `argo-mcp.yaml` | `kagent.dev/v1alpha1 MCPServer` resource that ran `argocd-mcp@latest` via `npx stdio` and connected directly into kagent |

---

### `agents/` — AI Agent Definition

Contains the kagent `Agent` custom resource that defines the **AIRE Agent** (AI Remediation Engine) — a declarative, GitOps-aware Kubernetes expert agent.

| File | Description |
|------|-------------|
| `aire-agent.yaml` | Full agent definition including system prompt, A2A skills, model config, MCP tool bindings, and Agent Skills reference |

The AIRE Agent exposes three [A2A](https://google.github.io/A2A/) skills:

- **Cluster Diagnostics** — Analyze and diagnose Kubernetes cluster issues
- **Resource Management** — Manage and optimize Kubernetes resources
- **Security Audit** — Audit and enhance Kubernetes security posture

The agent is bound to two MCP tool servers:

- `agw-mcp-servers` (Agent Gateway) — GitHub tools: `create_pull_request`, `create_branch`, `get_file_contents`, `search_code`, `create_or_update_file`, etc.
- `kagent-tool-server` — Kubernetes tools: `k8s_get_resources`, `k8s_describe_resource`, `k8s_get_pod_logs`, `k8s_execute_command`, etc.

Its core behavior follows a **GitOps-first** remediation approach: all fixes are implemented by creating a branch, committing a patch, and opening a pull request in the target GitHub repository — never by directly mutating cluster state.

## Architecture Overview

```
┌─────────────────────────────────────────────┐
│                  AIRE Agent                  │
│           (kagent Agent CRD)                 │
│    + Agent Skills (OCI: httproute:v2)        │
└────────────┬──────────────────┬─────────────┘
             │                  │
     GitHub tools          k8s_* tools
             │                  │
┌────────────▼──────────────────▼─────────────┐
│              Agent Gateway                   │
│         (agentgateway GatewayClass)          │
│              mcp-config/                     │
└──────┬──────────────┬───────────────┬────────┘
       │              │               │
  ┌────▼────┐   ┌─────▼─────┐  ┌─────▼──────┐
  │  ArgoCD │   │  GitHub   │  │   kagent   │
  │   MCP   │   │   MCP     │  │   tools    │
  │ Server  │   │  Server   │  │   server   │
  │  mcp/   │   │   mcp/    │  │            │
  └─────────┘   └───────────┘  └────────────┘
```

## Prerequisites

- Kubernetes cluster
- [kagent](https://www.kagent.dev) installed
- [Agent Gateway](https://agentgateway.dev) CRDs installed
- ArgoCD installed in the `argocd` namespace
- A GitHub personal access token (stored as `github-mcp-secret` in the `mcp` namespace)
- An ArgoCD API token (stored as `argocd-mcp-secret` in the `mcp` namespace)

## Applying the Manifests

```bash
# Deploy application workloads, assuming you have k8s installed
kubectl apply -f yamls/

# Deploy MCP servers, assuming you have kmcp installed
kubectl apply -f mcp/

# Deploy Agent Gateway and backend configuration
kubectl apply -f mcp-config/

# Deploy the AI agent - assuming you have kagent installed
kubectl apply -f agents/
```
