# Memory: Complete Guide

**CRITICAL: Memory ≠ History ≠ State ≠ Storage**

---

## Key Mental Model

```
Memory (Agent-Level)
├─ What it is: Persistent knowledge agents retrieve
├─ Scope: Per-crew, across executions
├─ Storage: External (database, file)
├─ Retrieval: By relevance
├─ Purpose: Learn from past
└─ Enabled by: memory=True in Crew

History (Execution Record)
├─ What it is: What happened during execution
├─ Scope: Per-task, current execution
├─ Storage: In-memory during task
├─ Retrieval: Linear (latest interactions)
├─ Purpose: Context for current thinking
└─ Enabled by: Automatic during execution

State (Flow-Level)
├─ What it is: Current workflow data
├─ Scope: Per-flow-execution
├─ Storage: self.state dictionary
├─ Retrieval: Direct key access
├─ Purpose: Coordinate flow steps
└─ Enabled by: self.state in Flow

Storage (Backend)
├─ What it is: Persistent data store
├─ Scope: Long-term, multi-system
├─ Storage: Database, files, cloud
├─ Retrieval: Queries
├─ Purpose: Permanent record
└─ Enabled by: Custom implementation
```

---

## Memory Enabled: memory=True

```python
from crewai import Crew

crew = Crew(
    agents=[agent1, agent2],
    tasks=[task1, task2],
    memory=True  # ← Enable memory
)

result = crew.kickoff()
```

**What Happens**:
1. Each agent interaction stored
2. Task outputs recorded
3. Agents can retrieve relevant past info
4. Next crew execution accesses same memory

### Memory Types

**Short-Term Memory**
```
Current & recent interactions
├─ What's happening now
├─ Last few task outputs
└─ Current execution context
```

**Long-Term Memory**
```
All past interactions
├─ Historical tasks
├─ Findings from previous runs
├─ Patterns and insights
└─ Retrieved by relevance search
```

**Entity Memory**
```
Key facts about entities
├─ Person: "Alice is an expert in AI"
├─ Topic: "Climate change: impacts..."
├─ Organization: "Acme Corp: founded..."
└─ Retrieved when entity mentioned
```

---

## Memory Storage

### Default Storage

```python
# CrewAI v1.15.22 uses:
# - LanceDB (default)
# - Embeddings for similarity search
# - Persistent storage location

crew = Crew(
    agents=[...],
    tasks=[...],
    memory=True  # Uses LanceDB by default
)
```

### How Memory Works

```
1. Agent completes task
    ↓
2. Output stored with embeddings
    ├─ Text: "Found that AI trends include agents"
    ├─ Embeddings: [0.23, -0.15, 0.88, ...]
    └─ Metadata: timestamp, task_id, agent_id
    ↓
3. Next time agent needs context
    ├─ Similarity search: "What about agents?"
    ├─ Find similar embeddings
    └─ Retrieve: "Found that AI trends include agents"
    ↓
4. Retrieved info injected into prompt
    ├─ Agent sees: "Previously found: ..."
    └─ Agent uses it for thinking
```

### Multi-Crew Memory

```python
# Multiple crews can share memory
crew1 = Crew(
    agents=[researcher],
    tasks=[research_task],
    memory=True
)

crew1.kickoff()  # Stores findings in memory

crew2 = Crew(
    agents=[writer],
    tasks=[write_task],
    memory=True  # ← Same memory!
)

result = crew2.kickoff()
# Writer can retrieve researcher's findings
```

---

## When Memory Helps

✅ **Good for:**
- Multi-task workflows where tasks build on each other
- Learning from previous runs
- Cross-agent knowledge sharing
- Complex investigations
- Long-running systems

❌ **Not ideal for:**
- Simple one-off tasks
- Real-time/latency-critical systems (memory lookup adds time)
- Small, independent tasks
- Privacy-sensitive work (memory stores everything)

---

## Memory vs State: Key Differences

| Aspect | Memory | State |
|--------|--------|-------|
| **Level** | Agent-level (Crew) | Application-level (Flow) |
| **Scope** | Across executions | Single flow execution |
| **Storage** | Database (persistent) | Dictionary (temporary) |
| **Retrieval** | Similarity search | Direct key access |
| **Purpose** | Learn from past | Coordinate workflow |
| **Persistence** | Yes (automatic) | No (manual if needed) |
| **Access** | LLM retrieves automatically | Code accesses directly |
| **Example** | "I remember researching this..." | "self.state['findings']" |

**Correct Usage**:
```python
# Memory: Agent learns from past
crew = Crew(..., memory=True)  # Agent can retrieve past findings

# State: Flow coordinates workflow
class MyFlow(Flow):
    @listen(step1)
    def step2(self):
        self.state["checkpoint"] = "reached_step2"
```

---

## Memory vs History

