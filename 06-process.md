# Process: Sequential vs Hierarchical

## What is Process?

**Process = How tasks are orchestrated within a Crew**

```python
from crewai import Crew, Process

crew = Crew(
    agents=[...],
    tasks=[...],
    process=Process.sequential  # or Process.hierarchical
)
```

A Process answers: "In what order do tasks execute? Who decides the order?"

---

## Sequential Process

### How It Works

```
Task 1
  ├─ Agent executes
  └─ Produces output
    ↓
Task 2 (receives Task 1 output in context)
  ├─ Agent executes
  └─ Produces output
    ↓
Task 3 (receives Task 2 output in context)
  ├─ Agent executes
  └─ Produces output
    ↓
Done
```

### Configuration

```python
from crewai import Crew, Process

crew = Crew(
    agents=[researcher, writer, reviewer],
    tasks=[research_task, write_task, review_task],
    process=Process.sequential  # Default
)

result = crew.kickoff()
```

### Characteristics

| Property | Sequential |
|----------|-----------|
| Order | Fixed, predetermined |
| Decider | Task list order |
| Dependencies | Explicit (task1 → task2) |
| Debugging | Easy (clear execution path) |
| Predictability | High (same order always) |
| Flexibility | Low (can't change order) |
| Speed | Limited by sequential nature |
| LLM overhead | Minimal (no manager deliberation) |

### When to Use Sequential

✅ **Good for:**
- Clear task dependencies
- Deterministic workflows
- Research → Write → Review pattern
- Extract → Analyze → Summarize
- Step-by-step processes

❌ **Not ideal for:**
- Dynamic task assignment
- Complex task graphs
- Conditional branching (use Flow instead)

### Example: Document Processing

```python
from crewai import Agent, Task, Crew, Process

# Agents
extractor = Agent(
    role="Document Extractor",
    goal="Extract structured data",
    tools=[pdf_reader]
)

validator = Agent(
    role="Data Validator",
    goal="Validate extracted data",
    tools=[]
)

enricher = Agent(
    role="Data Enricher",
    goal="Add context and links",
    tools=[web_search, db_query]
)

# Tasks
extract_task = Task(
    description="Extract data from PDF",
    expected_output="Structured JSON data",
    agent=extractor,
    output_json=ExtractedData
)

validate_task = Task(
    description="Validate extracted data",
    expected_output="Validation report",
    agent=validator,
    context=[extract_task],  # Gets extract output
    guardrail=validation_passes
)

enrich_task = Task(
    description="Enrich data with context",
    expected_output="Enhanced data",
    agent=enricher,
    context=[validate_task]  # Gets validation report
)

# Crew
crew = Crew(
    agents=[extractor, validator, enricher],
    tasks=[extract_task, validate_task, enrich_task],
    process=Process.sequential  # ← Sequential
)

result = crew.kickoff()
```

**Execution**:
1. Extract → Produces structured data
2. Validate → Reads extraction, validates it
3. Enrich → Reads validation, adds context
4. Return final enhanced data

---

## Hierarchical Process

### How It Works

```
Manager Agent receives all tasks
├─ Analyzes all tasks
├─ Decides task assignment
│
├─ Decision 1: "Specialist A should handle Task 1"
├─ Delegates to Specialist A
│   └─ Specialist A executes, returns result
│
├─ Analyzes result
├─ Decision 2: "Specialist B should handle Task 2"
├─ Delegates to Specialist B
│   └─ Specialist B executes, returns result
│
├─ ... (repeats for all tasks)
│
├─ Synthesizes results
└─ Returns final answer
```

### Configuration

```python
from crewai import Crew, Process

manager = Agent(
    role="Project Manager",
    goal="Coordinate team to solve problem",
    allow_delegation=True
)

specialist1 = Agent(
    role="Specialist 1",
    goal="Specialize in domain 1",
    tools=[...]
)

specialist2 = Agent(
    role="Specialist 2",
    goal="Specialize in domain 2",
    tools=[...]
)

crew = Crew(
    agents=[manager, specialist1, specialist2],
    tasks=[complex_task1, complex_task2],  # Just one or few tasks
    process=Process.hierarchical,  # ← Hierarchical
    manager_agent=manager,  # ← Required
    manager_llm=ChatOpenAI(model="gpt-4"),  # Optional (same as other agents by default)
    planning=True  # Manager creates plan first
)

result = crew.kickoff()
```

### Characteristics

| Property | Hierarchical |
|----------|-------------|
| Order | Dynamic (LLM decides) |
| Decider | Manager Agent |
| Dependencies | Implicit (manager decides) |
| Debugging | Hard (LLM-driven) |
| Predictability | Low (depends on LLM reasoning) |
| Flexibility | High (can change order dynamically) |
| Speed | Slower (manager deliberation overhead) |
| LLM overhead | High (multiple manager calls) |
| Token cost | Higher (manager reasoning) |

### Manager Agent Behavior

```
Input: All tasks to accomplish

Manager thinks:
├─ "Task A is about research, requires tools"
├─ "Task B is about writing, doesn't need tools"
├─ "Task C depends on both A and B"
└─ Decides: "Do A first (complex), then B, then C"

OR

Manager thinks:
├─ "Task A is simple writing"
├─ "Task B is complex analysis"
├─ "These don't depend on each other"
└─ Decides: "Do both in parallel" (in some frameworks)
```

### When to Use Hierarchical

✅ **Good for:**
- Dynamic task assignment
- Complex problem decomposition
- Manager coordinates diverse specialists
- Task order isn't predetermined
- "Manage this team of agents" scenarios

❌ **Not ideal for:**
- Simple linear workflows (overkill)
- When determinism is critical
- When cost must be minimized (expensive)
- Real-time systems (slow)

### Example: Bug Investigation Crew

```python
manager = Agent(
    role="Tech Lead",
    goal="Coordinate team to investigate and fix bug",
    allow_delegation=True
)

logger_specialist = Agent(
    role="Log Analyzer",
    goal="Find relevant log entries",
    tools=[log_query, metric_viewer]
)

code_specialist = Agent(
    role="Code Reviewer",
    goal="Find code issues",
    tools=[repo_search, diff_viewer]
)

test_specialist = Agent(
    role="Test Engineer",
    goal="Create reproduction test",
    tools=[test_runner]
)

# Single task to manager
investigation_task = Task(
    description="""
    A production bug is affecting 5% of users.
    Investigate:
    1. Check logs for errors
    2. Find related code
    3. Create test to reproduce
    4. Provide fix recommendation
    """,
    expected_output="Investigation report with fix recommendation",
    agent=manager
)

crew = Crew(
    agents=[manager, logger_specialist, code_specialist, test_specialist],
    tasks=[investigation_task],
    process=Process.hierarchical,
    manager_agent=manager
)

result = crew.kickoff()

# Manager delegated to specialists as needed:
# → Log Analyzer checked logs
# → Code Reviewer found related code
# → Test Engineer created reproduction
# → Manager synthesized findings
```

---

## Comparison: Sequential vs Hierarchical

### Scenario 1: Research & Writing

**Sequential** (BETTER):
```
Research Agent → produces findings
    ↓
Writer Agent → reads findings, writes article
```

**Why**: Clear order, no need for manager.

**Hierarchical** (OVERKILL):
```
Manager Agent
├─ "Research needed"
├─ Delegates to Researcher
│   └─ Researcher produces findings
├─ "Now writing needed"
├─ Delegates to Writer
│   └─ Writer writes article
├─ Synthesizes
└─ Returns
```

**Why not**: Extra cost, no benefit.

### Scenario 2: Complex Problem Solving

**Hierarchical** (BETTER):
```
Manager Agent receives complex problem
├─ Analyzes: "This needs research, analysis, and writing"
├─ Task decomposition: "Research first, then analysis, then writing"
├─ Delegates to specialists based on expertise
└─ Synthesizes final answer
```

**Why**: Manager intelligently decomposes problem.

**Sequential** (HARDER):
```
Would need to pre-specify all tasks
Risk: Wrong order if we didn't anticipate
```

**Why not**: Less flexible.

### Decision Tree

```
Do you know the exact task order?
├─ YES → Sequential (clearer, cheaper)
└─ NO → Hierarchical (flexible, expensive)

Is the problem well-defined?
├─ YES → Sequential
└─ NO (complex decomposition needed) → Hierarchical

Do you need cost optimization?
├─ YES → Sequential (lower token usage)
└─ NO → Hierarchical is OK

Do you need determinism?
├─ YES → Sequential (always same order)
└─ NO → Hierarchical is OK

Is this a "one-shot" problem?
├─ YES → Hierarchical might be better
└─ NO (reusable workflow) → Sequential better
```

---

## Process Implementation Details

### Sequential Execution Steps

```python
# Internal: What CrewAI does with Sequential

for task in tasks:
    # Get context from previous tasks
    context = []
    for prev_task_idx in range(task_idx):
        prev_output = results[prev_task_idx]
        context.append(prev_output)
    
    # Add context to task
    task.context = context
    
    # Execute task
    agent = task.agent
    output = agent.execute(task)
    
    # Store result
    results.append(output)

return CrewOutput(tasks_output=results)
```

### Hierarchical Execution Steps

```python
# Internal: What CrewAI does with Hierarchical

# Manager plans
manager_prompt = format_all_tasks(tasks)
plan = manager_agent.think(manager_prompt)

# Manager executes plan
for step in plan:
    # Manager delegates
    agent = step.assigned_agent
    task = step.task
    
    # Agent executes
    output = agent.execute(task)
    
    # Store result
    results[task.id] = output

# Manager synthesizes
final_output = manager_agent.synthesize(results)

return CrewOutput(raw=final_output)
```

---

## Best Practices

### For Sequential

✅ **DO:**
- Order tasks logically
- Use context=[task] to pass outputs
- Keep task descriptions clear
- Test each task independently

❌ **DON'T:**
- Expect tasks to reorder themselves
- Skip context if next task needs previous output
- Create circular dependencies

### For Hierarchical

✅ **DO:**
- Give manager clear task descriptions
- Assign specialists with clear expertise
- Use when task decomposition is complex
- Budget for higher token cost

❌ **DON'T:**
- Use for simple linear workflows (wasted tokens)
- Expect deterministic ordering
- Forget: more expensive than sequential
- Use if you need exact same order every time

---

## Migration: Sequential to Hierarchical

If your Sequential crew becomes complex:

```python
# BEFORE: Sequential (becoming hard to manage)
crew = Crew(
    agents=[agent1, agent2, agent3, agent4],
    tasks=[task1, task2, task3, task4],
    process=Process.sequential
)

# AFTER: Hierarchical (manager handles complexity)
manager = Agent(role="Manager", goal="Coordinate", allow_delegation=True)

crew = Crew(
    agents=[manager, agent1, agent2, agent3, agent4],
    tasks=[big_task],  # Manager receives big task, decomposes
    process=Process.hierarchical,
    manager_agent=manager
)
```

---

## Process & Token Cost

```python
# Sequential: ~N iterations (one per task) + small manager overhead
tokens_sequential = (task1_tokens + task2_tokens + task3_tokens) + overhead

# Hierarchical: ~N iterations + manager deliberation at each step
tokens_hierarchical = (manager_tokens + task1_tokens + manager_tokens + 
                       task2_tokens + manager_tokens + task3_tokens) + overhead

# Hierarchical is typically 20-40% more expensive
```

---

## Summary

| Aspect | Sequential | Hierarchical |
|--------|-----------|-------------|
| **Order** | Predetermined | Manager-decided |
| **Cost** | Lower | Higher |
| **Debugging** | Easier | Harder |
| **Flexibility** | Lower | Higher |
| **Predictability** | High | Low |
| **Best for** | Known workflows | Complex problem-solving |
| **Overhead** | Minimal | Significant |
| **When to use** | Most cases | When needed |

**Rule of thumb**: Start with Sequential. Only use Hierarchical if Sequential becomes difficult to manage or you need dynamic task assignment.

---

**Next**: [Flow](./07-flow.md) for application-level control beyond processes.
