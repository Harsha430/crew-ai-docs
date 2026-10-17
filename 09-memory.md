# Memory: Complete Guide


# CrewAI Memory, History, State, and Storage — Updated Guide

> **Core rule: Memory ≠ Conversation History ≠ Flow State ≠ Storage.**
>
> This guide reflects the current CrewAI unified `Memory` API documented for CrewAI v1.15.28. Older tutorials may describe separate short-term, long-term, and entity-memory components. The current API consolidates those responsibilities behind one `Memory` abstraction.

## 1. The mental model

```text
CrewAI Memory (unified API)
├── Purpose: Save and retrieve useful facts/knowledge
├── Organization: Hierarchical scopes, categories, metadata, importance
├── Retrieval: Semantic similarity + recency + importance
├── Analysis: LLM can infer missing scope/categories/importance
└── Storage: LanceDB by default; configurable backend

Conversation / task context
├── Purpose: Give an agent context for the current task
├── Lifetime: Depends on the task/agent execution and what is passed forward
└── Not the same thing as a durable memory record

Flow state
├── Purpose: Carry structured workflow data between Flow methods
├── Access: self.state
├── Formats: dict-like unstructured state or Pydantic structured state
└── Persistence: Optional; use @persist for persisted Flow state

Storage backend
├── Purpose: Physically keep memory records or Flow state
├── Memory default: LanceDB under ./.crewai/memory
└── Flow persistence default: SQLiteFlowPersistence
```

These are distinct responsibilities. A memory record can persist without representing a resumable workflow state. A Flow state can be persisted without automatically becoming a semantic memory that an agent recalls.

---

## 2. Does CrewAI still have short-term and long-term memory?

**Yes as concepts; not as separate required memory classes in the current unified API.**

The current `Memory` class replaces the older separate short-term, long-term, entity, and external-memory abstractions. In practice:

- **Short-term memory** is a useful label for information relevant to the current task/session or recent interaction.
- **Long-term memory** is a useful label for facts intended to be recalled across tasks, executions, or sessions.
- **Entity memory** is a useful label for facts about people, organizations, products, and other entities.

In the unified API, you organize these records with scopes, categories, metadata, importance, and retention rules you define. Do not assume CrewAI automatically labels every record as `SHORT_TERM` or `LONG_TERM`, or automatically deletes short-lived records when a task ends.

### Example: temporary context versus durable preference

```python
from crewai import Memory

memory = Memory()

# Intended for one report/session.
memory.remember(
    "Retrieved three products for report R101.",
    scope="/sessions/R101",
    categories=["execution-context"],
)

# Intended to be useful in future sessions.
memory.remember(
    "Customer prefers concise financial comparisons.",
    scope="/customers/C101/preferences",
    categories=["user-preference"],
    importance=0.9,
)
```

The labels and paths are chosen by the developer in this example. A scope does not itself implement an expiration policy. If session memory must be removed after a run, explicitly delete/reset the relevant scope or implement a lifecycle policy. If memory is customer-specific, enforce authentication and authorization in your application.

---

## 3. How unified Memory works

The current system can use an LLM at several points:

1. **Save analysis:** If you omit scope, categories, or importance, the LLM can infer them from the content and existing scope tree.
2. **Memory extraction:** `extract_memories()` can split a long piece of text into smaller, discrete facts.
3. **Recall analysis:** Deep recall can analyze a complex query and guide retrieval.
4. **Consolidation:** When new content resembles existing records, the LLM can decide whether the existing record should be kept, updated, deleted, or supplemented with a new record.

This is not a guarantee that the LLM always classifies information correctly. If the classification matters for authorization, retention, or compliance, supply the scope and metadata explicitly and validate them in application code.

### Save and recall

```python
from crewai import Memory

memory = Memory()

memory.remember(
    "We decided to use PostgreSQL for the user database."
)

matches = memory.recall("What database did we choose?")

for match in matches:
    print(f"[{match.score:.2f}] {match.record.content}")
```

