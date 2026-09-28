# CrewAI Documentation Index

**Complete Engineering Reference for CrewAI v1.15.22**

## Documentation Created

### Foundation (Start Here)
1. **[README.md](./README.md)** - Navigation guide, learning roadmap, quick facts
2. **[01-what-is-crewai.md](./01-what-is-crewai.md)** - Problem solved, mental model, core concepts
3. **[02-architecture.md](./02-architecture.md)** - System design, data flow, component interactions

### Core API Reference
4. **[03-agent.md](./03-agent.md)** - Agent parameters, execution flow, patterns, debugging
5. **[04-task.md](./04-task.md)** - Task specification, context, output formats, validation
6. **[05-crew.md](./05-crew.md)** - Crew orchestration, processes, execution patterns

### Critical Reference
7. **[10-runtime-output.md](./10-runtime-output.md)** - **ESSENTIAL**: All return types, structures, access patterns
8. **[CRITICAL-DISTINCTIONS.md](./CRITICAL-DISTINCTIONS.md)** - 20+ concept clarifications for interviews

### Interview Preparation
9. **[19-interview-questions.md](./19-interview-questions.md)** - 155 questions across 7 levels + answers

## Document Map by Use Case

### "I want to understand CrewAI from scratch"
→ README → 01-what-is-crewai → 02-architecture → 03-agent → 04-task → 05-crew

### "I'm building an agent"
→ 03-agent.md (parameters, best practices, patterns)

### "I'm building a crew"
→ 05-crew.md (orchestration, processes, patterns)

### "My code returns CrewOutput. What is it?"
→ 10-runtime-output.md (structures, fields, access patterns)

### "I'm confused about X concept"
→ CRITICAL-DISTINCTIONS.md (find your concept, see explanation)

### "I'm in an interview"
→ CRITICAL-DISTINCTIONS.md (review 20 concepts) → 19-interview-questions.md (practice with answers)

### "My code has a bug"
→ Find relevant chapter (03/04/05), scroll to "Debugging" section

## Key Chapters at a Glance

| File | Sections | Key Topics |
|------|----------|-----------|
| 01-what-is-crewai | Problems, Solutions, Mental model, Abstractions, When to use | Core concepts, decision trees |
| 02-architecture | Overview, Components, Execution flow, Cross-cutting concerns | System design, data flow |
| 03-agent | Parameters, Execution loop, Patterns, Debugging | WHO, identity, thinking |
| 04-task | Parameters, Execution, Context, Patterns, Debugging | WHAT, specification, validation |
| 05-crew | Parameters, Processes, Patterns, Debugging | Orchestration, team |
| 10-runtime-output | CrewOutput, TaskOutput, Token usage, Debugging | Return values, access |
| CRITICAL-DISTINCTIONS | 20 concept pairs with clear distinctions | Interview clarity |
| 19-interview-questions | 155 Q&A across 7 levels | Interview prep |

## Quick Reference Tables

### Agent Parameters (Most Important)

| Parameter | Default | Typical Use |
|-----------|---------|----------|
| `role` | required | Identity: "Researcher" |
| `goal` | required | Objective: "Find accurate information" |
| `backstory` | "" | Persona: "20 years of experience" |
| `tools` | [] | Capabilities: [web_search, api_call] |
| `llm` | CrewAI default | Model: ChatOpenAI(model="gpt-4") |
| `memory` | True | Learn from past: True/False |
| `max_iter` | 10 | Thinking loops: usually 10-15 |
| `max_retry_limit` | 2 | Retry on failure: usually 1-2 |
| `allow_delegation` | True | Can delegate: True for managers, False for specialists |
| `verbose` | False | Debug output: True in dev, False in prod |

### Task Parameters (Most Important)

| Parameter | Default | Typical Use |
|-----------|---------|----------|
| `description` | required | What to do: "Research AI trends" |
| `expected_output` | required | What success looks like: "Markdown report" |
| `agent` | required | WHO does it: researcher_agent |
| `context` | [] | Input from previous: [task1, task2] |
| `output_pydantic` | None | Structured output: MyModel class |
| `guardrail` | None | Validation: validation_function |
| `guardrail_max_retries` | 1 | Retry on fail: usually 1-2 |
| `human_input` | False | Ask human: True for approval |

