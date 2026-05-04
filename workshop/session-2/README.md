# Session 2: Adding Redis Cache

## Objective

Enhance the CLI client from Session 1 by adding Redis as a cache layer to store and retrieve previous question-answer pairs, reducing API calls and improving response times.

## Prerequisites

Before starting this session, ensure you have:

1. **Completed Session 1**: Understanding of the basic CLI client

1. **Redis Server Running**: Redis must be running on `localhost:6379`
   ```sh
   # From the project root directory
   docker-compose up -d redis
   ```

1. **AISHE Server Running**: The AISHE server must be running on `http://localhost:8000`
   ```sh
   docker-compose up -d aishe
   ```

1. Optional: **Redis CLI** or [Redis Insight](https://github.com/redis/RedisInsight/releases): For inspecting cache
   ```sh
   docker exec -it aishe-redis redis-cli
   ```

## Architecture Diagram

```
┌─────────────┐
│    User     │
└──────┬──────┘
       │ Question
       ▼
┌──────────────────────────────────────┐
│         CLI Application              │
│  ┌───────────────────────────────┐   │
│  │  1. Normalize & Hash Question │   │
│  └───────────────────────────────┘   │
│  ┌───────────────────────────────┐   │
│  │  2. Check Redis Cache         │   │
│  └───────────────────────────────┘   │
│         │                │           │
│    Cache Hit        Cache Miss       │
│         │                │           │
│         │                ▼           │
│         │  ┌───────────────────────┐ │
│         │  │ 3. Call AISHE API     │ │
│         │  └───────────────────────┘ │
│         │                │           │
│         │                ▼           │
│         │  ┌───────────────────────┐ │
│         │  │ 4. Save to Cache      │ │
│         │  └───────────────────────┘ │
│         │                │           │
│         └────────┬───────┘           │
│                  ▼                   │
│  ┌───────────────────────────────┐   │
│  │  5. Display Response          │   │
│  └───────────────────────────────┘   │
└──────────────────────────────────────┘
```

## Implementation Overview

### Cache Key Generation

Generate unique, consistent cache keys for questions:

- **Normalize the question**: Convert to lowercase and trim whitespace to ensure "What is Python?" and "what is python?" produce the same cache key
- **Hash the normalized question**: Use a hashing algorithm (e.g., SHA-256) to create a fixed-length key
- **Add a namespace prefix**: Use a prefix like `aishe:question:` to organize keys and avoid collisions

**Example cache key format**:
```
aishe:question:a3c5f8d9e2b1c4a7f6e8d9c2b5a4e7f1d3c6b9a2e5f8d1c4b7a6e9f2d5c8b1a4
```

### Cache Lookup Flow

Before making an API call, check the cache:

```
1. Receive user question
2. Generate cache key from normalized question
3. Query Redis for the cache key
4. If found (cache hit):
   - Return cached response immediately
   - Display cache hit indicator
5. If not found (cache miss):
   - Make API call to AISHE server
   - Store response in Redis with expiration
   - Return API response
```

### Redis Data Structure

Store responses as JSON strings:

**Key**: `aishe:question:{hash}`

**Value** (JSON string):
```json
{
  "answer": "The generated answer",
  "sources": [
    {
      "number": 1,
      "title": "Wikipedia Article Title",
      "url": "https://en.wikipedia.org/wiki/..."
    }
  ],
  "processing_time": 2.45
}
```

**TTL (Time To Live)**: 24 hours (86400 seconds)

### Cache Operations

#### Reading from Cache
```
1. Generate cache key
2. Execute GET command on Redis
3. If value exists:
   - Deserialize JSON string to object
   - Return cached data
4. If value is null:
   - Return null/none to indicate cache miss
```

#### Writing to Cache
```
1. Generate cache key
2. Serialize response data to JSON string
3. Execute SETEX command with:
   - Key: cache key
   - Value: JSON string
   - Expiration: 86400 seconds (24 hours)
```

## Testing Your Implementation

### Test Cache Miss (First Query)

```bash
# First time asking this question
./your-cli "What is Redis?"
```

Expected output:
```
Asking: What is Redis?
✗ Not in cache, calling AISHE API...
Waiting for response...
✓ Response saved to cache

======================================================================
ANSWER:
======================================================================
Redis is an in-memory data structure store...

======================================================================
Processing time: 2.34 seconds
======================================================================
```

### Test Cache Hit (Repeat Query)

```bash
# Ask the same question again
./your-cli "What is Redis?"
```

Expected output:
```
Asking: What is Redis?
✓ Found in cache! (no API call needed)

======================================================================
ANSWER:
======================================================================
Redis is an in-memory data structure store...

======================================================================
Source: Redis Cache
======================================================================
```

Notice: No processing time, instant response!


### Verify Cache in Redis

Use Redis CLI to inspect cached data:

```bash
# Connect to Redis
docker exec -it aishe-redis redis-cli

# List all AISHE cache keys
KEYS aishe:question:*

# View a specific cached value
GET aishe:question:{hash}

# Check TTL (time to live)
TTL aishe:question:{hash}

# Exit Redis CLI
exit
```

### Test Cache Expiration

```bash
# Set a short TTL for testing (e.g., 10 seconds)
# Ask a question, wait 11 seconds, ask again
# Should see cache miss on second attempt
```

## Performance Comparison

### Without Cache (Session 1)
- Every question requires an API call
- Processing time: ~2-5 seconds per question
- Network and computation overhead for each request

### With Cache (Session 2)
- Repeated questions served from cache
- Cache hit response time: ~10-50 milliseconds
- **50-500x faster** for cached responses
- Reduced load on AISHE server

## Common Issues

### Redis Connection Refused
```
Error: Could not connect to Redis at localhost:6379
```
**Solution**: Make sure Redis is running:
```bash
docker-compose up -d redis
docker-compose ps  # Check if redis container is running
```

### Cache Not Working (Always Cache Miss)
**Possible causes**:
1. Cache key generation is inconsistent
2. Question normalization not working
3. Redis connection issues

**Debug steps**:
```bash
# Check Redis is accessible
docker exec -it aishe-redis redis-cli ping
# Should return: PONG

# Monitor Redis commands in real-time
docker exec -it aishe-redis redis-cli monitor
# Run your CLI and watch the commands
```

### Stale Cache Data
If the AISHE server's answers improve but cache returns old answers:

**Solution**: Clear the cache:
```bash
# Clear all AISHE cache keys
docker exec -it aishe-redis redis-cli --scan --pattern "aishe:question:*" | xargs docker exec -i aishe-redis redis-cli DEL

# Or flush entire Redis database (use with caution)
docker exec -it aishe-redis redis-cli FLUSHDB
```

---

## STRETCH TASK: implement semantic caching

AISHE + regular Redis cache = blazingly fast *exact* prompt text lookups

AISHE + semantic Redis cache = still-extremely-fast *meaning-based* lookups

To implement semantic caching you'll need to:

1) Vectorize a prompt
   > Text is split into tokens. An embedding model transforms
   > these tokens into a list of numbers (an n-dimensional vector).