When the scope is omitted, CrewAI can infer a placement. The exact inferred scope and category are not deterministic business rules; inspect the result if your application depends on them.

### Explicit scope and category

```python
memory.remember(
    "The API rate limit is 1000 requests per minute.",
    scope="/product/api-limits",
    categories=["technical-fact", "api"],
    importance=0.8,
)
```

Use explicit values when you already know where the record belongs.

---

## 4. How does CrewAI choose which memories to retrieve?

The current documentation describes a composite ranking using three signals:

```text
score =
    semantic_weight * semantic_similarity
  + recency_weight  * recency_decay
  + importance_weight * importance
```

Where:

- **Semantic similarity** estimates how relevant the record is to the query.
- **Recency decay** gives newer records a higher recency score.
- **Importance** is the record's importance value.

The documented recency decay is:

```text
decay = 0.5 ** (age_days / recency_half_life_days)
```

For example, with a 30-day half-life, the recency component is approximately 1 for a new record and 0.5 after 30 days. This changes ranking; it does **not** delete the record after 30 days.

You can tune the weights:

```python
from crewai import Memory

memory = Memory(
    semantic_weight=0.5,
    recency_weight=0.3,
    importance_weight=0.2,
    recency_half_life_days=30,
)

memory.remember(
    "The application uses PostgreSQL.",
    scope="/project/alpha/architecture",
    categories=["database", "architecture"],
    importance=0.9,
)

matches = memory.recall(
    "Which database does the application use?",
    scope="/project/alpha",
    limit=5,
)
```

The weights are examples and can be changed for your use case. A recent project-retrospective assistant might favor recency; a stable architecture knowledge base might favor semantic relevance and importance.

### Shallow versus deep recall

```python
# Fast direct vector search; suitable for straightforward queries.
matches = memory.recall(
    "Which database did we choose?",
    depth="shallow",
    limit=5,
)

# Deep recall can use LLM-assisted query analysis and exploration.
matches = memory.recall(
    "Summarize the important architecture decisions made this quarter.",
    depth="deep",
    limit=10,
)
```

Deep recall may take longer and may require LLM calls. Use it for complex queries rather than assuming every retrieval needs the most elaborate path.

---

## 5. `memory=True` in a Crew

You can enable the default unified memory system on a crew:

```python
from crewai import Crew, Process

crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, writing_task],
    process=Process.sequential,
    memory=True,
    verbose=True,
)

result = crew.kickoff()
```

With `memory=True`, the crew creates a default `Memory()` instance. The crew's embedder configuration is passed through. Unless an agent has its own memory configured, agents in the crew share the crew memory.

According to the current documentation:

1. Before a task, relevant memories can be recalled and injected into the task prompt.
2. After a task, discrete facts can be extracted from the task output and stored.
3. Later tasks can recall relevant saved information.

This does **not** mean every token from every interaction is necessarily stored as a perfect transcript. The memory flow extracts facts from task output and saves memory records.

### Use a custom Memory instance

```python
from crewai import Crew, Memory, Process

memory = Memory(
    semantic_weight=0.4,
    recency_weight=0.4,
    importance_weight=0.2,
    recency_half_life_days=14,
)

crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, writing_task],
    process=Process.sequential,
    memory=memory,
)
```

This gives you control over memory retrieval ranking rather than only enabling default settings.

### Agent-specific memory scope

```python
from crewai import Agent, Memory

shared_memory = Memory()

researcher = Agent(
    role="Researcher",
    goal="Find and analyze accurate information",
    backstory="A careful researcher.",
    memory=shared_memory.scope("/agent/researcher"),
)

writer = Agent(
    role="Writer",
    goal="Create a clear report",
    backstory="A concise technical writer.",
    # If the crew has memory enabled and no agent memory is specified,
    # it can use the crew's shared memory.
)
```

