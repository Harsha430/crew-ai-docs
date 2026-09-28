# CrewAI v1.15.22 Engineering Documentation

**Complete reference for production-ready multi-agent systems**

This documentation is designed for engineers who:
- Have studied CrewAI basics and want deep implementation knowledge
- Need to build production-ready multi-agent systems
- Are preparing for GenAI/Agentic AI interviews
- Want to understand every parameter, return type, and execution path

## Quick Facts

- **CrewAI Version**: v1.15.22
- **Python**: 3.10+
- **Focus**: Engineering, architecture, runtime outputs, production patterns
- **Not**: A beginner tutorial

## Learning Roadmap

### Phase 1: Mental Model (Chapters 1-2)
- [What is CrewAI?](./01-what-is-crewai.md) — Problem, solution, mental model
- [Architecture](./02-architecture.md) — System design, components, data flow

### Phase 2: Core APIs (Chapters 3-7)
- [Agent](./03-agent.md) — Complete parameter reference, runtime behavior
- [Task](./04-task.md) — Task definition, context, output structures
- [Crew](./05-crew.md) — Orchestration, execution, return types
- [Process](./06-process.md) — Sequential vs hierarchical, manager-driven routing
- [Flow](./07-flow.md) — Application-level workflows, state management

### Phase 3: Integration (Chapters 8-9)
- [Tools](./08-tools.md) — Tool creation, schema, execution lifecycle
- [LLM Integration](./09-llm.md) — Provider configuration, model fallback

### Phase 4: Runtime & Output (Chapter 10)
- [Runtime Output Anatomy](./10-runtime-output.md) — **CRITICAL**: Every return type, field, and access pattern

### Phase 5: Production Patterns (Chapters 11-15)
- [Common Mistakes](./11-common-mistakes.md) — Detection, fix, mental model correction
- [Reliability](./12-reliability.md) — Retry, fallback, circuit breaker, graceful degradation
- [Security](./13-security.md) — Prompt injection, PII, data isolation, compliance
- [Observability](./14-observability.md) — Logging, metrics, tracing, token tracking
- [Deployment](./15-deployment.md) — Docker, environment, phased rollout

### Phase 6: Evaluation & Comparison (Chapters 16-18)
- [Evaluation](./16-evaluation.md) — Metrics, golden datasets, LLM-as-judge
- [vs AutoGen](./17-autogen-comparison.md) — Architecture, orchestration, APIs
- [vs LangGraph](./18-langgraph-comparison.md) — State, workflow, routing

### Phase 7: Interview Preparation (Chapters 19-21)
- [Critical Distinctions](./CRITICAL-DISTINCTIONS.md) — 30+ concept comparisons
- [Interview Questions](./19-interview-questions.md) — 150+ questions across 7 levels
- [Scenario Questions](./20-scenario-questions.md) — Real-world problems and solutions
- [System Design](./21-system-design.md) — 20+ architecture scenarios

## Important Distinctions

**Must understand first:**

| Concept | vs | Distinction |
|---------|----|----|
| Agent | Task | WHO performs (Agent) vs WHAT to do (Task) |
| Crew | Flow | Autonomous execution vs Workflow control |
| Process | Flow | How tasks are ordered vs How application flows |
| State | Memory | Current workflow data vs Historical knowledge |
| Expected Output | Guardrail | Contract vs Validation |
| Tool | API | Agent capability vs External service |
| Retry | Fallback | Same attempt vs Alternative approach |
| max_iter | Tool calls | Agent thinking steps vs Tool invocations |

See [CRITICAL-DISTINCTIONS.md](./CRITICAL-DISTINCTIONS.md) for 30+ comparisons.

## Key Concepts at a Glance

