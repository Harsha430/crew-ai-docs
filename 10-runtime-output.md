# Runtime Output Anatomy (CRITICAL)

**THIS CHAPTER IS ESSENTIAL**

You must know exactly what Python object is returned from every API call and what fields it contains.

## What crew.kickoff() Returns

```python
result = crew.kickoff()
type(result)  # <class 'crewai.crew.CrewOutput'>
```

### CrewOutput Object Structure

```
CrewOutput
├── raw: str
│   └── The final answer as plain text
│
├── json_dict: dict | None
│   └── Parsed JSON object (if output is valid JSON)
│
├── pydantic: BaseModel | None
│   └── Typed Pydantic object (if structured output configured)
│
├── tasks_output: list[TaskOutput]
│   └── Results from each task
│       ├── TaskOutput[0]
│       ├── TaskOutput[1]
│       └── TaskOutput[N]
│
└── token_usage: dict
    ├── "input_tokens": int
    ├── "output_tokens": int
    └── "total_tokens": int
```

### Accessing CrewOutput

```python
result = crew.kickoff()

# Raw text answer
answer_text = result.raw
print(type(answer_text))  # <class 'str'>
print(answer_text)        # "Here are the key findings..."

# JSON (if output is valid JSON)
json_output = result.json_dict
print(type(json_output))  # <class 'dict'> or None
print(json_output)        # {"key": "value"} or None

# Structured (if Pydantic model configured)
typed_output = result.pydantic
print(type(typed_output))  # <class 'your_model'> or None
print(typed_output.field_name)

# All task results
all_tasks = result.tasks_output
print(type(all_tasks))    # <class 'list'>
print(len(all_tasks))     # Number of tasks

# Token usage
usage = result.token_usage
print(type(usage))        # <class 'dict'>
print(usage["total_tokens"])  # Integer total
```

## TaskOutput Object Structure

When you access `result.tasks_output[i]`, you get a `TaskOutput`:

```
TaskOutput
├── description: str
│   └── Original task description
│
├── summary: str
│   └── Agent's summary of what it did
│
├── raw: str
│   └── The actual output the agent produced
│
├── pydantic: BaseModel | None
│   └── Typed output (if task has output_pydantic)
│
├── json_dict: dict | None
│   └── Parsed JSON (if output is valid JSON)
│
├── agent: Agent
│   └── Reference to the agent who executed it
│
├── output_format: str
│   └── "raw" | "json" | "pydantic"
│
└── token_usage: dict
    ├── "input_tokens": int
    ├── "output_tokens": int
    └── "total_tokens": int
```

### Accessing TaskOutput

```python
result = crew.kickoff()
first_task = result.tasks_output[0]

# What was the task?
print(first_task.description)  # Task description

# What did agent produce?
print(first_task.raw)          # Raw output text

# How much did it cost?
print(first_task.token_usage["total_tokens"])

# What was the format?
print(first_task.output_format)  # "raw", "json", or "pydantic"

# Iterate all tasks
for task_result in result.tasks_output:
    print(f"Task: {task_result.description}")
    print(f"Output: {task_result.raw}")
    print(f"Tokens: {task_result.token_usage['total_tokens']}")
```

## Output Formats in Detail

### Format 1: Raw Output

```python
task = Task(
    description="Write a poem",
    expected_output="A 10-line poem",
    agent=agent
    # NO output_json or output_pydantic
)

crew = Crew(agents=[agent], tasks=[task])
result = crew.kickoff()

# What you get:
print(result.raw)              # "Here's a poem: Roses are red..."
print(result.json_dict)        # None (no JSON)
print(result.pydantic)         # None (no Pydantic)
print(result.tasks_output[0].output_format)  # "raw"
```

**Use when**: Output doesn't need to be structured.

### Format 2: JSON Output

