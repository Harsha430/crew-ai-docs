# CrewAI Interview Questions (150+)

**Use these to prepare for GenAI/Agentic AI interviews**

---

## Level 1: Fundamentals (30 Questions)

### Conceptual

**Q1: What is CrewAI?**  
A: A Python framework for building multi-agent systems where specialized agents (with tools, memory, reasoning) collaborate on tasks through orchestrated processes (sequential or hierarchical) within crews or flows.

**Q2: What problem does CrewAI solve?**  
A: Single LLM calls lack specialization, orchestration, memory, validation, and observability. CrewAI enables building teams of specialized agents that work together reliably.

**Q3: What is the core mental model?**  
A: Agent (WHO) + Task (WHAT) + Crew (collaboration) + Process (orchestration) + Flow (application control).

**Q4: When should you use CrewAI instead of just calling an LLM API?**  
A: When you need multiple specialized agents, task dependencies, validation, memory across tasks, or complex workflows. For simple single-request LLM calls, use OpenAI/Anthropic SDK directly.

**Q5: What's the relationship between Agent and Task?**  
A: Agent is WHO (identity, tools, capabilities). Task is WHAT (specific work). One agent can do many tasks; one task uses one agent.

**Q6: What's the relationship between Crew and Flow?**  
A: Crew is autonomous task orchestration (fixed structure). Flow is application-level control with routing, conditions, and state.

**Q7: When would you use a Flow?**  
A: When you need conditional routing, human approval loops, multiple crews, or state management across steps.

**Q8: What is delegation?**  
A: In hierarchical process, a manager agent decides task assignment and delegates to specialist agents.

### Core Components

**Q9: List the core CrewAI components**  
A: Agent, Task, Crew, Process, Flow, Tool, Memory, Guardrail.

**Q10: What does an Agent contain?**  
A: Role, goal, backstory, tools, LLM, memory, constraints (max_iter, max_retry_limit), reasoning level.

**Q11: What does a Task contain?**  
A: Description, expected_output, agent assignment, context (previous task outputs), guardrail, callbacks.

**Q12: What does a Crew contain?**  
A: Agents, tasks, process (orchestration strategy), memory, callbacks.

**Q13: What processes does CrewAI support?**  
A: Sequential (task1 → task2 → task3) and Hierarchical (manager decides order).

**Q14: What does a Tool consist of?**  
A: Function + schema (name, description, parameters).

**Q15: What is memory?**  
A: Persistent context that agents retrieve by relevance to inform thinking.

### Outputs

**Q16: What does crew.kickoff() return?**  
A: CrewOutput object with: raw (text), json_dict (if valid JSON), pydantic (if structured), tasks_output (list of TaskOutput), token_usage.

**Q17: What is TaskOutput?**  
A: Result from one task containing: description, summary, raw output, pydantic, json_dict, agent reference, token_usage.

**Q18: What does flow.kickoff() return?**  
A: Whatever the last @listen method returns (could be CrewOutput, string, dict, any Python object).

**Q19: What is flow state?**  
A: Mutable dictionary (self.state) that persists across flow steps within one execution.

**Q20: How do you access token usage?**  
A: `result.token_usage["total_tokens"]` or `result.tasks_output[i].token_usage`.

### Execution

**Q21: Describe agent execution loop**  
A: Agent thinks (LLM generates) → decides (tool call or final answer) → if tool: execute and continue, if answer: return.

**Q22: What does max_iter control?**  
A: Thinking iterations (not tool calls). Each iteration: think + maybe tool call. max_iter=10 allows many tool calls.

**Q23: What happens at guardrail?**  
A: Task output validated. Pass → task complete. Fail → retry or escalate.

**Q24: How is context passed between tasks?**  
A: Task2 has `context=[Task1]` → Task1 output injected into Task2 prompt.

**Q25: How do agents learn?**  
A: Through memory (long-term) and recent context (short-term).

### Best Practices

**Q26: When should you set verbose=True?**  
A: Development/debugging only. Not in production (wastes tokens).

**Q27: What's a reasonable max_iter?**  
A: 10-15 for most tasks. Increase if agent needs many tool calls.

**Q28: How do you prevent agent loops?**  
A: Set reasonable max_iter, clear role/goal, good tool descriptions.

**Q29: When should you enable memory?**  
A: When agents work on related tasks that build on each other.

**Q30: Should you give one agent many tools or few?**  
A: Few well-described tools (3-5) is better than many. Agent gets confused with too many options.

---

