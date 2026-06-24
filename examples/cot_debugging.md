# CoT Example: Debugging a Race Condition

## The Problem

A Python web server intermittently returns stale data from cache. The bug is not reproducible on every request.

## Prompt (Few-shot CoT)

```
System: You are a senior backend engineer. Debug by reasoning through 
the execution flow step by step. Show your analysis before concluding.

Example:
Code: cache.get(key) or db.query(key)
Step 1: cache.get returns None if key missing
Step 2: db.query runs only when cache misses
Step 3: But what if cache.get returns stale data? It's truthy, db.query never runs
Step 4: Need cache TTL or invalidation
Answer: Stale cache - add TTL or explicit invalidation on write

Now debug this:
@app.get("/user/{id}")
async def get_user(id: int):
    cached = redis.get(f"user:{id}")
    if cached:
        return json.loads(cached)
    user = await db.fetch_user(id)
    redis.set(f"user:{id}", json.dumps(user))
    return user
```

## Expected Output

```
Step 1: Request comes in, check Redis cache
Step 2: If cache hit, return immediately - no DB call
Step 3: If cache miss, fetch from DB and cache the result
Step 4: Problem: redis.set has no TTL - data cached forever
Step 5: When user updates profile, cache still holds old data
Step 6: Race condition: two concurrent requests for same user, 
        both miss cache, both query DB, both write to cache -
        the second write may overwrite with slightly older data
Answer: Two issues - (1) no TTL on cache, add ex=300 for 5min expiry, 
(2) use redis.setnx or a lock to prevent concurrent cache population
```
