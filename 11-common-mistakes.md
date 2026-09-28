# Common Mistakes & How to Fix Them

**Diagnosis + Fix + Mental Model Correction for each**

---

## Agent Mistakes

### Mistake 1: Agent Loops Infinitely

**Symptom**: Agent keeps calling same tool, never returns.

**Root Cause**: 
- max_iter too high
- Agent doesn't understand when to stop
- Tool results don't help agent progress

**Diagnosis**:
```python
agent = Agent(
    role="Worker",
    max_iter=50  # ← Way too high!
)

# Run with verbose=True to see iterations
agent.verbose = True
task.execute()
# See agent calling search_web 30+ times
```

**Fix**:
```python
# Option 1: Lower max_iter
agent = Agent(
    role="Worker",
    max_iter=10  # More reasonable
)

# Option 2: Make goal clearer
agent = Agent(
    role="Researcher",
    goal="Find 3 key facts about AI and stop",  # Clear stopping condition
    max_iter=10
)

# Option 3: Make tool feedback more helpful
@tool
def search(query: str) -> str:
    """Search the web"""
    results = internet_search(query)
    return f"Found {len(results)} results:\n" + format_results(results)
    # Better feedback helps agent stop

# Option 4: Clarify backstory
agent = Agent(
    role="Researcher",
    goal="Find accurate information",
    backstory="You are efficient and know when you have enough information to stop.",
    max_iter=10
)
```

**Mental Model**: Agent doesn't loop because it's dumb. It loops because:
- It doesn't know how many iterations are enough
- Tool feedback doesn't indicate progress
- max_iter allows too many tries

---

### Mistake 2: Agent Doesn't Use Tools

**Symptom**: Agent just thinks and returns answer, never calls tools.

**Root Cause**:
- Tools not provided
- Tool descriptions unclear
- Agent role/goal doesn't suggest tool use

**Diagnosis**:
```python
# Check: Are tools provided?
print(agent.tools)  # [] ← Empty!

# Check: What does agent think tools are?
# Set verbose=True to see agent's internal prompt
agent.verbose = True
task.execute()
# See: "Tools available: None"
```

**Fix**:
```python
# Problem: No tools
agent = Agent(
    role="Researcher",
    goal="Find information",
    # No tools provided!
)

# Solution: Add tools
@tool
def web_search(query: str) -> str:
    """Search the web for current information"""
    return search_implementation(query)

agent = Agent(
    role="Researcher",
    goal="Find accurate, current information using web_search",
    tools=[web_search],
    backstory="You always use web_search to find current facts"
)

# Also make goal explicit
task = Task(
    description="Research AI trends. Use web_search to find current information.",
    expected_output="Report with current trends",
    agent=agent
)
```

**Mental Model**: Agents use tools when:
1. Tools are provided
2. Tool descriptions are clear
3. Agent role/goal suggests tool use is needed

---

### Mistake 3: Agent Calls Wrong Tool Repeatedly

**Symptom**: Agent calls same tool multiple times with same parameters.

**Root Cause**:
- Tool result doesn't help agent
- Similar tools confuse agent
- Tool feedback unclear

**Diagnosis**:
```python
# verbose output shows:
# Iteration 1: Calling web_search("AI")
# Result: [3 results]
# Iteration 2: Calling web_search("AI")  ← Same query!
# Result: [3 results]
# Iteration 3: Calling web_search("AI")
```

**Fix**:
```python
# Problem: Tool results don't help
@tool
def search(query: str) -> str:
    return search_implementation(query)

# Solution: Make tool results clearer
@tool
def search(query: str) -> str:
    """Search the web"""
    results = search_implementation(query)
    if not results:
        return "No results found for this query. Try different terms."
    return f"Found {len(results)} results:\n" + format_results(results)

# Also give agent instructions
task = Task(
    description="""
    Research the topic. 
    Use web_search to find information.
    After each search, analyze results.
    If results are relevant, synthesize into answer.
    If not relevant, try different search terms.
    Stop when you have good information.
    """,
    agent=agent
)
```

**Mental Model**: Agents repeat tools when feedback doesn't indicate progress. Make feedback explicit: "Found X results", "No results", "Information is outdated", etc.

---

## Task Mistakes

### Mistake 4: Task Context is Empty

**Symptom**: Task 2 doesn't have Task 1's output.

**Root Cause**:
- Forgot `context=[task1]`
- Task 1 failed
- Context not injected into prompt

**Diagnosis**:
```python
# Enable verbose
crew = Crew(..., verbose=True)
# OR check output directly
result = crew.kickoff()
task2_output = result.tasks_output[1]
print(task2_output.raw)  # Doesn't reference Task 1 findings
```

