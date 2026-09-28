# Scenario Questions: Real-World Problems

**Solve these scenarios to test production-ready thinking**

---

## Scenario 1: Agent Loops Forever

**Situation**:
Your research agent has been running for 2 hours. System monitor shows it's making 100+ tool calls. Users are reporting the API is timing out.

**Problem Diagnosis**:
What could be causing this? What evidence would you look for?

**Root Cause Identification**:
A) max_iter set to 100 (agent keeps thinking)  
B) Tool results unclear (agent doesn't know when to stop)  
C) Infinite loop in tool (tool keeps returning data)  
D) Agent recursively calling itself

**Most Likely**: B - Tool results unclear

**Solution**:
1. **Immediate**: Lower max_iter to 10, restart agent
2. **Root cause**: Make tool results clearer:
   ```python
   @tool
   def search(query: str) -> str:
       results = search_impl(query)
       return f"Found {len(results)} results. Most relevant:\n" + format_results(results[:3])
   ```
3. **Monitoring**: Set alert if tool_calls > 50 per task

**Key Takeaway**: Don't just increase max_iter. Understand why agent isn't stopping.

---

## Scenario 2: Task Context Empty

**Situation**:
Task 1 researches a topic and produces findings.
Task 2 should write an article based on findings.
But Task 2's output doesn't reference Task 1's findings at all.

**Problem**: Why isn't Task 1 output flowing to Task 2?

**Debugging Steps**:
1. Check: Is `context=[task1]` set on task2?
2. Check: Did task1 succeed? (check `result.tasks_output[0].raw`)
3. Check: Is task2's description clear? (does it ask agent to use findings?)

**Solution**:
```python
# Before (task2 doesn't reference task1)
task2 = Task(
    description="Write an article",
    agent=writer
    # No context!
)

# After (task2 gets task1 output)
task2 = Task(
    description="Write an article based on the research findings from the previous task",
    expected_output="Article that references research findings",
    agent=writer,
    context=[task1]  # ← Add this
)
```

**Key Takeaway**: Context doesn't flow automatically. Explicitly declare dependencies.

---

## Scenario 3: High Token Cost

**Situation**:
First implementation costs $5 per request.
You need to reduce to $0.50 per request (10x reduction).

**Optimization Strategies** (ranked by impact):

1. **Model Choice** (biggest impact):
   - Switch GPT-4 → GPT-3.5: ~50% cost reduction
   - Switch Claude Opus → Haiku: ~70% cost reduction

2. **Fewer Tools** (medium impact):
   - Remove unused tools: ~20% cost reduction
   - Each tool description adds tokens

3. **Shorter Prompts** (medium impact):
   - Shorter backstory: ~10% cost reduction
   - Fewer examples in context

4. **Fewer Tasks** (low-medium impact):
   - Combine tasks: ~15% cost reduction
   - Each task adds context

5. **Lower max_iter** (medium impact):
   - Reduce max_iter 10→5: ~20% cost reduction
   - Each iteration = LLM call

**Implementation Plan**:
```python
# High-cost agent
researcher = Agent(
    role="Expert Researcher",
    goal="Find exhaustive research",
    backstory="You have 20 years of experience...",  # Long
    llm=ChatOpenAI(model="gpt-4"),  # Expensive
    max_iter=15,  # High
    tools=[web_search, api, database, ...]  # Many
)

# Optimized agent
researcher = Agent(
    role="Researcher",
    goal="Find key research points",
    backstory="You research efficiently.",  # Short
    llm=ChatOpenAI(model="gpt-3.5-turbo"),  # Cheaper
    max_iter=5,  # Lower
    tools=[web_search]  # Fewer
)
```

**Measurement**:
- Before: $5.00 per request
- After: ~$0.80 (16% cost)
- Further optimization: ~$0.50 (10% cost) ← target

**Key Takeaway**: Model choice > number of tools > other factors.

---

## Scenario 4: Guardrail Keeps Failing

**Situation**:
You added a guardrail that requires output > 1000 words.
Agent runs out of retries and fails.
But the content is actually good, just under 1000 words.

**Problem**: Guardrail too strict?

**Analysis**:
- Is 1000 words achievable?
- Are you measuring correctly?
- Is guardrail logically sound?

**Solution Options**:

