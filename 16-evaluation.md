# Evaluation: Measuring Quality

**How to measure if your CrewAI system actually works**

---

## Problem with "It Works"

```
"It runs" ≠ "It works well"

Running:
├─ Agent completes task
├─ No errors
└─ Returns output

Working well:
├─ Output is correct
├─ Output is complete
├─ Output is efficient
├─ Output is reliable
├─ Output is cost-effective
└─ Output is safe
```

---

## Evaluation Levels

### Level 1: Output Correctness

```python
# Does the output answer the question?

def evaluate_correctness(output: str, expected: str) -> bool:
    """Check if output matches expected"""
    
    # Exact match
    if output == expected:
        return True
    
    # Keyword matching
    keywords = extract_keywords(expected)
    return all(kw in output for kw in keywords)

# Example
output = "The capital of France is Paris"
expected = "Paris"
evaluate_correctness(output, expected)  # True
```

### Level 2: Output Completeness

```python
# Does the output include all required information?

def evaluate_completeness(output: str, requirements: list[str]) -> float:
    """Score completeness: 0.0-1.0"""
    
    met = sum(1 for req in requirements if req in output)
    return met / len(requirements)

# Example
requirements = ["temperature", "humidity", "wind_speed"]
output = "Temperature is 72°F, humidity 65%"
# Met 2/3 = 0.67 score
```

### Level 3: Output Format

```python
# Is output in correct format?

def evaluate_format(output: str, expected_format: str) -> bool:
    """Check format compliance"""
    
    if expected_format == "json":
        try:
            json.loads(output)
            return True
        except:
            return False
    
    elif expected_format == "markdown":
        return "#" in output or "##" in output
    
    elif expected_format == "csv":
        return "," in output

# Example
evaluate_format("""
# Research Findings
- Finding 1
- Finding 2
""", "markdown")  # True
```

### Level 4: Efficiency

```python
# How efficiently did the system produce output?

def evaluate_efficiency(result) -> dict:
    """Measure efficiency metrics"""
    
    tokens_per_task = result.token_usage["total_tokens"] / len(result.tasks_output)
    iterations_per_task = total_iterations / len(result.tasks_output)
    tool_calls_per_task = total_tool_calls / len(result.tasks_output)
    
    return {
        "tokens_per_task": tokens_per_task,
        "iterations_per_task": iterations_per_task,
        "tool_calls_per_task": tool_calls_per_task,
        "efficiency_score": efficiency_score(tokens_per_task, iterations_per_task)
    }
```

### Level 5: Reliability

```python
# Does it work consistently?

def evaluate_reliability(results: list) -> dict:
    """Measure consistency"""
    
    total = len(results)
    successful = sum(1 for r in results if r["success"])
    correct = sum(1 for r in results if r["correct"])
    
    return {
        "success_rate": successful / total,
        "correctness_rate": correct / total,
        "consistency": consistency_score(results)
    }

# Run 100 times, measure consistency
```

---

## Metrics Framework

### Quantitative Metrics

```python
class EvaluationMetrics:
    def __init__(self):
        self.metrics = {}
    
    def record(self, run_id: str, metrics: dict):
        """Record metrics for a run"""
        self.metrics[run_id] = {
            "success": bool,           # Task completed?
            "correctness": 0.0-1.0,    # Is output correct?
            "completeness": 0.0-1.0,   # All info included?
            "latency_ms": int,         # How long?
            "tokens_used": int,        # How expensive?
            "cost": float,             # $cost
            "retries": int,            # How many retries?
            "tool_calls": int          # How many tools called?
        }
    
    def get_average(self, metric_name: str) -> float:
        """Average across all runs"""
        values = [m[metric_name] for m in self.metrics.values() if metric_name in m]
        return sum(values) / len(values) if values else 0
```

### Qualitative Metrics

```python
# User satisfaction
# Trustworthiness of output
# Helpfulness of response
# Appropriateness for use case

def get_user_feedback(output: str) -> dict:
    """Survey user satisfaction"""
    
    questions = {
        "Is the output helpful?": 1-5,
        "How trustworthy is the output?": 1-5,
        "Would you use this again?": yes/no,
        "Any concerns or issues?": free_text
    }
    
    return questions
```

---

## Golden Dataset

```python
# Create set of known good inputs/outputs

GOLDEN_DATASET = [
    {
        "input": {"topic": "climate change", "depth": "detailed"},
        "expected_output": {
            "summary": "Climate change is...",
            "key_facts": ["Fact 1", "Fact 2", "Fact 3"],
            "confidence": 0.95
        },
        "requirements": [
            "mentions greenhouse gases",
            "mentions human impact",
            "mentions solutions"
        ]
    },
    {
        "input": {"topic": "quantum computing", "depth": "simple"},
        "expected_output": {
            "summary": "Quantum computers...",
            "key_facts": ["Fact 1", "Fact 2"],
            "confidence": 0.80
        },
        "requirements": [
            "mentions qubits",
            "mentions superposition"
        ]
    }
]

def run_golden_test() -> dict:
    """Test against golden dataset"""
    
    results = {}
    for test in GOLDEN_DATASET:
        output = crew.kickoff(inputs=test["input"])
        
        results[test["input"]["topic"]] = {
            "correctness": evaluate_correctness(output.raw, test["expected_output"]),
            "completeness": evaluate_completeness(output.raw, test["requirements"]),
            "format": evaluate_format(output.raw, "markdown")
        }
    
    return results
```

---

## LLM-as-Judge