## Level 2: API & Implementation (35 Questions)

### Agent Configuration

**Q31: What are required Agent parameters?**  
A: `role` and `goal`. Everything else optional.

**Q32: What does backstory do?**  
A: Provides persona/context to LLM to inform reasoning and behavior.

**Q33: How do you configure the LLM?**  
A: Pass `llm` parameter: `llm=ChatOpenAI(model="gpt-4", temperature=0.3)`.

**Q34: What does allow_delegation do?**  
A: If True, agent can ask other agents to help. Used in hierarchical process.

**Q35: What's respect_context_window?**  
A: If True, agent stops rather than exceeding token context limit.

**Q36: How do you set up agent tools?**  
A: `tools=[tool1, tool2, ...]` where each is a @tool decorated function.

**Q37: Can one agent have tools from different domains?**  
A: Yes, but it's confusing for the agent. Better to give agent tools for its specialized domain.

**Q38: How do you disable agent memory?**  
A: `memory=False`.

### Task Configuration

**Q39: What are required Task parameters?**  
A: `description`, `expected_output`, and `agent`.

**Q40: How do you set task context?**  
A: `context=[task1, task2, ...]` - previous task outputs injected into prompt.

**Q41: What's output_json vs output_pydantic?**  
A: output_json: agent outputs JSON (parsed to dict). output_pydantic: agent outputs JSON parsed to typed Pydantic model.

**Q42: How do you add a guardrail?**  
A: `guardrail=my_validation_function` that returns True/False.

**Q43: How do you add a callback to a task?**  
A: `callback=my_callback_function` triggered during execution.

**Q44: What's human_input?**  
A: If True, task pauses and requests human feedback before completing.

**Q45: Can one task use multiple agents?**  
A: No, one task = one agent. For multiple agents, use multiple tasks + context.

### Crew Configuration

**Q46: What are required Crew parameters?**  
A: `agents` and `tasks`. Process defaults to sequential.

**Q47: How do you set crew process?**  
A: `process=Process.sequential` or `process=Process.hierarchical`.

**Q48: How do you configure hierarchical process?**  
A: `process=Process.hierarchical` + `manager_agent=your_agent`.

**Q49: How do you enable crew memory?**  
A: `memory=True` in Crew initialization.

**Q50: Can you use same agent in multiple crews?**  
A: Yes, same agent object can be in multiple crews.

**Q51: What does crew.kickoff(inputs={...}) do?**  
A: Passes inputs to crew; typically used in tasks with placeholders like {topic}.

**Q52: Can you kickoff same crew multiple times?**  
A: Yes, each kickoff is independent execution.

### Flow Configuration

**Q53: What does @start() do?**  
A: Marks entry point method, called first, returns initial value.

**Q54: What does @listen(source) do?**  
A: Marks method as listener to another method; receives its return value.

**Q55: What does @router(source) do?**  
A: Marks method that receives source output and returns route string.

**Q56: How do you route to different @listen methods?**  
A: @router returns string route name; @listen("route_name") handles it.

**Q57: How do you access flow state?**  
A: `self.state["key"]` within Flow methods.

**Q58: Is flow state persisted?**  
A: No, only for one execution. You must implement persistence.

**Q59: Can you create dynamic Flow routes?**  
A: Yes, @router can return any string matching a @listen decorator.

### Tooling

**Q60: How do you create a tool?**  
A: Use @tool decorator: `@tool` above function with docstring as description.

**Q61: What's tool schema?**  
A: Name (function name), description (docstring), parameters (function args with type hints).

**Q62: Can tools have complex parameters?**  
A: Yes, use Pydantic model for complex inputs.

**Q63: What should tool return?**  
A: Anything serializable (str, dict, list). Agent reads and acts on it.

**Q64: How do you handle tool errors?**  
A: Tool can raise exception (agent sees error and retries) or return error message.

**Q65: Can agent create new tools dynamically?**  
A: No, agent can only use tools provided at init.

### Execution & Output

**Q66: How do you handle CrewOutput?**  
A: Check: raw (always), json_dict (if JSON), pydantic (if structured), tasks_output (if multiple tasks), token_usage.

**Q67: How do you iterate task outputs?**  
A: `for task_result in result.tasks_output: ...`

**Q68: How do you check if Pydantic output succeeded?**  
A: `if result.pydantic: ...` (None if not configured or parsing failed).

**Q69: How do you calculate cost?**  
A: `(input_tokens / 1M) * input_price + (output_tokens / 1M) * output_price`.