1. **Lower requirement**:
   ```python
   def validate(output: str) -> bool:
       return len(output.split()) > 500  # Reduced from 1000
   ```

2. **Make task clearer**:
   ```python
   task = Task(
       description="Write a detailed article with at least 500 words",
       expected_output="Comprehensive article",
       agent=agent
   )
   ```

3. **Make guardrail multi-part**:
   ```python
   def validate(output: str) -> bool:
       word_count = len(output.split())
       has_structure = "Introduction" in output and "Conclusion" in output
       has_depth = word_count > 300  # More achievable
       return has_depth and has_structure
   ```

4. **Reduce retries**:
   ```python
   task = Task(
       ...,
       guardrail_max_retries=1  # Don't keep retrying
   )
   ```

**Key Takeaway**: Guardrails must be achievable. If agent can never pass, fix the guardrail, not the agent.

---

## Scenario 5: Manager Agent Doesn't Delegate

**Situation**:
You set up hierarchical process with a manager agent.
Manager is supposed to delegate to 3 specialist agents.
But manager does all the work itself instead of delegating.

**Problem**: Why doesn't manager delegate?

**Diagnosis**:
1. Does manager have `allow_delegation=True`?
2. Are specialist agents available?
3. Is task description clear about needing specialists?

**Solution**:
```python
# Before (manager doesn't see need to delegate)
manager = Agent(
    role="Manager",
    goal="Solve problem",
    allow_delegation=True  # Enabled, but...
)

specialist1 = Agent(role="Data Analyst", ...)
specialist2 = Agent(role="Writer", ...)

crew = Crew(
    agents=[manager, specialist1, specialist2],
    tasks=[single_big_task],
    process=Process.hierarchical,
    manager_agent=manager
)

# After (make delegation explicit)
manager = Agent(
    role="Project Manager",
    goal="Coordinate team to solve this problem",
    backstory="You are skilled at delegating to specialists",
    allow_delegation=True
)

single_big_task = Task(
    description="""
    This requires analysis and writing.
    Delegate data analysis to the analyst.
    Delegate writing to the writer.
    Synthesize final result.
    """,
    agent=manager
)

# Also make specialists clear in agent descriptions
specialist1 = Agent(
    role="Data Analyst",
    goal="Analyze data accurately",
    backstory="You are the team's data specialist"
)
```

**Key Takeaway**: Managers delegate when:
1. allow_delegation=True
2. Task needs specialized skills
3. Specialist agents are available
4. Task description suggests delegation

---

## Scenario 6: Output Type Mismatch

**Situation**:
You configured `output_pydantic=MyModel`.
Agent completed task, but `result.pydantic` is None.

**Debugging**:
```python
result = crew.kickoff()

# Check what we got
print(f"pydantic: {result.pydantic}")  # None
print(f"raw: {result.raw[:200]}")  # Raw output
print(f"json_dict: {result.json_dict}")  # None or dict?
```

**Possible Causes**:
1. Agent didn't output JSON
2. JSON doesn't match schema
3. Pydantic parsing failed silently

**Solution**:
```python
# Make requirement explicit
task = Task(
    description="""
    Output VALID JSON matching this schema:
    {
        "findings": ["item1", "item2"],
        "confidence": 0.9,
        "sources": 3
    }
    """,
    expected_output="Valid JSON with findings, confidence, and source count",
    agent=agent,
    output_pydantic=MyModel,
    guardrail=lambda output: validate_json(output, MyModel)
)

# Validation function
def validate_json(output: str, model_class) -> bool:
    try:
        import json
        data = json.loads(output)
        model_class(**data)  # Try to construct model
        return True
    except:
        return False
```

**Key Takeaway**: Pydantic parsing requires:
1. Valid JSON output
2. JSON matches schema
3. No parsing errors

---

## Scenario 7: Flow Routes to Wrong Branch

**Situation**:
Your Flow has conditional routing:
- If score > 0.8 → approve
- If score <= 0.8 → revise

But it keeps going to "revise" when it should go to "approve".

**Debugging**:
```python
@router(analyze_step)
def route(self, analysis_result):
    score = analysis_result.quality_score
    print(f"Score: {score}, Type: {type(score)}")
    
    if score > 0.8:
        return "approve"
    else:
        return "revise"
```

**Common Issues**:
1. Score is string, not float: "0.9" > 0.8 → False (string comparison!)
2. Score is None
3. Route name doesn't match @listen decorator