```python
from pydantic import BaseModel

class ResearchFindings(BaseModel):
    findings: list[str]
    confidence: float

task = Task(
    description="Research AI trends",
    expected_output="JSON with findings and confidence",
    agent=agent,
    output_json=ResearchFindings  # Request JSON
)

result = crew.kickoff()

# What you get:
print(result.raw)
# Raw text is still here, but the agent tried to format it as JSON

print(result.json_dict)        # dict parsed from JSON
print(type(result.json_dict))  # <class 'dict'>
print(result.json_dict["findings"])  # list[str]
print(result.json_dict["confidence"])  # float

print(result.pydantic)         # None (not pydantic format)
print(result.tasks_output[0].output_format)  # "json"
```

**Use when**: You want structured output parsed from JSON.

### Format 3: Pydantic Output

```python
from pydantic import BaseModel

class Report(BaseModel):
    title: str
    sections: list[str]
    quality_score: int

task = Task(
    description="Write a report",
    expected_output="A structured report",
    agent=agent,
    output_pydantic=Report  # Request Pydantic
)

result = crew.kickoff()

# What you get:
print(result.raw)              # Raw text
print(result.json_dict)        # dict (can be converted)
print(result.pydantic)         # Report object (typed!)
print(type(result.pydantic))   # <class '__main__.Report'>

# Access typed fields
report = result.pydantic
print(report.title)            # str
print(report.sections)         # list[str]
print(report.quality_score)    # int
```

**Use when**: You want strongly typed, validated output.

## Token Usage Tracking

### At Crew Level

```python
result = crew.kickoff()

usage = result.token_usage
print(usage)
# {
#   "input_tokens": 12500,
#   "output_tokens": 3200,
#   "total_tokens": 15700
# }

print(usage["input_tokens"])     # int: all input tokens across all tasks
print(usage["output_tokens"])    # int: all output tokens across all tasks
print(usage["total_tokens"])     # int: total
```

### At Task Level

```python
result = crew.kickoff()

for i, task_result in enumerate(result.tasks_output):
    task_usage = task_result.token_usage
    print(f"Task {i}: {task_usage['total_tokens']} tokens")
```

### Token Cost Calculation

```python
# Claude 3.5 Sonnet pricing (as of Sept 2026)
# Input: $3 per 1M tokens
# Output: $15 per 1M tokens

result = crew.kickoff()
usage = result.token_usage

input_cost = (usage["input_tokens"] / 1_000_000) * 3
output_cost = (usage["output_tokens"] / 1_000_000) * 15
total_cost = input_cost + output_cost

print(f"Cost: ${total_cost:.4f}")
```

## Flow Output

When you use `flow.kickoff()`:

```python
class MyFlow(Flow):
    @start()
    def start(self):
        return {"topic": "AI"}
    
    @listen(start)
    def research(self, topic):
        result = crew.kickoff({"topic": topic["topic"]})
        return result
    
    @router(research)
    def route(self, research_result):
        return "review" if research_result else "done"

flow = MyFlow()
final_result = flow.kickoff()

# What is final_result?
# - It's whatever the last @listen returns
# - Could be CrewOutput
# - Could be string
# - Could be any Python object
# Depends on your flow logic!

print(type(final_result))  # Depends on what you return
```

### Flow Return Types

```python
# Scenario 1: Flow returns Crew result
@listen("last_step")
def final_step(self):
    crew = Crew(...)
    return crew.kickoff()  # Returns CrewOutput

# flow.kickoff() returns CrewOutput

# Scenario 2: Flow returns processed result
@listen("last_step")
def final_step(self, result):
    return result.raw  # Returns str

# flow.kickoff() returns str

# Scenario 3: Flow returns state
@listen("last_step")
def final_step(self):
    return self.state  # Returns dict

# flow.kickoff() returns dict
```

## Router Output

When you have a @router:

```python
@router(source_method)
def my_router(self, source_output):
    if some_condition(source_output):
        return "route_a"
    else:
        return "route_b"
```

The router returns:
- Type: `str` (route identifier)
- Purpose: Tells Flow which @listen method to call next

```python
@listen("route_a")
def handle_a(self, value):
    return ...

@listen("route_b")
def handle_b(self, value):
    return ...
```

## State Structure

