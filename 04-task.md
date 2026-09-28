# Task: Complete Reference

## What is a Task?

A Task is:
- **WHAT** needs to be done (not WHO does it - that's Agent)
- A work specification with description, context, validation
- Assigned to one Agent
- Produces one TaskOutput

**Key mental model**: Task = Assignment

```
Agent = "I'm a researcher with tools and memory"
Task = "Research AI trends for 2026, output markdown, >1000 words"
```

## Task Parameters (v1.15.22)

### Required

```python
from crewai import Task

task = Task(
    description="Research latest AI trends",
    expected_output="Markdown document with key trends",
    agent=researcher_agent
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `description` | `str` | **required** | What work needs to be done |
| `expected_output` | `str` | **required** | What the output should look like |
| `agent` | `Agent` | **required** | Which agent performs this task |

### Context & Dependencies

```python
research_task = Task(...)
write_task = Task(
    description="Write a report",
    expected_output="Final report",
    agent=writer_agent,
    context=[research_task]  # Use research output as input
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `context` | `list[Task]` | `[]` | Previous task outputs to use as context |

**Why**: Allows task chaining - Task 2 reads Task 1 output.

**How it works**:
1. Task 1 completes → produces output
2. Task 1 output → formatted into prompt
3. Task 2 agent receives prompt + Task 1 output in context

### Output Format

```python
from pydantic import BaseModel

class ResearchOutput(BaseModel):
    findings: list[str]
    sources: int

task = Task(
    description="Research",
    expected_output="Findings and source count",
    agent=agent,
    output_json=ResearchOutput,  # Expect JSON, parse to dict
    # OR
    output_pydantic=ResearchOutput  # Parse to Pydantic model
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `output_json` | `Model` | `None` | Tell agent: output JSON matching this schema |
| `output_pydantic` | `Model` | `None` | Tell agent: output JSON, CrewAI parses to Pydantic |

**Difference**:
- `output_json`: Result has `json_dict` (untyped dict)
- `output_pydantic`: Result has `pydantic` (typed object)
- Use `output_pydantic` for strongly typed requirements

### Validation & Retry

```python
def validate_research(output: str) -> bool:
    return len(output) > 500 and "finding" in output.lower()

task = Task(
    description="Research",
    expected_output="Report",
    agent=agent,
    guardrail=validate_research,
    guardrail_max_retries=2  # Retry up to 2 times if validation fails
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `guardrail` | `Callable` | `None` | Function that validates output (returns True/False) |
| `guardrail_max_retries` | `int` | `1` | How many retries if guardrail fails |

**Behavior**:
```
Agent produces output
    ↓
guardrail(output) → True/False
    ├─ True: Task complete
    └─ False: 
       ├─ Retries < max? → Agent retries
       └─ Retries exhausted? → Fail/escalate
```

### Asynchronous Execution

```python
task = Task(
    description="Long task",
    expected_output="Result",
    agent=agent,
    async_execution=True  # Run without waiting
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `async_execution` | `bool` | `False` | Run asynchronously? (for Crew.kickoff_async) |

### Human-in-the-Loop

```python
task = Task(
    description="Important decision",
    expected_output="Decision",
    agent=agent,
    human_input=True  # Pause for human feedback
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `human_input` | `bool` | `False` | Pause task for human feedback? |

**When to use**: Critical decisions, risky actions, high-stakes outputs.

### Callbacks & Observability

```python
def on_task_start(task):
    print(f"Task starting: {task.description}")

def on_task_end(output):
    print(f"Task output: {output.raw[:100]}...")

task = Task(
    description="Do work",
    expected_output="Result",
    agent=agent,
    callback=on_task_start,  # Called when task starts
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `callback` | `Callable` | `None` | Function called at task events |

### Markdown & Documentation

```python
task = Task(
    description="Write documentation",
    expected_output="README.md",
    agent=agent,
    markdown=True  # Output is markdown
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `markdown` | `bool` | `False` | Treat output as markdown? |

### Files & Output

```python
task = Task(
    description="Generate report",
    expected_output="PDF report",
    agent=agent,
    output_file="report.pdf"  # Save output to file
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `output_file` | `str` | `None` | Save output to this file |

## Task Execution

### Sequential Execution (Crew level)

```
Crew.kickoff()
    ↓
Task 1
├─ Agent thinks
├─ Guardrail validates
└─ TaskOutput created
    ↓
Task 2 (receives Task 1 output)
├─ Agent thinks (with context from Task 1)
├─ Guardrail validates
└─ TaskOutput created
    ↓
Task 3
└─ ... (same pattern)
    ↓
CrewOutput returned
└─ Contains all TaskOutputs
```

### Context Injection

```python
research_task = Task(
    description="Research AI trends",
    expected_output="List of trends",
    agent=researcher
)

write_task = Task(
    description="Write report about trends",
    expected_output="Markdown report",
    agent=writer,
    context=[research_task]  # ← This is the key
)
```

**What happens**:
1. Research task completes
2. Output: "AI trends: 1) LLMs 2) Agents 3) Multimodal"
3. Write task agent receives prompt that includes:
   ```
   You are a writer.
   Here are findings from previous research:
   [full research output here]
   
   Now write a report about these trends.
   ```
4. Writer reads research findings and writes report

### Multiple Contexts

```python
research_task = Task(description="Research", agent=researcher)
analysis_task = Task(description="Analyze", agent=analyst)

review_task = Task(
    description="Write summary",
    agent=reviewer,
    context=[research_task, analysis_task]  # Multiple contexts
)
```

**What happens**: Reviewer receives both previous outputs in context.

## Common Task Patterns

### Pattern 1: Simple Task

```python
task = Task(
    description="Summarize this article",
    expected_output="One-paragraph summary",
    agent=summarizer
)
```

**When**: Simple, standalone work.

### Pattern 2: Task with Structured Output

```python
class CompanyAnalysis(BaseModel):
    name: str
    revenue: float
    growth_rate: float

task = Task(
    description="Analyze company financial data",
    expected_output="Structured company analysis",
    agent=analyst,
    output_pydantic=CompanyAnalysis
)
```

**When**: You need typed, validated output.

### Pattern 3: Task with Context

```python
research = Task(description="Research", agent=researcher)
write = Task(
    description="Write article",
    agent=writer,
    context=[research]
)
```

**When**: Task depends on previous task output.

### Pattern 4: Task with Validation

```python
def validate_quality(output: str) -> bool:
    word_count = len(output.split())
    has_sources = "source" in output.lower()
    return word_count > 300 and has_sources

task = Task(
    description="Research topic",
    agent=agent,
    guardrail=validate_quality,
    guardrail_max_retries=2
)
```

**When**: Quality requirements (minimum length, required content, etc.).

### Pattern 5: Human-Approved Task

```python
task = Task(
    description="Propose contract",
    expected_output="Contract",
    agent=lawyer,
    human_input=True  # Pause for approval
)
```

**When**: Risky actions, important decisions.

## Task Return Value

Task doesn't return CrewOutput. Task produces TaskOutput.

```python
from crewai import Task

task = Task(
    description="Do work",
    expected_output="Result",
    agent=agent
)

# To get TaskOutput, must run in Crew
crew = Crew(agents=[agent], tasks=[task])
result = crew.kickoff()

# result is CrewOutput
task_output = result.tasks_output[0]  # This is TaskOutput
```

## Task Debugging

### When Task Context is Empty

**Problem**: Task 2 doesn't have Task 1 output.

**Diagnosis**:
1. Check `context=[task1]` is set
2. Check task1 succeeded (check result.tasks_output[0].raw)
3. Check task1 output is readable (not error message)

**Fix**:
```python
# Debug: Print task 1 output
result = crew.kickoff()
print("Task 1 output:")
print(result.tasks_output[0].raw)

# Then check Task 2 sees it in prompt (enable verbose)
```

### When Guardrail Keeps Failing

**Problem**: Guardrail validation never passes.

**Diagnosis**:
1. Validation function too strict?
2. Agent can't achieve requirement?
3. Guardrail logic wrong?

**Fix**:
```python
# Debug: Print what agent produced
result = crew.kickoff()
output = result.tasks_output[0].raw

# Manual validation
print(f"Length: {len(output)}")
print(f"Has 'key': {'key' in output}")

# Adjust guardrail
def validate(output):
    # Make less strict
    return len(output) > 200  # Was 500
```

### When Task Output Format is Wrong

**Problem**: Output is not JSON when expected.

**Diagnosis**:
1. `output_json` or `output_pydantic` set?
2. Agent understood format requirement?
3. LLM returned valid JSON?

**Fix**:
```python
# Make requirement explicit in description
task = Task(
    description="Research. Output VALID JSON with: {\"findings\": [...]}",
    expected_output="Valid JSON object",
    agent=agent,
    output_pydantic=MyModel
)
```

### When Task Takes Too Long

**Problem**: Task never completes.

**Diagnosis**:
1. Agent stuck in loop? (check max_iter)
2. Tool making slow external call?
3. LLM timeout?

**Fix**:
```python
agent = Agent(
    role="Quick worker",
    goal="Complete quickly",
    max_iter=5  # Lower from 10
)

task = Task(description="Quick task", agent=agent)
```

## Task Best Practices

✅ **DO:**
- Write clear, specific descriptions
- Set reasonable expected_output (tells agent what success looks like)
- Use context to build on previous work
- Set guardrails for quality requirements
- Use output_pydantic for structured needs
- Test tasks with verbose=True

❌ **DON'T:**
- Give conflicting descriptions/expected_output
- Use too many contexts (confuses agent)
- Make guardrails unrealistic
- Expect raw task execution (use Crew)
- Set guardrail_max_retries too high (wastes tokens)

## Task vs Workflow

| Task | Workflow |
|------|----------|
| One agent, one work item | Multiple tasks, multiple agents |
| Synchronous (within Crew) | Can be sync or async (Flow) |
| Produces TaskOutput | Produces workflow result |
| Part of Crew | Task collection + orchestration |

---

**Next**: [Crew](./05-crew.md)
