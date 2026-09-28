# Observability: Monitoring & Metrics

**See what's happening in your CrewAI system**

---

## Three Pillars

```
Logging (What happened?)
├─ Events: task started, agent thinking, tool called, guardrail failed
├─ Data: timestamps, durations, inputs, outputs
└─ Use: Debugging, audit trail, compliance

Metrics (How much/how long?)
├─ Counts: tasks completed, failures, retries
├─ Timing: agent latency, crew duration, tool latency
├─ Resources: tokens used, cost, memory
└─ Use: Performance tracking, alerting, trending

Tracing (How did one request flow?)
├─ Request flow: crew → task → agent → tool → result
├─ Dependencies: which components called what
├─ Latencies: where is time spent?
└─ Use: Performance analysis, debugging complex workflows
```

---

## Logging

### Structured Logging

```python
import json
import logging
from datetime import datetime

logging.basicConfig(
    format='%(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

def log_structured(event_type: str, **data):
    """Log as JSON for easy parsing"""
    entry = {
        "timestamp": datetime.now().isoformat(),
        "event": event_type,
        **data
    }
    logger.info(json.dumps(entry))

# Usage
log_structured("crew_start", crew_id="research_crew", agent_count=3)
log_structured("task_complete", 
    task_id="research",
    duration_ms=1250,
    success=True,
    tokens_used=3500
)
log_structured("agent_iteration",
    agent_id="researcher",
    iteration=1,
    tool_called="web_search",
    tool_duration_ms=450
)
```

### Log Levels

```python
logger.DEBUG("Detailed info for developers")     # Development
logger.INFO("Important events")                  # Normal operations
logger.WARNING("Something unexpected")           # Potential issues
logger.ERROR("Something failed")                 # Actual problems
logger.CRITICAL("System failure")                # Must alert
```

### What to Log

```python
# Agent level
log_structured("agent_start", 
    agent_id=agent.id,
    role=agent.role,
    tools_available=len(agent.tools)
)

# Task level
log_structured("task_start",
    task_id=task.id,
    description=task.description
)
log_structured("task_complete",
    task_id=task.id,
    duration_ms=elapsed,
    output_length=len(output)
)

# Tool level
log_structured("tool_called",
    tool_name=tool.name,
    parameters=params,
    duration_ms=elapsed,
    success=True
)

# Crew level
log_structured("crew_complete",
    crew_id=crew.id,
    total_duration_ms=elapsed,
    total_tokens=tokens,
    total_cost=cost,
    tasks_completed=len(tasks)
)

# Error level
log_structured("error",
    error_type=type(error).__name__,
    error_message=str(error),
    component="agent",
    component_id=agent.id,
    recovery="retry"
)
```

---

## Metrics

### Core Metrics

```python
class CrewMetrics:
    def __init__(self):
        self.tasks_started = 0
        self.tasks_completed = 0
        self.tasks_failed = 0
        self.tool_calls_total = 0
        self.tool_calls_failed = 0
        self.total_tokens = 0
        self.total_cost = 0
        self.latencies = []
    
    def record_task_start(self):
        self.tasks_started += 1
    
    def record_task_success(self, duration_ms: float, tokens: int, cost: float):
        self.tasks_completed += 1
        self.total_tokens += tokens
        self.total_cost += cost
        self.latencies.append(duration_ms)
    
    def record_task_failure(self):
        self.tasks_failed += 1
    
    def record_tool_call(self, success: bool):
        self.tool_calls_total += 1
        if not success:
            self.tool_calls_failed += 1
    
    def get_stats(self) -> dict:
        return {
            "tasks_completed": self.tasks_completed,
            "tasks_failed": self.tasks_failed,
            "failure_rate": self.tasks_failed / max(self.tasks_started, 1),
            "tool_call_failure_rate": self.tool_calls_failed / max(self.tool_calls_total, 1),
            "avg_latency_ms": sum(self.latencies) / len(self.latencies) if self.latencies else 0,
            "p99_latency_ms": sorted(self.latencies)[int(0.99 * len(self.latencies))] if len(self.latencies) > 100 else max(self.latencies or [0]),
            "total_tokens": self.total_tokens,
            "total_cost": f"${self.total_cost:.2f}"
        }

# Usage
metrics = CrewMetrics()

# In crew execution
metrics.record_task_start()
try:
    result = crew.kickoff()
    metrics.record_task_success(
        duration_ms=elapsed,
        tokens=result.token_usage["total_tokens"],
        cost=calculate_cost(result.token_usage)
    )
except Exception as e:
    metrics.record_task_failure()

print(metrics.get_stats())
```

### Performance Percentiles

```python
def calculate_percentiles(latencies: list[float]) -> dict:
    """Calculate p50, p95, p99"""
    sorted_latencies = sorted(latencies)
    n = len(sorted_latencies)
    
    return {
        "p50": sorted_latencies[int(0.50 * n)] if n > 0 else 0,
        "p95": sorted_latencies[int(0.95 * n)] if n > 0 else 0,
        "p99": sorted_latencies[int(0.99 * n)] if n > 0 else 0,
        "max": max(latencies) if latencies else 0
    }

# Usage
latencies = [100, 150, 200, 250, 300, ...]  # ms
perf = calculate_percentiles(latencies)
print(f"p50: {perf['p50']}ms, p95: {perf['p95']}ms, p99: {perf['p99']}ms")
```

---

## Token & Cost Tracking

### Per-Task Tracking