```python
class MyFlow(Flow):
    @start()
    def start(self):
        # State starts empty
        print(self.state)  # {}
        
        # Initialize state
        self.state["counter"] = 0
        self.state["results"] = []
        
        return {"initial": True}
    
    @listen(start)
    def step_1(self, value):
        # Access state
        self.state["counter"] += 1
        
        # Mutate state
        self.state["results"].append("Step 1")
        
        return "done"
```

**State type**: `dict[str, Any]`  
**Persistence**: Per-flow-execution only (not automatic saved)

## Type Checking with CrewOutput

```python
from crewai.crew import CrewOutput

result = crew.kickoff()

if isinstance(result, CrewOutput):
    print(f"Got CrewOutput with {len(result.tasks_output)} tasks")
    
    for task_result in result.tasks_output:
        if task_result.pydantic:
            print(f"Task returned Pydantic: {type(task_result.pydantic)}")
```

## Inspecting Unknown Objects

```python
def inspect_crewai_object(obj):
    """Helper to understand any CrewAI object"""
    print(f"Type: {type(obj)}")
    print(f"Type name: {type(obj).__name__}")
    
    if hasattr(obj, "__dict__"):
        print("Attributes:")
        for key, value in obj.__dict__.items():
            if not key.startswith("_"):
                print(f"  {key}: {type(value).__name__}")
    
    if hasattr(obj, "__annotations__"):
        print("Annotations:")
        for key, type_hint in obj.__annotations__.items():
            print(f"  {key}: {type_hint}")

# Usage
result = crew.kickoff()
inspect_crewai_object(result)
```

## Common Output Mistakes

### Mistake 1: Assuming raw is JSON

```python
# ❌ WRONG
result = crew.kickoff()
parsed = json.loads(result.raw)  # May fail!

# ✅ CORRECT
if result.json_dict:
    parsed = result.json_dict
else:
    parsed = json.loads(result.raw)  # Fallback with error handling
```

### Mistake 2: Expecting Pydantic when not configured

```python
task = Task(
    description="Do something",
    agent=agent
    # NO output_pydantic
)

result = crew.kickoff()
if result.pydantic:  # Will be None!
    print(result.pydantic.field)  # ERROR: NoneType has no field
```

### Mistake 3: Iterating tasks incorrectly

```python
# ❌ WRONG - assumes 3 tasks always
result = crew.kickoff()
print(result.tasks_output[0])
print(result.tasks_output[1])
print(result.tasks_output[2])  # IndexError if < 3 tasks!

# ✅ CORRECT
for task_result in result.tasks_output:
    print(task_result.raw)
```

### Mistake 4: Token usage in wrong place

```python
# ❌ WRONG - checking wrong level
result = crew.kickoff()
print(result.tasks_output[0]["token_usage"])  # AttributeError: no dict subscript

# ✅ CORRECT
print(result.tasks_output[0].token_usage["total_tokens"])
print(result.token_usage["total_tokens"])
```

## Debugging Output Issues

### Check what type you got

```python
result = crew.kickoff()

print(f"Output type: {type(result)}")
print(f"Raw type: {type(result.raw)}")
print(f"JSON dict type: {type(result.json_dict)}")
print(f"Pydantic type: {type(result.pydantic)}")
print(f"Tasks output type: {type(result.tasks_output)}")
```

### Print full structure

```python
import json

result = crew.kickoff()

# Serialize what you can
output = {
    "raw": result.raw,
    "json_dict": result.json_dict,
    "task_count": len(result.tasks_output),
    "token_usage": result.token_usage
}

print(json.dumps(output, indent=2, default=str))
```

### Inspect task outputs

```python
result = crew.kickoff()

for i, task_result in enumerate(result.tasks_output):
    print(f"\n=== Task {i} ===")
    print(f"Description: {task_result.description}")
    print(f"Format: {task_result.output_format}")
    print(f"Has JSON: {task_result.json_dict is not None}")
    print(f"Has Pydantic: {task_result.pydantic is not None}")
    print(f"Tokens: {task_result.token_usage['total_tokens']}")
    print(f"Raw (first 200 chars):\n{task_result.raw[:200]}")
```

---

**Next**: [Critical Distinctions](./CRITICAL-DISTINCTIONS.md)
