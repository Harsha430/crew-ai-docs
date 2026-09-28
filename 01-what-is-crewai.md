# What is CrewAI?

## The Problem CrewAI Solves

### Single LLM API Limitations

Calling an LLM API directly gives you:
```python
response = client.messages.create(
    model="claude-3-5-sonnet",
    messages=[{"role": "user", "content": "Analyze this document"}]
)
```

**Limitations:**
- No specialization: One model does everything poorly instead of many models doing specific things well
- No tools: Model can't interact with systems, data, or external services
- No memory: Forgets context between requests
- No guardrails: No validation or safety constraints
- No orchestration: Each request is independent
- No persistence: No tracking of what was done
- No observability: Can't see why it failed or what it cost
- No fallback: When it fails, you have nothing
- No human loop: No way to pause for human approval
- No evaluation: No way to measure if it actually worked

### The Multi-Agent Solution

Instead of one model doing everything, deploy **specialists**:

```
Research Agent (specialized for finding information)
    ↓
Writer Agent (specialized for clear communication)
    ↓
Reviewer Agent (specialized for quality)
    ↓
Human (final approval)
```

Each agent:
- Has a specific **role** and **goal**
- Can use **tools** to interact with systems
- Has access to **memory** about past work
- Receives **context** from previous agents
- Must meet **guardrails** (validation rules)
- Can be **evaluated** separately

**Benefits:**
✅ Specialization: Each agent excels at one thing
✅ Tools: Agents can search, analyze, write, compute
✅ Memory: Agents remember what happened before
✅ Guardrails: Outputs are validated
✅ Orchestration: Agents work together deterministically or flexibly
✅ Persistence: Full audit trail
✅ Observability: See costs, latencies, retries
✅ Fallback: If one agent fails, route to another
✅ Human-in-loop: Stop for approval when needed
✅ Evaluation: Measure quality at every stage

## What CrewAI Is

**CrewAI is a Python framework for building orchestrated multi-agent systems.**

It provides abstractions for:

| Concept | Purpose |
|---------|---------|
| **Agent** | WHO performs work (LLM + tools + memory) |
| **Task** | WHAT needs to be done (description, output format, guardrails) |
| **Crew** | WHO works together (agents + tasks + process) |
| **Process** | HOW tasks are ordered (sequential, hierarchical, or custom) |
| **Flow** | HOW the workflow is controlled (entry point, routing, state) |
| **Tool** | WHAT external capability an agent can use |
| **Memory** | WHAT agents remember between tasks |
| **Guardrail** | WHAT validates outputs before they proceed |

## Core Mental Model

### The Hierarchy

```
Application
    ↓
Flow (orchestrates execution path)
    ↓
Crew (orchestrates agents on tasks)
    ├─ Process (how tasks are ordered)
    │
    ├─ Task 1
    │   ├─ Agent 1
    │   │   ├─ LLM (thinks)
    │   │   ├─ Tools (acts)
    │   │   └─ Memory (remembers)
    │   └─ Guardrail 1 (validates)
    │
    ├─ Task 2 (context from Task 1)
    │   ├─ Agent 2 (or same agent)
    │   │   ├─ LLM
    │   │   ├─ Tools
    │   │   └─ Memory
    │   └─ Guardrail 2
    │
    └─ Task N
        └─ ...
    ↓
Output (CrewOutput with task results)
    ↓
Application continues (or routes in Flow)
```

### Agent ≠ Task

**Agent**: WHO and HOW
- Role: "I am a Research Expert"
- Goal: "I find accurate information"
- Tools: Web search, API calls, database queries
- Memory: I remember previous research
- Reasoning: I think step-by-step

**Task**: WHAT
- Description: "Research modern RAG architectures"
- Expected output: "Markdown document with 5 key patterns"
- Guardrail: "Output must be > 500 words"
- Context: "Use findings from previous research task"

One agent can perform many tasks. One task uses one agent.

### Crew ≠ Flow

**Crew**: Autonomous task execution within a fixed structure
- Fixed sequence of tasks (or manager decides order)
- All execution controlled by the Crew
- Returns final aggregated output
- When to use: Structured workflows, clear task dependencies

```python
crew = Crew(
    agents=[researcher, writer, reviewer],
    tasks=[research_task, write_task, review_task],
    process=Process.sequential  # Tasks run in order
)

result = crew.kickoff()
# All tasks execute, returns CrewOutput
```

**Flow**: Application-level workflow with conditional routing
- Flexible routing between steps
- Application has control at each decision point
- Can branch, loop, pause for human input
- Can run multiple crews or single-agent tasks
- When to use: Complex workflows, conditional logic, HITL

