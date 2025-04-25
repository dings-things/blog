---
date: "2024-12-06"
draft: false
author: dingyu
categories: ["eda"]
tags:
- kafka
- flink
- apache
- fds
- dynamic job
title: '[EDA] Flink Dynamic Job Case Study'
summary: Do we really need to redeploy the job every time the policy changes? A deep dive into executing Flink jobs dynamically.
description: This post explores dynamic rule-based stream processing using Apache Flink for real-time fraud detection. It covers key topics such as dynamic key partitioning, broadcast state for rule updates, and custom window processing to efficiently evaluate transactions without redeploying jobs. The implementation ensures low-latency fraud detection by minimizing shuffle overhead, dynamically applying grouping keys, and leveraging stateful processing. Additionally, it discusses event retention strategies, performance considerations, and architecture trade-offs for building a scalable, high-performance fraud detection system.
cover:
    image: img/flink.png
    relative: true
comments: true
---

> #### REFS
- [Rules Based Stream Processing with Apache Flink's Broadcast Pattern](https://brggs.co.uk/blog/broadcast-state-pattern-rules-based-flink/)
- [Advanced Flink Application Patterns Vol.1: Case Study of a Fraud Detection System](https://flink.apache.org/2020/01/15/advanced-flink-application-patterns-vol.1-case-study-of-a-fraud-detection-system/)
- [Build a dynamic rules engine with Amazon Managed Service for Apache Flink](https://aws.amazon.com/ko/blogs/big-data/build-a-dynamic-rules-engine-with-amazon-managed-service-for-apache-flink/)


# Research Background

- We needed a case study on building a frequency-based detection policy using filtering conditions within a `Window Time`.
- Research was necessary on how to handle all policy processing within a **single job code**.
  - Allocating instances per policy—even when using container virtualization—is resource inefficient.
- Policies are managed by admins and should not require job redeployment upon updates.

# Prerequisites

### Dynamic Partitioning
Kafka uses the **event key** to **hash** and **modulo** into partitions.

In frameworks like Kafka Streams and Flink, using a **non-key field for group by (keyBy)** causes **reshuffling**, which involves cross-network data movement among Task Managers—posing a significant overhead.

To solve this, ensure that **transactions with the same grouping key** are handled by the **same subtask**.

Example: Pre-partitioning by target key (e.g., ID or IP) via separate Kafka topics.

### Terminology
(Abbreviated explanation of JobManager, TaskManager, SubTask, Broadcast, etc.)

# Implementation Strategy

1. Merge **action events** with **active policy events** (via CDC from rule DB) and publish to a topic.
   - If there’s 1 action and N active policies → publish N merged events.
2. Use `DynamicKeyFunction` to dynamically partition source stream by group-by condition.
   - Handles reshuffling **dynamically** without job redeployment.
   - Existing keyBy still processed by current TaskSlots.
3. In `DynamicEvaluationFunction`, evaluate whether each event satisfies a rule → emit restriction event if it does.

## Broadcast State
Broadcast one stream to all parallel subtasks so they can share consistent configuration/rules.

Typical use case: low-throughput control/config stream broadcasted to high-throughput action stream.

### Broadcast Architecture
- Source stream: Payment events
- Broadcast stream: Policy rules (with infinite retention)
- Merge both → evaluate

## Dynamic Data Partitioning
Create a system that can **add/remove rules at runtime** without redeploying jobs.

### Static vs. Dynamic Keys
| Type | Static Key | Dynamic Key |
|------|------------|-------------|
| Definition | Pre-defined field | Runtime-decided field |
| Flexibility | Low | High |
| Implementation | Simple | Complex (rule parsing required) |
| Performance | Optimized | Slight overhead |

Example: If rules are grouped by `id`, all relevant events will go to the same subtask, even if the logic per rule differs.

Policies can share subtasks if their `groupingKey` is the same.

## Rule Broadcasting
Use a broadcast source (e.g., from a rule DB CDC topic) to continuously update the active rules.

Each time a rule is added/modified, it is inserted into the broadcast state.

If `rule.disabled = true`, it is removed.

## Custom Window Processing
Flink offers multiple window types: Tumbling, Sliding, Session.

But... **each has limitations** for fraud detection.
- Tumbling: May miss events at window boundaries.
- Sliding: Has inherent delay and overlapping evaluations.

→ Solution: Implement **Custom Windowing** using state and timestamps.

Events stored as:
```java
MapState<Long, Set<PaymentEvent>> windowState;
```
> Since the state backend is a key-value store, it doesn't support list types directly.
> This means we need to iterate over all timestamps in the map to find valid entries...
> More research is needed here, but since we’re only iterating timestamps (not full events), memory impact may be minimal — though CPU usage could be a concern depending on loop cost.

---

#### Considerations on Event Retention (TTL)
![](image-12.png)

How should we determine the retention period, i.e., the **Time-To-Live (TTL)** for each event?

In `DynamicEvaluationFunction()`, it is possible to receive payment events with the same key scope but evaluate them under **different rules** with **different time windows**.

Therefore, at the time a rule is consumed from the `Rule Stream (Broadcast Stream)`, we must **update and store the longest rule duration** for that key.

##### Example: `UpdateWidestWindow`
```java
@Override
public void processBroadcastElement(Rule rule, Context ctx, Collector<Alert> out) {
  ...
  updateWidestWindowRule(rule, broadcastState);
}

private void updateWidestWindowRule(Rule rule, BroadcastState<Integer, Rule> broadcastState) {
  Rule widestWindowRule = broadcastState.get(WIDEST_RULE_KEY);
  if (widestWindowRule == null) {
    broadcastState.put(WIDEST_RULE_KEY, rule);
    return;
  }
  if (widestWindowRule.getWindowMillis() < rule.getWindowMillis()) {
    broadcastState.put(WIDEST_RULE_KEY, rule);
  }
}
```

In summary, **Dynamic Evaluation** uses the rule with the longest duration to determine the TTL for the event.

> Since the state backend is a key-value store, it doesn't support list types directly.
>
> This means we need to iterate over all timestamps in the map to find valid entries…
>
> More research is needed here, but since we’re only iterating timestamps (not full events), memory impact may be minimal — though CPU usage could be a concern depending on loop cost.

