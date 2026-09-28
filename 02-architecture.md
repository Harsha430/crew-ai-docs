# CrewAI Architecture

## System Overview

### High-Level Flow

```
┌─────────────────────────────────────────────────────────────┐
│ APPLICATION                                                 │
│ ┌───────────────────────────────────────────────────────┐   │
│ │ FLOW (Control Flow Layer)                             │   │
│ │ ┌──────────┐ ┌──────────┐ ┌──────────┐               │   │
│ │ │ @start   │→│ @listen  │→│ @router  │→ ...          │   │
│ │ └──────────┘ └──────────┘ └──────────┘               │   │
│ │     ↓            ↓            ↓                        │   │
│ │ (State, Routing, Conditions, HITL)                    │   │
│ └───────────────────────────────────────────────────────┘   │
│                        ↓                                     │
│ ┌───────────────────────────────────────────────────────┐   │
│ │ CREW (Task Orchestration Layer)                       │   │
│ │ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   │   │
│ │ │ PROCESS      │ │ MEMORY       │ │ CALLBACKS    │   │   │
│ │ │ - Sequential │ │ - History    │ │ - Events     │   │   │
│ │ │ - Hierarchal │ │ - Entities   │ │ - Metrics    │   │   │
│ │ └──────────────┘ └──────────────┘ └──────────────┘   │   │
│ │        ↓                                               │   │
│ │ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │   │
│ │ │ TASK 1   │→│ TASK 2   │→│ TASK 3   │→│ TASK N   │ │   │
│ │ │ context: │ │ context: │ │ context: │ │ context: │ │   │
│ │ │ [input]  │ │ [out 1]  │ │ [out 2]  │ │ [out N-1]│ │   │
│ │ └──────────┘ └──────────┘ └──────────┘ └──────────┘ │   │
│ │        ↓           ↓           ↓           ↓         │   │
│ │    ┌─────────────────────────────────────────┐       │   │
│ │    │ CrewOutput (aggregated results)         │       │   │
│ │    │ - raw, json_dict, pydantic              │       │   │
│ │    │ - tasks_output: TaskOutput[]            │       │   │
│ │    │ - token_usage                           │       │   │
│ │    └─────────────────────────────────────────┘       │   │
│ └───────────────────────────────────────────────────────┘   │
│                        ↓                                     │
│ ┌───────────────────────────────────────────────────────┐   │
│ │ OUTPUT HANDLING                                       │   │
│ │ - Store result                                        │   │
│ │ - Human approval                                      │   │
│ │ - Route to next Flow step                             │   │
│ └───────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Task Execution Layer

```
┌──────────────────────────────────────────────────────┐
│ TASK EXECUTION                                       │
│ ┌────────────────────────────────────────────────┐   │
│ │ TASK                                           │   │
│ │ - description: str                             │   │
│ │ - expected_output: str                         │   │
│ │ - agent: Agent                                 │   │
│ │ - context: [Task] (previous task outputs)      │   │
│ │ - guardrail: Callable                          │   │
│ │ - human_input: bool                            │   │
│ └────────────────────────────────────────────────┘   │
│              ↓                                        │
│ ┌────────────────────────────────────────────────┐   │
│ │ AGENT EXECUTION LOOP                           │   │
│ │                                                 │   │
│ │ iteration = 0                                   │   │
│ │ while iteration < max_iter:                    │   │
│ │   ├─ LLM thinks (given context + tools)        │   │
│ │   ├─ Decision:                                 │   │
│ │   │  ├─ Call tool? → Execute tool → Get result │   │
│ │   │  │                                          │   │
│ │   │  └─ Return answer                           │   │
│ │   └─ iteration += 1                             │   │
│ │                                                 │   │
│ └────────────────────────────────────────────────┘   │
│              ↓                                        │
│ ┌────────────────────────────────────────────────┐   │
│ │ OUTPUT VALIDATION (Guardrail)                  │   │
│ │                                                 │   │
│ │ if guardrail(output):                          │   │
│ │   ├─ PASS → Task complete                      │   │
│ │   └─ Return TaskOutput                         │   │
│ │ else:                                          │   │
│ │   ├─ FAIL → Retry or escalate                 │   │
│ │   └─ feedback to agent → Re-execute           │   │
│ │                                                 │   │
│ └────────────────────────────────────────────────┘   │
│              ↓                                        │
│ ┌────────────────────────────────────────────────┐   │
│ │ TaskOutput                                      │   │
│ │ - description: str                             │   │
│ │ - summary: str                                 │   │
│ │ - raw: str                                     │   │
│ │ - pydantic: PydanticModel (optional)           │   │
│ │ - json_dict: dict (optional)                   │   │
│ │ - agent: Agent                                 │   │
│ │ - output_format: Enum                          │   │
│ │ - token_usage: {...}                           │   │
│ └────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────┘
```

### Agent Execution Details

```
┌────────────────────────────────────────────────────────────┐
│ AGENT                                                      │
│                                                            │
│ ┌──────────────────────────────────────────────────────┐  │
│ │ Identity                                             │  │
│ │ - role: "Research Expert"                            │  │
│ │ - goal: "Find accurate information"                  │  │
│ │ - backstory: context/persona                         │  │
│ └──────────────────────────────────────────────────────┘  │
│              ↓                                             │
│ ┌──────────────────────────────────────────────────────┐  │
│ │ Capabilities                                         │  │
│ │ ┌─────────────────┐ ┌────────────────────────────┐  │  │
│ │ │ LLM             │ │ Tools                      │  │  │
│ │ │ - Model name    │ │ - web_search              │  │  │
│ │ │ - Temperature   │ │ - api_query               │  │  │
│ │ │ - Max tokens    │ │ - file_read               │  │  │
│ │ │ - API key       │ │ - ... (agent-specific)    │  │  │
│ │ └─────────────────┘ └────────────────────────────┘  │  │
│ │                                                      │  │
│ │ ┌─────────────────┐ ┌────────────────────────────┐  │  │
│ │ │ Memory          │ │ Constraints                │  │  │
│ │ │ - Enabled       │ │ - max_iter: 10             │  │  │
│ │ │ - Storage       │ │ - max_retry_limit: 2      │  │  │
│ │ │ - Retrieval     │ │ - respect_context_window  │  │  │
│ │ └─────────────────┘ └────────────────────────────┘  │  │
│ └──────────────────────────────────────────────────────┘  │
│              ↓                                             │
│ ┌──────────────────────────────────────────────────────┐  │
│ │ Execution (for each iteration)                       │  │
│ │                                                      │  │
│ │ 1. Format prompt                                    │  │
│ │    - role, goal, backstory                          │  │
│ │    - task description                               │  │
│ │    - tool descriptions                              │  │
│ │    - context (previous task outputs)                │  │
│ │                                                      │  │
│ │ 2. Call LLM                                         │  │
│ │    - LLM generates thought + decision               │  │
│ │                                                      │  │
│ │ 3. Parse response                                   │  │
│ │    - Tool call? → Yes/No                            │  │
│ │    - Which tool?                                    │  │
│ │    - What parameters?                               │  │
│ │                                                      │  │
│ │ 4. Decision                                         │  │
│ │    ├─ If tool call:                                 │  │
│ │    │  ├─ Tool allowed? (check permissions)          │  │
│ │    │  ├─ Execute tool                               │  │
│ │    │  ├─ Get result                                 │  │
│ │    │  └─ Continue (next iteration)                  │  │
│ │    └─ If final answer:                              │  │
│ │       └─ Return answer                              │  │
│ │                                                      │  │
│ │ 5. Increment iteration counter                      │  │
│ │    - If max_iter reached → Return answer            │  │
│ │                                                      │  │
│ └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

