---
title: "[Go] Profiling Worker Pool vs. Async Processing"
date: "2024-10-27"
draft: false
author: dingyu
categories: ["optimization"]
tags: ["go", "async", "sync", "pprof"]
summary: Is it better to distribute tasks across multiple workers or handle each task asynchronously? Let’s find out through performance profiling.
cover:
  image: img/go-thumbnail.png
  relative: true
comments: true
description: This post explores profiling and optimizing worker pools vs. asynchronous execution in Go using pprof. It analyzes the performance impact of concurrent HTTP requests, comparing sync worker pools (10 vs. 100 workers) and a single async worker in terms of throughput, CPU overhead, and memory allocation. Profiling results reveal that worker pools suffer from high concurrency overhead, while asynchronous execution significantly improves throughput with minimal memory cost. Additionally, the post discusses when to use worker pools vs. async processing, highlighting key trade-offs for IO-bound vs. CPU-bound tasks.
---

![](img/image.png)
> For how to apply `pprof`, refer to [Tuning GC with pprof](https://velog.io/@wjddn3711/pprof%EB%A1%9C-GC-%ED%8A%9C%EB%8B%9D%ED%95%98%EA%B8%B0)

## Background

Our service server was performing one HTTP request per event—triggered frequently and handled asynchronously. However, the more requests we sent, the more linear the delay became.

![](img/image-1.png)

### Problems
1. **Excessive network requests** on every event
2. **Increased latency** due to queuing
3. **Lack of scalability** — implementation differences between services caused repeated maintenance issues

## Architecture Improvement

![](img/image-2.png)

### Solutions
1. Queue events locally and send batch requests when either buffer size or interval is exceeded.
2. Reduce latency via batching.
3. Build a common library with loose coupling between service servers and the batch loader.

## Design Decisions

We decided to build a shared module called **Batch Processor** to manage event queues.

### Requirements
- Optimize for **IO-bound tasks**
- Manage goroutine lifecycle cleanly

### Option 1: Worker Pool (Sync IO)

![](img/image-3.png)

#### Pros
- Avoids deep copy overhead
- Tunable performance via pool size

#### Cons
- Potentially multiple HTTP requests per interval
- Hard to tune optimal pool size for every service

### Option 2: Single Worker + Async HTTP

![](img/image-4.png)

#### Pros
- Only one HTTP request per interval
- Simpler integration without tuning

#### Cons
- Minor CPU/memory overhead from deep copy
- GC pressure may increase due to heap allocations

Given the trade-offs, we opted for **Option 2**, but needed to validate that deep copy overhead wouldn’t impact performance.

## Profiling Goals

- Measure **throughput** at 2000 RPS
- Quantify **memory impact** of `deepCopy()`
- Analyze **GC overhead** from buffer copies

```go
func deepCopy[T any](src []T) []T {
    if src == nil {
        return nil
    }
    dst := make([]T, len(src))
    copy(dst, src)
    return dst
}
```

## Methodology

Compare 3 implementations:
- **10 Workers + Sync IO**
- **100 Workers + Sync IO**
- **1 Worker + Async IO**

Track:
- Throughput via logs
- Heap profile with:
```bash
curl {endpoint}/debug/pprof/heap?seconds=30 --output {output_path}
```

Log parsing example:
```text
2024-10-14T05:11:06Z INF count=1020
2024-10-14T05:11:07Z INF count=1000
2024-10-14T05:11:07Z INF stopping BatchProcessor...
```

## CPU Profiling

### 100 Workers + Sync IO
![](img/image-5.png)
- 85% of time spent on `sellock` and `acquireSudog`
- High contention on channel access

### 10 Workers + Sync IO
![](img/image-6.png)
- Lock contention reduced to 66%

### 1 Worker + Async IO
![](img/image-7.png)
- Deep copy overhead ~10ms
- 50% time split between API calls and deep copy

## Heap Profiling

### Worker Pool
![](img/image-8.png)
- ~8.2MB total, ~7.7MB from HTTP
- No deep copy impact

### 1 Worker + Async IO
![](img/image-9.png)
- ~12.2MB total, ~11.4MB from HTTP
- Deep copy impact: 150kB (~1.22%) — negligible

## Results

| Setup              | Throughput/min | CPU Overhead | Memory Overhead |
|-------------------|----------------|--------------|-----------------|
| 10 Workers        | 83,663         | 66%          | 0%              |
| 100 Workers       | 84,042         | 85%          | 0%              |
| 1 Worker + Async  | 119,720        | 50%          | 1.22%           |

## Conclusion

- Worker pools introduce significant concurrency overhead
- Increasing worker count doesn’t scale linearly
- **Async execution outperforms worker pools** for IO-bound tasks
- Worker pools remain ideal for CPU-bound tasks
- When order matters, prefer worker pool even for IO-bound jobs