**Q70: Can you access intermediate task outputs?**  
A: Yes, via `result.tasks_output[i]` for task i.

---

## Level 3: Runtime Output & Debugging (25 Questions)

### Output Structures

**Q71: What's the difference between raw, json_dict, and pydantic?**  
A: raw = plain text, json_dict = parsed JSON dict (untyped), pydantic = parsed JSON as typed Pydantic model.

**Q72: When is json_dict None?**  
A: When task output is not valid JSON or output_json not configured.

**Q73: When is pydantic None?**  
A: When task doesn't have output_pydantic configured or parsing failed.

**Q74: What's the structure of token_usage?**  
A: Dict with "input_tokens", "output_tokens", "total_tokens".

**Q75: Can you modify TaskOutput after creation?**  
A: Shouldn't; TaskOutput is immutable after task completes.

**Q76: What information does TaskOutput.description contain?**  
A: Original task description.

**Q77: What information does TaskOutput.summary contain?**  
A: Agent's summary of what it accomplished.

**Q78: How do you know which agent executed a task?**  
A: `task_output.agent` gives agent reference.

**Q79: Can you distinguish task failures in CrewOutput?**  
A: Check `tasks_output[i].raw` for error message or check for empty/error markers.

**Q80: What's the type of tasks_output?**  
A: list[TaskOutput].

### Debugging

**Q81: Your agent doesn't use tools. What could be wrong?**  
A: Tools not provided, tool description unclear, agent role doesn't suggest tool use, agent thinks it can answer without tools.

**Q82: Your agent loops infinitely. How do you fix it?**  
A: Lower max_iter, check tool results are informative, check role/goal clarity.

**Q83: Your task output is wrong format. What do you check?**  
A: Check expected_output clarity, check output_json/pydantic config, check guardrail validation.

**Q84: Guardrail keeps failing. What's the problem?**  
A: Guardrail too strict, task description unclear to agent, agent can't achieve what guardrail expects.

**Q85: Agent context window exceeded. What happened?**  
A: Too many iterations, too long previous task outputs, too much tool history.

**Q86: Token usage much higher than expected. Why?**  
A: Many iterations, large context, large tool results, many tool calls.

**Q87: Agent calls wrong tool repeatedly. Why?**  
A: Tool description confusing, similar tools, tool results don't help agent.

**Q88: Task context empty or wrong. What's the issue?**  
A: Context task failed, context index wrong, context wasn't properly injected into prompt.

**Q89: Flow enters wrong route. What to check?**  
A: Router logic, route string spelling matches @listen, router receives expected input type.

**Q90: Flow state is None when accessed. Why?**  
A: State not initialized, key doesn't exist (use state.get("key", default) safely).

**Q91: CrewOutput.pydantic is None unexpectedly. Why?**  
A: output_pydantic not configured on task, Pydantic parsing failed, agent didn't output valid JSON.

**Q92: Crew execution hangs. What could cause it?**  
A: Tool making external request that's slow, LLM timeout, agent infinite loop (if max_iter very high).

**Q93: Agent output is completely different than expected. What happened?**  
A: Role/goal confusing, temperature too high, LLM changed, prompt injection via context.

**Q94: Same crew gives different results each run. Is that normal?**  
A: Yes, if temperature > 0 (randomness). Set temperature=0 for deterministic output.

**Q95: How do you test an agent without crew?**  
A: Create task, set agent, call task.execute() directly (if method exists) or verify tool descriptions.

### Production Output

**Q96: Should you return raw or pydantic?**  
A: Pydantic if available (typed, validated). Raw if you need unstructured text.

**Q97: How do you handle failed guardrails in production?**  
A: Check if retry happened; if still failed, log error, either return partial result or escalate.

**Q98: How do you log CrewOutput for monitoring?**  
A: Log raw (or pydantic), token_usage, execution time, success/failure.

**Q99: Should you store raw output or json_dict?**  
A: Store whichever matches your downstream processing; json_dict is more queryable.

**Q100: How do you version outputs?**  
A: Include agent version, LLM model, CrewAI version in metadata with output.

---

## Level 4: Architecture & Design (20 Questions)

### System Design

**Q101: Design a research → write → review crew. What's the process?**  
A: Sequential: Research Agent → Writer Agent → Reviewer Agent. Each task context=[previous].

**Q102: Design a hierarchical crew. When would you use it?**  
A: When manager agent should dynamically decide task assignment based on complexity/team availability.