### Crew Parameters (Most Important)

| Parameter | Default | Typical Use |
|-----------|---------|----------|
| `agents` | required | Team: [agent1, agent2, ...] |
| `tasks` | required | Work: [task1, task2, ...] |
| `process` | sequential | Orchestration: sequential or hierarchical |
| `manager_agent` | required if hierarchical | Manager: only for hierarchical |
| `memory` | True | Shared memory: True for multi-task |
| `verbose` | False | Debug: True in dev |

### Return Values

```
crew.kickoff() → CrewOutput
  ├─ raw: str (always present)
  ├─ json_dict: dict | None (if JSON)
  ├─ pydantic: BaseModel | None (if structured)
  ├─ tasks_output: list[TaskOutput]
  │   └─ TaskOutput[i]
  │       ├─ raw: str
  │       ├─ pydantic: Model | None
  │       ├─ json_dict: dict | None
  │       ├─ agent: Agent
  │       └─ token_usage: dict
  └─ token_usage: dict (total)
```

## Critical Distinctions (Quick Reference)

| Concept 1 | vs | Concept 2 | Remember |
|-----------|----|---------:|----------|
| Agent | vs | Task | WHO vs WHAT |
| Crew | vs | Flow | Autonomous vs Orchestrated |
| State | vs | Memory | Temporary vs Persistent |
| max_iter | vs | Tool calls | Iterations vs Executions |
| Raw | vs | JSON | Unstructured vs Structured |
| Sequential | vs | Hierarchical | Predetermined vs Manager-decided |
| Delegation | vs | Context | Agent → Agent vs Task → Task |
| Retry | vs | Fallback | Same approach vs Different approach |
| Expected_output | vs | Guardrail | Guidance vs Validation |

See [CRITICAL-DISTINCTIONS.md](./CRITICAL-DISTINCTIONS.md) for full explanations.

## Common Code Patterns

### Minimal Agent
```python
from crewai import Agent
agent = Agent(role="Worker", goal="Do work")
```

### Agent with Tools
```python
agent = Agent(
    role="Researcher",
    goal="Find information",
    tools=[web_search, database_query],
    memory=True
)
```

### Minimal Task
```python
from crewai import Task
task = Task(
    description="Do something",
    expected_output="Result",
    agent=agent
)
```

### Task with Validation
```python
task = Task(
    description="Research",
    expected_output="Detailed report",
    agent=agent,
    guardrail=lambda output: len(output) > 500,
    guardrail_max_retries=2
)
```

### Minimal Crew
```python
from crewai import Crew, Process
crew = Crew(
    agents=[agent1, agent2],
    tasks=[task1, task2],
    process=Process.sequential
)
result = crew.kickoff()
```

### Minimal Flow
```python
from crewai import Flow

class MyFlow(Flow):
    @start()
    def start_step(self):
        return {"topic": "AI"}
    
    @listen(start_step)
    def research_step(self, input_data):
        crew = Crew(...)
        return crew.kickoff()

flow = MyFlow()
result = flow.kickoff()
```

## Interview Preparation Checklist

### Understand These First (Foundation)
- [ ] What is CrewAI?
- [ ] Agent vs Task
- [ ] Crew vs Flow
- [ ] Process vs Flow
- [ ] Sequential vs Hierarchical

### Know These Cold (Distinctions)
- [ ] All 20 distinctions from CRITICAL-DISTINCTIONS.md
- [ ] Agent parameters (role, goal, tools, memory, max_iter)
- [ ] Task parameters (description, expected_output, context, guardrail)
- [ ] Crew parameters (agents, tasks, process)

### Be Able to Explain (Depth)
- [ ] How agent execution loop works
- [ ] How context flows between tasks
- [ ] What CrewOutput and TaskOutput contain
- [ ] How max_iter differs from tool calls
- [ ] When to use hierarchical vs sequential