**Fix**:
```python
# Problem: No context
task2 = Task(
    description="Write article",
    expected_output="Article",
    agent=writer
    # No context!
)

# Solution: Add context
task2 = Task(
    description="Write article based on research",
    expected_output="Article",
    agent=writer,
    context=[task1]  # ← Add this
)

# Also check task1 succeeded
result = crew.kickoff()
if result.tasks_output[0].raw and len(result.tasks_output[0].raw) > 0:
    print("Task 1 succeeded")
else:
    print("Task 1 failed - Task 2 won't have context")
```

**Mental Model**: Context is explicit. Must be declared with `context=[...]`. Task outputs don't automatically flow to next task.

---

### Mistake 5: Guardrail Keeps Failing

**Symptom**: Guardrail validation never passes, task retries forever.

**Root Cause**:
- Validation too strict
- Agent can't achieve requirement
- Guardrail logic wrong

**Diagnosis**:
```python
def validate(output: str) -> bool:
    return len(output) > 10000  # Require 10K words!

# With verbose=True, see repeated failures
agent retries → agent retries → agent retries
# Agent can't write 10K words, validation fails

# Check: Can output ever pass?
test_output = "x" * 5000  # 5K words
print(validate(test_output))  # False - too strict
```

**Fix**:
```python
# Problem: Too strict
def validate(output: str) -> bool:
    return len(output) > 10000

# Solution: Reasonable requirement
def validate(output: str) -> bool:
    word_count = len(output.split())
    has_key_points = "key point" in output.lower()
    return word_count > 500 and has_key_points  # Achievable

# Also reduce retries
task = Task(
    description="Write report",
    expected_output="Detailed report",
    agent=agent,
    guardrail=validate,
    guardrail_max_retries=2  # Don't retry forever
)

# OR make requirement explicit
task = Task(
    description="Write detailed report with at least 3 key points",
    expected_output="Report with: intro, 3 key points, conclusion",
    agent=agent,
    guardrail=validate
)
```

**Mental Model**: Guardrails must be achievable. If agent can never pass, it will fail. Make guardrails and expected_output consistent.

---

## Crew Mistakes

### Mistake 6: Wrong Task Order (Hierarchical)

**Symptom**: Tasks execute in unexpected order.

**Root Cause**:
- Hierarchical process (order is dynamic)
- Manager didn't understand task dependencies
- Task descriptions unclear

**Diagnosis**:
```python
# With sequential, order is guaranteed
# With hierarchical, order depends on manager

result = crew.kickoff()
for i, task_output in enumerate(result.tasks_output):
    print(f"Task {i}: {task_output.description}")
    # Order might be different from input order!
```

**Fix**:
```python
# Problem: Using hierarchical for deterministic workflow
crew = Crew(
    agents=[...],
    tasks=[task1, task2, task3],  # Want this order
    process=Process.hierarchical  # But manager might change it!
)

# Solution 1: Use sequential if order matters
crew = Crew(
    agents=[...],
    tasks=[task1, task2, task3],
    process=Process.sequential  # ← Guarantees order
)

# Solution 2: If must use hierarchical, make dependencies explicit
task2 = Task(
    description="Task 2 - MUST run after Task 1 for context",
    context=[task1]  # Manager knows task1 is dependency
)

# Solution 3: Give manager clear instructions
task1 = Task(
    description="FIRST: Extract data",
    agent=extractor
)

task2 = Task(
    description="SECOND: Analyze extracted data",
    context=[task1]
    agent=analyzer
)

task3 = Task(
    description="THIRD: Summarize analysis",
    context=[task2]
    agent=summarizer
)
```

**Mental Model**: Sequential = Fixed order. Hierarchical = Manager decides order. Choose based on needs.

---

### Mistake 7: Tasks Interfere With Each Other

**Symptom**: Task B output disrupts Task C, causes wrong results.

**Root Cause**:
- Task B output not formatted well
- Task B output too long
- Too much context confuses agent

**Diagnosis**:
```python
result = crew.kickoff()
print("Task B output:")
print(result.tasks_output[1].raw)
# Is it messy? Too long? Poorly formatted?

print("Task C sees (first 200 chars of context):")
print(result.tasks_output[2].raw[:200])
# Does it reference Task B correctly?
```

**Fix**:
```python
# Problem: Task B output is messy
task_b = Task(
    description="Analyze something",
    agent=agent_b
    # No guardrail on output format
)

# Solution 1: Add summary task between
task_b = Task(
    description="Analyze something",
    agent=agent_b
)

task_b_summary = Task(
    description="Summarize analysis in 2-3 bullet points",
    agent=summarizer,
    context=[task_b],  # Get analysis
    expected_output="Bullet points only"
)

task_c = Task(
    description="Use summary for next step",
    agent=agent_c,
    context=[task_b_summary]  # ← Clean input!
)

# Solution 2: Limit context window
crew = Crew(
    agents=[...],
    tasks=[...],
    context_window=2000  # Limit context size
)

# Solution 3: Make task output expectations clear
task_b = Task(
    description="Analyze something",
    expected_output="Analysis in clear bullet points",
    agent=agent_b,
    guardrail=output_is_clean  # Validate format
)
```

