# Homekey Backend Assessment

## Context

When a property goes on the market, buyers submit offers on our website. A popular listing can receive multiple offers in a few hours.

The codebase uses:
- **Flask** for the API layer
- **MongoDB** as the primary database
- **Redis** for caching
- **A multi-agent AI system** for property valuation

---

## Task 1

**Format:** code + explanation

A listing received a burst of concurrent offer submissions. Under load, you observe:
- Duplicate offers appearing in the database for the same buyer
- MongoDB CPU spiking to 87%
- Stale offer lists being served to the seller's dashboard

There are multiple bugs in file `app/services/offer_service.py`. Fix only the most critical one. 

For the other ones, explain
1. What the bug is
2. What it causes under load
3. How you would fix it (code or pseudocode)
4. Why you chose not to fix it first

The constraint is intentional. We want to see how you triage.

---

### Other Issues

#### 1. Redis Cache Staleness

**Bug**

`get_offers()` caches offer lists in Redis but the cache is never invalidated when a new offer is submitted.

**Impact under load**

Sellers may see outdated offer lists on the dashboard.

**Fix (pseudocode)**

```python
self.redis.delete(f"{self.OFFERS_CACHE_PREFIX}{property_id}")
```
This should be executed after a successful offer insertion.

**Why not fixed first**

This issue affects UI freshness but does not compromise transaction correctness.

#### 2. N+1 Query Problem

**Bug**

`get_offers()` performs a user lookup for every offer:

```python
offer["buyer"] = self.db.users.find_one({"_id": offer["buyer_id"]})
```
**Impact under load**

If a property has many offers, this generates a large number of database queries and increases MongoDB CPU usage.

**Fix (pseudocode)**
```python
buyer_ids = [offer["buyer_id"] for offer in offers]
buyers = db.users.find({"_id": {"$in": buyer_ids}})
```
Batch loading buyers reduces database calls from O(N) to O(1).

**Why not fixed first**

This is primarily a performance issue, while the duplicate offer bug affects data integrity and transaction correctness, making it higher priority.

## Task 2

**Format:** explanation

A colleague proposes storing offers embedded in the property document:

```json
{
  "_id": "property_abc",
  "address": "123 Main St",
  "asking_price": 1200000,
  "offers": [
    { "buyer_id": "u1", "amount": 1150000, "status": "pending", "submitted_at": "2025-11-01T10:00:00Z", ... },
    { "buyer_id": "u2", "amount": 1180000, "status": "accepted", "submitted_at": "2025-11-01T11:30:00Z", ... }
  ]
}
```

This works fine in staging.

1. What can break in production, and why?
2. Propose an alternative schema.
3. There is no schema that solves everything. What does your alternative make harder compared to the embedded approach?

### 1. What can break in production, and why?

Embedding offers inside the property document can cause the document to grow very large as popular listings receive many offers. MongoDB has a **16MB document size limit**, which could eventually be exceeded. Each new offer also requires rewriting the entire property document, creating **write contention on a hot document** when many buyers submit offers concurrently. This increases replication pressure and reduces write throughput. Querying or updating individual offers also becomes inefficient because the full document must be modified.

---

### 2. Proposed alternative schema

Store offers in a **separate collection** and reference the property.

Example:

```json
properties
{
  "_id": "property_abc",
  "address": "123 Main St",
  "asking_price": 1200000
}

offers
{
  "_id": ObjectId(),
  "property_id": "property_abc",
  "buyer_id": "u1",
  "amount": 1150000,
  "status": "pending",
  "submitted_at": ISODate("2025-11-01T10:00:00Z")
}
```

Recommended indexes:

(property_id) |  (property_id, buyer_id) unique | (property_id, submitted_at)

This keeps property documents small and allows offers to scale independently.

### 3. What does the alternative make harder compared to the embedded approach?
Using separate collections requires additional queries or joins at the application layer when retrieving a property and its offers. Reads may require aggregation pipelines or multiple queries, which adds complexity. Maintaining consistency across collections may also require transactions in certain workflows. Compared to embedding, this can introduce slightly higher read latency. However, it significantly improves scalability for write-heavy workloads.

---

## Task 3

**Format:** code + explanation

Review `get_comparable_sales` method in `app/services/valuation_service.py`. 
Leave comments on any issue you'd flag and order them by priority as we scale up.

---

## Task 3

### Code Review: `get_comparable_sales`

Below are issues I would flag during code review, ordered by priority as the system scales.

---

### 1. Missing Database Index (High Priority)

**Issue**

The query filters by `property_id` and sorts by `sale_price`, but there is no guarantee that a compound index exists.

```python
self.db.comparable_sales
    .find({"property_id": property_id})
    .sort("sale_price", -1)
    .limit(3)
```
Without an index, MongoDB may perform a collection scan and in-memory sort.

**Impact**

Under heavy traffic this increases database CPU usage and latency.

**Fix**

Create a compound index:
```python
db.comparable_sales.create_index(
    [("property_id", 1), ("sale_price", -1)]
)
```

### 2. Duplicate Database Query (Medium Priority)

**Issue**

The function performs two database queries:

```python
top_comps = list(...)
total = self.db.comparable_sales.count_documents(...)
```

Both queries scan the same dataset.

**Impact**

This doubles database load and increases latency.

**Fix**

Use an aggregation pipeline:

```python
pipeline = [
    {"$match": {"property_id": property_id}},
    {"$facet": {
        "top": [{"$sort": {"sale_price": -1}}, {"$limit": 3}],
        "count": [{"$count": "total"}]
    }}
]
```

This allows MongoDB to compute results in a single query.

### 3. Cache Stampede Risk (Medium Priority)

**Issue**

If the cache expires, many concurrent requests may simultaneously hit the database.

```python
cached = self.redis.get(cache_key)
```

**Impact**

This creates bursts of identical database queries.

**Fix**

Use a short lock or request coalescing pattern:
```python
SETNX lock_key
```

Only one request populates the cache while others wait.


### 4. Missing Field Projection (Low Priority)

**Issue**

The query retrieves the full document:

```python
.find({"property_id": property_id})
```

**Impact**

Unnecessary fields increase network payload and serialization cost.

**Fix**

Limit returned fields:

```python
.find(
  {"property_id": property_id},
  {"address": 1, "sale_price": 1, "sold_at": 1, "distance_km": 1}
)
```

### 5. Cache Key Namespace Consistency (Low Priority)

**Issue**

The cache key "comps:{property_id}" is inconsistent with the service prefix pattern.

```python
cache_key = f"comps:{property_id}"
```

**Impact**

Inconsistent naming makes cache invalidation and debugging harder.

**Fix**

Use a consistent prefix:

```python
cache_key = f"{self.VALUATION_CACHE_PREFIX}comps:{property_id}"
```

## Task 4

**Format:** explanation

An AI system routes user requests through an Orchestrator that manages a stateful workflow. Each downstream agent runs a multi-step LLM chain. Agents call external tools through a service layer. Some workflows pause mid-execution waiting for user input.

In production you observe p99 response times exceeding 50 seconds, occasional cascading failures when a downstream agent is slow, and paused workflows that are never cleaned up.

You have 1 week before a product demo.

1. What reliability problems would you expect in a system like this?
2. How would you prioritize fixing them, and what does your prioritization give up?
3. Given all of the knowledge you know, provide a 1-week timeline of your implementation details and how it will impact the users.
4. What makes the tool service layer harder to make resilient than a typical stateless API?
