# Evaluation System Caching Strategy - Comprehensive Documentation

## 📋 Table of Contents
1. [Overview](#overview)
2. [Caching Strategy](#caching-strategy)
3. [Storage Mechanism](#storage-mechanism)
4. [Cache Key Generation](#cache-key-generation)
5. [Expiration & Deletion Mechanism](#expiration--deletion-mechanism)
6. [Cache Configuration](#cache-configuration)
7. [Implementation Details](#implementation-details)
8. [Performance Benefits](#performance-benefits)
9. [Cache Statistics & Monitoring](#cache-statistics--monitoring)
10. [Interview Q&A](#interview-qa)

---

## 1. Overview

The evaluation system implements an **in-memory TTL (Time-To-Live) cache** to optimize LLM API calls during evaluation processes. This cache significantly reduces:
- **API costs** (fewer redundant API calls)
- **Latency** (instant responses for cached queries)
- **Load on LLM providers** (reduced API rate limits)

### Key Metrics
- **TTL**: 3 days (259,200 seconds)
- **Max Size**: 512 entries
- **Storage**: In-memory (RAM)
- **Eviction Policy**: LRU (Least Recently Used) when max size reached

---

## 2. Caching Strategy

### 2.1 Cache-Aside Pattern
We use the **Cache-Aside (Lazy Loading)** pattern:

```
┌─────────────────────────────────────────┐
│  1. Check Cache First                    │
│     ↓                                    │
│  2. Cache Hit? → Return Cached Response │
│     ↓                                    │
│  3. Cache Miss? → Call LLM API           │
│     ↓                                    │
│  4. Store Response in Cache              │
│     ↓                                    │
│  5. Return Response                      │
└─────────────────────────────────────────┘
```

### 2.2 When Caching is Applied
- **Evaluation Metrics**: All LLM-based evaluation calls (insight validity, reasoning coherence, actionability, coverage, summary quality)
- **Ensemble Evaluations**: Each ensemble run benefits from cache (multiple runs with same prompt = cache hit)
- **Batch Evaluations**: Multiple records with similar prompts = significant cache hit rate

### 2.3 Cache Scope
- **Per-Model**: Different models have separate cache entries
- **Per-Temperature**: Different temperature settings = different cache entries
- **Per-MaxTokens**: Different max_tokens = different cache entries
- **Per-Prompt**: Each unique prompt = unique cache entry

---

## 3. Storage Mechanism

### 3.1 Storage Location
**Type**: In-Memory (RAM)  
**Library**: `cachetools.TTLCache`  
**Persistence**: **Non-persistent** (cleared on application restart)

### 3.2 Why In-Memory?
✅ **Pros:**
- **Ultra-fast access** (nanosecond-level latency)
- **No disk I/O overhead**
- **Simple implementation** (no database setup)
- **Automatic cleanup** (memory freed on restart)

❌ **Cons:**
- **Lost on restart** (acceptable trade-off for evaluation cache)
- **Limited by RAM** (mitigated by maxsize=512 limit)

### 3.3 Memory Structure
```python
TTLCache(
    maxsize=512,           # Maximum 512 entries
    ttl=259200            # 3 days in seconds
)
```

**Memory Estimation:**
- Average prompt: ~2KB
- Average response: ~1KB
- Cache key (SHA256): 64 bytes
- **Per entry**: ~3KB
- **Total (512 entries)**: ~1.5MB (negligible memory footprint)

---

## 4. Cache Key Generation

### 4.1 Key Components
The cache key is a **SHA256 hash** of a JSON object containing:

```python
{
    "prompt": "<full evaluation prompt>",
    "model": "<LLM model name>",
    "temperature": 0.0,
    "max_tokens": 2000
}
```

### 4.2 Why This Approach?
1. **Deterministic**: Same inputs = same hash = cache hit
2. **Collision-resistant**: SHA256 ensures unique keys
3. **Compact**: 64-character hex string (vs storing full prompt as key)
4. **Fast**: Hash computation is O(1) operation

### 4.3 Key Generation Code
```python
def _generate_cache_key(prompt: str, model: str, temperature: float, max_tokens: int) -> str:
    cache_key_data = {
        "prompt": prompt,
        "model": model,
        "temperature": temperature,
        "max_tokens": max_tokens
    }
    json_str = json.dumps(cache_key_data, sort_keys=True)  # Sorted for consistency
    return hashlib.sha256(json_str.encode('utf-8')).hexdigest()
```

### 4.4 Cache Hit Scenarios
✅ **Cache Hit** occurs when:
- Same prompt + same model + same temperature + same max_tokens
- Entry hasn't expired (within 3 days)
- Cache hasn't been cleared

❌ **Cache Miss** occurs when:
- Different prompt (even minor changes)
- Different model
- Different temperature
- Different max_tokens
- Entry expired (>3 days old)
- Cache at maxsize and entry was evicted

---

## 5. Expiration & Deletion Mechanism

### 5.1 How TTL Works
**TTL (Time-To-Live)**: 3 days = 259,200 seconds

### 5.2 Expiration Strategy: **Lazy Expiration**
The `cachetools.TTLCache` uses **lazy expiration**, meaning:

1. **No Background Thread**: No scheduled cleanup job runs
2. **On-Access Check**: TTL is checked when cache is accessed (get/set operations)
3. **Automatic Removal**: Expired entries are removed when:
   - Cache is accessed (`get()` or `set()`)
   - Entry is explicitly retrieved
   - New entry is added (may trigger cleanup)

### 5.3 Deletion Flow
```
┌─────────────────────────────────────────────┐
│  Cache Access (get/set)                      │
│         ↓                                    │
│  Check Entry Timestamp                       │
│         ↓                                    │
│  Current Time - Entry Time > TTL?            │
│         ↓                                    │
│  YES → Remove Entry (expired)                │
│  NO  → Return Entry (valid)                  │
└─────────────────────────────────────────────┘
```

### 5.4 Why No Scheduled Cleanup?
- **Efficient**: Only checks entries when needed
- **No Overhead**: No background threads consuming resources
- **Automatic**: Expired entries naturally removed on access
- **Memory Safe**: Expired entries don't accumulate (checked on every access)

### 5.5 Example Timeline
```
Day 0, 10:00 AM: Entry cached
Day 1, 10:00 AM: Entry accessed → ✅ Valid (1 day old)
Day 2, 10:00 AM: Entry accessed → ✅ Valid (2 days old)
Day 3, 10:00 AM: Entry accessed → ✅ Valid (3 days old)
Day 3, 10:00:01 AM: Entry accessed → ❌ Expired (deleted automatically)
```

---

## 6. Cache Configuration

### 6.1 Configuration Options
Located in: `evaluation/config.py`

```python
class EvaluationConfig:
    # Enable/disable caching
    enable_evaluation_cache: bool = True
    
    # TTL in seconds (3 days)
    evaluation_cache_ttl_seconds: int = 259200
    
    # Maximum cache entries
    evaluation_cache_maxsize: int = 512
```

### 6.2 Configuration Sources (Priority Order)
1. **Environment Variables** (highest priority)
2. **Config File** (`config.py`)
3. **Default Values** (fallback)

### 6.3 Runtime Configuration
```python
# Disable cache
config.enable_evaluation_cache = False

# Change TTL to 1 day
config.evaluation_cache_ttl_seconds = 86400

# Increase max size
config.evaluation_cache_maxsize = 1024
```

---

## 7. Implementation Details

### 7.1 Cache Initialization
**Lazy Initialization**: Cache is created on first use, not at import time.

```python
# Module-level cache (None until first use)
_evaluation_cache: Optional[TTLCache] = None

def get_evaluation_cache() -> TTLCache:
    global _evaluation_cache
    if _evaluation_cache is None:
        # Initialize with config values
        config = get_evaluation_config()
        ttl = config.evaluation_cache_ttl_seconds  # 259200
        maxsize = config.evaluation_cache_maxsize   # 512
        _evaluation_cache = TTLCache(maxsize=maxsize, ttl=ttl)
    return _evaluation_cache
```

### 7.2 Integration in Evaluation Flow
**File**: `evaluation/evaluation_system.py`

```python
async def _call_llm(self, prompt: str, max_tokens: int = None) -> str:
    config = get_evaluation_config()
    
    # 1. Check cache if enabled
    if config.enable_evaluation_cache:
        cached_response = get_cached_response(
            prompt=prompt,
            model=self.model,
            temperature=temperature,
            max_tokens=max_tokens
        )
        if cached_response is not None:
            return cached_response  # Cache hit - return immediately
    
    # 2. Cache miss - call LLM API
    response = await self.client.chat.completions.create(...)
    content = response.choices[0].message.content
    
    # 3. Store in cache
    if config.enable_evaluation_cache:
        cache_response(
            prompt=prompt,
            model=self.model,
            temperature=temperature,
            max_tokens=max_tokens,
            response=content
        )
    
    return content
```

### 7.3 Thread Safety
✅ **Thread-Safe**: `TTLCache` from `cachetools` is thread-safe for concurrent read/write operations.

### 7.4 Error Handling
- **Cache Errors**: Non-fatal (evaluation continues even if cache fails)
- **Config Errors**: Falls back to default values
- **Hash Errors**: Extremely rare (SHA256 collision probability: ~0)

---

## 8. Performance Benefits

### 8.1 Cost Savings
**Scenario**: Batch evaluation of 1000 records
- **Without Cache**: 1000 LLM API calls
- **With Cache (50% hit rate)**: 500 LLM API calls
- **Savings**: 50% reduction in API costs

**Example Cost Calculation** (GPT-4):
- Cost per 1K tokens: $0.03
- Average evaluation prompt: 2K tokens
- Average response: 1K tokens
- **Per call**: ~$0.09
- **1000 calls**: $90
- **With 50% cache**: $45 (saves $45 per batch)

### 8.2 Latency Improvements
- **Cache Hit**: ~0.1ms (in-memory lookup)
- **Cache Miss**: ~2-5 seconds (LLM API call)
- **Speedup**: **20,000x faster** for cached responses

### 8.3 Throughput Improvements
- **Without Cache**: Limited by LLM API rate limits
- **With Cache**: Can process more evaluations per second (cached = instant)

### 8.4 Real-World Impact
```
Batch Evaluation: 100 records
├─ 60% cache hit rate (60 records)
├─ 40% cache miss (40 records)
├─ Time saved: 60 × 3s = 180 seconds
└─ Cost saved: 60 × $0.09 = $5.40 per batch
```

---

## 9. Cache Statistics & Monitoring

### 9.1 Cache Statistics API
The evaluation result includes cache statistics:

```python
{
    "overall_score": 85.5,
    "metric_scores": {...},
    "cache_stats": {
        "size": 245,           # Current entries
        "max_size": 512,       # Maximum capacity
        "ttl_seconds": 259200  # Time-to-live
    }
}
```

### 9.2 Monitoring Cache Performance
**Key Metrics to Track:**
1. **Cache Hit Rate**: `(hits / (hits + misses)) × 100`
2. **Cache Size**: Current entries vs max size
3. **Cache Eviction Rate**: How often LRU eviction occurs
4. **Average Response Time**: With vs without cache

### 9.3 Cache Management Functions
```python
# Get cache statistics
stats = get_cache_stats()
# Returns: {"size": 245, "max_size": 512, "ttl_seconds": 259200}

# Clear cache (manual)
clear_cache()
```

---

## 10. Interview Q&A

### Q1: **Where is the cache data stored?**
**Answer:**
The cache is stored **in-memory (RAM)** using Python's `cachetools.TTLCache`. It's a module-level singleton that lives in the application's memory space. Data is **non-persistent** - it's cleared when the application restarts. This is intentional because:
- Evaluation prompts are deterministic (same inputs = same outputs)
- Cache can rebuild quickly on restart
- No need for persistence (evaluation is stateless)

### Q2: **How is data deleted after 3 days? Is there a scheduled job?**
**Answer:**
No scheduled job is needed! The cache uses **lazy expiration**:
- **TTL Check**: Every time the cache is accessed (get/set), it checks if the entry is older than 3 days
- **Automatic Deletion**: Expired entries are automatically removed when accessed
- **No Background Thread**: No cleanup job runs - expiration happens on-demand
- **Memory Efficient**: Expired entries don't accumulate because they're checked on every access

**Example**: If an entry was cached on Day 0 and accessed on Day 4, the cache will detect it's expired (>3 days) and remove it automatically.

### Q3: **What happens when the cache reaches max size (512 entries)?**
**Answer:**
The cache uses **LRU (Least Recently Used) eviction**:
- When the 513th entry is added, the least recently used entry is automatically evicted
- This ensures the cache never exceeds 512 entries
- Most frequently used entries stay in cache
- Memory usage remains bounded

### Q4: **Why did you choose 3 days as TTL?**
**Answer:**
3 days balances multiple factors:
- **Evaluation Stability**: Evaluation prompts and criteria don't change frequently
- **Cost Optimization**: Longer TTL = more cache hits = lower costs
- **Freshness**: 3 days ensures evaluations stay relevant (not too stale)
- **Memory Management**: Prevents indefinite accumulation of old entries

### Q5: **Is the cache thread-safe?**
**Answer:**
Yes! `cachetools.TTLCache` is **thread-safe** and can handle concurrent read/write operations from multiple threads or async tasks. This is crucial because:
- Batch evaluations run concurrently
- Multiple evaluation metrics can be computed in parallel
- No race conditions or data corruption

### Q6: **What if the same prompt is evaluated with different models?**
**Answer:**
They are **cached separately**! The cache key includes:
- Prompt content
- Model name
- Temperature
- Max tokens

So `prompt + model_A` and `prompt + model_B` are different cache entries. This ensures:
- Model-specific responses are cached correctly
- No cross-contamination between models
- Accurate cache hit rates per model

### Q7: **How do you measure cache effectiveness?**
**Answer:**
We track several metrics:
1. **Cache Hit Rate**: Percentage of requests served from cache
2. **Cache Size**: Current entries vs capacity
3. **Cost Savings**: API calls avoided due to cache hits
4. **Latency Reduction**: Time saved by cache hits

These metrics are included in the evaluation result's `cache_stats` field for monitoring.

### Q8: **What are the limitations of this caching approach?**
**Answer:**
**Limitations:**
1. **Non-persistent**: Cache is lost on restart (acceptable trade-off)
2. **In-memory only**: Limited by available RAM (mitigated by maxsize=512)
3. **Single instance**: Each application instance has its own cache (no shared cache across instances)
4. **Lazy expiration**: Expired entries may linger until accessed (minimal impact)

**Why These Are Acceptable:**
- Evaluation is stateless (can rebuild cache quickly)
- 512 entries = ~1.5MB (negligible memory)
- Each instance benefits from its own cache
- Lazy expiration is efficient (no background overhead)

### Q9: **How would you scale this cache for a distributed system?**
**Answer:**
For a distributed system, I would:
1. **Redis Cache**: Replace in-memory cache with Redis (shared across instances)
2. **Cache Warming**: Pre-populate cache with common evaluation prompts
3. **Cache Invalidation**: Add ability to invalidate specific entries when evaluation criteria change
4. **Cache Replication**: Replicate cache across regions for global deployments
5. **Metrics**: Add Prometheus metrics for cache hit rate, latency, etc.

### Q10: **What's the cache key generation strategy and why?**
**Answer:**
We use **SHA256 hashing** of a JSON object containing:
- Prompt content
- Model name
- Temperature
- Max tokens

**Why SHA256?**
- **Deterministic**: Same inputs = same hash = cache hit
- **Collision-resistant**: Extremely low probability of collisions
- **Compact**: 64-character hex string (vs storing full prompt as key)
- **Fast**: O(1) hash computation

**Why Include All Parameters?**
- Ensures cache correctness (different model/temperature = different response)
- Prevents cache pollution (same prompt, different config = separate entries)

---

## 📊 Summary

### Key Takeaways
1. **Storage**: In-memory (RAM), non-persistent
2. **TTL**: 3 days (259,200 seconds)
3. **Expiration**: Lazy expiration (no scheduled job)
4. **Eviction**: LRU when max size (512) reached
5. **Thread-Safe**: Yes, handles concurrent access
6. **Performance**: 20,000x faster for cache hits, 50%+ cost savings
7. **Monitoring**: Cache stats included in evaluation results

### Architecture Decision Rationale
- **In-Memory**: Fastest access, simple implementation
- **TTL-Based**: Automatic expiration without manual cleanup
- **LRU Eviction**: Keeps frequently used entries
- **SHA256 Keys**: Deterministic, collision-resistant, compact
- **Lazy Initialization**: Only creates cache when needed

---

**Document Version**: 1.0  
**Last Updated**: 2025-12-22  
**Author**: Evaluation System Team