## Components Deep Dive

### LLM Component

```
Agent → LLM Provider Abstraction
            ↓
        ├─ OpenAI (GPT-4, etc.)
        ├─ Anthropic (Claude)
        ├─ Groq
        ├─ Azure OpenAI
        ├─ Local/Ollama
        └─ Custom provider
            ↓
        Configuration:
        ├─ model_name: "gpt-4"
        ├─ api_key: from environment
        ├─ temperature: 0.7
        ├─ max_tokens: 4096
        ├─ base_url: custom endpoint
        └─ timeout: 30s
```

**Why separated**: Easy to swap providers, configure per-agent, implement fallback.

### Tool Component

```
Tool = Function + Schema
    ├─ Function: Actual implementation
    ├─ Name: What agent sees
    ├─ Description: What agent understands
    ├─ Parameters: Input schema (Pydantic)
    └─ Return: Output to agent
    
Process:
    Agent → "I'll call web_search"
       ↓
    CrewAI validates:
       ├─ Does agent have permission?
       ├─ Are parameters valid?
       └─ Does agent allow_delegation allow it?
       ↓
    Execute function
       ↓
    Return result to agent
```

### Memory Component

```
Memory = Persistent Context

Types:
├─ Short-term (current execution)
│   └─ Current task context + recent iterations
│
├─ Long-term (across crew runs)
│   └─ All interactions, retrieved by relevance
│
└─ Entity memory (specific facts)
    └─ Key entities and their attributes

Retrieval:
    Agent + Context
       ↓
    Similarity search (embeddings)
       ↓
    Relevant memories retrieved
       ↓
    Injected into agent prompt
```

