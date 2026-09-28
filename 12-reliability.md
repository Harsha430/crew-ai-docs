# Reliability: Production Patterns

**Building systems that work when things break**

---

## Failure Types

```
Transient (recoverable)
├─ Network timeout → retry
├─ Rate limit → wait + retry
├─ Temporary LLM error → retry
└─ Tool service temporarily down → retry

Permanent (not recoverable)
├─ Invalid API key → fail fast
├─ Malformed input → fail fast
├─ Tool doesn't exist → fail fast
└─ Wrong credentials → fail fast

Business (domain-specific)
├─ Quality too low → retry or escalate
├─ Content violates policy → reject
├─ Confidence too low → escalate
└─ Requires human decision → escalate

Safety (security/compliance)
├─ PII detected in output → redact
├─ Toxic content → block
├─ Unauthorized access → deny
└─ Compliance violation → log + alert
```

---

## Retry Strategy

### Immediate Retry

```python
# Use for: Transient errors that usually pass immediately
@tool
def api_call(endpoint: str) -> str:
    """Call API with immediate retry"""
    for attempt in range(3):
        try:
            return requests.get(f"https://api.example.com/{endpoint}").json()
        except requests.ConnectionError:
            if attempt == 2:
                raise
            continue  # Try again immediately
```

### Exponential Backoff

```python
import time

def call_with_backoff(func, max_retries=5):
    """Retry with exponential backoff: 1s, 2s, 4s, 8s, 16s"""
    for attempt in range(max_retries):
        try:
            return func()
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            
            wait_time = 2 ** attempt  # 1, 2, 4, 8, 16
            print(f"Attempt {attempt + 1} failed. Retrying in {wait_time}s...")
            time.sleep(wait_time)

# Usage in tool
@tool
def api_call(query: str) -> str:
    """Call API with exponential backoff"""
    return call_with_backoff(
        lambda: requests.get(f"https://api.example.com/search?q={query}").json()
    )
```

### Jittered Backoff

```python
import time
import random

def call_with_jitter(func, max_retries=5):
    """Backoff + jitter prevents thundering herd"""
    for attempt in range(max_retries):
        try:
            return func()
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            
            # Exponential backoff + random jitter
            base_wait = 2 ** attempt
            jitter = random.uniform(0, base_wait)
            wait_time = base_wait + jitter
            
            print(f"Attempt {attempt + 1} failed. Retrying in {wait_time:.2f}s...")
            time.sleep(wait_time)
```

### When to Retry

```python
# Retry these:
RETRYABLE_ERRORS = {
    "timeout",
    "connection",
    "temporarily unavailable",
    "rate limit",
    "service unavailable"
}

# Don't retry these:
PERMANENT_ERRORS = {
    "invalid api key",
    "not found",
    "unauthorized",
    "malformed request"
}

def should_retry(error_message: str) -> bool:
    error_lower = error_message.lower()
    return any(e in error_lower for e in RETRYABLE_ERRORS)
```

---

## Fallback Strategy

### Tool Fallback

```python
@tool
def search_with_fallback(query: str) -> str:
    """Try primary search, fallback to cache if fails"""
    try:
        # Try primary API
        return primary_search(query)
    except Exception as e:
        print(f"Primary search failed: {e}")
        
        # Fallback to cache
        cached = get_from_cache(query)
        if cached:
            return f"(cached) {cached}"
        
        # Last resort: return error
        return f"ERROR: Could not search. {str(e)}"
```

### Agent Fallback

```python
# Primary agent fails, use backup
primary_agent = Agent(role="Specialist", goal="...", tools=[specialized_tools])
backup_agent = Agent(role="Generalist", goal="...", tools=[general_tools])

# In Flow/Task:
def execute_with_fallback(primary_agent, backup_agent, task):
    try:
        return primary_agent.execute(task)
    except Exception as e:
        print(f"Primary agent failed: {e}")
        return backup_agent.execute(task)
```

### Model Fallback

```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic

def get_llm_with_fallback():
    """Try GPT-4, fallback to Claude"""
    try:
        llm = ChatOpenAI(model="gpt-4")
        # Test it works
        llm.invoke("test")
        return llm
    except Exception as e:
        print(f"GPT-4 failed: {e}, using Claude")
        return ChatAnthropic(model="claude-opus")

agent = Agent(
    role="Thinker",
    goal="Think deeply",
    llm=get_llm_with_fallback()
)
```

---

## Circuit Breaker

```
Closed (normal)
├─ Requests flow through
├─ Failures counted
└─ If failures exceed threshold → Open

Open (broken)
├─ Requests immediately fail
├─ No calls to broken service
└─ After timeout → Half-Open

Half-Open (testing)
├─ Try one request
├─ If succeeds → Close
└─ If fails → Open
```

### Implementation

```python
import time
from enum import Enum

class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time = None
    
    def call(self, func, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.timeout:
                self.state = CircuitState.HALF_OPEN
                self.failure_count = 0
            else:
                raise Exception("Circuit breaker is open")
        
        try:
            result = func(*args, **kwargs)
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise

    def on_success(self):
        self.failure_count = 0
        self.state = CircuitState.CLOSED
    
    def on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN

# Usage in tool
breaker = CircuitBreaker(failure_threshold=3, timeout=30)

@tool
def external_api(query: str) -> str:
    """Call external API with circuit breaker"""
    return breaker.call(lambda: requests.get(f"https://api.example.com/{query}").json())
```

