# Critical Distinctions: Must Understand These

**These are the most common confusions. Master these distinctions for interviews.**

## 1. Agent vs Task

| Agent | Task |
|-------|------|
| **WHO** performs work | **WHAT** needs to be done |
| Identity: role, goal, backstory | Work description: what output should look like |
| Capabilities: tools, memory, reasoning | Instructions: expected output, guardrails |
| Long-lived (can do many tasks) | Short-lived (one execution) |
| ` agent = Agent(role="...", goal="...")` | `task = Task(description="...", agent=agent)` |
| Example: "I am a researcher" | Example: "Research AI trends" |
| You have 1 agent, 1 task is like "a person, one assignment" | |

**Interview Q**: "What's the difference between Agent and Task?"

**Answer**: Agent is WHO (the identity and capabilities), Task is WHAT (the specific work to do). One agent can perform many tasks. One task uses one agent. Agent = "Researcher person", Task = "Researched AI document".

## 2. Crew vs Flow

| Crew | Flow |
|------|------|
| **Autonomous** execution | **Application-level** workflow |
| Fixed orchestration (process) | Flexible routing and conditions |
| Agents complete tasks deterministically | Application controls flow |
| Returns: `CrewOutput` | Returns: whatever last step returns |
| When: Structured workflows | When: Complex business logic |
| `crew.kickoff()` | `flow.kickoff()` |
| "Execute these tasks in order" | "If X happens, route to Y, else Z" |
| Example: Research → Write → Review | Example: Research → if good? → Approve/Revise |

**Interview Q**: "When do you use Crew vs Flow?"

**Answer**: Crew when you have a fixed sequence of tasks (research, then write, then review). Flow when you need conditional logic, branching, human approval, or multi-step application workflows. Crew is "execute", Flow is "orchestrate".

## 3. Process vs Flow

| Process | Flow |
|---------|------|
| **Inside Crew**: How tasks are ordered | **Outside Crew**: Application-level control |
| Sequential or Hierarchical | Decorators: @start, @listen, @router |
| Agent behavior (manager delegates) | Application behavior (routing) |
| Part of Crew configuration | Separate from Crew |
| `process=Process.sequential` | `class MyFlow(Flow): ...` |
| "How do I sequence these tasks?" | "How does my whole application flow?" |

**Interview Q**: "Explain Process vs Flow"

**Answer**: Process determines how tasks within a Crew are ordered (sequential = task1→task2→task3, or hierarchical = manager decides order). Flow is application-level workflow that routes between steps, Crews, and conditions. Flow contains Process.

## 4. Sequential vs Hierarchical Process

| Sequential | Hierarchical |
|-----------|-------------|
| Task 1 → Task 2 → Task 3 | Manager Agent decides order |
| Deterministic order | LLM-driven (flexible but less predictable) |
| Easy to debug | Complex to debug |
| Agent doesn't decide order | Manager Agent decides task assignment |
| `process=Process.sequential` | `process=Process.hierarchical` |
| Predictable execution path | Execution depends on LLM reasoning |
| Example: Research → Write → Review | Example: Manager sees all tasks, picks who does what |

**Interview Q**: "When do you use hierarchical vs sequential?"

**Answer**: Sequential when you have a clear task order that shouldn't change. Hierarchical when you want a manager agent to dynamically decide task assignment based on reasoning (more flexible but less predictable). Sequential is more common in production.

## 5. Delegation vs Handoff vs Task Context

| Delegation | Task Context | Handoff |
|-----------|------|---------|
| Agent delegates to another agent | Agent2 receives Agent1's output | Not a CrewAI term |
| Hierarchical process only | Sequential process | Used in other frameworks |
| Manager says "do this for me" | Task2 description says "use previous result" | Similar to handoff but CrewAI uses context |
| `allow_delegation=True` | `context=[task1]` | N/A |
| "Manager → Specialist" | "Task 1 result → Task 2 input" | |

**Interview Q**: "What's the difference between delegation and task context?"

**Answer**: Delegation (hierarchical) is when one agent asks another agent to do work ("I'll delegate this to you"). Task context (sequential) is when one task output becomes the input/context to another task ("here's what the previous agent found"). Different mechanisms, different use cases.

## 6. State vs Memory

| State | Memory |
|-------|--------|
| **Current** workflow data | **Historical** knowledge |
| Per-flow-execution only | Persists across executions |
| Mutable dictionary | Retrieved by relevance |
| `self.state["key"] = value` in Flow | Enabled with `memory=True` in Crew |
| Application-level | Agent-level |
| No automatic persistence | Can be stored in database |
| Example: `state["findings"] = "..."` | Example: Agent remembers it researched X before |

**Interview Q**: "What's the difference between Flow state and agent memory?"

