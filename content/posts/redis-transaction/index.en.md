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

# Purpose

For typical RDB transactions, we rely on isolation levels, rollback, and commit mechanisms.

But what about **Redis**?

Redis does not offer clear options to ensure data consistency and atomicity out-of-the-box. 

This document records the approach to **isolate a transaction** and achieve **All-or-Nothing atomicity** in Redis.

---

# Options

1. Redis TxPipeline
2. Lua Script

---

## Redis TxPipeline

When handling multiple commands in Redis, Pipeline is a natural choice.

Pipeline allows sending multiple commands to the Redis server in a batch and receiving multiple responses at once.

However, **standard Pipeline does NOT guarantee transactional integrity**.  
During execution, data might still be modified by other commands, resulting in inconsistencies.

### Edge Cases

1. **Network Latency**: Command and response order may mismatch due to network delay.
2. **Multi-threaded Environment**: Redis allows concurrent client requests, making command execution order non-deterministic.
3. **Redis Server Configuration**: Replication settings (e.g., `slaveof`) can impact consistency.

**TxPipeline** solves these issues.

### Pros

- **Transactional Guarantee**: Commands are treated as one atomic transaction.
- **Performance Boost**: Reduces network overhead by batching commands.
- **Atomicity**: All commands succeed or fail together (but no rollback).

### Cons

- **Memory Usage**: Large transactions consume significant memory.
- **Complexity**: Careful management is needed, especially when mixing with regular pipelines.

### Test

![](2.png)

> Using `WATCH` ensures monitored keys are checked for changes. If any change occurs, the transaction fails.

Example Transaction:

```bash
MULTI
SET key1 value1
SET key2 value2
SET key3 value3
EXEC
```

If an error occurs at `SET key2`:

- **Memory Exhaustion Case**:

```bash
OOM command not allowed when used memory > 'maxmemory'
```

- **Wrong Type Error**:

```bash
WRONGTYPE Operation against a key holding the wrong kind of value
```

The most common realistic failure is client disconnection.

Even if you attempt rollback manually after disconnection, it would likely fail —
thus **TxPipeline guarantees consistency but not perfect atomicity**.

---

## Lua Script

Execute multiple Redis commands atomically inside a Lua script.

### Edge Cases

1. **Network Failure During Script Upload**: Script might never reach the server.
2. **Network Failure During Response Reception**: Script executes but the client may not receive the result.

### Pros

- **Lightweight and Fast**: Lua scripts execute efficiently.
- **Built-in Scripting**: Extends Redis functionality (like raw SQL vs ORM).
- **Readable Syntax**: Clean, easy-to-understand language.

### Cons

- **Smaller Ecosystem**: Fewer libraries and community support.
- **Limited Data Types**: Integer/floating point distinctions are loose.
- **Strict Syntax**: Steeper learning curve for beginners.
- **Threading Limitations**: Lua is fundamentally single-threaded.

### Test

Lua scripts also **do NOT support rollback**.

Worst-case scenario remains: network disconnection during execution.

**Example**: Insert key1 ~ key5, simulate error at key3.

```go
package main

import (
	"context"
	"fmt"
	"github.com/go-redis/redis/v8"
)

var ctx = context.Background()

func main() {
	rdb := redis.NewClient(&redis.Options{
		Addr: "localhost:6379",
	})

	luaScript := `
    for i = 1, #KEYS do
        if KEYS[i] == 'key3' then
            error('Error on setting key3')
        else
            redis.call('SET', KEYS[i], ARGV[i])
        end
    end
    `

	keys := []string{"key1", "key2", "key3", "key4", "key5"}
	values := []interface{}{"value1", "value2", "value3", "value4", "value5"}

	err := rdb.Eval(ctx, luaScript, keys, values...).Err()
	if err != nil {
		fmt.Printf("Lua script execution error: %v\n", err)
		return
	}

	fmt.Println("Lua script executed successfully")
}
```

[Result]

![](1.png)

---

# Summary

## Pros & Cons

|                | TxPipeline | Lua Script |
| -------------- | ------------------------------------------------------------ | ---------------------------------------------------------------- |
| **Key Features** | - Standard Redis syntax<br>- Ensures data integrity<br>- Optimistic lock (`WATCH`) prevents conflict | - Requires Lua scripting<br>- Ensures data integrity<br>- Executes script as a single unit |
| **Transaction Rollback** | Not supported | Not supported |
| **Drawbacks** | - Slightly lower performance than Lua<br>- Complexity increases with large transactions | - Management overhead for Lua scripts<br>- Learning curve |

---

- Both options guarantee **data integrity**, but neither supports rollback after failure.
- **Validation before insertion** reduces chances of runtime errors.
- **Worst-case scenario**:
  - Redis client connection drops during transaction.
  - Cannot roll back manually if disconnected.
- Consider Redis Sentinel if high availability edge cases need to be addressed.

## Benchmark

Testing 1,000 key insertions into Redis:

| Test Item | Test Count | Avg Execution Time (ms) |
| --------- | ---------- | ----------------------- |
| Lua Script | 1424 | 0.83 |
| TxPipeline | 460 | 2.56 |
| Pipeline | 506 | 2.34 |

- Performance differences are not critical.
- **TxPipeline or Lua Script is recommended** to ensure data consistency.