```python
class ResearchFlow(Flow):
    @start()
    def start(self):
        return "climate change"
    
    @listen(start)
    def research(self, topic):
        # Run crew or task
        return result
    
    @router(research)
    def route(self, research_result):
        if "controversial" in research_result.raw:
            return "review"
        return "finish"
    
    @listen("review")
    def human_review(self):
        return human_approval

flow = ResearchFlow()
result = flow.kickoff()
```

## When to Use CrewAI

✅ **Good fit:**
- Multi-agent workflows where agents specialize
- Tasks that need to validate outputs before proceeding
- Workflows that need memory across tasks
- Systems that need observability (costs, latencies, retries)
- Applications that need human approval loops
- Production systems with fallback and retry logic

❌ **Not a good fit:**
- Simple single-request LLM calls (use OpenAI SDK directly)
- Highly dynamic workflows needing low latency (use LangGraph)
- Deterministic procedural code (just write functions)
- Real-time systems (latency overhead)

## When to Use Each Abstraction

### Agent Only
```python
agent = Agent(role="Researcher", goal="Find accurate info", tools=[...])
# Manual execution in your code
```
Use when: You want one AI entity to think and act, but control execution yourself.

### Agent + Task (No Crew)
```python
task = Task(
    description="Research climate science",
    expected_output="Report",
    agent=agent
)
task.execute()  # Manually execute
```
Use when: You want structured task definition and validation, but don't need orchestration.

### Agent + Task + Crew
```python
crew = Crew(agents=[agent1, agent2], tasks=[task1, task2], process=Process.sequential)
result = crew.kickoff()
```
Use when: Multiple agents working on interdependent tasks in a fixed sequence.

### Crew + Flow
```python
class MyFlow(Flow):
    @start()
    def start(self):
        return inputs
    
    @listen(start)
    def research_step(self, inputs):
        crew = Crew(...)
        return crew.kickoff()
    
    @router(research_step)
    def route(self, result):
        return "review" if needs_review else "done"

flow = MyFlow()
result = flow.kickoff()
```
Use when: Application-level workflow with routing, conditional logic, or HITL.

## How CrewAI Executes

### Sequential Process

```
Start
  ↓
Task 1
  ├─ Agent thinks with tools
  ├─ Output validated by guardrail
  └─ Output becomes context for Task 2
  ↓
Task 2
  ├─ Agent thinks with previous context
  ├─ Output validated by guardrail
  └─ Output becomes context for Task 3
  ↓
Task 3
  ├─ Agent thinks with previous context
  ├─ Output validated by guardrail
  └─ Task finished
  ↓
Return CrewOutput
```

**Behavior**: Deterministic, predictable, clear dependencies.

### Hierarchical Process

```
Start
  ↓
Manager Agent
  ├─ Analyzes task
  ├─ Decides: "Agent A should research"
  ├─ Delegates to Agent A
  │   └─ Agent A returns result
  ├─ Analyzes result
  ├─ Decides: "Agent B should review"
  ├─ Delegates to Agent B
  │   └─ Agent B returns result
  ├─ Synthesizes final answer
  └─ Returns
  ↓
Return CrewOutput
```

**Behavior**: Model-driven, more flexible, less predictable.

### Agent Execution Loop

```
Agent receives task + context
  ↓
LLM thinks
  ├─ Generates thought
  ├─ Decides: "I need tool X"
  └─ Calls Tool X (if allowed)
      └─ Gets result
  ↓
LLM continues (max_iter iterations)
  ├─ Re-reads context + tool results
  ├─ Thinks more
  ├─ Decides: "I need another tool" or "I'm done"
  └─ Repeats
  ↓
Agent returns final answer
  ↓
Guardrail validates
  ├─ Pass → Task complete
  └─ Fail → Retry or escalate
```

## Key Components

### Agent

An Agent is:
- An identity: "I am a Research Expert"
- A capability set: Tools, memory, constraints
- A thinking engine: LLM with prompts

```python
from crewai import Agent

researcher = Agent(
    role="Research Expert",
    goal="Find accurate, current information",
    backstory="You are a veteran research expert with 20 years of experience.",
    tools=[web_search, database_query],
    llm=claude,
    memory=True,
    verbose=True
)
```

### Task

A Task is:
- What needs doing: Description and context
- What success looks like: Expected output and guardrails
- Who does it: Agent assignment
- What came before: Context from previous tasks

```python
from crewai import Task

research_task = Task(
    description="Research the latest advances in RAG",
    expected_output="A markdown report with 5 key advances",
    agent=researcher,
    guardrail=output_must_be_long_enough,
    context=[previous_context_task]  # Uses previous task output
)
```

### Crew

A Crew is:
- A set of agents
- A sequence of tasks
- An orchestration strategy (process)
- Shared resources (memory, callbacks)

```python
from crewai import Crew, Process

crew = Crew(
    agents=[researcher, writer, reviewer],
    tasks=[research_task, write_task, review_task],
    process=Process.sequential,
    memory=True,
    verbose=True
)

result = crew.kickoff()
```

### Process

