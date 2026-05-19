# kagent-sre-agent

Kubernetes manifests that wire up a **SigNoz-powered SRE / observability agent** in
[kagent](https://kagent.dev). The bundle deploys the SigNoz MCP server inside the
cluster, registers it with kagent as a `RemoteMCPServer`, configures an LLM via
OpenRouter, and declares a `kagent.dev/v1alpha2` `Agent` that uses both to answer
questions about logs, metrics, traces, alerts, and dashboards stored in SigNoz.

## Architecture

```
            ┌──────────────────────────┐
            │     kagent Agent         │
            │   (signoz-agent.yaml)    │
            └────────────┬─────────────┘
                         │ uses
          ┌──────────────┼──────────────────────┐
          ▼                                     ▼
┌────────────────────┐              ┌────────────────────────┐
│ ModelConfig        │              │ RemoteMCPServer        │
│ openrouter-model   │              │ signoz-mcpserver       │
│ (openrouter-       │              │ (signoz-               │
│  modelconfig.yaml) │              │  remotemcpserver.yaml) │
└──────────┬─────────┘              └────────────┬───────────┘
           │ HTTPS                               │ HTTP /mcp
           ▼                                     ▼
   openrouter.ai/api/v1               ┌────────────────────────┐
                                      │ signoz-mcp Deployment  │
                                      │ + Service              │
                                      │ (signoz-mcp-server.    │
                                      │  yaml)                 │
                                      └────────────┬───────────┘
                                                   │ SigNoz API
                                                   ▼
                                      ┌────────────────────────┐
                                      │ SigNoz (ClickHouse,    │
                                      │ OTel Collector, etc.)  │
                                      │ instrumented via       │
                                      │ signoz-agent-          │
                                      │ values.yaml            │
                                      └────────────────────────┘
```

The agent never talks to SigNoz directly. It calls MCP tools exposed by the
in-cluster `signoz-mcp` server, which translates those tool calls into SigNoz
API requests.

## Repository layout

```
.
├── kagent/                          # resources in the kagent namespace
│   ├── kagent-values.yaml           #   Helm values for the kagent chart (with OTel)
│   ├── openrouter-modelconfig.yaml  #   kagent ModelConfig (OpenRouter)
│   ├── signoz-remotemcpserver.yaml  #   kagent RemoteMCPServer → signoz-mcp-svc
│   └── signoz-agent.yaml            #   kagent Agent definition + tool allowlist
└── signoz/                          # resources in the signoz namespace
    ├── signoz-agent-values.yaml     #   Helm values for k8s-infra OTel agent
    └── signoz-mcp-server.yaml       #   Deployment + Service for SigNoz MCP server
```

## Prerequisites

- A Kubernetes cluster with:
  - [kagent](https://kagent.dev) installed (CRDs `Agent`, `ModelConfig`,
    `RemoteMCPServer` in API group `kagent.dev/v1alpha2`) in the `kagent`
    namespace.
  - [SigNoz](https://signoz.io) installed in the `signoz` namespace (the
    manifests assume the standard Helm chart service names
    `signoz.signoz.svc.cluster.local` and
    `signoz-otel-collector.signoz.svc.cluster.local`).
- A SigNoz API key.
- An [OpenRouter](https://openrouter.ai) API key.

## Files

### `kagent/openrouter-modelconfig.yaml`
A kagent `ModelConfig` named **`openrouter-model`** in the `kagent` namespace.
It points the agent at OpenRouter using the OpenAI-compatible API:

- `model: openai/gpt-oss-120b:free` — the LLM the agent will use.
- `provider: OpenAI` with `openAI.baseUrl: https://openrouter.ai/api/v1` so the
  OpenAI SDK is repointed at OpenRouter.
- `apiKeySecret: openrouter-secret` / `apiKeySecretKey: api-key` — kagent reads
  the key from a Kubernetes `Secret` you must create yourself:

  ```bash
  kubectl -n kagent create secret generic openrouter-secret \
    --from-literal=api-key=sk-or-...
  ```

Swap the model string for any other OpenRouter-hosted model if you prefer.

### `signoz/signoz-mcp-server.yaml`
A `Deployment` + `ClusterIP` `Service` for the SigNoz MCP server, deployed into
the `signoz` namespace.

- Image: `signoz/signoz-mcp-server`.
- Env:
  - `SIGNOZ_API_KEY` — **replace the placeholder `<SIGNOZ_API_KEY`** with your
    real key, or refactor to read from a `Secret` (recommended).
  - `SIGNOZ_URL: http://signoz.signoz.svc.cluster.local:8080` — in-cluster URL
    of the SigNoz query service.
  - `TRANSPORT_MODE: http` — exposes MCP over HTTP (vs stdio) so kagent can
    reach it as a `RemoteMCPServer`.
  - `MCP_SERVER_PORT: 8000`.
- Service `signoz-mcp-svc` exposes port `8000` with `appProtocol: mcp` so
  the URL `http://signoz-mcp-svc.signoz.svc.cluster.local:8000/mcp` resolves
  from inside the cluster.

### `kagent/signoz-remotemcpserver.yaml`
A kagent `RemoteMCPServer` named **`signoz-mcpserver`** in the `kagent`
namespace. This is the bridge between kagent and the in-cluster MCP server:

- `url: http://signoz-mcp-svc.signoz.svc.cluster.local:8000/mcp` — the Service
  defined above.
- `timeout: 30s`, `sseReadTimeout: 5m0s` — request and streaming timeouts the
  agent uses when calling MCP tools.

Once applied, the kagent control plane discovers the MCP tools and makes them
available for `Agent`s to reference.

### `kagent/signoz-agent.yaml`
The core `Agent` resource (`signoz-agent`, namespace `kagent`). It is a
`Declarative` agent that:

- Uses `modelConfig: openrouter-model` (defined above).
- Streams responses (`stream: true`).
- Carries a long `systemMessage` instructing the LLM how to behave as a
  SigNoz observability assistant — investigation workflow, filter syntax,
  default time windows, read-only safety rules, and the expected markdown
  response format.
- Lists the MCP tools it is allowed to call under `tools[].mcpServer`,
  referencing the `RemoteMCPServer` `signoz-mcpserver`:
  - `signoz_list_services`
  - `signoz_search_logs`
  - `signoz_query_metrics`, `signoz_list_metrics`
  - `signoz_search_traces`, `signoz_aggregate_traces`, `signoz_get_trace`
  - `signoz_list_alerts`, `signoz_list_alert_rules`, `signoz_get_alert`,
    `signoz_get_alert_history`
  - `signoz_list_dashboards`, `signoz_get_dashboard`
  - `signoz_get_field_keys`, `signoz_get_field_values`

Only the tools enumerated here are exposed to the LLM — even if the MCP server
advertises more.

### `kagent/kagent-values.yaml`
Helm `values.yaml` for the **kagent** chart that enables OpenTelemetry traces
and logs so the agent's own activity is observable in SigNoz:

- `providers.default: openAI` with `providers.openAI.apiKey: $OPENAI_API_KEY`
  — the LLM provider kagent itself uses for audit/observability features.
  Export `OPENAI_API_KEY` in the install environment, or swap this for an
  in-chart secret reference.
- `otel.tracing.enabled: true` and `otel.logging.enabled: true` — turn on
  OTLP emission from kagent's control-plane components.
- Both exporters use `otlp` to
  `http://signoz-otel-collector.signoz.svc.cluster.local:4317` with
  `insecure: true` and a 15s timeout — gRPC OTLP into the in-cluster SigNoz
  collector (note port `4317` here vs `4318` HTTP used by the k8s-infra agent).

Install kagent with observability enabled:

```bash
helm install kagent kagent/kagent -n kagent --create-namespace \
  -f kagent/kagent-values.yaml
```

After install, kagent's traces and logs will appear in SigNoz alongside the
rest of the cluster's telemetry — useful for debugging agent invocations and
MCP tool calls.

### `signoz/signoz-agent-values.yaml`
Helm `values.yaml` for the **`k8s-infra`** (a.k.a. SigNoz OpenTelemetry
collector agent) chart. It is **not** a kagent manifest; it instruments the
cluster so SigNoz has data to query.

- `global.cloud: others`, `clusterName: my-cluster`, `deploymentEnvironment: dev`
  — labels that tag every signal SigNoz receives.
- `otelCollectorEndpoint: http://signoz-otel-collector.signoz.svc.cluster.local:4318`
  with `otelInsecure: true` — sends OTLP/HTTP to the in-cluster SigNoz
  collector.
- `presets.otlphttpExporter.enabled: true`, `loggingExporter.enabled: false` —
  forward to OTLP, don't log every span to stdout.
- `otelAgent.useHostPort: false` — avoids binding host ports on nodes.

Apply with:

```bash
helm install k8s-infra signoz/k8s-infra -n signoz \
  -f signoz/signoz-agent-values.yaml
```

## Installation order

1. Install SigNoz in the `signoz` namespace.
2. (Optional) Install the k8s-infra OTel collector agent using
   `signoz/signoz-agent-values.yaml` so SigNoz starts receiving cluster telemetry.
3. Install kagent with observability enabled, pointing its OTLP exporters at
   the in-cluster SigNoz collector:
   ```bash
   export OPENAI_API_KEY=sk-...
   helm install kagent kagent/kagent -n kagent --create-namespace \
     -f kagent/kagent-values.yaml
   ```
4. Create the OpenRouter API key secret:
   ```bash
   kubectl -n kagent create secret generic openrouter-secret \
     --from-literal=api-key=sk-or-...
   ```
5. Edit `signoz/signoz-mcp-server.yaml` to set a real `SIGNOZ_API_KEY` (or
   replace the env entry with a `secretKeyRef`).
6. Apply the manifests:
   ```bash
   kubectl apply -f signoz/signoz-mcp-server.yaml
   kubectl apply -f kagent/signoz-remotemcpserver.yaml
   kubectl apply -f kagent/openrouter-modelconfig.yaml
   kubectl apply -f kagent/signoz-agent.yaml
   ```
7. Open the kagent UI and chat with `signoz-agent`.

## Verification

```bash
kubectl -n signoz get deploy signoz-mcp
kubectl -n signoz get svc signoz-mcp-svc
kubectl -n kagent get remotemcpserver signoz-mcpserver
kubectl -n kagent get modelconfig openrouter-model
kubectl -n kagent get agent signoz-agent
```

If the agent reports "no tools available", check:
- The `RemoteMCPServer` status — kagent must be able to reach
  `signoz-mcp-svc:8000/mcp`.
- The MCP server pod logs for SigNoz auth errors (placeholder API key not
  replaced).

## Notes / hardening

- `signoz-mcp-server.yaml` ships with a literal placeholder
  (`"<SIGNOZ_API_KEY"`) for the API key — replace it or migrate to a `Secret`
  before production use.
- The agent's `systemMessage` explicitly tells the model the deployment is
  **read-only**. The tool list also excludes any mutating SigNoz MCP tools.
- The model `openai/gpt-oss-120b:free` is a free-tier OpenRouter route; rate
  limits and quality differ from paid models. Swap it for a stronger model for
  real incident response.
