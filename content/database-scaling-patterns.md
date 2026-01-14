+++
title = "Database Scaling Patterns: From Single Instance to Global Distribution"
date = 2026-01-11
description = "A practical guide to scaling databases as your application grows from hundreds to millions of users."
[taxonomies]
tags = ["system-design", "databases", "scaling"]
categories = ["System Design"]
+++

Your application is successful. Traffic is growing. And your database is starting to sweat. Before reaching for a complete rewrite, understand the scaling patterns available to you—and when each one makes sense.

<!-- more -->

## Stage 1: Vertical Scaling

The simplest solution is often the best first step: get a bigger machine.

Modern cloud instances offer up to 24TB of RAM and 448 vCPUs. That's enough for many applications that think they need horizontal scaling. Vertical scaling has massive advantages:

- No application changes required
- No distributed systems complexity
- Transactions work normally
- JOINs remain fast

**When to move on**: When you've maxed out available instance sizes, when you need geographic distribution, or when the cost becomes prohibitive.

## Stage 2: Read Replicas

Most applications are read-heavy. Read replicas let you scale reads horizontally while keeping writes on a single primary.

```
           ┌─────────────┐
           │   Primary   │
           │  (writes)   │
           └──────┬──────┘
                  │ replication
        ┌─────────┼─────────┐
        ▼         ▼         ▼
   ┌─────────┐ ┌─────────┐ ┌─────────┐
   │ Replica │ │ Replica │ │ Replica │
   │ (reads) │ │ (reads) │ │ (reads) │
   └─────────┘ └─────────┘ └─────────┘
```

**Implementation considerations**:

- **Replication lag**: Replicas are eventually consistent. A user might write data and not see it on the next read. Route reads-after-writes to the primary, or accept the inconsistency.

- **Connection management**: Your application needs to route queries appropriately. Many ORMs and connection pools support this natively.

```python
# Example: Read from replica, write to primary
class DatabaseRouter:
    def db_for_read(self, model):
        return 'replica'

    def db_for_write(self, model):
        return 'primary'
```

**When to move on**: When your write volume exceeds what a single primary can handle, or when you need lower latency for geographically distributed users.

## Stage 3: Caching

Before sharding your database, consider whether you need to hit it at all.

**Read-through caching**: Check cache first; on miss, read from database and populate cache.

```python
def get_user(user_id):
    cached = cache.get(f"user:{user_id}")
    if cached:
        return cached

    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    cache.set(f"user:{user_id}", user, ttl=3600)
    return user
```

**Write-through caching**: Update cache when database is updated.

**Cache invalidation**: The two hardest problems in computer science are cache invalidation, naming things, and off-by-one errors. Use TTLs as a safety net, and invalidate explicitly when data changes.

**Cache stampede**: When a popular cache entry expires, many requests hit the database simultaneously. Use locking or probabilistic early expiration:

```python
def get_with_stampede_protection(key, ttl=3600):
    value, expiry = cache.get_with_expiry(key)

    # Probabilistically refresh before expiry
    remaining = expiry - time.time()
    if remaining < ttl * 0.1 * random.random():
        refresh_in_background(key)

    return value
```

## Stage 4: Sharding

When a single database can't handle your write volume, you split the data across multiple databases.

### Shard Key Selection

The shard key determines which shard holds each row. This decision is critical and hard to change later.

**Good shard keys**:
- Have high cardinality (many distinct values)
- Distribute writes evenly
- Keep related data together

**Common strategies**:

- **User ID**: Each user's data lives on one shard. Great for user-centric applications.
- **Geographic region**: Data stays close to users. Good for latency-sensitive applications.
- **Tenant ID**: Natural for multi-tenant SaaS applications.

**Anti-patterns**:

- **Timestamp**: Creates hot spots on the "current" shard.
- **Sequential IDs**: Same problem—recent data concentrates on one shard.

### Cross-Shard Queries

Once data is sharded, queries that span shards become expensive:

```sql
-- Easy: user on a single shard
SELECT * FROM orders WHERE user_id = 123;

-- Hard: aggregation across all shards
SELECT COUNT(*) FROM orders WHERE created_at > '2024-01-01';
```

Design your data model to minimize cross-shard queries. Denormalize when necessary. Consider separate analytics infrastructure for aggregations.

### Rebalancing

As data grows unevenly, you'll need to move data between shards. Consistent hashing minimizes data movement when adding shards:

```
Traditional: Add shard, rehash everything
Consistent:  Add shard, move only affected range
```

## Stage 5: Multi-Region Deployment

For global applications, a single-region database adds latency for distant users. Multi-region deployments bring data closer to users.

### Active-Passive

One region handles writes; others are read replicas. Simple, but writes have high latency for users far from the primary region.

### Active-Active

All regions accept writes. Now you need conflict resolution:

- **Last-write-wins**: Simple but loses data.
- **Vector clocks**: Track causality, surface conflicts to applications.
- **CRDTs**: Data structures designed for concurrent updates without conflicts.

Most applications are better served by active-passive with regional failover than true active-active.

## The Operational Dimension

Scaling isn't just technical—it's operational:

**Monitoring**: Shard hotspots, replication lag, cache hit rates.

**Backup and recovery**: Can you restore a single shard? How long does it take?

**Schema migrations**: Rolling out changes across shards requires coordination.

**Runbooks**: When shard 3 is overloaded at 3 AM, what do you do?

## When to Use Managed Services

Cloud-managed databases (Aurora, Cloud Spanner, CockroachDB) handle much of this complexity:

- Automatic sharding and rebalancing
- Built-in replication
- Managed failover

The tradeoff is cost and vendor lock-in. But for many teams, the operational simplicity is worth it.

## Conclusion

Database scaling is a journey, not a destination. Start simple:

1. Optimize queries and add indexes
2. Vertical scaling
3. Read replicas
4. Caching
5. Sharding (only if truly necessary)
6. Multi-region (only if truly necessary)

Each step adds complexity. Move to the next stage only when you've exhausted the current one. The best architecture is the simplest one that meets your requirements.