A Process is:
- **Sequential**: Tasks run one after another in order
- **Hierarchical**: A manager agent decides task order and assigns work
- **Custom**: You define the logic

### Flow

A Flow is:
- An entry point: `@start()` method
- Listener steps: `@listen(previous_step)` method
- Routing logic: `@router(source)` determines next step
- State management: `self.state` persists data across steps

```python
from crewai import Flow

class ResearchFlow(Flow):
    @start()
    def kickoff_step(self):
        return {"topic": "AI trends"}
    
    @listen(kickoff_step)
    def research_step(self, topic):
        crew = Crew(...)
        return crew.kickoff()
    
    @router(research_step)
    def decide_next(self, research_output):
        if quality_score(research_output) > 0.8:
            return "publish"
        else:
            return "review"
    
    @listen("publish")
    def publish_step(self):
        return "Done"

flow = ResearchFlow()
result = flow.kickoff()
```

### Tool

A Tool is:
- A Python function or class the agent can call
- Has a schema (name, description, input parameters)
- Returns a result the agent can read and act on

```python
from crewai_tools import tool

@tool
def web_search(query: str) -> str:
    """Search the web for information"""
    # Implementation
    return search_results

agent = Agent(
    role="Researcher",
    tools=[web_search]
)
```

### Memory

Memory allows agents to:
- Remember previous work across tasks
- Retrieve relevant past interactions
- Build on earlier findings

```python
crew = Crew(
    agents=[...],
    tasks=[...],
    memory=True  # Enable memory
)
```

Memory types:
- **Short-term**: Task execution context (current and recent)
- **Long-term**: Historical information across crew executions
- **Entity memory**: Key facts about entities

### Guardrail

A Guardrail is:
- A validation rule on task output
- Runs after agent completes task
- Can retry or escalate if validation fails

```python
def validate_output(output: str) -> bool:
    return len(output) > 500 and "key finding" in output

task = Task(
    description="Research something",
    expected_output="A detailed report",
    agent=agent,
    guardrail=validate_output
)
```

## Output: CrewOutput

Every `crew.kickoff()` returns a `CrewOutput`:

```python
result = crew.kickoff()

# Access outputs
print(result.raw)           # Final answer as string
print(result.json_dict)     # Parsed JSON (if valid)
print(result.pydantic)      # Typed Pydantic object
print(result.tasks_output)  # List of TaskOutput objects
print(result.token_usage)   # Token stats
```

Each `TaskOutput` contains:
- `description`: What the task was
- `summary`: Agent's summary
- `raw`: Raw output string
- `pydantic`: Typed output
- `json_dict`: JSON parsed output

See [Runtime Output Anatomy](./10-runtime-output.md) for complete details.

## Flow State

A Flow has state that persists across steps:

```python
class MyFlow(Flow):
    @start()
    def start(self):
        self.state["research_topic"] = "AI"
        return self.state["research_topic"]
    
    @listen(start)
    def next_step(self, topic):
        self.state["findings"] = do_research(topic)
        return self.state["findings"]
    
    # State is accessible via self.state dictionary
```

State is:
- Per-flow-execution
- Mutable
- Accessible across all steps
- Not automatically persisted (you must implement persistence)

## Why These Abstractions Exist

| Abstraction | Problem It Solves |
|-------------|-------------------|
| **Agent** | Single LLM can't specialize; needs identity, tools, memory |
| **Task** | Need to describe work, validate output, pass context |
| **Crew** | Need to orchestrate multiple agents predictably |
| **Process** | Need different orchestration strategies (sequential vs manager-driven) |
| **Flow** | Crew is autonomous; need application control for routing, conditions, HITL |
| **Tool** | Agents need to interact with systems (can't read arbitrary APIs) |
| **Memory** | Agents need context of previous work |
| **Guardrail** | Need validation before outputs are used |

## Next Steps

1. **Understand Architecture**: Read [Architecture](./02-architecture.md)
2. **Learn Agent**: Read [Agent](./03-agent.md)
3. **Learn Task**: Read [Task](./04-task.md)
4. **See It Work**: Check [Common Mistakes](./11-common-mistakes.md) for common "gotchas"
5. **Build**: Pick a use case and start implementing

## Summary

| What | How | Why |
|------|-----|-----|
| **Agent** | LLM + tools + memory | Specialized thinking |
| **Task** | Structured work + validation | Reliable outputs |
| **Crew** | Multiple agents + process | Coordinated work |
| **Process** | Sequential or hierarchical | Flexible orchestration |
| **Flow** | Routing + conditions + state | Application control |
| **Memory** | Persistent context | Learn from past |
| **Guardrail** | Output validation | Quality assurance |
| **Tool** | Agent capability | External interaction |

CrewAI turns **"call an LLM API"** into **"build a thinking team"**.

---

**Next**: [Architecture](./02-architecture.md)
