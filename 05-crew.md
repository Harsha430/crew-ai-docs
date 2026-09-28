# Crew: Complete Reference

## What is a Crew?

A Crew is:
- **Orchestration** of agents on coordinated tasks
- Provides shared resources (memory, callbacks)
- Executes tasks in a specified process (sequential or hierarchical)
- Returns CrewOutput with all results

**Key mental model**: Crew = Team

```
Crew
├─ Agents: [Researcher, Writer, Reviewer]
├─ Tasks: [Research, Write, Review]
├─ Process: Sequential (Research → Write → Review)
├─ Memory: Shared across all agents
└─ Returns: CrewOutput with all results
```

## Crew Parameters (v1.15.22)

### Required

```python
from crewai import Crew, Agent, Task, Process

crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task]
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `agents` | `list[Agent]` | **required** | Agents who will work |
| `tasks` | `list[Task]` | **required** | Tasks to accomplish |

### Process

```python
from crewai import Process

# Sequential: Task1 → Task2 → Task3
crew = Crew(
    agents=[...],
    tasks=[...],
    process=Process.sequential
)

# Hierarchical: Manager decides order
crew = Crew(
    agents=[...],
    tasks=[...],
    process=Process.hierarchical,
    manager_agent=manager,  # Required for hierarchical
    manager_llm=custom_llm   # Optional
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `process` | `Process` | `Process.sequential` | How to orchestrate tasks |
| `manager_agent` | `Agent` | Required if hierarchical | Agent who decides task order |
| `manager_llm` | `LLM` | Default | LLM for manager (if different from agents) |

**When to use each**:
- **Sequential**: Clear task order, predictable, easier to debug
- **Hierarchical**: Dynamic task assignment, manager decides, less predictable

### Memory & Observability

```python
crew = Crew(
    agents=[...],
    tasks=[...],
    memory=True,  # Agents remember past interactions
    verbose=True,  # Print thinking/actions
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `memory` | `bool` | `True` | Should agents share memory? |
| `verbose` | `bool` | `False` | Print execution details? |

**Memory behavior**:
- `True`: All agents can access shared memory, retrieve relevant context
- `False`: No memory, each task independent

### Callbacks

```python
def on_task_completed(output):
    print(f"Task done: {output.description}")

def on_crew_completed(output):
    print(f"Crew done! Tokens: {output.token_usage['total_tokens']}")

crew = Crew(
    agents=[...],
    tasks=[...],
    callbacks=[on_task_completed],  # Task-level callbacks
    # crew-level callbacks in later versions
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `callback` | `Callable` | `None` | Function called at task events |

### Advanced Configuration

```python
crew = Crew(
    agents=[...],
    tasks=[...],
    planning=True,  # Manager creates plan (hierarchical)
    knowledge="./knowledge",  # Path to knowledge base
    context_window=None  # Max context size
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `planning` | `bool` | `False` | Should manager create a plan first? |
| `knowledge` | `str` | `None` | Path to knowledge files for agents |

## Crew Execution

### Sequential Execution

```
crew.kickoff()
    ↓
Task 1
├─ Agent 1 executes
├─ Produces TaskOutput
└─ Saved to results
    ↓
Task 2 (gets Task 1 output as context)
├─ Agent 2 executes
├─ Produces TaskOutput
└─ Saved to results
    ↓
Task 3
└─ ... same pattern
    ↓
CrewOutput
├─ raw: Final answer
├─ tasks_output: [TaskOutput1, TaskOutput2, TaskOutput3]
└─ token_usage: Total tokens
```

### Hierarchical Execution

```
crew.kickoff()
    ↓
Manager Agent receives all tasks
├─ Analyzes all tasks
├─ Creates plan (if planning=True)
├─ Decides: "Agent A should do Task 1"
├─ Delegates to Agent A
│   └─ Agent A produces output
├─ Analyzes result
├─ Decides: "Agent B should do Task 2"
├─ Delegates to Agent B
│   └─ Agent B produces output
├─ ... (repeats for all tasks)
├─ Synthesizes final answer
└─ Returns
    ↓
CrewOutput
```

## Crew Inputs

```python
crew = Crew(agents=[...], tasks=[...])

# Static execution
result = crew.kickoff()

# With inputs (for variable substitution)
result = crew.kickoff(inputs={"topic": "AI trends", "year": 2026})
```

**Inputs usage**: If task description has `{topic}` or `{year}`, they get substituted:

```python
task = Task(
    description="Research {topic} for {year}",
    expected_output="Report on {topic}",
    agent=agent
)

result = crew.kickoff(inputs={"topic": "AI", "year": 2026})
# Description becomes: "Research AI for 2026"
```

## Crew Return Value

```python
result = crew.kickoff()

# type(result) = CrewOutput
print(result.raw)  # Final text answer
print(result.json_dict)  # If JSON-formatted
print(result.pydantic)  # If structured
print(result.tasks_output)  # List of task outputs
print(result.token_usage)  # Token stats
```

### CrewOutput Structure

```
CrewOutput
├─ raw: str (always)
│   └─ Final answer as text
├─ json_dict: dict | None
│   └─ Parsed JSON (if output is JSON)
├─ pydantic: BaseModel | None
│   └─ Typed object (if structured output)
├─ tasks_output: list[TaskOutput]
│   ├─ TaskOutput for each task
│   └─ Contains: description, raw, pydantic, json_dict, token_usage
└─ token_usage: dict
    ├─ "input_tokens": int
    ├─ "output_tokens": int
    └─ "total_tokens": int
```

## Common Crew Patterns

### Pattern 1: Simple Sequential Crew

```python
researcher = Agent(role="Researcher", goal="Find info", tools=[web_search])
writer = Agent(role="Writer", goal="Write clearly", tools=[])

research_task = Task(
    description="Research AI trends",
    expected_output="List of trends",
    agent=researcher
)

write_task = Task(
    description="Write article",
    expected_output="Markdown article",
    agent=writer,
    context=[research_task]  # ← Use research output
)

crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    process=Process.sequential,
    memory=True
)

result = crew.kickoff()
print(result.raw)  # Final article
```

**When**: Clear task order, each builds on previous.

### Pattern 2: Hierarchical Crew with Manager

```python
manager = Agent(
    role="Project Manager",
    goal="Coordinate team",
    allow_delegation=True
)

specialist1 = Agent(role="Specialist 1", goal="...", tools=[...])
specialist2 = Agent(role="Specialist 2", goal="...", tools=[...])

task1 = Task(description="Do X", agent=specialist1)
task2 = Task(description="Do Y", agent=specialist2)

crew = Crew(
    agents=[manager, specialist1, specialist2],
    tasks=[task1, task2],
    process=Process.hierarchical,
    manager_agent=manager,
    planning=True  # Manager creates plan first
)

result = crew.kickoff()
```

**When**: Dynamic task assignment, flexible ordering.

### Pattern 3: Multi-Step Analysis

```python
extractor = Agent(role="Extractor", tools=[pdf_reader])
analyzer = Agent(role="Analyzer", tools=[data_tools])
reviewer = Agent(role="Reviewer", tools=[])

extract_task = Task(description="Extract data", agent=extractor)
analyze_task = Task(
    description="Analyze extracted data",
    agent=analyzer,
    context=[extract_task]
)
review_task = Task(
    description="Review analysis",
    agent=reviewer,
    context=[analyze_task],
    guardrail=quality_check
)

crew = Crew(
    agents=[extractor, analyzer, reviewer],
    tasks=[extract_task, analyze_task, review_task],
    process=Process.sequential
)

result = crew.kickoff()
```

**When**: Quality gates at each step, progressive refinement.

### Pattern 4: Structured Output Crew

```python
from pydantic import BaseModel

class ResearchFindings(BaseModel):
    topic: str
    findings: list[str]
    confidence: float

task = Task(
    description="Research and structure findings",
    expected_output="Structured research output",
    agent=agent,
    output_pydantic=ResearchFindings
)

crew = Crew(agents=[agent], tasks=[task])
result = crew.kickoff()

# Get typed output
findings: ResearchFindings = result.pydantic
print(findings.topic)
print(findings.findings)
print(findings.confidence)
```

**When**: You need strongly typed, validated output.

## Crew Configuration Best Practices

### Sequential vs Hierarchical Decision

**Use Sequential when:**
- Task order is predetermined and won't change
- Each task clearly depends on previous
- Debugging is important (execution is transparent)
- Cost predictability matters

**Use Hierarchical when:**
- Task order might vary based on complexity
- You want a manager agent to coordinate
- Flexibility is more important than transparency
- Team composition might change

### Memory Consideration

```python
# With memory
crew = Crew(..., memory=True)
# Agents remember findings across tasks
# Larger context (more tokens)
# Better continuity

# Without memory
crew = Crew(..., memory=False)
# Each task is independent
# Smaller context (fewer tokens)
# Less continuity but faster
```

### Verbose for Development

```python
# Development
crew = Crew(..., verbose=True)
# See all thinking, tools, decisions
# Helps debug issues

# Production
crew = Crew(..., verbose=False)
# No debug output
# Saves tokens, cleaner logs
```

## Crew Execution Patterns

### Pattern: Execute Once

```python
result = crew.kickoff()
# Returns CrewOutput
```

### Pattern: Execute Multiple Times

```python
crew = Crew(...)

# Multiple independent runs
result1 = crew.kickoff(inputs={"param": "value1"})
result2 = crew.kickoff(inputs={"param": "value2"})

# Each execution independent (unless memory enabled)
```

### Pattern: Async Execution

```python
# Requires async_execution=True on tasks
import asyncio

async def run_crew():
    result = await crew.kickoff_async()
    return result

# Use with asyncio.run() or in async context
```

## Handling Crew Output

### Extract Values

```python
result = crew.kickoff()

# Get final answer
final_answer = result.raw

# Get all task results
for i, task_result in enumerate(result.tasks_output):
    print(f"Task {i}: {task_result.description}")
    print(f"Output: {task_result.raw}")
    print(f"Tokens: {task_result.token_usage['total_tokens']}")
```

### Check Success

```python
result = crew.kickoff()

# Check if output is empty
if not result.raw or len(result.raw) < 10:
    print("Crew failed to produce output")
else:
    print("Success:", result.raw)

# Check token count
if result.token_usage['total_tokens'] > 50000:
    print("High token usage:", result.token_usage)
```

### Store for Later

```python
import json

result = crew.kickoff()

# Save to file
output = {
    "raw": result.raw,
    "token_usage": result.token_usage,
    "task_count": len(result.tasks_output),
    "success": len(result.raw) > 0
}

with open("crew_result.json", "w") as f:
    json.dump(output, f)
```

## Debugging Crew Issues

### When Crew Hangs

**Problem**: crew.kickoff() never returns.

**Diagnosis**:
1. Agent in infinite loop? Check max_iter
2. Tool making slow external call?
3. Manager deliberating too long?

**Fix**:
```python
# Lower iteration limits
agent = Agent(..., max_iter=5)

# Use timeout (if supported)
result = crew.kickoff(timeout=60)
```

### When Task Order is Wrong

**Problem**: Tasks execute in unexpected order (hierarchical process).

**Diagnosis**:
1. Manager agent confused about task dependencies
2. Manager agent didn't understand tasks

**Fix**:
```python
# Make task descriptions very clear
task1 = Task(
    description="FIRST: Extract data from document",
    agent=extractor
)

task2 = Task(
    description="SECOND: Analyze extracted data from Task 1",
    agent=analyzer,
    context=[task1]  # Explicit dependency
)
```

### When Crew Output is Wrong

**Problem**: Final output doesn't match expectations.

**Diagnosis**:
1. Agent role/goal unclear
2. Task expected_output unclear
3. Guardrail not enforcing quality

**Fix**:
```python
task = Task(
    description="Produce detailed analysis",
    expected_output="Analysis with: summary, key points, recommendations",
    agent=agent,
    guardrail=validate_analysis,
    guardrail_max_retries=2
)
```

### When Multiple Tasks Interfere

**Problem**: Task B output disrupts Task C.

**Diagnosis**:
1. Task B output not suitable for Task C
2. Context injection confused agent

**Fix**:
```python
# Summarize Task B output before passing to Task C
summary_task = Task(
    description="Summarize findings for next team member",
    expected_output="Concise summary",
    agent=summarizer,
    context=[task_b]
)

task_c = Task(
    description="Use summary to do next step",
    agent=agent_c,
    context=[summary_task]  # Cleaner input
)
```

## Crew Best Practices

✅ **DO:**
- Use sequential by default, hierarchical only if needed
- Give agents clear, specialized roles
- Order tasks logically (build on previous results)
- Use context to pass results forward
- Enable memory for multi-task workflows
- Set guardrails for quality gates
- Use verbose=True during development
- Test with reasonable max_iter (10-15)

❌ **DON'T:**
- Use hierarchical without good reason (harder to debug)
- Give conflicting agent roles
- Use too many tasks (overwhelming)
- Forget to add context to dependent tasks
- Leave verbose=True in production
- Set max_iter too high (wastes time/tokens)
- Expect perfect determinism (LLMs vary)

---

**Next**: [Flow](./07-flow.md) for application-level orchestration