2) Perform similarity search
   > Compute how similar this prompt's meaning is to previous
   > prompts in a vector database.

3) If a similar one exists, reuse cached answer

4) Otherwise cache this vector embedding and its answer

```
         ┌─────────────┐
         │    User     │
         │  question   │
         └──────┬──────┘
                │
                ▼
      ┌───────────────────┐
      │   Embedding       │
      │     model         │
      │ (tokens → vector) │
      └─────────┬─────────┘
                │
                │  n-dim vector
                ▼
    ┌───────────────────────┐
    │   Redis vector DB     │
    │  similarity search    │
    │  over past prompts    │
    └───────────┬───────────┘
                │
     ┌──────────┴──────────┐
     │                     │
  similar hit          no match
  (score ≥ threshold)  (cache miss)
     │                     │
     ▼                     ▼
┌─────────────┐     ┌─────────────┐
│   Cached    │     │    AISHE    │
│   answer    │     │   server    │
│  (reused)   │     │ (full RAG)  │
└──────┬──────┘     └──────┬──────┘
       │                   │
       │                   │  answer
       │                   ▼
       │            ┌─────────────┐
       │            │ Store vector│
       │            │  + answer   │
       │            │  in Redis   │
       │            └──────┬──────┘
       │                   │
       └─────────┬─────────┘
                 │
                 ▼
          ┌─────────────┐
          │    User     │
          └─────────────┘
```

### Before proceeding

You need a working session 2 client before proceeding. Complete the regular task
first, then revisit this one.

This task involves modifying your existing AISHE client. No server changes are necessary.

For a more comprehensive overview of semantic caching, check out
[this article](https://wso2.com/library/blogs/what-is-semantic-caching/).

Again, you are free to use AI agents for this task.

### How to use AI to implement this task

Sometimes, you're barely familiar with the subject. You may not know how to
store embeddings in Redis or compute semantic similarity.

But you know what a working solution looks like. You have a well-defined task
and don't know the implementation details.

In these cases, you can *reverse the development process* (compared to stretch task 1):
- We let AI implement our task with as many requirements as we know.
- Later we study and refine the implementation piece-by-piece.

1. Create a **detailed prompt** and have your AI agent implement the task outright:

> I want you to implement semantic caching with Redis inside my Python client in
> workshop/session-2/python/main.py.
>
> To do this you'll need to:
> 1) Vectorize a prompt: text is split into tokens. An embedding model transforms
>                        these tokens into a list of numbers (an n-dimensional vector)
> 2) Perform similarity search: compute how similar this prompt's meaning is to
>                               previous prompts in a vector database
> 3) If a similar one exists, reuse cached answer
> 4) Otherwise cache this vector embedding and its answer
>
> Use the LangCache model https://huggingface.co/redis/langcache-embed-v2. Show me
> the simplest instructions to install it and set it up manually.
>
> Use my existing plain Redis OSS to store vector embeddings in whatever way you deem best.
>
> Make all code changes only in my main.py file. Optimize the code to be easy to read
> for a software engineering student and adopt my personal style of coding.

2. **Assert** it's working:

```bash
./your-cli "What is Redis?"
```

and

```bash
./your-cli "Explain Redis"
```

Should hit the cache and return the same response.

3. Inspect, **study**, and question the implementation:

> Hey, I'm curious: how does this code determine if two prompts are similar?

4. **Refine** implementation chunk by chunk:

> What similarity threshold would I need to set to match looser prompts like "Where is Redis used?"
