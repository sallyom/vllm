# DBO and EPLB Metrics Implementation

## Overview

This implementation adds Prometheus metrics for DBO (Dual Batch Overlap) and EPLB (Expert-Parallel Load Balancing) to address critical observability gaps in wide-EP deployments.

## Problems Being Addressed

### 1. **Throughput Variance in WideEP Deployments**

**Problem**: Production WideEP deployments exhibit unpredictable throughput degradation, but we lack visibility into the root causes.

**Symptoms**:
- Inconsistent tokens/second across identical requests
- Periodic performance drops with no clear trigger
- Wide variance in end-to-end latency

**Root Causes** (previously invisible):
- DBO falling out due to empty second ubatches at 256-token boundaries
- Expert load imbalance causing EP rank stalls
- Coordination failures between DP ranks during ubatching

**Solution**: DBO metrics (`dbo_active`, `dbo_fallout_total`, `ubatch_token_count`) expose when and why DBO disengages.

### 2. **Expert Load Imbalance in WideEP**

**Problem**: Expert parallelism can create severe load imbalance where some EP ranks process 3-5x more tokens than others, causing stragglers.

**Impact**:
- All-reduce operations blocked by slowest rank
- Underutilization of fast ranks
- Cascading latency across batches

**Solution**: EPLB metrics (`eplb_balancedness_ratio`, `eplb_avg_tokens_per_rank`, `eplb_max_tokens_per_rank`) quantify imbalance and track rebalancing effectiveness.

### 3. **Instance-Level Capacity Signals for External Load Balancers**

**Problem**: External LBs (K8s, nginx) lack visibility into instance-level EPLB state, leading to:
- Routing requests to instances mid-rebalancing (high latency)
- Poor scheduling decisions during expert rearrangement
- No feedback loop for intelligent routing

**Solution**: `vllm:eplb_rebalancing` binary flag signals when instance is rebalancing, enabling LB to avoid that instance.

### 4. **Per-Engine Throughput Tracking**

**Problem**: DP deployments hide per-engine variance - aggregate metrics mask individual engine issues.

**Solution**: Per-engine throughput gauges (`prompt_throughput_toks_per_s`, `generation_throughput_toks_per_s`) expose engine-level variance.

### 5. **Deep Debugging with Opt-in High-Cardinality Metrics**

**Problem**: When aggregated metrics show imbalance, we can't identify which specific experts are hot without code instrumentation.

**Solution**: Ephemeral per-expert debug metrics (`expert_load_per_expert_tokens_DEBUG`) provide expert-level granularity for short debugging windows.

## Context

See: `llm-d/guides/wide-ep-lws` for the WideEP deployment guide.

**Related Work**: Builds on Mark McLoughlin's EPLB observability proposal ([markmc/vllm:epblb-metrics-claude](https://github.com/vllm-project/vllm/compare/main...markmc:vllm:epblb-metrics-claude)).

## Metrics Added

### DBO (Dual Batch Overlap) Metrics

| Metric | Type | Description | Labels |
|--------|------|-------------|--------|
| `vllm:dbo_active` | Gauge | Whether DBO is currently engaged (1=active, 0=inactive) | model_name, engine, phase (prefill/decode) |
| `vllm:dbo_fallout_total` | Counter | Count of DBO fallout events | model_name, engine, reason |
| `vllm:ubatch_token_count` | Histogram | Distribution of ubatch sizes in tokens | model_name, engine, ubatch_index (first/second) |

**Fallout Reasons:**
- `empty_second_ubatch` - Second ubatch would be empty after padding
- `coordination_failure` - DP ranks disagreed on ubatching
- `other` - Other reasons

### EPLB (Expert-Parallel Load Balancing) Metrics

| Metric | Type | Description | Labels |
|--------|------|-------------|--------|
| `vllm:eplb_balancedness_ratio` | Gauge | Load balancedness ratio (avg/max load across ranks) | model_name, engine, layer |
| `vllm:eplb_max_tokens_per_rank` | Gauge | Maximum token load across all EP ranks | model_name, engine, layer |
| `vllm:eplb_avg_tokens_per_rank` | Gauge | Average token load per EP rank | model_name, engine, layer |
| `vllm:eplb_rebalancing` | Gauge | Whether EPLB is currently rebalancing (1=yes, 0=no) | model_name, engine |
| `vllm:eplb_rearrangements_total` | Counter | Count of EPLB expert rearrangement events | model_name, engine |
| `vllm:eplb_rearrangement_duration_seconds` | Histogram | Duration of EPLB rearrangement operations | model_name, engine |
| `vllm:expert_load_per_expert_tokens_DEBUG` | Gauge | DEBUG: Per-expert token load (HIGH CARDINALITY) | model_name, engine, layer, expert_id |