A scope limits memory operations to a branch of the scope tree. This is useful for organizing context, but do not treat a scope as a substitute for application-level authorization checks.

---

## 6. Scope, category, importance, and source are different

These fields answer different questions:

| Field | Question answered | Example |
|---|---|---|
| Scope | Where should this record be organized and searched? | `/customers/C101/preferences` |
| Category | What kind of information is it? | `user-preference` |
| Importance | How significant is it for ranking? | `0.9` |
| Source | Where did the record come from? | `user:C101` |
| Recency | How recent is the record? | Derived from record age |
| Semantic similarity | How relevant is it to the query? | Derived from vector search |

Example:

```python
memory.remember(
    "Customer prefers concise comparisons.",
    scope="/customers/C101/preferences",
    categories=["user-preference"],
    importance=0.9,
    source="user:C101",
)
```

A `source` tag can help with provenance and filtering. The documentation also supports private records, which are visible only when the source matches unless privileged access is requested. Avoid storing secrets such as API keys in memory even if a private flag exists; use a secret manager instead.

---

## 7. Memory storage and persistence

### Default memory backend

The current CrewAI documentation specifies:

- **Backend:** LanceDB by default.
- **Default path:** `./.crewai/memory`.
- **Environment override:** `$CREWAI_STORAGE_DIR/memory`.
- **Custom path:** `Memory(storage="path/to/dir")`.
- **Custom backend:** An implementation of the `StorageBackend` protocol can be supplied.

The default embedder is OpenAI `text-embedding-3-large` unless you configure a different embedder. The memory LLM used for analysis defaults to `gpt-4o-mini` in the documented configuration. These defaults can change between framework versions, so verify the documentation for the version pinned by your project.

### Use a custom storage path

```python
from crewai import Memory

memory = Memory(storage="./data/crewai_memory")
```

### Inspect stored memory

```python
print(memory.tree())
print(memory.info("/customers/C101"))
print(memory.list_scopes("/"))
print(memory.list_categories())
print(memory.list_records(scope="/customers/C101", limit=20))
```

### Reset memory

```python
# Reset all records in this Memory instance.
memory.reset()

# Or reset only a particular subtree.
memory.reset(scope="/project/old")
```

Resetting is destructive. Ensure the target scope is correct before using it in a production application.

### Persistence does not equal retention policy

A durable backend means records can survive a process restart. It does not decide your business retention rules for you. Define:

- Which records are allowed to be saved.
- How customer data is isolated.
- When session records expire.
- How users request deletion.
- Which facts need verification before being reused.

---

## 8. Flow state is separate from Memory

Use Flow state for data required to control the current workflow: validated input, IDs, retrieved documents, approval status, and outputs.

CrewAI supports unstructured dictionary-like state and structured Pydantic state.

### Unstructured state

```python
from crewai.flow.flow import Flow, start, listen

class ResearchFlow(Flow):
    @start()
    def gather_data(self):
        self.state["query"] = "Compare database options"
        self.state["validated"] = True
        return self.state["query"]

    @listen(gather_data)
    def retrieve(self, query):
        self.state["documents"] = [
            "PostgreSQL overview",
            "MySQL overview",
        ]
        return self.state["documents"]

flow = ResearchFlow()
result = flow.kickoff()
```

### Structured state

```python
from pydantic import BaseModel
from crewai.flow.flow import Flow, start, listen

class ResearchState(BaseModel):
    query: str = ""
    validated: bool = False
    documents: list[str] = []

class ResearchFlow(Flow[ResearchState]):
    @start()
    def gather_data(self):
        self.state.query = "Compare database options"
        self.state.validated = True
        return self.state.query

    @listen(gather_data)
    def retrieve(self, query):
        self.state.documents = [
            "PostgreSQL overview",
            "MySQL overview",
        ]
        return self.state.documents

flow = ResearchFlow()
result = flow.kickoff()
```

