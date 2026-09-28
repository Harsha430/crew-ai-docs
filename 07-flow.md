# Flow: Application-Level Orchestration

## What is a Flow?

**Flow = Application-level workflow with routing, state, and control**

While Crew is autonomous, **Flow lets your application decide what happens next**.

```python
from crewai import Flow

class ResearchFlow(Flow):
    @start()
    def kickoff_step(self):
        return {"topic": "AI trends"}
    
    @listen(kickoff_step)
    def research_step(self, topic_dict):
        crew = Crew(...)
        return crew.kickoff()
    
    @router(research_step)
    def decide_next(self, research_output):
        if quality_good(research_output):
            return "approve"
        else:
            return "revise"

flow = ResearchFlow()
result = flow.kickoff()
```

---

## Core Decorators

### @start()

**Entry point** - Called first, returns initial value.

```python
class MyFlow(Flow):
    @start()
    def entry_point(self):
        print("Flow starting")
        return {"user_input": "query"}
    
    @listen(entry_point)
    def next_step(self, data):
        print(f"Received: {data}")
```

**Rules**:
- One @start per Flow
- Called first
- Returns value passed to listeners
- No parameters (except self)

### @listen(source)

**Listener** - Receives previous step's return value, does work, returns new value.

```python
@listen(entry_point)
def step_1(self, input_data):
    # input_data = whatever entry_point returned
    result = do_work(input_data)
    return result

@listen(step_1)
def step_2(self, input_data):
    # input_data = whatever step_1 returned
    ...
```

**Rules**:
- Source can be method name or @router string
- Receives return value from source
- Can do work, call crews, etc.
- Returns value for next step (or @router)

### @router(source)

**Routing decision** - Receives value, decides next step, returns route string.

```python
@router(research_step)
def decide_path(self, research_result):
    if "good" in research_result.raw.lower():
        return "approve"  # Routes to @listen("approve")
    else:
        return "revise"   # Routes to @listen("revise")

@listen("approve")
def approve_step(self):
    return "done"

@listen("revise")
def revise_step(self):
    return "revised"
```

**Rules**:
- Source receives return value
- Must return string (route name)
- Route string must match @listen decorator
- Can be used for branching logic

---

## Flow State

```python
class MyFlow(Flow):
    @start()
    def start(self):
        # Initialize state
        self.state["counter"] = 0
        self.state["results"] = []
        return "start"
    
    @listen(start)
    def process(self, value):
        # Access state
        self.state["counter"] += 1
        
        # Mutate state
        self.state["results"].append("item")
        
        return self.state
    
    @listen(process)
    def finish(self, state_dict):
        # Read final state
        print(f"Processed {state_dict['counter']} items")
```

**State Properties**:
- Type: `dict[str, Any]`
- Access: `self.state["key"]`
- Scope: Per-flow-execution only
- Persistence: Manual (you implement saving)

---

## Complete Flow Example

### Human-in-Loop Approval

```python
class ApprovalFlow(Flow):
    @start()
    def start(self):
        """Entry: Get topic"""
        self.state["topic"] = "AI trends"
        return self.state["topic"]
    
    @listen(start)
    def research(self, topic):
        """Research using crew"""
        crew = Crew(
            agents=[researcher],
            tasks=[research_task]
        )
        self.state["research"] = crew.kickoff()
        return self.state["research"]
    
    @router(research)
    def check_quality(self, research_output):
        """Route: Is quality good?"""
        score = analyze_quality(research_output)
        self.state["quality_score"] = score
        
        if score > 0.8:
            return "approve"
        else:
            return "review"
    
    @listen("review")
    def human_review(self):
        """Pause for human feedback"""
        self.state["status"] = "pending_human"
        
        # In real system: save state, wait for human
        human_input = input("Approve? (yes/no): ")
        
        self.state["human_approved"] = human_input == "yes"
        
        if human_input == "yes":
            return "finalize"
        else:
            return "revise"
    
    @listen("revise")
    def revise_research(self):
        """Re-run crew with feedback"""
        # ... revised crew execution
        return "done"
    
    @listen("approve")
    def finalize(self):
        """Final step"""
        self.state["status"] = "completed"
        return self.state

flow = ApprovalFlow()
result = flow.kickoff()
```

---

## Flow Execution Sequences

### Linear Sequence

```
@start() → data1
    ↓
@listen(start) → data2
    ↓
@listen(prev) → data3
    ↓
End (return data3)
```

### Branching with Router

```
@start() → data1
    ↓
@listen(start) → data2
    ↓
@router(listen) → "path_a" or "path_b"
    ├─ data2 → "path_a"
    │   ↓
    │ @listen("path_a") → data3a
    │   ↓
    │ Done (return data3a)
    │
    └─ data2 → "path_b"
        ↓
      @listen("path_b") → data3b
        ↓
      Done (return data3b)
```

### Multiple Routes with Convergence

```
@start()
    ↓
@listen(start)
    ├─ does work
    ↓
@router
    ├─ "fast_path"
    │   ├─ @listen("fast_path")
    │   └─ Returns data_fast
    │
    └─ "slow_path"
        ├─ @listen("slow_path")
        └─ Returns data_slow
        
(No automatic convergence - must be explicit)
```

---

## Flow vs Crew Decision Tree

```
Do you need conditional routing?
├─ YES → Flow
└─ NO → Crew (simpler)

Do you need state management?
├─ YES → Flow
└─ NO → Crew

Do you need human approval steps?
├─ YES → Flow
└─ NO → Crew

Do you need multiple crews?
├─ YES → Flow (coordinates crews)
└─ NO → Crew

Is workflow deterministic?
├─ YES → Crew (simpler)
└─ NO (depends on logic) → Flow
```