---

## Graceful Degradation

### Feature Degradation

```
Full system:
├─ Vision analysis
├─ Text extraction  
├─ Entity linking
├─ Sentiment analysis
└─ Returns: Rich analysis

If Vision fails:
├─ Text extraction only
├─ Reduced Entity linking
├─ Sentiment analysis
└─ Returns: Partial analysis (degraded)

If multiple fail:
├─ Text extraction
└─ Returns: Basic extraction (very degraded)
```

### Implementation

```python
class RichAnalysisTask:
    def execute(self, document):
        results = {}
        
        # Try vision analysis
        try:
            results["vision"] = self.analyze_vision(document)
        except Exception as e:
            print(f"Vision failed: {e}")
            results["vision"] = None
        
        # Try extraction (always attempt)
        try:
            results["extraction"] = self.extract_text(document)
        except Exception as e:
            print(f"Extraction failed: {e}")
            results["extraction"] = None
        
        # Try entity linking (if extraction succeeded)
        if results["extraction"]:
            try:
                results["entities"] = self.link_entities(results["extraction"])
            except Exception as e:
                print(f"Entity linking failed: {e}")
                results["entities"] = None
        
        # Synthesize what we have
        return self.synthesize_results(results)
    
    def synthesize_results(self, results):
        """Use whatever succeeded"""
        if results["vision"]:
            return "Full analysis: " + results["vision"]
        elif results["extraction"]:
            return "Text analysis: " + results["extraction"]
        else:
            return "ERROR: Could not process document"
```

---

## Idempotency

**Problem**: Retry causes duplicate side effects

```
Request 1: "Create payment"
  ├─ Sent to API
  ├─ API processes
  ├─ Response lost (network timeout)
  └─ We don't know if it succeeded

Retry: "Create payment"
  ├─ Sent again
  ├─ API creates ANOTHER payment
  └─ Duplicate charge!
```

### Solution: Idempotency Key

```python
import uuid

@tool
def create_payment(amount: float, customer_id: str) -> dict:
    """Create payment with idempotency key"""
    
    # Generate unique key for this operation
    idempotency_key = str(uuid.uuid4())
    
    response = requests.post(
        "https://payment-api.example.com/payments",
        json={"amount": amount, "customer_id": customer_id},
        headers={"Idempotency-Key": idempotency_key}
    )
    
    # API returns same result if called with same key
    return response.json()

# On retry: same idempotency_key → API returns cached result
# No duplicate charge!
```

### Idempotent vs Non-Idempotent

| Operation | Idempotent? | Why |
|-----------|------------|-----|
| Read user | ✅ Yes | Same result every time |
| Update name | ✅ Yes | Doing it twice = same as once |
| Create payment | ❌ No | Each call = new charge |
| Delete file | ❌ No | First delete succeeds, second fails |
| Increment counter | ❌ No | Each call increments |

**Safe to retry**: Read, update (idempotent), delete (with idempotency key)  
**Unsafe to retry**: Create, delete (without key), increment

---

## Error Classification

```python
class ErrorClassifier:
    @staticmethod
    def classify(error: Exception) -> str:
        error_msg = str(error).lower()
        
        if "timeout" in error_msg or "deadline" in error_msg:
            return "transient"
        elif "rate limit" in error_msg or "429" in error_msg:
            return "transient"
        elif "invalid" in error_msg or "malformed" in error_msg:
            return "permanent"
        elif "unauthorized" in error_msg or "401" in error_msg:
            return "permanent"
        elif "not found" in error_msg or "404" in error_msg:
            return "permanent"
        elif "quality" in error_msg or "confidence" in error_msg:
            return "business"
        elif "unsafe" in error_msg or "blocked" in error_msg:
            return "safety"
        else:
            return "unknown"

# Usage
error_type = ErrorClassifier.classify(error)

if error_type == "transient":
    retry()
elif error_type == "permanent":
    fail_fast()
elif error_type == "business":
    escalate_to_human()
elif error_type == "safety":
    log_alert_and_block()
```

---

## Timeout Strategy

```python
# LLM call timeout
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(
    model="gpt-4",
    timeout=30  # 30 second timeout
)

agent = Agent(role="...", goal="...", llm=llm)

# Tool timeout
import signal

def timeout_handler(signum, frame):
    raise TimeoutError("Operation exceeded timeout")

signal.signal(signal.SIGALRM, timeout_handler)
signal.alarm(60)  # 60 second timeout

try:
    result = slow_operation()
finally:
    signal.alarm(0)  # Cancel timeout
```

---

## Production Reliability Checklist

✅ **DO:**
- Classify errors (transient vs permanent)
- Retry with exponential backoff (not immediate)
- Implement circuit breakers for external APIs
- Use idempotency keys for mutations
- Degrade gracefully (don't fail completely)
- Monitor retry rates (high = problem)
- Log all failures with context
- Test failure scenarios

❌ **DON'T:**
- Retry everything blindly
- Retry forever (max attempts needed)
- Ignore circuit breaker signals
- Create duplicate side effects (use idempotency)
- Fail completely (degrade gracefully)
- Leave retries silent (log them)
- Trust one attempt (test with failures)

---

**Next**: [Security](./13-security.md)