### Throughput Metrics

| Metric | Type | Description | Labels |
|--------|------|-------------|--------|
| `vllm:prompt_throughput_toks_per_s` | Gauge | Prompt throughput in tokens/s | model_name, engine |
| `vllm:generation_throughput_toks_per_s` | Gauge | Generation throughput in tokens/s | model_name, engine |

## Implementation Status

### ✅ Completed

1. **Metric Definitions** (`vllm/v1/metrics/loggers.py`)
   - Added all Prometheus metric definitions to `PrometheusStatLogger.__init__()`
   - Added recording methods:
     - `record_eplb_stats(avg_tokens, max_tokens, balancedness, layer, model_name, engine_idx)`
     - `record_eplb_rearrangement(duration, engine_idx)`
     - `record_dbo_state(active, phase, engine_idx)`
     - `record_dbo_fallout(reason, engine_idx)`
     - `record_ubatch_size(token_count, ubatch_index, engine_idx)`
   - Throughput tracking integrated into `record()` method (10s update interval)

2. **Stats Dataclasses** (`vllm/v1/metrics/stats.py`)
   ```python
   @dataclass
   class EplbStats:
       avg_tokens_per_rank: float = 0.0
       max_tokens_per_rank: float = 0.0
       balancedness: float = 0.0
       layer: int = 0
       model_name: str = ""
       rearrangement_duration: float = 0.0
       is_rebalancing: bool = False
       per_expert_loads: list[float] | None = None  # Debug mode only

   @dataclass
   class DboStats:
       prefill_active: bool = False
       decode_active: bool = False
       first_ubatch_tokens: int = 0
       second_ubatch_tokens: int = 0
       fallout_reason: str = ""
   ```
   - Added to `SchedulerStats` as optional fields

3. **Stats Pipeline Wiring** (`vllm/v1/metrics/loggers.py`)
   - Updated `PrometheusStatLogger.record()` to consume `eplb_stats` and `dbo_stats` from `SchedulerStats`
   - Calls appropriate recording methods when stats are present

4. **EPLB Instrumentation** (`vllm/distributed/eplb/eplb_state.py`)
   - Added `latest_balancedness_metrics` dict to store per-model metrics
   - Added `latest_per_expert_loads` for debug mode per-expert tracking
   - Added `last_rearrangement_duration` to track rearrangement time
   - Added `is_rebalancing` flag to track rebalancing state
   - Modified `step()` to populate balancedness metrics when `log_stats=True`
   - Modified `rearrange()` to set/clear `is_rebalancing` flag and track duration

5. **EPLB Metrics Wiring** (`vllm/v1/worker/gpu_model_runner.py`)
   - Wired EPLB metrics from `eplb_state` to `EplbStats` dataclass
   - Populated stats with balancedness metrics, rebalancing state, and per-expert loads
   - Attached `EplbStats` to `SchedulerStats` for flow to Prometheus logger

6. **DBO Instrumentation** (`vllm/v1/worker/dp_utils.py`)
   - Modified `_post_process_ubatch()` to return fallout reasons:
     - `"coordination_failure"` when DP ranks disagree on ubatching
     - `"empty_second_ubatch"` when second ubatch would be empty
   - Modified `coordinate_batch_across_dp()` to create and return `DboStats`
   - Tracked `prefill_active`/`decode_active` based on ubatch decision and phase
   - Wired `DboStats` through return values to flow to `SchedulerStats`

7. **Debug Mode Configuration** (`vllm/config/parallel.py`)
   - Added `debug_per_expert_metrics` flag to `EPLBConfig` for opt-in high-cardinality metrics
   - Added `debug_metrics_duration` for auto-disable after timeout (recommended: 300 seconds)
   - Implemented auto-disable logic in `PrometheusStatLogger` based on elapsed time

## Testing