**Q103: Design human-in-loop approval. How?**  
A: In Flow: @listen for task, @router to check quality, if poor → @listen for human approval.

**Q104: Design agent fallback. How?**  
A: Have multiple agents with similar roles; if first fails, route to second.

**Q105: Design cost optimization. What strategies?**  
A: Lower temperature (deterministic), fewer tools, lower max_iter, use smaller models for simple tasks.

**Q106: Design for multi-tenant. Security considerations?**  
A: Isolate agent memory per tenant, validate tool access, audit logging, separate LLM configurations.

**Q107: Design for high reliability. How?**  
A: Implement retry logic, fallback agents, circuit breaker for external APIs, health checks.

**Q108: Design state persistence in Flow. How?**  
A: Store state to database after each step, implement resume from checkpoint.

**Q109: Design multi-crew orchestration. How?**  
A: Flow coordinates multiple Crews; routes output from Crew 1 to Crew 2 as input.

**Q110: Design agent specialization. How many tools per agent?**  
A: 3-5 closely related tools. More than that, split into multiple specialized agents.

### Integration

**Q111: How do you integrate CrewAI with an API?**  
A: Create tool that calls API, agent uses tool to interact.

**Q112: How do you integrate CrewAI with database?**  
A: Create tools for queries/writes, agents use them.

**Q113: How do you integrate CrewAI with external services (Slack, email)?**  
A: Tools can call these APIs; agent uses tools as needed.

**Q114: Should agents directly call APIs or via tools?**  
A: Always via tools (sandboxed, logged, error handling).

**Q115: How do you version crews for A/B testing?**  
A: Create versions with different agents/prompts, route by experiment flag.

**Q116: How do you handle rate limits from external APIs?**  
A: Tool catches rate limit error, reports to agent, agent retries after backoff.

**Q117: How do you ensure audit trail?**  
A: Log all agent decisions, tool calls, outputs. Store CrewOutput + metadata.

**Q118: How do you handle large outputs?**  
A: If too large, summarize or chunk. Use guardrail to limit output size.

**Q119: How do you handle streaming outputs?**  
A: CrewAI typically collects full output, not streaming. For real-time, would need custom implementation.

**Q120: How do you handle secrets in tools?**  
A: Store in environment variables, don't hardcode. Tool reads from env at runtime.

---

## Level 5: Production & Reliability (20 Questions)

### Reliability

**Q121: What's the difference between retry and fallback?**  
A: Retry = same approach again. Fallback = different approach.

**Q122: How do you implement exponential backoff?**  
A: In tool/wrapper: first retry after 1s, then 2s, 4s, 8s, etc.

**Q123: What's a circuit breaker?**  
A: Pattern: if too many failures, stop trying (circuit open), then periodically retry (half-open), if success resume (close).

**Q124: How do you prevent infinite loops?**  
A: Set max_iter, use guardrails, verify tool feedback is informative.

**Q125: What's graceful degradation?**  
A: If full system fails, use reduced-capability version (fewer agents, fewer tools).

**Q126: How do you handle partial failures?**  
A: If Crew has 5 tasks, if task 3 fails, skip task 3+, summarize partial results.

**Q127: When should you escalate to humans?**  
A: When confidence is low, when changes are risky, when guardrail keeps failing.

**Q128: How do you implement idempotency?**  
A: Tool generates unique ID, checks if already done before executing, returns cached result if exists.

**Q129: How do you timeout long-running tasks?**  
A: Tool/agent has timeout setting; if exceeded, return partial result or error.

**Q130: How do you implement monitoring?**  
A: Log all operations (agent, tool, guardrail), track metrics (success rate, latency, cost, tokens).

### Security

**Q131: How do you prevent prompt injection?**  
A: Sanitize user inputs, separate untrusted data boundaries, use guardrails on output.

**Q132: How do you protect PII?**  
A: Detect and redact before passing to agent, audit outputs for leaks.

**Q133: How do you prevent data leakage via memory?**  
A: Memory is per-user/tenant, validate what gets stored.

**Q134: How do you secure tool permissions?**  
A: Agent only given tools it needs, tool can validate caller, rate limit per agent.

**Q135: How do you handle authentication?**  
A: Tools authenticate to external services, credentials from env, not hardcoded.

**Q136: How do you handle authorization?**  
A: Tool checks if agent allowed to access resource before executing.

**Q137: How do you audit tool calls?**  
A: Log all tool calls (who, what, when, result), store for compliance.

