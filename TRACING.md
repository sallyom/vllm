# OpenTelemetry Tracing in vLLM

vLLM supports OpenTelemetry distributed tracing to monitor LLM inference performance and latency metrics.

## Quick Start

### V1 Engine (Recommended) - Zero Configuration

vLLM V1 automatically detects when OpenTelemetry auto-instrumentation is active and enables tracing without additional configuration.

```bash
# Install auto-instrumentation
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap --action=install

# Start vLLM with auto-instrumentation (zero config)
opentelemetry-instrument vllm serve facebook/opt-125m
```

### V0 Engine - Manual Configuration

The V0 engine still works as it did before V1, requiring explicit OTLP endpoint configuration:

```bash
vllm serve facebook/opt-125m --otlp-traces-endpoint http://localhost:4317
```

## Kubernetes Deployment with OpenTelemetry Operator

### Prerequisites

Install the OpenTelemetry Operator:
```bash
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
```

### Setup

Create two Custom Resources:

**1. OpenTelemetry Collector:**
```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-collector
  namespace: default
spec:
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    
    processors:
      batch:
    
    exporters:
      # Configure your backend (e.g., Jaeger, Zipkin, etc.)
      jaeger:
        endpoint: jaeger-collector:14250
        tls:
          insecure: true
    
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [jaeger]
```

**2. Instrumentation Configuration:**
```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: vllm-instrumentation
  namespace: default
spec:
  exporter:
    endpoint: http://otel-collector:4317
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "1"
  python:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:latest
```

**3. Deploy vLLM with Instrumentation:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-server
spec:
  template:
    metadata:
      annotations:
        instrumentation.opentelemetry.io/inject-python: "vllm-instrumentation"
    spec:
      containers:
      - name: vllm
        image: vllm/vllm-openai:latest
        command: ["vllm", "serve", "facebook/opt-125m"]
        # No additional tracing flags needed - auto-detected!
```

## Span Attributes

vLLM automatically captures these OpenTelemetry attributes:

### Request Metadata
- `gen_ai.request.id` - Unique request identifier
- `gen_ai.request.model` - Model name
- `gen_ai.request.max_tokens` - Maximum tokens requested
- `gen_ai.request.temperature` - Sampling temperature
- `gen_ai.request.top_p` - Top-p sampling parameter

### Token Usage
- `gen_ai.usage.prompt_tokens` - Input token count
- `gen_ai.usage.completion_tokens` - Generated token count

### Latency Metrics
- `gen_ai.latency.e2e` - End-to-end request latency
- `gen_ai.latency.time_to_first_token` - Time to first token generation
- `gen_ai.latency.time_in_queue` - Time spent in request queue
- `gen_ai.latency.time_in_model_prefill` - Prefill phase latency
- `gen_ai.latency.time_in_model_decode` - Decode phase latency
- `gen_ai.latency.time_in_model_inference` - Total inference latency

## Manual Configuration (Advanced)

For custom setups, you can still explicitly configure the OTLP endpoint:

```bash
# V1 Engine with explicit endpoint
vllm serve facebook/opt-125m --otlp-traces-endpoint http://your-collector:4317

# V0 Engine (legacy)
VLLM_USE_V1=0 vllm serve facebook/opt-125m --otlp-traces-endpoint http://your-collector:4317
```

## Trace Context Propagation

vLLM automatically extracts and propagates trace context from incoming HTTP requests, enabling distributed tracing across your application stack.

## Documentation Links

- [OpenTelemetry Operator](https://opentelemetry.io/docs/kubernetes/operator/)
- [OpenTelemetry Python Auto-instrumentation](https://opentelemetry.io/docs/languages/python/automatic/)
- [OTLP Exporter Configuration](https://opentelemetry.io/docs/specs/otlp/)
- [Jaeger Tracing](https://www.jaegertracing.io/docs/getting-started/)