### Be Ready to Solve (Scenarios)
- [ ] Design a research → write → review workflow
- [ ] Fix an agent that loops infinitely
- [ ] Explain how to handle failures
- [ ] Design a human-in-loop approval system
- [ ] Describe token cost optimization

### Practice Questions (from 19-interview-questions.md)
- [ ] Answer all Level 1 (30 questions)
- [ ] Answer all Level 2 (35 questions)
- [ ] Answer all Level 3 (25 questions)
- [ ] Answer Level 4-7 (65 questions)

## Quick Debugging Guide

**Agent doesn't use tools?**
→ [03-agent.md](./03-agent.md) "Debugging Agent Behavior"

**Task context empty?**
→ [04-task.md](./04-task.md) "Task Debugging"

**Guardrail keeps failing?**
→ [04-task.md](./04-task.md) "When Guardrail Keeps Failing"

**Crew hangs?**
→ [05-crew.md](./05-crew.md) "When Crew Hangs"

**Wrong return type?**
→ [10-runtime-output.md](./10-runtime-output.md) "Common Output Mistakes"

**Confused about concept?**
→ [CRITICAL-DISTINCTIONS.md](./CRITICAL-DISTINCTIONS.md)

## Documentation Statistics

| Component | Count | Details |
|-----------|-------|---------|
| **Total Chapters** | 9 | Foundation, API, Reference, Interview |
| **Parameters Documented** | 40+ | Agent, Task, Crew with all options |
| **Interview Questions** | 155 | Across 7 difficulty levels with answers |
| **Concepts Clarified** | 20 | Critical distinctions with examples |
| **Code Patterns** | 50+ | Agent, Task, Crew, Flow examples |
| **Debugging Sections** | 30+ | Specific problems with diagnosis & fixes |
| **Tables & Charts** | 50+ | Quick reference, architecture, comparisons |

## What's Covered

✅ **Core API**
- Agent (parameters, execution, patterns)
- Task (specification, context, validation)
- Crew (orchestration, processes, execution)
- Flow (control flow, state, routing)
- Tools (creation, schema, execution)

✅ **Runtime**
- CrewOutput structure and access
- TaskOutput structure and access
- Token usage tracking
- Output formats (raw, JSON, Pydantic)

✅ **Architecture**
- System design and data flow
- Component interactions
- Execution models (sequential, hierarchical)
- Cross-cutting concerns

✅ **Production**
- Reliability patterns (retry, fallback, circuit breaker)
- Security considerations
- Observability and monitoring
- Deployment patterns

✅ **Learning**
- 155 interview questions with answers
- Real-world scenarios
- Best practices
- Common mistakes and fixes

## What's Not Covered (Out of Scope)

❌ CrewAI Cloud/AMP REST API (use official docs)
❌ UI/Platform features (use official docs)
❌ Third-party integrations (Slack, Email, etc.) beyond concepts
❌ Older versions (<1.15)
❌ Installation & setup (use official docs)

## How to Update This Documentation

If CrewAI v1.15.22 changes:
1. Update the parameter tables
2. Verify return types in chapter 10
3. Check execution flow diagrams
4. Add new interview questions
5. Update distinctions if concepts change

Current baseline: CrewAI v1.15.22 (Sept 2026)

---

## Getting Started

**First time user?**
→ Start with [README.md](./README.md), then [01-what-is-crewai.md](./01-what-is-crewai.md)

**Need to build something?**
→ Pick your use case from README, go to relevant chapter

**Preparing for interview?**
→ Read [CRITICAL-DISTINCTIONS.md](./CRITICAL-DISTINCTIONS.md), practice [19-interview-questions.md](./19-interview-questions.md)

**Debugging an issue?**
→ Find your issue in the relevant chapter's debugging section

---

**Last Updated**: September 28, 2026
**CrewAI Version**: 1.15.22
**Python Version**: 3.10+
**Status**: Complete reference for engineering-level knowledge