**Solution**:
```python
# Before (buggy)
@router(analyze_step)
def route(self, analysis_result):
    score = analysis_result.quality_score  # Could be string!
    if score > 0.8:
        return "approve"
    else:
        return "revise"

# After (fixed)
@router(analyze_step)
def route(self, analysis_result):
    score = float(analysis_result.quality_score)  # Ensure float
    
    if score > 0.8:
        route_name = "approve"
    else:
        route_name = "revise"
    
    print(f"Router: score={score}, sending to {route_name}")
    return route_name

# And verify listener exists
@listen("approve")
def approve_step(self):
    return "Done"

@listen("revise")
def revise_step(self):
    return "Revised"
```

**Key Takeaway**: Type matters. String "0.9" is not > 0.8.

---

## Scenario 8: Memory Not Retrieved

**Situation**:
You enabled `memory=True` on Crew.
Agent completes task, memory should store findings.
Next crew run, agent should retrieve previous findings.
But agent acts like it doesn't remember anything.

**Diagnosis**:
```python
# Check 1: Is memory enabled?
crew = Crew(..., memory=True)

# Check 2: Does agent know to use memory?
agent = Agent(
    role="Researcher",
    goal="Find information",
    backstory="You remember previous research"  # ← Hint to use memory
)

# Check 3: Is query similar enough for retrieval?
# "I researched AI" won't match "Tell me about machine learning"
```

**Solution**:
```python
# Make agent memory-aware
agent = Agent(
    role="Research Expert",
    goal="Find accurate information, using previous research where relevant",
    backstory="""
    You are an expert researcher.
    You remember findings from previous research tasks.
    You build upon previous findings rather than repeating research.
    """,
    memory=True
)

# Make task ask for memory use
task = Task(
    description="Research AI trends. Check if you've researched this before.",
    agent=agent
)

# Run multiple times
crew = Crew(agents=[agent], tasks=[task], memory=True)

# First run
result1 = crew.kickoff(inputs={"topic": "AI trends"})

# Second run (agent should retrieve first run)
result2 = crew.kickoff(inputs={"topic": "latest AI developments"})
# Agent might say: "I recall researching AI trends before..."
```

**Key Takeaway**: Memory requires:
1. `memory=True` on Crew
2. Agent backstory mentioning memory
3. Task suggesting memory use
4. Similarity between queries for retrieval

---

## Scenario 9: Cost Spike

**Situation**:
Your system was costing $100/month.
This month: $1,500/month.
What could cause 15x spike?

**Investigation**:
```python
# Check token usage
if token_usage_this_month > token_usage_last_month * 10:
    print("Token usage spike - check logs")

# Check model changes
if "gpt-4" in logs and "gpt-3.5" in old_logs:
    print("Model switched - that's expensive!")

# Check retry rate
if retry_rate > 50%:
    print("High retries - costing lots")

# Check max_iter changes
if max_iter_new > max_iter_old * 5:
    print("More iterations per task")
```

**Common Causes**:
1. Model switched to expensive version
2. max_iter increased
3. New tool with expensive operations
4. Bug causing infinite retries
5. More agents/tasks in system
6. Memory enabled (adds token cost)

**Investigation & Fix**:
```python
# 1. Check what changed
git diff HEAD~1 config.py

# 2. Estimate impact of each change
estimate_cost_change(
    old_model="gpt-3.5",
    new_model="gpt-4",
    impact="4x more expensive"
)

# 3. Revert expensive changes
agent = Agent(..., llm=ChatOpenAI(model="gpt-3.5-turbo"))

# 4. Monitor cost daily
log_daily_cost()
alert_if_cost_increases_10_percent()
```

**Key Takeaway**: Monitor costs continuously. Investigate spikes immediately.

---

## Scenario 10: Production Outage

**Situation**:
System was working fine at noon.
At 2 PM: Customers report all requests failing.
Error logs show: "Tool timeout after 30 seconds"

**Incident Response**:
1. **Immediate**: Declare incident, notify team
2. **Status**: Check tool status
3. **Mitigate**: Enable fallback or graceful degradation
4. **Investigate**: Why are timeouts happening?
5. **Fix**: Address root cause
6. **Monitor**: Watch for recurrence