```python
def track_cost(result):
    """Track tokens and cost"""
    tokens = result.token_usage
    
    # Pricing (example: Claude 3.5 Sonnet)
    INPUT_PRICE = 3 / 1_000_000     # $3 per 1M input tokens
    OUTPUT_PRICE = 15 / 1_000_000   # $15 per 1M output tokens
    
    input_cost = tokens["input_tokens"] * INPUT_PRICE
    output_cost = tokens["output_tokens"] * OUTPUT_PRICE
    total_cost = input_cost + output_cost
    
    return {
        "input_tokens": tokens["input_tokens"],
        "output_tokens": tokens["output_tokens"],
        "total_tokens": tokens["total_tokens"],
        "input_cost": f"${input_cost:.4f}",
        "output_cost": f"${output_cost:.4f}",
        "total_cost": f"${total_cost:.4f}"
    }

result = crew.kickoff()
cost_breakdown = track_cost(result)
print(f"Total cost: {cost_breakdown['total_cost']}")
```

### Cost Optimization

```python
def optimize_for_cost(crew):
    """Suggestions to reduce cost"""
    results = []
    
    # Check token usage
    if total_tokens > 100_000:
        results.append("High token usage - consider shorter prompts")
    
    # Check retry rate
    if retry_count / task_count > 0.3:
        results.append("High retry rate - improve task clarity")
    
    # Check model choice
    if using_gpt4:
        results.append("Consider gpt-3.5 for simpler tasks")
    
    # Check agent count
    if agent_count > 5:
        results.append("Many agents - consolidate roles?")
    
    return results
```

---

## Tracing

### Trace a Request

```python
import uuid
from contextlib import contextmanager

class RequestTracer:
    def __init__(self):
        self.traces = {}
    
    @contextmanager
    def trace_span(self, span_name: str, trace_id: str = None):
        """Trace a span (unit of work)"""
        if trace_id is None:
            trace_id = str(uuid.uuid4())
        
        start_time = datetime.now()
        
        try:
            yield trace_id
        except Exception as e:
            log_structured("span_error", 
                span_name=span_name,
                trace_id=trace_id,
                error=str(e)
            )
            raise
        finally:
            duration = (datetime.now() - start_time).total_seconds()
            log_structured("span_complete",
                span_name=span_name,
                trace_id=trace_id,
                duration_ms=duration * 1000
            )

# Usage
tracer = RequestTracer()

with tracer.trace_span("crew_execution") as trace_id:
    
    with tracer.trace_span("task_1", trace_id):
        result1 = task1.execute()
    
    with tracer.trace_span("task_2", trace_id):
        result2 = task2.execute()
    
    with tracer.trace_span("synthesis", trace_id):
        final = combine(result1, result2)
```

---

## Dashboards

### Simple CSV Export

```python
import csv
from datetime import datetime

def export_metrics(metrics_list: list[dict], filename: str):
    """Export metrics to CSV for analysis"""
    
    with open(filename, 'w', newline='') as f:
        writer = csv.DictWriter(f, fieldnames=metrics_list[0].keys())
        writer.writeheader()
        writer.writerows(metrics_list)

# Usage
metrics_history = []
for i in range(100):
    result = crew.kickoff()
    metrics_history.append({
        "timestamp": datetime.now().isoformat(),
        "duration_ms": result_duration,
        "tokens": result.token_usage["total_tokens"],
        "cost": calculate_cost(result.token_usage),
        "success": len(result.raw) > 0
    })

export_metrics(metrics_history, "metrics.csv")
# Open in Excel/Sheets to visualize trends
```

### Prometheus Integration

```python
from prometheus_client import Counter, Histogram, Gauge

# Define metrics
tasks_completed = Counter('crewaí_tasks_completed', 'Tasks completed')
task_duration = Histogram('crewai_task_duration_seconds', 'Task duration')
task_failures = Counter('crewai_task_failures', 'Task failures')
tokens_used = Gauge('crewai_tokens_used', 'Tokens used')

# Record metrics
tasks_completed.inc()
task_duration.observe(elapsed_seconds)
tokens_used.set(result.token_usage["total_tokens"])

# Prometheus scrapes these automatically
# Alert rules can trigger on these metrics
```

---

## Alerting

### Alert Conditions

```python
class AlertManager:
    def __init__(self):
        self.alerts = []
    
    def check_alerts(self, metrics: dict):
        """Check if any alert conditions are met"""
        
        # High failure rate
        if metrics["failure_rate"] > 0.1:
            self.alert("High task failure rate: {:.1%}".format(
                metrics["failure_rate"]
            ))
        
        # High latency
        if metrics["p99_latency_ms"] > 10_000:
            self.alert("Slow p99 latency: {:.0f}ms".format(
                metrics["p99_latency_ms"]
            ))
        
        # High cost
        if metrics["total_cost"] > 100:
            self.alert("High cost: ${}".format(metrics["total_cost"]))
        
        # Tool failures
        if metrics["tool_call_failure_rate"] > 0.2:
            self.alert("High tool failure rate: {:.1%}".format(
                metrics["tool_call_failure_rate"]
            ))
    
    def alert(self, message: str):
        """Send alert"""
        log_structured("alert", message=message)
        # Could also send email, Slack, PagerDuty, etc.
```

---

## Observability Checklist

✅ **DO:**
- Log all important events (structured JSON)
- Track key metrics (success rate, latency, cost)
- Monitor p99 latency (not just average)
- Alert on anomalies (not on every event)
- Keep audit trail (for compliance)
- Track token usage (for cost control)
- Correlate traces (request flow)
- Archive logs (for historical analysis)

❌ **DON'T:**
- Log in free-form text (hard to parse)
- Only track averages (outliers hidden)
- Alert on every metric (alert fatigue)
- Store sensitive data in logs (PII)
- Forget about retention (logs grow forever)
- Ignore cost tracking (surprises later)
- Mix related requests (hard to trace)
- Discard old metrics (no trending)

---

**Next**: [Scenario Questions](./20-scenario-questions.md)