**Mental Model**: Context is powerful but noisy. Summarize or filter between tasks if output is large.

---

## Output Mistakes

### Mistake 8: Assuming raw is JSON

**Symptom**: `json.loads(result.raw)` fails.

**Root Cause**:
- raw is always text, not always JSON
- Agent didn't output JSON
- JSON parsing failed

**Diagnosis**:
```python
result = crew.kickoff()
import json

try:
    parsed = json.loads(result.raw)
except json.JSONDecodeError:
    print(f"Not JSON: {result.raw[:100]}")
```

**Fix**:
```python
# Problem: Assume raw is JSON
result = crew.kickoff()
data = json.loads(result.raw)  # Might fail!

# Solution: Check first
result = crew.kickoff()
if result.json_dict:
    data = result.json_dict  # Use parsed JSON
elif result.pydantic:
    data = result.pydantic  # Use typed object
else:
    # Fall back to parsing raw
    try:
        data = json.loads(result.raw)
    except:
        data = result.raw  # Plain text

# Better: Use output_pydantic or output_json
task = Task(
    description="...",
    agent=agent,
    output_pydantic=MyModel  # ← Ensures structured output
)

result = crew.kickoff()
# Now result.pydantic is guaranteed to be present
```

**Mental Model**: raw is always text. Use json_dict or pydantic for structured data.

---

### Mistake 9: Pydantic Output Parsing Failed

**Symptom**: `result.pydantic` is None when expected.

**Root Cause**:
- output_pydantic not configured
- Agent didn't output valid JSON
- JSON didn't match schema

**Diagnosis**:
```python
result = crew.kickoff()
if result.pydantic is None:
    print("Pydantic parsing failed")
    print("Raw output:")
    print(result.raw[:500])
    # Does it contain JSON?
    # Does JSON match schema?
```

**Fix**:
```python
# Problem: Pydantic output not working
result = crew.kickoff()
if result.pydantic is None:
    # Either not configured, or parsing failed
    
    # Check if configured
    if task.output_pydantic is None:
        # Not configured
        task = Task(
            ...,
            output_pydantic=MyModel  # Add this
        )
    
    # If configured but still None:
    # Agent didn't output valid JSON
    # Make task more explicit
    task = Task(
        description="Output VALID JSON like: {...}",
        expected_output="Valid JSON matching schema",
        agent=agent,
        output_pydantic=MyModel,
        guardrail=validate_json  # Validate JSON
    )
```

**Mental Model**: Pydantic output requires agent to output valid JSON matching schema. Both must be true.

---

### Mistake 10: Iterating tasks_output Incorrectly

**Symptom**: IndexError when accessing `result.tasks_output[3]`.

**Root Cause**:
- Assuming fixed number of tasks
- Fewer tasks than expected

**Diagnosis**:
```python
result = crew.kickoff()
print(f"Number of tasks: {len(result.tasks_output)}")

# Wrong:
print(result.tasks_output[3])  # IndexError if fewer than 4 tasks

# Right:
for task_result in result.tasks_output:
    print(task_result.raw)
```

**Fix**:
```python
result = crew.kickoff()

# Wrong way
task3 = result.tasks_output[3]  # Assumes 4+ tasks

# Right way
for i, task_result in enumerate(result.tasks_output):
    print(f"Task {i}: {task_result.description}")

# Or with checking
if len(result.tasks_output) > 3:
    task3 = result.tasks_output[3]
else:
    print("Only", len(result.tasks_output), "tasks completed")
```

**Mental Model**: Don't assume task count. Always iterate or check length.

---

## Common Misconceptions

| Wrong | Correct |
|-------|---------|
| max_iter=10 means 10 tool calls | max_iter=10 means 10 thinking iterations |
| raw is always JSON | raw is always text, json_dict is JSON |
| Context flows automatically | Context must be explicit: context=[...] |
| Sequential guarantees parallel execution | Sequential is sequential, no parallelization |
| Flow state persists | Flow state is per-execution only |
| Delegation is same as context | Delegation is agent→agent, context is task→task |
| Crew is same as Flow | Crew is autonomous, Flow has application control |

---

## Debugging Checklist

When something goes wrong:

1. **Enable verbose**: `verbose=True` on Agent/Crew
2. **Check outputs**: `result.tasks_output[i].raw` for each task
3. **Verify token usage**: High usage might indicate loops
4. **Check return types**: `type(result)`, `type(result.pydantic)`
5. **Validate inputs**: Are task descriptions clear?
6. **Test components**: Test task/agent individually
7. **Read error messages**: CrewAI provides helpful error details
8. **Compare with examples**: Check [README](./README.md) examples

---

**Next**: [Interview Questions](./19-interview-questions.md) for practice