**Root Cause Analysis**:
```python
# Check tool latency
avg_latency_before = 500ms
avg_latency_now = 35000ms  # 70x slower!

# Why?
# Option A: Tool service degraded
response = requests.get("https://tool-service.example.com/health")
# → Status 500, very slow

# Option B: Network issue
# Run: ping, traceroute, DNS check

# Option C: Our side increased load
# Check: Did we deploy anything new?
```

**Solution (Immediate)**:
```python
# Fallback to cached data
@tool
def search_with_fallback(query: str) -> str:
    try:
        result = call_tool_with_timeout(query, timeout=30)
        return result
    except TimeoutError:
        cached = get_from_cache(query)
        if cached:
            return f"(cached) {cached}"
        else:
            raise
```

**Solution (Long-term)**:
```python
# 1. Increase timeout to 60s
@tool
def search(query: str) -> str:
    return call_with_timeout(query, timeout=60)

# 2. Add circuit breaker
breaker = CircuitBreaker(failure_threshold=5)

# 3. Monitor tool health
monitor_tool_health()

# 4. Set alert on p99 latency > 40s
```

**Post-incident**:
- Document what happened
- Add monitoring/alerts
- Implement fallback
- Train team on incident response
- Update runbooks

**Key Takeaway**: Resilience planning (circuit breakers, fallbacks, degradation) prevents outages.

---

## Scenario 11: Agent Calls Unauthorized Tool

**Situation**:
Research agent should only use web_search.
But in production, it called delete_file tool (dangerous!).
The tool executed and deleted user data.

**How Did This Happen?**
```python
# Agent has tool available
agent = Agent(
    role="Researcher",
    tools=[web_search, delete_file, send_email]  # Too many tools!
)

# Agent got confused, called delete_file
```

**Prevention**:
```python
# Option 1: Limit tools per agent
researcher = Agent(
    role="Researcher",
    tools=[web_search]  # Only what's needed
)

# Option 2: Implement permissions
AGENT_PERMISSIONS = {
    "researcher": ["web_search"],
    "admin": ["all"]
}

@tool
def delete_file(path: str):
    if not has_permission(current_agent, "delete_file"):
        raise PermissionError("Not allowed")
    return filesystem.delete(path)

# Option 3: Make audit trail
AuditLog.record("Tool used", tool="delete_file", agent="researcher", timestamp=...)

# Option 4: Require approval for dangerous tools
dangerous_tools = ["delete_file", "send_email", "modify_database"]

@tool
def dangerous_operation(operation: str):
    if operation in dangerous_tools:
        approval = request_human_approval(operation)
        if not approval:
            return "Operation blocked - pending approval"
    return execute(operation)
```

**Key Takeaway**: Least privilege. Give each agent only the tools it needs.

---

## Scenario 12: Compliance Audit Fails

**Situation**:
Regulator asks: "Show me all processing of user ID 12345's data"
You have no audit trail. Audit fails.

**Prevention**:
```python
# Log everything touching user data
def log_user_data_access(user_id: str, action: str, agent_id: str, data: any):
    AuditLog.record({
        "timestamp": datetime.now().isoformat(),
        "user_id": user_id,
        "action": action,
        "agent_id": agent_id,
        "data_size": len(str(data)),
        "status": "success"
    })

# During crew execution
log_user_data_access(user_id=12345, action="research", agent_id="researcher", data=findings)

# During result retrieval
log_user_data_access(user_id=12345, action="output_generated", agent_id="synthesizer", data=result)

# For audit
def audit_user_data(user_id: str) -> list:
    return AuditLog.get_entries(user_id=user_id)

# Regulator asks
audit_trail = audit_user_data(12345)
# → Returns complete trace of all processing
```

**Key Takeaway**: Audit logging isn't optional for production. Plan for compliance from day 1.

---

## Your Turn

Now that you've seen 12 scenarios, try solving these:

**Challenge 1**: Agent uses same tool 50+ times. How would you diagnose and fix?

**Challenge 2**: Task outputs 100MB of data. Next task's context becomes noisy. Solution?

**Challenge 3**: System fails every Monday at 9 AM. What's your investigation approach?

**Challenge 4**: Cost is $500/month but should be $50/month. Design optimization.

**Challenge 5**: Manager agent stops delegating mid-execution. Why? How to fix?

---

**Next**: [System Design](./21-system-design.md)
