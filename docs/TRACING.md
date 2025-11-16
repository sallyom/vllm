# Distributed Tracing for vLLM

This document describes OpenTelemetry distributed tracing support in vLLM.

## Overview

vLLM implements OpenTelemetry tracing with custom spans to provide observability into inference workloads. The instrumentation captures latency metrics, resource utilization, and request characteristics while following metadata-only tracing principles (no prompts or generated text).

### Main Spans

- **`llm_request`**: Full request lifecycle from arrival to completion (SERVER span)
  - Captures: TTFT, E2E latency, queue time, prefill/decode time, token counts
  - Propagates trace context from incoming requests
  - Created per-request in OutputProcessor when request finishes

## Configuration

Tracing is configured via environment variables:

```bash
# Enable tracing by setting OTLP endpoint in observability config
# The endpoint can be configured in your VllmConfig:
vllm_config = VllmConfig(
    observability_config=ObservabilityConfig(
        otlp_traces_endpoint="http://otel-collector:4317"
    )
)

# OTLP protocol (optional, default: grpc)
export OTEL_EXPORTER_OTLP_TRACES_PROTOCOL="grpc"  # or "http/protobuf"
```

## Initialization

Tracing is automatically initialized when `otlp_traces_endpoint` is set in the ObservabilityConfig:

```python
from vllm import LLM, SamplingParams
from vllm.config import VllmConfig, ObservabilityConfig

# Create config with tracing enabled
vllm_config = VllmConfig(
    # ... other config ...
    observability_config=ObservabilityConfig(
        otlp_traces_endpoint="http://localhost:4317"
    )
)

# Create LLM engine
llm = LLM.from_vllm_config(vllm_config)
```

## Spans and Attributes

### llm_request (SERVER)

Created in `OutputProcessor.do_tracing()` when a request completes.

**Latency Attributes:**
- `gen_ai.latency.time_to_first_token` (float): Time to first token (TTFT) in seconds
- `gen_ai.latency.e2e` (float): End-to-end latency in seconds
- `gen_ai.latency.time_in_queue` (float): Time spent waiting in queue
- `gen_ai.latency.time_in_model_prefill` (float): Time spent in prefill phase
- `gen_ai.latency.time_in_model_decode` (float): Time spent in decode phase
- `gen_ai.latency.time_in_model_inference` (float): Total inference time (prefill + decode)

**Usage Attributes:**
- `gen_ai.usage.prompt_tokens` (int): Number of prompt tokens
- `gen_ai.usage.completion_tokens` (int): Number of generated tokens

**Request Parameter Attributes:**
- `gen_ai.request.id` (string): Unique request identifier
- `gen_ai.request.top_p` (float): Top-p sampling parameter (if set)
- `gen_ai.request.max_tokens` (int): Maximum tokens to generate (if set)
- `gen_ai.request.temperature` (float): Temperature parameter (if set)
- `gen_ai.request.n` (int): Number of completions to generate (if set)

## Trace Context Propagation

vLLM automatically propagates W3C trace context from incoming requests:

1. Extract trace headers (`traceparent`, `tracestate`) from incoming HTTP requests
2. Pass trace headers to `LLMEngine.add_request()` via `trace_headers` parameter
3. Trace context is automatically extracted and used when creating the `llm_request` span

Example with OpenAI API compatibility layer:
```python
# Trace headers are automatically extracted from HTTP requests
# and passed to the engine
```

## Example Trace

A complete trace for an inference request:

```
llm_request (2150ms)
├── Attributes:
│   ├── gen_ai.latency.time_to_first_token: 0.045
│   ├── gen_ai.latency.e2e: 2.150
│   ├── gen_ai.latency.time_in_queue: 0.012
│   ├── gen_ai.latency.time_in_model_prefill: 0.033
│   ├── gen_ai.latency.time_in_model_decode: 2.105
│   ├── gen_ai.usage.prompt_tokens: 128
│   ├── gen_ai.usage.completion_tokens: 512
│   ├── gen_ai.request.temperature: 0.7
│   └── gen_ai.request.max_tokens: 512
```

## Viewing Traces

### With Jaeger

1. Deploy Jaeger and OpenTelemetry Collector:
```bash
# docker-compose.yml
version: '3'
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # Jaeger UI
      - "14250:14250"  # gRPC collector

  otel-collector:
    image: otel/opentelemetry-collector:latest
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "4317:4317"  # OTLP gRPC
```

2. Configure vLLM:
```python
vllm_config = VllmConfig(
    observability_config=ObservabilityConfig(
        otlp_traces_endpoint="http://localhost:4317"
    )
)
```

3. View traces at http://localhost:16686

### Query Examples

**Find all vLLM requests:**
```
service="vllm.llm_engine" AND operation="llm_request"
```

**Find slow requests (>1s TTFT):**
```
gen_ai.latency.time_to_first_token > 1.0
```

**Find requests with high token counts:**
```
gen_ai.usage.prompt_tokens > 1000 OR gen_ai.usage.completion_tokens > 1000
```

## Security Considerations

The instrumentation follows metadata-only tracing:

**Captured:**
- ✅ Token counts (prompt and completion)
- ✅ Latency metrics (TTFT, E2E, queue time)
- ✅ Request parameters (temperature, top_p, max_tokens)
- ✅ Request IDs

**NOT Captured:**
- ❌ Actual prompts or prompt text
- ❌ Generated completions or output text
- ❌ Token IDs or vocabulary
- ❌ User identifiers or sensitive metadata

## Performance Impact

- **Overhead when tracing disabled**: ~0% (no tracer initialized)
- **Overhead when tracing enabled**: <1% latency increase
  - Span creation happens only on request completion
  - Uses OpenTelemetry BatchSpanProcessor for efficient export
  - Minimal overhead from attribute setting

## Integration with Gateway

When vLLM is called by the llm-d gateway, trace context is automatically propagated via HTTP headers:

```python
# Gateway sends trace headers with request
headers = {
    "traceparent": "00-trace_id-span_id-01",
    "tracestate": "..."
}

# vLLM extracts and continues the trace
```

This creates a distributed trace spanning:
```
gateway.request (SERVER)
├── gateway.director.handle_request (INTERNAL)
├── gateway.scheduler.schedule (INTERNAL)
└── [HTTP call to vLLM]
    └── llm_request (SERVER) ← trace continues here
```

## References

- [OpenTelemetry Python Documentation](https://opentelemetry.io/docs/languages/python/)
- [GenAI Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
- [vLLM Tracing Implementation](https://github.com/vllm-project/vllm/blob/main/vllm/tracing.py)
- [llm-d Distributed Tracing Proposal](../../llm-d/docs/proposals/distributed-tracing.md)