State lets your code explicitly pass and update execution data. It does not automatically turn every field into a semantic memory record.

### Persist Flow state with `@persist`

```python
from pydantic import BaseModel
from crewai.flow.flow import Flow, start, listen
from crewai.flow.persistence import persist

class CounterState(BaseModel):
    counter: int = 0

@persist
class CounterFlow(Flow[CounterState]):
    @start()
    def increment(self):
        self.state.counter += 1
        print("Counter:", self.state.counter)

    @listen(increment)
    def next_step(self):
        self.state.counter += 1
        print("Counter:", self.state.counter)

flow = CounterFlow()
flow.kickoff()
```

`@persist` enables Flow state persistence. The documented default persistence backend is `SQLiteFlowPersistence`. Persisted state has an ID that can be used to resume or fork execution, according to the Flow persistence API.

This is different from `Memory.remember()`:

- `Memory.remember()` saves information for later retrieval.
- `@persist` saves Flow state for workflow continuation.

---

## 9. Memory vs. history vs. state vs. storage

| Aspect | Memory | Conversation/task context | Flow state | Storage backend |
|---|---|---|---|---|
| Purpose | Retrieve useful information from the past | Provide context to the current task/model | Coordinate workflow execution | Physically retain records |
| Typical content | Preferences, facts, decisions, extracted findings | Messages and task inputs/outputs | IDs, flags, retrieved docs, step results | Serialized records/state |
| Retrieval | Semantic/relevance-based recall | Passed through execution/context mechanisms | Direct field/key access | Backend-specific query/read |
| Survives restart? | If its backend is persistent and reused | Not guaranteed | Only if persisted | Depends on backend |
| Typical API | `remember`, `recall`, `forget` | Depends on agent/task implementation | `self.state` | LanceDB, SQLite, or custom backend |

**Important:** CrewAI does not promise that all execution history is automatically stored as memory. Task outputs may be used to extract discrete facts, which are then saved to the unified memory system.

---

## 10. Custom memory: simple JSON example

Use a custom implementation only if you need behavior different from the framework's unified memory API. The example below is a simple keyword-based JSON store, not a replacement for semantic vector retrieval.

```python
import json
from datetime import datetime, timezone
from pathlib import Path
from typing import Any

class CustomJsonMemory:
    def __init__(self, filepath: str = "memory.json"):
        self.filepath = Path(filepath)
        self.records: list[dict[str, Any]] = self._load()

    def _load(self) -> list[dict[str, Any]]:
        if not self.filepath.exists():
            return []

        try:
            with self.filepath.open("r", encoding="utf-8") as file:
                data = json.load(file)
            return data if isinstance(data, list) else []
        except (json.JSONDecodeError, OSError):
            # In production, log and handle corruption explicitly.
            return []

    def remember(
        self,
        key: str,
        value: str,
        *,
        scope: str = "/",
        categories: list[str] | None = None,
    ) -> None:
        record = {
            "key": key,
            "value": value,
            "scope": scope,
            "categories": categories or [],
            "timestamp": datetime.now(timezone.utc).isoformat(),
        }
        self.records.append(record)
        self._save()

    def recall(self, query: str, limit: int = 5) -> list[dict[str, Any]]:
        terms = set(query.casefold().split())
        scored = []

        for record in self.records:
            text = f"{record['key']} {record['value']}".casefold()
            score = sum(1 for term in terms if term in text)
            if score:
                scored.append((score, record))

        scored.sort(key=lambda item: item[0], reverse=True)
        return [record for _, record in scored[:limit]]

    def forget_scope(self, scope: str) -> None:
        prefix = scope.rstrip("/")
        self.records = [
            record
            for record in self.records
            if not (
                record["scope"] == prefix
                or record["scope"].startswith(prefix + "/")
            )
        ]
        self._save()

    def _save(self) -> None:
        self.filepath.parent.mkdir(parents=True, exist_ok=True)
        with self.filepath.open("w", encoding="utf-8") as file:
            json.dump(self.records, file, indent=2, ensure_ascii=False)

# Example
memory = CustomJsonMemory()

memory.remember(
    key="database_decision",
    value="The project uses PostgreSQL.",
    scope="/project/alpha/decisions",
    categories=["architecture", "database"],
)

print(memory.recall("Which database does the project use?"))
```