```python
from anthropic import Anthropic

def llm_evaluate(output: str, criteria: str, reference: str = None) -> dict:
    """Use LLM to evaluate output"""
    
    client = Anthropic()
    
    prompt = f"""
    Evaluate this output based on the criteria.
    
    Output: {output}
    
    Criteria: {criteria}
    
    {f"Reference/Expected: {reference}" if reference else ""}
    
    Provide:
    1. Score (0-100)
    2. Reasoning
    3. Strengths
    4. Weaknesses
    5. Suggestions
    """
    
    response = client.messages.create(
        model="claude-opus",
        max_tokens=1000,
        messages=[{"role": "user", "content": prompt}]
    )
    
    return {
        "evaluation": response.content[0].text,
        "score": extract_score(response.content[0].text)
    }

# Usage
eval_result = llm_evaluate(
    output="The capital of France is Paris",
    criteria="Accuracy and completeness",
    reference="Correctly identify capital of France"
)
```

---

## Regression Testing

```python
# Ensure new versions don't break existing functionality

class RegressionTest:
    def __init__(self, baseline_version: str):
        self.baseline = load_results(baseline_version)
        self.current = None
    
    def run_test(self, current_version: str) -> dict:
        """Compare current vs baseline"""
        
        self.current = run_evaluation(current_version)
        
        regressions = []
        improvements = []
        
        for test_case in self.baseline:
            baseline_score = self.baseline[test_case]["score"]
            current_score = self.current[test_case]["score"]
            
            if current_score < baseline_score * 0.95:  # >5% drop
                regressions.append({
                    "test": test_case,
                    "baseline": baseline_score,
                    "current": current_score,
                    "change": current_score - baseline_score
                })
            elif current_score > baseline_score:
                improvements.append({
                    "test": test_case,
                    "baseline": baseline_score,
                    "current": current_score,
                    "change": current_score - baseline_score
                })
        
        return {
            "regressions": regressions,
            "improvements": improvements,
            "regression_count": len(regressions),
            "improvement_count": len(improvements),
            "safe_to_deploy": len(regressions) == 0
        }
```

---

## Cost-Quality Trade-offs

```python
# Plot quality vs cost to find optimal point

import matplotlib.pyplot as plt

def analyze_trade_offs(experiments: list[dict]):
    """Find optimal cost/quality balance"""
    
    costs = [exp["cost"] for exp in experiments]
    qualities = [exp["quality_score"] for exp in experiments]
    
    # Find Pareto frontier (no better point exists)
    pareto = []
    for i, (cost, quality) in enumerate(zip(costs, qualities)):
        dominated = False
        for j, (other_cost, other_quality) in enumerate(zip(costs, qualities)):
            if other_cost < cost and other_quality > quality:
                dominated = True
                break
        if not dominated:
            pareto.append((cost, quality, experiments[i]["config"]))
    
    # Plot
    plt.scatter(costs, qualities)
    pareto_costs = [p[0] for p in pareto]
    pareto_qualities = [p[1] for p in pareto]
    plt.plot(pareto_costs, pareto_qualities, 'r-', label="Pareto frontier")
    plt.xlabel("Cost ($)")
    plt.ylabel("Quality Score (0-100)")
    plt.legend()
    plt.show()
    
    return pareto

# Example
experiments = [
    {"cost": 1.0, "quality_score": 60, "config": "gpt-3.5, max_iter=5"},
    {"cost": 2.0, "quality_score": 75, "config": "gpt-3.5, max_iter=10"},
    {"cost": 5.0, "quality_score": 85, "config": "gpt-4, max_iter=10"},
    {"cost": 10.0, "quality_score": 92, "config": "gpt-4, max_iter=20"}
]

optimal = analyze_trade_offs(experiments)
# Shows which configs are on Pareto frontier
```

---

## Evaluation Dashboard

```python
def create_evaluation_dashboard(results: dict) -> str:
    """Generate evaluation report"""
    
    report = f"""
    # CrewAI Evaluation Report
    
    ## Summary
    - Success Rate: {results['success_rate']*100:.1f}%
    - Correctness: {results['correctness_rate']*100:.1f}%
    - Avg Latency: {results['avg_latency_ms']:.0f}ms
    - Avg Cost: ${results['avg_cost']:.2f}
    
    ## Quality Metrics
    - Completeness: {results['completeness']:.2f}
    - Format Compliance: {results['format_compliance']*100:.1f}%
    - User Satisfaction: {results['user_satisfaction']:.1f}/5.0
    
    ## Efficiency Metrics
    - Tokens per Task: {results['tokens_per_task']:.0f}
    - Tool Calls per Task: {results['tool_calls_per_task']:.1f}
    - Iterations per Task: {results['iterations_per_task']:.1f}
    
    ## Regressions
    {format_regressions(results['regressions'])}
    
    ## Recommendations
    {generate_recommendations(results)}
    """
    
    return report
```

---

## Evaluation Checklist

✅ **DO:**
- Test with golden dataset (known good inputs/outputs)
- Measure correctness (does it answer the question?)
- Measure completeness (includes all required info?)
- Measure efficiency (tokens, latency, cost)
- Measure reliability (consistent across runs?)
- Use LLM-as-judge for subjective criteria
- Track regressions (new version better/worse?)
- Test cost-quality trade-offs
- Monitor real-world performance
- Iterate based on findings

❌ **DON'T:**
- Assume "it runs" = "it works well"
- Only test happy path
- Ignore edge cases
- Skip regression testing
- Trust single evaluation metric
- Deploy without evaluation
- Ignore user feedback
- Forget to measure efficiency
- Only evaluate final output (trace matters)

---

**Next**: [Comparisons](./17-autogen-comparison.md)