**Answer**: State is temporary workflow data (this Flow run only, stored in `self.state`). Memory is persistent knowledge (agents remember what happened before, retrieved by relevance). State is application-level, Memory is agent-level. State is lost when Flow ends; Memory persists.

## 7. Expected Output vs Guardrail

| Expected Output | Guardrail |
|---------|----------|
| **Contract**: What should the output look like? | **Validation**: Does output actually meet requirements? |
| Description in task definition | Function that validates output |
| Guidance to agent | Enforcement rule |
| `expected_output="Markdown document"` | `guardrail=validate_output_function` |
| "I expect a detailed report" | "Output must be >500 words AND contain 'key finding'" |
| Soft guidance | Hard validation |
| Affects prompt | Affects execution (retry if fails) |

**Interview Q**: "What's the difference between expected_output and guardrail?"

**Answer**: expected_output is what you tell the agent you want ("a 10-page report"). Guardrail is what you enforce ("output must be >1000 words, and if not, retry"). Expected output is guidance; guardrail is validation.

## 8. Tool vs API vs Tool Calling

| Tool | API | Tool Calling |
|------|-----|------|
| CrewAI abstraction: function + schema | External HTTP endpoint | LLM deciding to use a tool |
| Agent calls it via CrewAI | External system | LLM behavior |
| `@tool def search(): ...` | `POST /search` | Model sees tool, decides "I'll use it" |
| Sandboxed, controlled | No sandbox | LLM capability |
| Tool result → Agent context | Requires client code | Model decision |

**Interview Q**: "What is tool calling?"

**Answer**: Tool calling is the LLM's ability to decide "I should call this tool" and format the call correctly. It's a model capability. The "tool" in CrewAI is the abstraction: function + schema. Tool calling is the LLM behavior that uses tools.

## 9. Tool ≠ API

A tool can call an API internally, but they're different:

```python
# API: External endpoint
POST /v1/search?q=AI

# Tool: CrewAI wrapper around API + API
@tool
def search(query: str) -> str:
    """Search the web"""
    response = requests.get("https://api.search.com/v1/search", 
                            params={"q": query})
    return response.json()

agent = Agent(tools=[search])  # Agent uses tool, not API directly
```

Tool = Agent capability  
API = External service

## 10. max_iter vs Tool Calls

| max_iter | Tool Calls |
|----------|-----------|
| **Thinking iterations** | **Number of tool invocations** |
| Agent thinking steps | Actual tool executions |
| One iteration can have 0-N tool calls | One per decision |
| `max_iter=10` = 10 thinking loops | Not 10 tool calls |
| Includes "think about result" iteration | Only actual executions |
| ❌ "max_iter=10 means 10 tools" | ✅ "max_iter=10 means 10 thinking steps" |

**Example**:
```
Iteration 1: Think → call tool A → get result (1 tool call)
Iteration 2: Think → call tools B, C → get results (2 tool calls)
Iteration 3: Think → return answer (0 tool calls)
= 3 iterations, 3 tool calls
```

**Interview Q**: "What does max_iter actually control?"

**Answer**: max_iter controls thinking iterations, NOT tool calls. Each iteration the agent thinks and may call a tool (or may not). So max_iter=10 allows potentially many tool calls, not exactly 10.

## 11. Retry vs Fallback

| Retry | Fallback |
|-------|----------|
| **Same approach again** | **Different approach** |
| Try the same tool again | Use different tool or agent |
| Tool failed → retry | Tool fails → use backup tool |
| `max_retry_limit=2` | Custom logic: `if tool_fails: use_backup` |
| "Try again" | "Try something else" |
| Fixed limit | Custom decision |
| Example: API timeout → retry | Example: API fails → use cached data |

**Interview Q**: "When do you retry vs fallback?"

**Answer**: Retry when the failure is transient (network hiccup, temporary rate limit). Fallback when the approach itself won't work (API permanently down, tool not appropriate). Retry same thing, fallback to different thing.

## 12. Error vs Timeout vs Interruption

| Error | Timeout | Interruption |
|-------|---------|------|
| Tool/LLM returned error | Operation took too long | User/system stopped execution |
| `max_retry_limit` applies | Enforced by timeout setting | Manual stop |
| Recoverable (usually) | Not recoverable | Partial results |
| "Tool failed" | "Took too long" | "Stopped by user" |

## 13. CrewOutput vs TaskOutput

| CrewOutput | TaskOutput |
|-----------|-----------|
| **All tasks combined** | **One task result** |
| `crew.kickoff()` returns | Each element of `tasks_output[]` |
| `raw`: Final aggregated answer | `raw`: That task's output |
| `tasks_output`: List of all task results | `description`: That task's description |
| `token_usage`: Total tokens | `token_usage`: That task's tokens |
| Contains multiple TaskOutputs | Single task result |