Limitations of this example:

- Retrieval is keyword matching, not embedding similarity.
- There is no concurrency control or database transaction handling.
- There is no automatic fact extraction or LLM-assisted consolidation.
- It does not provide production-grade authorization or retention controls.

For production, use a supported persistent backend and implement access control, auditing, backup, and deletion policies.

---

## 11. When memory helps

### Good uses

- Reusing findings across multiple tasks or executions.
- Remembering stable user preferences.
- Sharing relevant information among agents.
- Building up project decisions and research findings.
- Reducing repeated discovery work.

### Cases requiring care

- Highly sensitive information.
- Facts that change frequently and must always be current.
- One-off tasks where retrieval overhead is unnecessary.
- Applications requiring strict data isolation, deletion, or retention.
- High-stakes decisions where recalled information must be verified.

Memory is not a source of truth for live balances, account status, or other transactional data. Query the authoritative system for those values.

---

## 12. Troubleshooting

### Memory is not being recalled

1. Confirm the crew has `memory=True` or a configured `Memory` instance.
2. Check that the same storage location/backend is being reused.
3. Inspect the scope tree with `memory.tree()`.
4. Inspect records and categories with `memory.list_records()` and `memory.list_categories()`.
5. Test recall with a concise, direct query.
6. Try `depth="shallow"` to isolate vector retrieval from deep query analysis.
7. Check embedding/model configuration and logs.

### Memory does not survive restart

1. Confirm the path is writable.
2. Confirm the process is using the same storage path.
3. Check `CREWAI_STORAGE_DIR` if configured.
4. Do not confuse memory storage with Flow state persistence.
5. For Flow resumption, configure `@persist` and use the appropriate state ID.

### Memory grows too large

- Store atomic facts rather than whole transcripts when possible.
- Use scopes to organize records.
- Remove obsolete scopes/records according to your retention policy.
- Monitor storage size and retrieval quality.
- Do not assume recency decay deletes old data.

### Memory analysis is slow or fails

- Deep recall may invoke an LLM for complex queries.
- Shallow recall is available for direct vector search.
- Verify LLM and embedding credentials.
- The documented implementation can fall back to defaults for some analysis failures, but storage or embedding failures can still cause errors.

---

## 13. Practical rules to remember

1. **Memory is not history.** The unified memory system stores retrievable records; it is not necessarily a complete transcript.
2. **Memory is not Flow state.** Memory is for recalling facts; state is for controlling execution.
3. **Short-term and long-term are intended uses/lifetimes.** They are not separate required classes in the current unified API.
4. **Scope is organization, not automatic expiration.**
5. **Importance and recency affect ranking, not deletion.**
6. **`memory=True` enables Crew memory behavior.** It does not replace explicit workflow state design.
7. **LanceDB is the default Memory backend in the current documentation.**
8. **SQLiteFlowPersistence is the default backend for persisted Flow state.**
9. **Persistent storage does not define your data-retention policy.**
10. **Validate critical metadata and enforce access control in application code.**

---

## 14. Official documentation

- CrewAI Memory: https://docs.crewai.com/en/concepts/memory
- CrewAI Flows and state management: https://docs.crewai.com/en/concepts/flows
- CrewAI Crews: https://docs.crewai.com/en/concepts/crews

**Version note:** This guide follows the current official documentation at the time of writing (CrewAI documentation version v1.15.28). Check these official pages against the version pinned in your environment before copying examples into a production project.