1. **Enable metrics:**
   ```yaml
   --disable-log-stats=false
   --enable-eplb
   --eplb-log-balancedness=true
   --enable-dbo
   ```

2. **Check Prometheus endpoint:** `/metrics`

3. **Expected metrics for WideEP:**
   ```
   vllm:eplb_balancedness_ratio{model_name="deepseek-ai/DeepSeek-R1-0528",engine="0",layer="0"} 0.75
   vllm:eplb_max_tokens_per_rank{model_name="deepseek-ai/DeepSeek-R1-0528",engine="0",layer="0"} 1200
   vllm:eplb_avg_tokens_per_rank{model_name="deepseek-ai/DeepSeek-R1-0528",engine="0",layer="0"} 900
   vllm:eplb_rebalancing{model_name="deepseek-ai/DeepSeek-R1-0528",engine="0"} 0
   ```

4. **Expected metrics for DBO:**
   ```
   vllm:dbo_active{model_name="...",engine="0",phase="decode"} 1
   vllm:dbo_fallout_total{model_name="...",engine="0",reason="empty_second_ubatch"} 5
   vllm:ubatch_token_count_bucket{model_name="...",engine="0",ubatch_index="first",le="256"} 100
   ```

## Grafana Dashboard Ideas

```
Panel 1: EPLB Balancedness Over Time
Query: vllm:eplb_balancedness_ratio

Panel 2: Expert Load Distribution
Query: vllm:eplb_max_tokens_per_rank - vllm:eplb_avg_tokens_per_rank (imbalance)

Panel 3: EPLB Rebalancing Activity
Query: vllm:eplb_rebalancing
Note: Shows when instance is performing expert rearrangement (1=rebalancing)

Panel 4: DBO Engagement Rate
Query: rate(vllm:dbo_fallout_total[5m]) by (reason)

Panel 5: Throughput Variance
Query: stddev_over_time(vllm:generation_throughput_toks_per_s[1m])

Panel 6: Ubatch Size Distribution
Query: histogram_quantile(0.95, vllm:ubatch_token_count_bucket)

Panel 7: Per-Expert Loads (Debug Mode Only)
Query: vllm:expert_load_per_expert_tokens_DEBUG{layer="0"}
Warning: Only enable when debug_per_expert_metrics=True
```

## Per-Expert Debug Metrics

### Configuration

Enable high-cardinality per-expert metrics for temporary debugging:

```bash
vllm serve model \
  --enable-eplb \
  --eplb-log-balancedness \
  --eplb-debug-per-expert-metrics \
  --eplb-debug-metrics-duration=300  # Auto-disable after 5 minutes
```

### Cardinality Warning

⚠️ **HIGH CARDINALITY**: Per-expert metrics create one time series per (layer, expert) combination.

For DeepSeek-R1 scale (60 layers × 256 experts):
- **Per-expert metrics**: ~15,360 time series
- **Aggregated metrics**: ~180 time series
- **Ratio**: 87x more expensive

### Usage Guidelines

1. **Temporary debugging only** - Use `--eplb-debug-metrics-duration` to auto-disable
2. **Short time windows** - Recommended: 300 seconds (5 minutes)
3. **Monitor Prometheus** - Watch for performance degradation
4. **Production**: Use aggregated metrics (`eplb_avg_tokens_per_rank`, `eplb_max_tokens_per_rank`)

### Example Queries

```promql
# Identify hottest expert
topk(5, vllm:expert_load_per_expert_tokens_DEBUG{layer="0"})

# Expert load variance
stddev(vllm:expert_load_per_expert_tokens_DEBUG{layer="0"})

# Compare expert 42 vs average
vllm:expert_load_per_expert_tokens_DEBUG{expert_id="42"}
  / on(model_name, engine)
  vllm:eplb_avg_tokens_per_rank
```

## Notes

- Throughput gauges update every 10 seconds in `PrometheusStatLogger.record()`
- EPLB metrics are only populated when `log_balancedness=True` in EPLB config
- DBO metrics require `enable_dbo=True` in parallel config
- All metrics use the `model_name` and `engine` labels for correlation
- `vllm:eplb_rebalancing` indicates when expert rearrangement is in progress (useful for external load balancers)

## Related Issues

- Throughput variance in WideEP deployments (from team discussion)
- Need for EPLB observability in production
- Debugging empty second ubatch problem at the 256 token threshold
