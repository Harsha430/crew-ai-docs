# Agent: Complete Reference

## What is an Agent?

An Agent is:
- An **identity**: Role, goal, backstory
- A **capability set**: Tools, memory, reasoning constraints
- A **thinking engine**: LLM with structured prompts
- An **executor**: Iteratively thinks and acts until complete

**NOT** a Task. An Agent can perform many tasks. A Task uses one Agent.

## Core Mental Model

```
Agent = Identity + Capabilities + Thinking
    ├─ Identity (who am I?)
    │   ├─ role: "Security Expert"
    │   ├─ goal: "Identify vulnerabilities"
    │   └─ backstory: "15 years in penetration testing"
    │
    ├─ Capabilities (what can I do?)
    │   ├─ tools: [port_scan, web_request, analyze_code]
    │   ├─ memory: Enabled (remember findings)
    │   ├─ reasoning: Deep (for complex analysis)
    │   └─ max_iter: 10 (thinking steps)
    │
    └─ Thinking (how do I work?)
        ├─ LLM: Claude 3.5 Sonnet
        ├─ Temperature: 0.7
        ├─ Max tokens: 4096
        └─ Prompt template: (system + role + tools)
```

## Agent Parameters (v1.15.22)

### Required

```python
from crewai import Agent

agent = Agent(
    role="Your Role",
    goal="Your Goal",
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `role` | `str` | **required** | Identity: "Security Expert", "Data Analyst", etc. |
| `goal` | `str` | **required** | Objective: "Find vulnerabilities", "Extract insights", etc. |

### Identity & Context

```python
agent = Agent(
    role="Data Analyst",
    goal="Find actionable insights",
    backstory="You are a data analyst with 10 years of experience...",
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `backstory` | `str` | `""` | Persona/context given to LLM to inform reasoning |

**Why**: Backstory helps the LLM understand context and nuance. A backstory of "veteran" vs "intern" changes how carefully the LLM approaches tasks.

### Tools

```python
from crewai_tools import tool

@tool
def web_search(query: str) -> str:
    """Search the web"""
    return search_results

agent = Agent(
    role="Researcher",
    goal="Find accurate information",
    tools=[web_search, another_tool],
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `tools` | `list[Tool]` | `[]` | What external capabilities the agent can use |

**Why**: Agents without tools can only think. With tools, they can act.

### LLM Configuration

```python
from crewai import Agent
from langchain_openai import ChatOpenAI

custom_llm = ChatOpenAI(
    model_name="gpt-4",
    temperature=0.3,
    max_tokens=2048
)

agent = Agent(
    role="Specialist",
    goal="Solve problem",
    llm=custom_llm,
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `llm` | `LLM` | CrewAI default (Claude or GPT) | Which language model to use |

**Note**: If not specified, CrewAI uses a default provider. Check your `CREWAI_LLM_*` environment variables.

### Memory

```python
agent = Agent(
    role="Researcher",
    goal="Learn and remember",
    memory=True,  # Enable memory
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `memory` | `bool` | `True` | Should agent remember past interactions? |

**Why**: 
- `True`: Agent has context of previous tasks, learns
- `False`: Each task is independent, fresh start

**Trade-off**: Memory increases context size (costs more tokens) but provides better continuity.

### Reasoning & Iteration

```python
agent = Agent(
    role="Analyst",
    goal="Provide detailed analysis",
    max_iter=15,  # How many thinking steps
    max_retry_limit=2,  # How many times to retry on failure
    reasoning="extended",  # More thoughtful reasoning
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `max_iter` | `int` | `10` | Max iterations (thinking + tool use loops) |
| `max_retry_limit` | `int` | `2` | How many times to retry on tool failure |
| `reasoning` | `str` | `"basic"` | Reasoning level ("basic" or "extended") |

**CRITICAL**: `max_iter` is NOT tool calls. It's thinking steps. One iteration might:
- Iterate 1: Think → call Tool A → get result
- Iterate 2: Think → call Tool B → get result
- Iterate 3: Think → return final answer

So `max_iter=10` allows many tool calls, not just 10.

### Verbosity & Logging

```python
agent = Agent(
    role="Worker",
    goal="Do work",
    verbose=True,  # Print thinking/actions
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `verbose` | `bool` | `False` | Print thinking, tools, decisions to stdout |

**Use in development**: `True` (debug)  
**Use in production**: `False` (or log to file)

### Delegation

```python
agent = Agent(
    role="Manager",
    goal="Coordinate team",
    allow_delegation=True,  # Can delegate tasks to other agents
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `allow_delegation` | `bool` | `True` | Can this agent delegate work to others? |

**When to disable** (`False`):
- You want agent to solve everything themselves
- You want to prevent infinite delegation loops
- Agent should be a "leaf" agent

**When to enable** (`True`):
- Agent is a manager/coordinator
- Hierarchical process where delegation makes sense

### Context Window & Tokens

```python
agent = Agent(
    role="Worker",
    goal="Accomplish goal",
    respect_context_window=True,  # Don't exceed context limits
)
```

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `respect_context_window` | `bool` | `True` | Fail if context window would be exceeded? |

**Behavior**:
- `True`: If adding context would exceed token limit, stop and return what you have
- `False`: Try anyway (may get token limit errors from LLM)

## Agent Execution Flow

```
1. Task assigns work to Agent
    ├─ task.description
    ├─ task.expected_output
    └─ task.context (previous task outputs)

2. Agent prepares prompt
    ├─ role, goal, backstory
    ├─ task description
    ├─ tool descriptions
    ├─ previous tool results
    └─ memory (if enabled)

3. Agent calls LLM (iteration 0)
    ├─ Input: Full prompt + context
    ├─ LLM generates: thought + decision
    └─ Output: "I should call tool X" OR "Final answer: ..."

4. Decision
    ├─ If tool call:
    │  ├─ Validate: Tool exists and agent has permission
    │  ├─ Parse: Tool name + parameters
    │  ├─ Execute: Call tool
    │  ├─ Get: Result
    │  └─ Continue (iteration += 1, go to 3)
    │
    └─ If final answer:
       └─ Return answer (go to 5)

5. Check termination
    ├─ If max_iter reached:
    │  └─ Return current answer
    └─ If answer returned:
       └─ Task continues to guardrail

6. Agent returns output
    ├─ raw: Final text answer
    └─ metadata: token usage, iterations used, tools called
```

## Common Agent Patterns

### Pattern 1: Specialist Agent (No Delegation)

```python
researcher = Agent(
    role="Research Expert",
    goal="Find accurate, current information",
    backstory="You have 20 years of research experience...",
    tools=[web_search, database_query],
    memory=True,
    allow_delegation=False,  # I don't delegate
    max_iter=15,
    verbose=True
)
```

**When to use**: Focused, specialized agent that should solve everything themselves.

### Pattern 2: Manager Agent (With Delegation)

```python
manager = Agent(
    role="Project Manager",
    goal="Coordinate team to complete project",
    backstory="You are an experienced PM...",
    allow_delegation=True,  # I delegate to specialists
    memory=True,
    max_iter=10
)
```

**When to use**: Hierarchical process where agent decides task assignment.

### Pattern 3: Tool-Rich Agent

```python
analyst = Agent(
    role="Data Analyst",
    goal="Extract insights from data",
    tools=[
        database_query,
        statistical_analysis,
        chart_generator,
        report_writer,
    ],
    memory=True,
    max_iter=20,  # More iterations because more tools
    verbose=True
)
```

**When to use**: Agent needs to orchestrate multiple tools.

### Pattern 4: Reasoning-Heavy Agent

```python
strategist = Agent(
    role="Strategy Consultant",
    goal="Provide strategic recommendations",
    backstory="20 years of consulting...",
    reasoning="extended",  # More careful thinking
    memory=True,
    max_iter=15,
    tools=[],  # No tools, just thinking
    verbose=True
)
```

**When to use**: Complex reasoning, less tool use.

## Agent Return Value

An agent doesn't return `CrewOutput`. The Task returns values from the agent.

```python
agent = Agent(...)
task = Task(
    description="Do something",
    agent=agent
)
# When task executes, agent returns text
# Agent return → Task wraps → TaskOutput created
```

Agent execution returns:
- `raw`: Final answer as string
- Internal: Iterations used, tools called, token count

This is wrapped by the Task into a `TaskOutput`.

## Agent-to-Agent Interaction

### Method 1: Task Context (Sequential)

```python
agent1 = Agent(role="Researcher", goal="Find info", tools=[web_search])
agent2 = Agent(role="Writer", goal="Write report", tools=[])

task1 = Task(
    description="Research AI",
    agent=agent1
)

task2 = Task(
    description="Write report based on research",
    agent=agent2,
    context=[task1]  # Agent2 gets Agent1's output
)

crew = Crew(agents=[agent1, agent2], tasks=[task1, task2], process=Process.sequential)
result = crew.kickoff()
```

Agent2 sees Agent1's output in the prompt context.

### Method 2: Delegation (Hierarchical)

```python
manager = Agent(
    role="Manager",
    goal="Coordinate",
    allow_delegation=True
)

specialist1 = Agent(role="Researcher", goal="Research", tools=[web_search])
specialist2 = Agent(role="Writer", goal="Write", tools=[])

crew = Crew(
    agents=[manager, specialist1, specialist2],
    tasks=[big_task],  # Just one task to manager
    process=Process.hierarchical,
    manager_agent=manager
)

# Manager decides: "specialist1 research first, then specialist2 write"
```

Manager dynamically delegates to specialists.

### Method 3: Direct Conversation (Not typical)

Agents don't directly call each other. They go through:
- Task context (sequential)
- Manager delegation (hierarchical)
- Tool calls (agent calls API which queries another system)

## Agent Constraints

### max_iter Confusion

❌ **WRONG**: "max_iter=10 means 10 tool calls"

✅ **CORRECT**: "max_iter=10 means 10 thinking iterations, each can have 0-N tool calls"

Example:
```
Iteration 1: LLM thinks → calls tool A → gets result
Iteration 2: LLM thinks → calls tool B → gets result
Iteration 3: LLM thinks → calls tools C and D → gets results
Iteration 4: LLM thinks → decides final answer
```

This is 4 iterations, 4 tool calls (one per iteration except last).

### Context Window Limits

```python
agent = Agent(
    role="Researcher",
    respect_context_window=True
)
```

If agent has:
- Large backstory
- Large previous task outputs
- Large tool results
- Many iterations of history

The context window can fill up. With `respect_context_window=True`, agent stops rather than erroring.

### Token Cost

Each iteration of agent thinking:
- Input tokens: Prompt + context + tool results
- Output tokens: Response

With `max_iter=15` on a complex task:
- Could be 15 * (2000 input + 500 output) = 37,500 tokens
- Cost: Depends on model

Monitor with `token_usage` tracking.

## Agent Best Practices

✅ **DO:**
- Give agents clear, specific roles
- Provide relevant tools (not too many, not too few)
- Set `verbose=True` during development
- Set reasonable `max_iter` (10-15 is usually enough)
- Enable memory for multi-task workflows
- Use `backstory` to establish expertise

❌ **DON'T:**
- Give agents conflicting goals
- Overload with irrelevant tools
- Leave `verbose=True` in production (costs output tokens)
- Set `max_iter=100` (you'll timeout)
- Give two agents the exact same role (differentiates by backstory if you must)

## Agent Testing

```python
from crewai import Agent, Task

# Test agent alone
agent = Agent(role="Tester", goal="Test things", tools=[])
task = Task(
    description="Test if this output is good: 'output here'",
    agent=agent,
    expected_output="Yes or No"
)

# Don't use crew, just execute task directly
result = task.execute()
print(result.raw)
```

## Debugging Agent Behavior

### When Agent Loops Infinitely

**Symptom**: Agent keeps calling same tool repeatedly, never returns.

**Diagnosis**:
1. Check `max_iter`: Is it too high? Set to 10-15.
2. Check tool results: Are they not giving agent what it needs?
3. Check prompt: Is role/goal confusing?

**Fix**:
```python
agent = Agent(
    role="Specific role",
    goal="Clear, achievable goal",
    max_iter=10,  # Reduce from 50
    verbose=True  # See what's happening
)
```

### When Agent Doesn't Use Tools

**Symptom**: Agent just thinks and returns answer, never calls tools.

**Diagnosis**:
1. Are tools provided? `tools=[...]`
2. Is agent allowed? No permission restrictions?
3. Are tool descriptions clear enough?

**Fix**:
```python
# Make tool descriptions very clear
@tool
def search(query: str) -> str:
    """SEARCH THE INTERNET. This is how you find information."""
    return internet_search(query)

agent = Agent(
    role="Researcher",
    goal="Find information. Use search tool.",  # Explicit
    tools=[search],
    verbose=True  # Debug
)
```

### When Agent Calls Tool With Wrong Parameters

**Symptom**: Tool execution fails because parameters are wrong format.

**Diagnosis**:
1. Tool schema might be unclear
2. Agent didn't understand parameter format
3. Context didn't explain what tool expects

**Fix**:
```python
@tool
def calculate(x: float, y: float) -> float:
    """Calculate x + y. x and y must be numbers."""
    return x + y

# Make agent aware
agent = Agent(
    role="Calculator",
    goal="Do calculations accurately",
    tools=[calculate],
    backstory="You understand mathematical operations..."
)
```

---

**Next**: [Task](./04-task.md)