```
USER INPUT
    ↓
FLOW (@start → @listen → @router)
    ↓
CREW (orchestrates agents)
    ├─ PROCESS (sequential or hierarchical)
    │   ├─ TASK 1
    │   │   ├─ Agent (LLM + Tools + Memory)
    │   │   └─ Expected Output + Guardrail
    │   ├─ TASK 2
    │   │   └─ Uses context from TASK 1
    │   └─ ...
    ├─ Manager Agent (if hierarchical)
    └─ Memory (persistent across tasks)
    ↓
OUTPUT (CrewOutput with TaskOutput list)
    ├─ raw (text)
    ├─ json_dict (parsed JSON)
    ├─ pydantic (typed object)
    └─ tasks_output (TaskOutput[])
```

## Version & Compatibility

- **Installed Version**: CrewAI v1.15.22
- **Source**: Official CrewAI docs + GitHub repository
- **Python**: 3.10, 3.11, 3.12, 3.13 verified
- **APIs Verified**: Against official API reference as of Sept 2026

## Documentation Principles

1. **PROBLEM → SOLUTION**: Every concept explains why it exists
2. **RUNTIME ANATOMY**: All return types, fields, nested structures documented
3. **NO ASSUMPTIONS**: Every parameter verified against source
4. **PRODUCTION FOCUS**: Security, reliability, observability, cost included
5. **INTERVIEW READY**: 150+ questions + scenario answers

## Quick Navigation

### By Use Case
- **"Build a basic agent"** → [Agent](./03-agent.md) + [Tools](./08-tools.md)
- **"Coordinate multiple agents"** → [Crew](./05-crew.md) + [Process](./06-process.md) + [Delegation](./05-crew.md#delegation)
- **"Complex workflows"** → [Flow](./07-flow.md) + [State](./07-flow.md#state)
- **"Production system"** → [Reliability](./12-reliability.md) + [Security](./13-security.md) + [Deployment](./15-deployment.md)
- **"Interview prep"** → Start with [Critical Distinctions](./CRITICAL-DISTINCTIONS.md), then [Interview Questions](./19-interview-questions.md)

### By Problem
- **"Agent loops"** → [Common Mistake: max_iter Confusion](./11-common-mistakes.md#max_iter)
- **"Task context empty"** → [Task Context Design](./04-task.md#context)
- **"Wrong return type"** → [Runtime Output Anatomy](./10-runtime-output.md)
- **"Cost too high"** → [Token Tracking](./14-observability.md#token-usage) + [Model Selection](./09-llm.md)
- **"Flow enters wrong branch"** → [Router Design](./07-flow.md#router)

## How to Use This Documentation

1. **First time**: Read [What is CrewAI?](./01-what-is-crewai.md) + [Architecture](./02-architecture.md)
2. **Building**: Use the relevant chapter + reference [Runtime Output Anatomy](./10-runtime-output.md) continuously
3. **Debugging**: Go directly to [Common Mistakes](./11-common-mistakes.md) or find the concept in the specific chapter
4. **Interview**: Use [Critical Distinctions](./CRITICAL-DISTINCTIONS.md) to think through concepts, then [Interview Questions](./19-interview-questions.md) to practice
5. **Production**: Read [Reliability](./12-reliability.md) → [Security](./13-security.md) → [Observability](./14-observability.md) → [Deployment](./15-deployment.md) sequentially

## Verified APIs in This Documentation

- ✅ Agent initialization and parameters
- ✅ Task definition and context passing
- ✅ Crew orchestration and kickoff()
- ✅ Sequential and Hierarchical processes
- ✅ Flow decorators (@start, @listen, @router)
- ✅ Flow state management
- ✅ Tool creation and schema
- ✅ CrewOutput and TaskOutput structures
- ✅ Memory and guardrails
- ✅ Callbacks and observability
- ✅ Token usage tracking
- ✅ Failure modes and recovery

## Not Covered (Out of Scope)

- CrewAI Cloud/AMP platform REST API
- UI-specific deployment details
- Third-party integrations (Slack, Email, etc.) beyond concepts
- Older CrewAI versions (<1.15)

---

**Next**: Start with [What is CrewAI?](./01-what-is-crewai.md)