### Process Component

```
Process = Orchestration Strategy

Sequential:
    Task 1 → Task 2 → Task 3 → Done
    
    Execution:
    ├─ Task 1 starts
    ├─ Task 1 completes
    ├─ Task 2 starts (gets Task 1 output as context)
    ├─ Task 2 completes
    └─ ...
    
    Properties:
    ├─ Deterministic
    ├─ Easy to debug
    ├─ Clear dependencies
    └─ Limited parallelism

Hierarchical:
    Manager Agent
    ├─ Analyzes all tasks
    ├─ Decides order and assignment
    ├─ Delegates to Agents
    └─ Synthesizes final answer
    
    Execution:
    ├─ Manager gets tasks
    ├─ Manager decides "Agent A → Task 1"
    ├─ Agent A completes
    ├─ Manager decides "Agent B → Task 2"
    ├─ Agent B completes
    ├─ Manager combines results
    └─ Done
    
    Properties:
    ├─ Model-driven (LLM decides)
    ├─ Flexible
    ├─ Harder to debug
    ├─ Can parallelize
    └─ Can handle dynamic task count
```

### Flow Component

```
Flow = Application-Level Workflow

States:
    Unexecuted → Starting → Listening → Routing → Decision → Next/Done

Structure:
    @start()
        ↓ (returns value)
    @listen(start)
        ↓ (receives start's return value)
        ↓ (can do work, return value)
    @router(previous_listen)
        ↓ (receives listen's return value)
        ↓ (returns route string)
    @listen(route_string)
        ↓ (receives router's route value)
        ↓ (can do work)
    ... more routes and listeners ...
    
Execution:
    flow.kickoff()
        ├─ Execute @start()
        ├─ For each @listen(previous_output):
        │  ├─ Execute @listen
        │  └─ If has @router:
        │     ├─ Call @router to get route
        │     └─ Go to next @listen(route) or end
        └─ Return final value

State Lifecycle:
    Created (empty) → Updated by each step → Can be queried by steps → Lost after flow ends
    (No automatic persistence)
```

## Cross-Cutting Concerns

### Observability

```
Every operation records:

Agent Think:
├─ Input (task + context)
├─ LLM call (model, tokens)
├─ Decision (tool? / answer?)
├─ Latency
└─ Cost

Tool Call:
├─ Which tool
├─ Parameters
├─ Execution time
├─ Result
└─ Error (if failed)

Task Completion:
├─ Duration
├─ Token usage
├─ Tool calls made
├─ Guardrail pass/fail
├─ Retry count
└─ Final output

Crew Execution:
├─ Total duration
├─ Total tokens
├─ Total cost
├─ Tasks completed
├─ Failures/retries
└─ Final output quality
```