**Q138: Should agents see raw API credentials?**  
A: No, tools handle auth. Agent never sees credentials.

**Q139: How do you handle sensitive outputs?**  
A: Log to secure storage, mask in non-prod, restrict access.

**Q140: How do you verify ground truth?**  
A: Use guardrails, LLM-as-judge, human review for critical decisions.

---

## Level 6: Evaluation (10 Questions)

### Metrics & Assessment

**Q141: What should you measure for agent quality?**  
A: Correctness, instruction-following, tool selection accuracy, hallucination rate, latency.

**Q142: What should you measure for task quality?**  
A: Output format compliance, guardrail pass rate, retry count, token efficiency.

**Q143: What should you measure for crew quality?**  
A: End-to-end success rate, task dependency correctness, delegation accuracy, cost.

**Q144: What's the difference between evaluation and observability?**  
A: Evaluation = measuring quality (does it work well?). Observability = tracking what happened (latency, errors, decisions).

**Q145: How do you evaluate tool selection?**  
A: Check if agent called right tool for task, didn't call unnecessary tools.

**Q146: How do you evaluate delegation?**  
A: Check if manager delegated to right agent for task type.

**Q147: How do you evaluate routing in Flow?**  
A: Check if router sent to correct next step given input.

**Q148: What's LLM-as-judge?**  
A: Use LLM to score outputs (similar to reference or rubric-based).

**Q149: What's a golden dataset?**  
A: Set of inputs with known correct outputs for regression testing.

**Q150: How do you measure cost-quality trade-off?**  
A: Plot: X=cost, Y=quality. Find Pareto frontier of optimal configurations.

---

## Level 7: System Design Scenarios (15 Questions)

### Real-World Scenarios

**Q151: Design enterprise document analysis. Multi-step workflow with 1M+ documents/day.**

**Answer**: 
- Flow as coordinator (state tracks document)
- Crew 1: Extraction (parse document structure)
- Crew 2: Analysis (extract key info)
- Crew 3: Classification (category, risk level)
- Crew 4: Review (human approval for high-risk)
- Observability: track per-document cost, latency, success rate
- Reliability: retry failed documents, fallback to simpler extraction if complex analysis fails
- Storage: save CrewOutput to database with document ID
- Scaling: async processing, batch optimization

**Q152: Design customer support chatbot with escalation.**

**Answer**:
- Flow: Listen for customer input
- Crew (Intent classifier): Determine issue type
- Router: If routine (FAQ, billing, technical) → Crew (Solver), else → Human queue
- Crew (Solver): Use tools (knowledge base, API, database) to resolve
- Flow: If guardrail fails (low confidence) → Human escalation, else → Send to customer
- Memory: Store conversation history, link to customer profile
- Security: Redact PII, audit all agent decisions

**Q153: Design financial data analysis with compliance requirements.**

**Answer**:
- Agents: Extraction specialist, Analysis specialist, Compliance reviewer
- Sequential crew: Extract → Analyze → Compliance check
- Guardrails: Compliance validator on final output, retries if validation fails
- Security: No sensitive data in logs, encrypted storage, audit trail
- Evaluation: Accuracy against known datasets, compliance rate
- Deployment: Separate staging for compliance testing

**Q154: Design multi-language content translation + localization workflow.**

**Answer**:
- Flow: Get content, language
- Crew 1: Translate (LLM + human review)
- Router: If confidence low → Crew (Expert review), else → Localize
- Crew 2: Localize (cultural adaptation, links, dates)
- Guardrail: Native speaker validation
- Storage: Store per-language versions

**Q155: Design research paper analysis system for academics.**

**Answer**:
- Agent 1: Extract methodology, contributions, related work
- Agent 2: Evaluate technical soundness
- Agent 3: Assess novelty and impact
- Sequential crew: All agents on same paper, share context
- Output: Pydantic model with structured assessment
- Memory: Cross-paper connections, author history
- Evaluation: Compare against peer reviews

---

## Quick Self-Test

**Before interviews, answer these WITHOUT notes:**

1. Agent vs Task?
2. Crew vs Flow?
3. Sequential vs Hierarchical?
4. max_iter meaning?
5. CrewOutput structure?
6. State vs Memory?
7. Retry vs Fallback?
8. Tool vs API?
9. raw vs json_dict vs pydantic?
10. @start vs @listen vs @router?

**If you can answer all 10 clearly, you're ready for interviews.**

---

**Next**: Study [Scenario Questions](./20-scenario-questions.md) for real-world problem-solving.