| Aspect | Memory | History |
|--------|--------|---------|
| **What** | Persistent knowledge | Execution record |
| **Scope** | Multi-execution | Single execution |
| **Storage** | Database | In-memory |
| **Retrieval** | Similarity search | Sequential/linear |
| **Enabled by** | `memory=True` | Automatic |
| **Access** | LLM automatic | Via callbacks/logs |
| **Purpose** | Learn from past | Track what happened |

**Correct Understanding**:
```python
# Memory stores and retrieves knowledge
crew = Crew(..., memory=True)
# Agent can say: "I remember from 3 weeks ago that..."

# History tracks execution
def on_task_complete(output):
    print(f"Task output: {output.raw}")
    # You're recording history

callback = CrewCallback()
# Callbacks record history
```

---

## Memory Optimization

### Memory Search Quality

```python
# Good memory setup
crew = Crew(
    agents=[
        Agent(
            role="Researcher",
            goal="Find accurate information",
            backstory="You remember findings from past research"
        )
    ],
    tasks=[...],
    memory=True  # Enables similarity search
)

# Result: Agent can find relevant past findings
```

### Memory Size Considerations

```python
# With memory enabled:
crew = Crew(..., memory=True)
# Token cost increases by ~20% (memory retrieval overhead)

# Without memory:
crew = Crew(..., memory=False)
# Faster, cheaper, but no learning from past
```

---

## Implementing Custom Memory

If you need different memory behavior:

```python
import json
from datetime import datetime

class CustomMemory:
    def __init__(self, filepath="memory.json"):
        self.filepath = filepath
        self.memory = self._load()
    
    def _load(self):
        try:
            with open(self.filepath) as f:
                return json.load(f)
        except:
            return []
    
    def store(self, key: str, value: str, metadata: dict = None):
        """Store information"""
        entry = {
            "key": key,
            "value": value,
            "timestamp": datetime.now().isoformat(),
            "metadata": metadata or {}
        }
        self.memory.append(entry)
        self._save()
    
    def retrieve(self, query: str, limit: int = 5) -> list:
        """Retrieve relevant information"""
        # Simple keyword matching (could use embeddings)
        results = [
            m for m in self.memory 
            if query.lower() in m["value"].lower()
        ]
        return results[:limit]
    
    def _save(self):
        with open(self.filepath, "w") as f:
            json.dump(self.memory, f)

# Usage
memory = CustomMemory()

# During crew execution
result = crew.kickoff()
memory.store(
    key="research_findings",
    value=result.raw,
    metadata={"task": "research", "date": "2026-09-28"}
)

# Later
findings = memory.retrieve("AI trends")
```

---

## Memory Best Practices

✅ **DO:**
- Enable memory for multi-task workflows
- Provide clear agent backstory (tells agent to use memory)
- Use similarity search (embeddings work better than keywords)
- Review memory periodically (clean up stale data)
- Monitor memory size (growth over time)

❌ **DON'T:**
- Use memory for single-task systems (unnecessary overhead)
- Store sensitive data in memory (retrieval is not gated)
- Rely on memory for critical data (implement backups)
- Mix memory with state (different purposes)
- Forget memory persists (consider privacy)

---

## Memory Debugging

### Memory Not Being Retrieved

**Problem**: Agent acts like it doesn't remember past findings.

**Diagnosis**:
```python
crew = Crew(..., memory=True)

# Check if memory is enabled
print(crew.memory)  # Should not be None/False

# Check if agent can retrieve
agent.verbose = True
# See if retrieval queries appear in output
```

**Fix**:
```python
# Make agent role/goal suggest memory use
agent = Agent(
    role="Researcher",
    goal="Find accurate information using past findings",
    backstory="You remember previous research and build upon it",
    memory=True
)

# Make task description explicit
task = Task(
    description="Research. Use past findings if relevant.",
    agent=agent
)
```

### Memory Growing Too Large

**Problem**: Memory database growing unboundedly.

**Diagnosis**:
```python
# Monitor memory size
import os
memory_file = "memory.db"  # LanceDB or similar
size_mb = os.path.getsize(memory_file) / (1024 * 1024)
print(f"Memory size: {size_mb} MB")
```

**Fix**:
```python
# Implement memory cleanup
def cleanup_old_memory(crew, days_old=30):
    # Delete entries older than 30 days
    # Implementation depends on storage backend
    pass

# Or disable memory if not needed
crew = Crew(..., memory=False)
```

---

## Summary: Three Different Things

| Term | What | Where | Why |
|------|------|-------|-----|
| **Memory** | Persistent knowledge | Agent-level (Crew) | Learn from past |
| **History** | Execution record | Callbacks/logs | Track what happened |
| **State** | Workflow data | Application-level (Flow) | Coordinate steps |

**Remember**: They're independent! You can use:
- Memory alone (agent learns)
- State alone (flow coordinates)
- Both together (coordinated learning)
- Neither (stateless operations)

---

**Next**: [Reliability](./12-reliability.md) for production resilience
