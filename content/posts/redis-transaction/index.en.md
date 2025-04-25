---
title: "[DB] Redis Transaction"
date: "2024-01-31"
draft: false
author: dingyu
categories: ["db"]
tags: ["redis", "transaction", "txpipeline"]
summary: How does Redis guarantee transactionality without traditional locks?
cover:
  image: img/redis.png
  relative: true
comments: true
description: Unlike RDBMS, Redis doesn’t have a traditional transaction mechanism. It primarily operates as a single-threaded cache, but transactions can still be made atomic using TX pipelines or Lua scripts. This post focuses on comparing the pros and cons of Lua scripts vs. TX pipelines.
---

## Purpose

Unlike traditional RDBMS, which implement transactions through isolation levels and mechanisms like rollback and commit, Redis has no native transactional locking system. This post discusses how to isolate a transaction and preserve **all-or-nothing atomicity** in Redis using available tools.

## Options

1. Redis TxPipeline
2. Lua Script

## Redis TxPipeline

A natural first approach is to use **Pipelines**, which send multiple commands in batch. However, Pipelines alone don’t ensure atomicity—commands can still be interleaved with others from different clients.

### Edge Cases
1. **Network Delay**: Commands and responses may arrive out of sync.
2. **Multithreaded Clients**: Multiple clients using pipelines simultaneously can cause inconsistent ordering.
3. **Replica Configuration**: Data consistency may not be guaranteed when replication settings like `slaveof` are active.

This is where **TxPipeline** shines.

### Pros
- **Atomic batch execution** via `MULTI`/`EXEC`.
- **Improved performance** by reducing round trips.
- **Transactional consistency** when used with `WATCH`.

### Cons
- **Memory overhead**: All commands and responses are buffered.
- **Complexity**: Mixed usage with normal pipelines may cause unintended behavior.

### Testing Example
![](img/2.png)

```bash
MULTI
SET key1 value1
SET key2 value2
SET key3 value3
EXEC
```

If `SET key2 value2` fails:
- `OOM`: Out of memory
- `WRONGTYPE`: Type mismatch on key

Even with `WATCH`, if the connection drops mid-transaction, rollback is not guaranteed. TxPipeline **ensures consistency**, but not full rollback support.

---

## Lua Script

Use Lua to execute multiple Redis commands **atomically**.

### Edge Cases
1. Script transmission failure due to network issues.
2. Post-execution result fetch failures on the client side.

### Pros
- **Lightweight and fast**
- **Atomic**: Entire script runs as one unit
- **Embedded scripting** for customization

### Cons
- Smaller ecosystem
- Fewer data types
- Tougher concurrency and threading

### Testing
Scripts cannot be rolled back either. Worst case? Network cut-off during the process.

```go
luaScript := `
  for i = 1, #KEYS do
    if KEYS[i] == 'key3' then
      error('error on key3')
    else
      redis.call('SET', KEYS[i], ARGV[i])
    end
  end
`
```

Keys: key1 to key5 → `key3` triggers error.

![](img/1.png)

---

## Summary

|               | TxPipeline                                                                 | Lua Script                                                            |
|---------------|----------------------------------------------------------------------------|------------------------------------------------------------------------|
| **Overview**  | Redis-like syntax, optimistic locking via `WATCH`, consistent             | Full atomic block via Lua, less intuitive for some users             |
| **Rollback**  | ❌ No rollback                                                             | ❌ No rollback                                                        |
| **Drawbacks** | Slightly slower than Lua, potential complexity                            | Extra script management, steeper learning curve                      |

- Both methods ensure **data consistency**, not rollback.
- Client-side validation is crucial.
- Network errors remain the weakest link.

## Benchmark
Setting 1,000 keys:

| Method        | Runs | Avg Time (ms) |
|---------------|------|----------------|
| Lua Script    | 1424 | 0.83           |
| TxPipeline    | 460  | 2.56           |
| Basic Pipeline| 506  | 2.34           |

While not dramatic, Lua is slightly faster—**but choose based on structure and consistency needs**.