---

## Common Flow Patterns

### Pattern 1: Simple Linear

```python
class SimpleFlow(Flow):
    @start()
    def start(self):
        return "input"
    
    @listen(start)
    def process(self, data):
        result = do_work(data)
        return result
```

### Pattern 2: Conditional Branching

```python
class BranchingFlow(Flow):
    @start()
    def start(self):
        return "data"
    
    @listen(start)
    def analyze(self, data):
        return {"score": calculate_score(data)}
    
    @router(analyze)
    def decide(self, analysis):
        if analysis["score"] > 80:
            return "approve"
        else:
            return "reject"
    
    @listen("approve")
    def approve_path(self):
        return "approved"
    
    @listen("reject")
    def reject_path(self):
        return "rejected"
```

### Pattern 3: State Management

```python
class StatefulFlow(Flow):
    @start()
    def start(self):
        self.state["items"] = []
        self.state["count"] = 0
        return None
    
    @listen(start)
    def process_item_1(self):
        self.state["items"].append("item1")
        self.state["count"] += 1
        return self.state["count"]
    
    @listen(process_item_1)
    def process_item_2(self):
        self.state["items"].append("item2")
        self.state["count"] += 1
        return self.state["count"]
    
    @listen(process_item_2)
    def summary(self):
        return f"Processed {self.state['count']} items"
```

### Pattern 4: Multi-Crew Orchestration

```python
class MultiCrewFlow(Flow):
    @start()
    def start(self):
        return {"topic": "AI"}
    
    @listen(start)
    def research_crew(self, data):
        crew = ResearchCrew()
        result = crew.kickoff(inputs=data)
        self.state["research"] = result
        return result
    
    @listen(research_crew)
    def analysis_crew(self, research_result):
        crew = AnalysisCrew()
        result = crew.kickoff(inputs={"findings": research_result.raw})
        self.state["analysis"] = result
        return result
    
    @listen(analysis_crew)
    def writing_crew(self, analysis_result):
        crew = WritingCrew()
        result = crew.kickoff(inputs={"analysis": analysis_result.raw})
        return result
```

---

## Flow Input/Output

### Kickoff with No Args

```python
flow = MyFlow()
result = flow.kickoff()
# result = whatever final @listen returns
```

### Kickoff with Inputs

```python
flow = MyFlow()
result = flow.kickoff(inputs={"topic": "AI", "year": 2026})

# Inputs available in state or methods
class MyFlow(Flow):
    @start()
    def start(self):
        # Access via self.state if needed
        return "start"
```

### Return Values

```python
# Flow returns whatever last step returns
# Could be:
# - CrewOutput (if crew.kickoff() called)
# - str (if step returns string)
# - dict (if step returns dict)
# - Any Python object

result = flow.kickoff()
print(type(result))  # Depends on last step
```

---

## Flow Debugging

### When Flow Enters Wrong Route

**Problem**: Router sends to wrong @listen

**Diagnosis**:
```python
@router(prev_step)
def router(self, value):
    print(f"Router input: {value}")
    print(f"Type: {type(value)}")
    route = decide_route(value)
    print(f"Router output: {route}")
    return route
```

**Fix**: Check router logic, ensure route string matches @listen decorator

### When Flow State Disappears

**Problem**: Accessing self.state but it's empty

**Diagnosis**:
```python
@listen(step)
def my_step(self):
    print(f"State keys: {self.state.keys()}")
    # Check if key exists before accessing
    if "key" in self.state:
        print(self.state["key"])
    else:
        print("Key not found")
```

**Fix**: Initialize state in @start, use `.get()` safely

### When Flow Hangs

**Problem**: flow.kickoff() never returns

**Diagnosis**:
1. Is a @listen expecting human input? (blocks on input())
2. Is a crew hung? (check crew debugging)
3. Is router infinite loop? (check router logic)

**Fix**:
```python
@listen(step)
def my_step(self):
    # Don't block forever
    timeout = 30
    result = wait_for_result(timeout=timeout)
    return result
```

---

## Flow Best Practices

✅ **DO:**
- Initialize state in @start()
- Use clear route names in @router
- Handle all routes (no dead ends)
- Return values that make sense for next step
- Document flow structure
- Log state changes for debugging

❌ **DON'T:**
- Use Flow for simple sequential work (Crew is simpler)
- Forget to handle all possible routes
- Rely on state persistence (implement it yourself)
- Make router logic too complex
- Block indefinitely (e.g., waiting for human)
- Return None without documenting why

---

## Flow State Persistence (Custom)

Flow state is NOT automatically persisted. Implement if needed:

```python
import json

class MyFlow(Flow):
    def __init__(self):
        super().__init__()
        self.state_file = "flow_state.json"
        self.load_state()
    
    def load_state(self):
        try:
            with open(self.state_file, 'r') as f:
                self.state = json.load(f)
        except FileNotFoundError:
            self.state = {}
    
    def save_state(self):
        with open(self.state_file, 'w') as f:
            json.dump(self.state, f)
    
    @listen(some_step)
    def save_and_continue(self):
        self.state["checkpoint"] = "step_x"
        self.save_state()
        return "continue"
```

---

## Summary: When to Use Each

| Need | Tool |
|------|------|
| Fixed task sequence | Crew (Process.sequential) |
| Dynamic task order | Crew (Process.hierarchical) |
| Conditional routing | Flow |
| State management | Flow |
| Human approval | Flow |
| Multiple crews | Flow |
| Simple workflow | Crew |
| Complex application | Flow |

---

**Next**: [Tools](./08-tools.md)
