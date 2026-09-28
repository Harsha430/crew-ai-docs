# Tools: Creation & Execution

## What is a Tool?

**Tool = Agent capability: function + schema**

A tool is:
- A Python function agents can call
- A schema (name, description, parameters)
- Executed by agent when needed
- Result fed back to agent for further thinking

```python
from crewai_tools import tool

@tool
def web_search(query: str) -> str:
    """Search the web for information"""
    return search_implementation(query)

agent = Agent(role="...", goal="...", tools=[web_search])
```

---

## Creating Tools

### Simple Function Tool

```python
from crewai_tools import tool

@tool
def calculate_total(price: float, tax_rate: float) -> float:
    """Calculate total price with tax"""
    return price * (1 + tax_rate)

# Schema auto-generated from function
# Name: calculate_total
# Description: "Calculate total price with tax"
# Parameters: price (float), tax_rate (float)
```

### Tool with Complex Parameters

```python
from crewai_tools import tool
from pydantic import BaseModel

class QueryParams(BaseModel):
    query: str
    limit: int = 10
    sort_by: str = "relevance"

@tool
def search_database(params: QueryParams) -> list[dict]:
    """Search database with advanced filters"""
    return database.search(
        query=params.query,
        limit=params.limit,
        sort_by=params.sort_by
    )
```

### Tool with Error Handling

```python
from crewai_tools import tool

@tool
def safe_api_call(endpoint: str, timeout: int = 5) -> str:
    """Call API endpoint with error handling"""
    try:
        response = requests.get(
            f"https://api.example.com/{endpoint}",
            timeout=timeout
        )
        response.raise_for_status()
        return response.json()
    except requests.Timeout:
        return "ERROR: Request timed out"
    except requests.HTTPError as e:
        return f"ERROR: HTTP {e.response.status_code}"
    except Exception as e:
        return f"ERROR: {str(e)}"
```

---

## Tool Schema

### Generated Schema

```python
@tool
def example(param1: str, param2: int) -> str:
    """Do something with param1 and param2"""
    return f"{param1}: {param2}"

# CrewAI generates:
# {
#   "name": "example",
#   "description": "Do something with param1 and param2",
#   "parameters": {
#       "type": "object",
#       "properties": {
#           "param1": {"type": "string"},
#           "param2": {"type": "integer"}
#       },
#       "required": ["param1", "param2"]
#   }
# }
```

### Schema Quality Matters

**Good schema** (agent understands):
```python
@tool
def search(query: str) -> list[dict]:
    """Search the internet for information about the query.
    Returns list of results with url and snippet."""
    ...
```

**Bad schema** (agent confused):
```python
@tool
def search(q: str) -> list:
    """Search"""
    ...
```

**Better**:
```python
@tool
def web_search(query: str) -> list[dict]:
    """
    Search the internet for current information.
    
    Args:
        query: Search terms (e.g., "AI trends 2026")
    
    Returns:
        List of search results, each with:
        - url: Website URL
        - title: Page title
        - snippet: Text preview
    """
    ...
```

---

## Tool Execution Flow

```
1. Agent receives task + tools
    ↓
2. LLM sees tool schemas (name, description, parameters)
    ↓
3. LLM decides: "I should call tool X"
    ↓
4. LLM generates: tool_name="web_search", parameters={"query": "AI"}
    ↓
5. CrewAI validates:
    ├─ Does agent have this tool?
    ├─ Are parameters valid type?
    └─ All required params provided?
    ↓
6. Execute tool: web_search(query="AI")
    ↓
7. Get result: [{"title": "...", "url": "..."}]
    ↓
8. Add to agent context: "Tool result: ..."
    ↓
9. LLM continues thinking with result
    ↓
10. Loop (next iteration or final answer)
```

---

## Tool Return Types

### String Return

```python
@tool
def get_weather(city: str) -> str:
    """Get current weather"""
    return "Sunny, 72°F"

# Agent sees: "Tool result: Sunny, 72°F"
```

### Dict Return

```python
@tool
def get_user(user_id: int) -> dict:
    """Get user info"""
    return {"id": 123, "name": "Alice", "email": "alice@example.com"}

# Agent sees formatted dict
```

### List Return

```python
@tool
def search(query: str) -> list[str]:
    """Search for results"""
    return ["result1", "result2", "result3"]

# Agent sees formatted list
```

### Complex Object

```python
from dataclasses import dataclass

@dataclass
class Report:
    title: str
    findings: list[str]
    confidence: float

@tool
def analyze(data: str) -> Report:
    """Analyze data and return structured report"""
    return Report(
        title="Analysis",
        findings=["Finding 1", "Finding 2"],
        confidence=0.92
    )

# Agent sees formatted structure
```

### Error Return

```python
@tool
def risky_operation(param: str) -> str:
    """Do something risky"""
    try:
        result = dangerous_operation(param)
        return result
    except Exception as e:
        return f"ERROR: Operation failed - {str(e)}"

# Agent sees error message and can retry or adjust
```

---

## Tool Organization

### One Tool

```python
agent = Agent(
    role="Researcher",
    goal="Find information",
    tools=[web_search]
)
```

### Few Related Tools

```python
agent = Agent(
    role="Data Analyst",
    goal="Analyze data",
    tools=[
        database_query,
        statistical_analysis,
        chart_generator
    ]
)
```

### Many Tools (Specialization)