**Access**:
```python
result = crew.kickoff()
all_tasks = result.tasks_output  # CrewOutput.tasks_output
first_task = result.tasks_output[0]  # TaskOutput
first_task_tokens = first_task.token_usage  # That task's tokens
all_tokens = result.token_usage  # All tasks' total
```

## 14. raw vs json_dict vs pydantic

| raw | json_dict | pydantic |
|-----|-----------|----------|
| **Plain text** | **Python dict** | **Typed object** |
| Always present | Only if valid JSON | Only if configured |
| `str` type | `dict` type | `BaseModel` type |
| `result.raw` | `result.json_dict` | `result.pydantic` |
| "Here's the answer..." | `{"key": "value"}` | `MyModel(key="value")` |
| No structure | Structured but untyped | Structured and typed |

**When each exists**:
```python
# All results have raw
print(result.raw)  # Always works

# json_dict only if output was JSON
if result.json_dict:
    print(result.json_dict)  # Might be None

# pydantic only if task has output_pydantic
if result.pydantic:
    print(result.pydantic)  # Might be None
```

## 15. Structured Output Formats

| Raw Only | JSON | Pydantic |
|----------|------|----------|
| No special config | `output_json=MyModel` | `output_pydantic=MyModel` |
| `result.raw` | `result.json_dict` or `result.raw` | `result.pydantic` or `result.raw` |
| Unstructured | Parsed JSON dict | Typed object |
| Agent decides format | Agent outputs JSON | Agent outputs JSON, CrewAI parses to Pydantic |
| Fastest | Balanced | Most reliable |

## 16. @start vs @listen vs @router

| @start | @listen | @router |
|-------|---------|---------|
| **Entry point** | **Listener to previous** | **Route decision** |
| One per Flow | Decorates method | Decorates method |
| `@start()` | `@listen(source_method_or_string)` | `@router(source_method)` |
| Returns initial value | Receives previous value | Receives previous value, returns route string |
| Called first | Called when source completes | Called to decide next step |
| Example: `return user_input` | Example: `receive input, do work` | Example: `if good: return "approve" else "revise"` |

**Flow**:
```
@start() → returns X
    ↓
@listen(start) → receives X, returns Y
    ↓
@router(listen) → receives Y, returns "route_a" or "route_b"
    ↓
@listen("route_a") → called if router returned "route_a"
```

## 17. Crew Manager vs Agent Manager

| Crew Manager Agent | Agent Manager Agent |
|------------------|-----|
| Orchestrates tasks | Doesn't exist (confusion) |
| `manager_agent` in Crew | Hierarchical process agent |
| Hierarchical process only | Specialized agent with role "Manager" |
| Decides task order/assignment | NOT the orchestrator |

**Correct**: In hierarchical process, the `manager_agent` is the orchestrator agent that gets all tasks and decides the plan.

## 18. Token Input vs Token Output

| Input Tokens | Output Tokens |
|--------------|---------|
| Tokens you send TO LLM | Tokens LLM generates FOR you |
| Your prompt + context | Agent's thinking + answer |
| Cheaper (usually 1x cost) | More expensive (usually 3-5x cost) |
| Increase with context size | Increase with answer length |
| Can optimize by removing context | Can optimize by constraining max_tokens |

**Cost calculation**:
```
Total cost = (input_tokens / 1M) * input_price + (output_tokens / 1M) * output_price

For Claude 3.5 Sonnet:
Total cost = (input_tokens / 1M) * $3 + (output_tokens / 1M) * $15
```

## 19. Callback vs Guardrail

| Callback | Guardrail |
|----------|----------|
| **Observe** execution | **Validate** output |
| Triggered at events | Triggered after task |
| No control (info only) | Can retry/reject |
| `callback=my_observer` | `guardrail=my_validator` |
| "Log that this happened" | "Check if this is good" |

## 20. Memory ≠ Storage

| Memory | Storage |
|--------|---------|
| Agent-level abstraction | Backend (database, file) |
| Automatic retrieval by relevance | Persistent storage |
| `memory=True` enables | LanceDB or custom backend |
| In-agent context | Outside agent |
| "Agent remembers" | "Data is stored somewhere" |

---

## Quick Interview Test

Can you answer these without looking above?

1. **Agent vs Task**: WHO vs WHAT
2. **Crew vs Flow**: Fixed orchestration vs Application control
3. **State vs Memory**: Temporary workflow data vs Persistent knowledge
4. **max_iter**: Thinking iterations, not tool calls
5. **Expected Output vs Guardrail**: Guidance vs Enforcement
6. **Delegation**: Hierarchical only, manager assigns work
7. **CrewOutput vs TaskOutput**: All tasks vs One task
8. **raw vs json_dict vs pydantic**: Text vs Dict vs Typed
9. **@start vs @listen vs @router**: Entry vs Listener vs Decision
10. **Sequential vs Hierarchical**: Deterministic order vs Manager decides

---

**Next**: [Interview Questions](./19-interview-questions.md)