### Error Handling

```
Error Categories:

LLM Error:
├─ Rate limit → Retry with backoff
├─ Timeout → Fallback to other model
├─ Invalid token → Fail
└─ Invalid API key → Fail fast

Tool Error:
├─ Input validation fail → Retry (agent fixes params)
├─ Execution fails → Fallback tool or skip
├─ Timeout → Retry or fallback
└─ Permission denied → Inform agent (tool unavailable)

Task Error:
├─ Guardrail fails → Agent retries
├─ Max iterations reached → Timeout/fail
├─ Agent internal error → Escalate
└─ Context missing → Fail

Crew Error:
├─ Task dependency broken → Skip dependent tasks
├─ Manager delegation fails → Escalate
└─ Overall timeout → Stop and return partial
```

### Security Boundaries

```
User Input
    ↓
Sanitization Layer
├─ Length limits
├─ Type validation
└─ Schema enforcement
    ↓
Agent Permission Layer
├─ Which tools can this agent use?
├─ Can agent delegate?
├─ Can agent use specific data sources?
└─ Rate limits per agent
    ↓
Tool Execution Layer
├─ Scope: Tool only accesses what it needs
├─ Isolation: Tool can't access other tools' resources
├─ Audit: All tool calls logged
└─ Retry: Tool can fail safely
    ↓
Output Validation Layer
├─ Does output contain PII?
├─ Is output factually grounded?
├─ Does output match expected format?
└─ Is output safe to send to user?
```

## Data Flow Example: Research Task

```
User: "Research AI trends for 2026"
    ↓
Flow.kickoff({"topic": "AI trends"})
    ↓
@start() returns topic
    ↓
@listen(start) receives topic
    ├─ Creates Crew with Research task
    ├─ Crew.kickoff() called
    │   └─ Task receives:
    │       ├─ description: "Research AI trends for 2026"
    │       ├─ agent: ResearchAgent
    │       ├─ context: [] (no previous tasks)
    │       └─ guardrail: output_must_be_comprehensive
    │   ↓
    │   Agent starts:
    │   ├─ LLM: "I should search for AI trends"
    │   ├─ Tool call: web_search("AI trends 2026")
    │   ├─ Tool returns: [search results]
    │   ├─ LLM: "I should search for more specifics"
    │   ├─ Tool call: web_search("generative AI 2026")
    │   ├─ Tool returns: [more results]
    │   ├─ LLM: "I have enough, let me compile"
    │   └─ Returns: "Here are the top 10 AI trends..."
    │   ↓
    │   Guardrail:
    │   ├─ Check: len(output) > 1000? YES
    │   ├─ Check: "trend" in output? YES
    │   └─ PASS
    │   ↓
    │   TaskOutput created:
    │   ├─ raw: "Here are the top 10 AI trends..."
    │   ├─ agent: ResearchAgent
    │   └─ token_usage: {input: 2500, output: 1200}
    ├─ Crew returns: CrewOutput with tasks_output = [TaskOutput]
    └─ @listen returns CrewOutput
    ↓
@router(crew_output) checks quality
├─ Quality score: 0.92
├─ Decision: "high quality"
└─ Returns: "publish"
    ↓
@listen("publish") receives crew output
├─ Stores results
└─ Returns result to user
```

## Why This Architecture?

| Component | Purpose | Benefit |
|-----------|---------|---------|
| **Flow** | Application control | Separate business logic from agent logic |
| **Crew** | Agent orchestration | Reusable multi-agent orchestration |
| **Process** | Task sequencing | Different workflows for different needs |
| **Task** | Structured work | Validation, context passing, guardrails |
| **Agent** | Specialized AI | Roles, tools, memory |
| **Tool** | External capability | Safe, controlled external interaction |
| **Memory** | Context persistence | Agents build on previous work |
| **Guardrail** | Output validation | Ensure quality before proceeding |

---

**Next**: [Agent](./03-agent.md)