```python
# Good: Each agent specializes
researcher = Agent(
    role="Researcher",
    tools=[web_search, academic_database]
)

analyst = Agent(
    role="Analyst",
    tools=[sql_query, statistical_tools]
)

writer = Agent(
    role="Writer",
    tools=[]  # No tools, just thinking
)
```

### Anti-Pattern: Tool Overload

```python
# ❌ BAD: Too many tools confuses agent
agent = Agent(
    role="General Worker",
    goal="Do everything",
    tools=[
        web_search,
        database_query,
        api_call,
        file_reader,
        calculator,
        translator,
        code_executor,
        email_sender,
        ...  # 20 tools total
    ]
)

# Agent: "Which tool should I use?"
# Result: Random/confused tool selection
```

---

## Tool Best Practices

### ✅ DO

- **Clear names**: `web_search` (clear), not `tool_1` (unclear)
- **Good docstrings**: Describe what tool does and returns
- **Single purpose**: One tool = one capability
- **Reasonable scope**: Tool does one focused thing
- **Error handling**: Tool handles and reports errors
- **Type hints**: Function parameters have types
- **Return consistent format**: Always same type

### ❌ DON'T

- **Vague names**: `get_data` (unclear), `do_stuff` (unclear)
- **No description**: Agent won't know what tool does
- **Too broad**: Tool tries to do too much
- **No error handling**: Tool crashes, agent confused
- **Silent failures**: Tool fails, returns empty/None
- **Random return types**: Sometimes str, sometimes dict
- **Side effects**: Tool shouldn't have hidden behavior

---

## Common Tools

### Pre-Built CrewAI Tools

```python
from crewai_tools import (
    FileReadTool,
    FileWriteTool,
    DirectoryReadTool,
    CodeInterpreterTool,
    BaseTool
)

file_reader = FileReadTool()
code_tool = CodeInterpreterTool()

agent = Agent(
    role="Developer",
    goal="Write code",
    tools=[file_reader, code_tool]
)
```

### External API Tools

```python
@tool
def news_api(query: str) -> list[dict]:
    """Get latest news about a topic"""
    import requests
    response = requests.get(
        f"https://newsapi.org/v2/everything",
        params={"q": query}
    )
    return response.json()["articles"]
```

### Database Tools

```python
@tool
def query_db(sql: str) -> list[dict]:
    """Execute SQL query"""
    import sqlite3
    conn = sqlite3.connect("data.db")
    cursor = conn.execute(sql)
    return [dict(row) for row in cursor.fetchall()]
```

### Calculation Tools

```python
@tool
def calculate(expression: str) -> float:
    """Calculate mathematical expression"""
    return eval(expression)  # Note: Use with caution in production
```

---

## Tool Debugging

### Tool Not Called by Agent

**Problem**: Agent never calls your tool.

**Diagnosis**:
1. Is tool in agents' tools list?
2. Is tool description clear?
3. Does agent role/goal suggest need for tool?
4. Is tool name obvious?

**Fix**:
```python
# Before: Vague
@tool
def search(q: str) -> str:
    """Search for stuff"""
    ...

# After: Clear
@tool
def search_web(query: str) -> list[dict]:
    """
    Search the internet for information.
    Use this when you need to find facts, news, or current information.
    
    Args:
        query: Search terms
    
    Returns:
        List of results with title, url, snippet
    """
    ...

# In agent description
agent = Agent(
    role="Researcher",
    goal="Find accurate information using web_search tool",
    tools=[search_web]
)
```

### Tool Called With Wrong Parameters

**Problem**: Tool fails because parameters are wrong.

**Diagnosis**:
1. Are parameter types clear?
2. Does docstring explain parameters?
3. Are defaults confusing?

**Fix**:
```python
# Before: Confusing
@tool
def search(q: str, l: int = 10) -> str:
    """Search"""
    ...

# After: Clear
@tool
def search(query: str, limit: int = 10) -> list[dict]:
    """
    Search the web.
    
    Args:
        query: Search terms (e.g., "AI")
        limit: Max results to return (default 10)
    
    Returns:
        List of search results
    """
    ...
```

### Tool Returns Unexpected Format

**Problem**: Agent confused by tool output format.

**Diagnosis**:
1. Is return type consistent?
2. Is format documented in docstring?

**Fix**:
```python
# Before: Inconsistent
@tool
def analyze(data: str):
    """Analyze data"""
    if len(data) > 100:
        return {"result": "long"}
    else:
        return "short"  # Different type!

# After: Consistent
@tool
def analyze(data: str) -> dict:
    """Analyze data"""
    if len(data) > 100:
        return {"result": "long", "type": "analysis"}
    else:
        return {"result": "short", "type": "analysis"}  # Always dict
```

---

## Tool vs API vs Function

```
Function:
├─ Pure Python
├─ Agent can't call directly
└─ Must wrap in Tool

Tool:
├─ Function + Schema
├─ Agent can call
└─ Result to agent context

API:
├─ External HTTP endpoint
├─ Tool internally calls API
└─ Tool returns result to agent

Architecture:
Agent → Tool → API → External System
         ↑        ↑
      Wrapper    HTTP
```

---

## Summary

| Aspect | Detail |
|--------|--------|
| **Purpose** | Give agents capabilities |
| **Creation** | @tool decorated function |
| **Schema** | Auto-generated from function signature |
| **Return** | Any type, but consistent format best |
| **Execution** | LLM decides when to call |
| **Error handling** | Tool reports errors as strings |
| **Naming** | Clear, descriptive names matter |
| **Documentation** | Good docstrings = better agent usage |

---

**Next**: [Common Mistakes](./11-common-mistakes.md)
