---
title: "[EDA] Schema Registry"
date: "2025-02-24"
draft: false
author: dingyu
categories: ["eda"]
tags: ["kafka", "avro", "protobuf", "json", "schema registry"]
summary: How can we ensure backward and forward compatibility for event schemas?
cover:
  image: img/kafka.png
  relative: true
comments: true
description: People often underestimate the importance of documenting schemas before starting to code, especially when working with stream processing. In this post, I’ll explain why using a schema registry is essential and why designing schemas upfront is crucial before diving into coding.
---

## Why Schema First?

While EDA promotes loose coupling, event schemas inherently form a **tight contract** between `producers` and `consumers`. Let’s explore why this contract matters and how a **Schema Registry** helps maintain compatibility.

## Purpose of Event Schemas
- Define structure and format of event data.
- Enforce data consistency between producers and consumers.
- Enable validation, compatibility, and documentation.

If you're familiar with REST APIs, this is similar to defining OpenAPI contracts between services:

![](img/image.png)

Event streams function the same way—producers emit events conforming to predefined schemas; consumers process them based on those expectations.

### Bad Example
```json
{
  "user_id": 123,
  "user_action": 1
} // action code instead of expected string
```

![](img/kafka-event-mismatch.png)

Even with agreed-upon schemas, schema drift can occur—leading to broken consumers. Much like skipping ERD when designing databases, **skipping event schemas is risky** in EDA.

## Common Schema Formats

| Format     | Pros                                                                                   | Cons                                                       |
|------------|-------------------------------------------------------------------------------------------|------------------------------------------------------------|
| **JSON**   | Human-readable, widely supported                                                        | Large size, lacks strong validation                        |
| **Protobuf** | Compact, fast, schema-enforced                                                         | Hard to debug, needs precompiled schema                    |
| **Avro**   | Compact binary, supports schema evolution                                                | Less widely adopted, tooling gaps in some ecosystems       |

Text formats like JSON are appealing for debugging. But size and speed matter in stream processing.

### JSON Suitability Checklist
Use JSON **only if**:
- ✅ Messages are small.
- ✅ You can tolerate slow (de)serialization.
- ✅ Strong type validation isn’t required.
- ✅ You don’t need a schema registry.
- ✅ Volume of messages will remain low.
- ✅ Debugging via raw payload is helpful.

### Advantages of Avro / Protobuf
- Strong typing and schema enforcement
- Fast (de)serialization
- Built-in **backward/forward compatibility**

![](img/image-1.png)

Even small messages show over 2x performance gains in binary formats. The difference increases with message size.

### Impact on Kafka
- Text-based formats like JSON consume more **storage** and **network bandwidth**.
- High volume = performance degradation at produce/consume phases.

## What Is a Schema Registry?

A Schema Registry stores and version-controls data schemas.

Benefits:
- Enforces compatibility
- Enables schema evolution (backward/forward)
- Minimizes payload size by referencing schemas via ID
- Centralized schema governance

### Schema Evolution in Action
![](img/image-2.png)
![](img/image-5.png)

1. Producer publishes an event with schema v2.
2. Consumer detects version mismatch and fetches v2 from registry.
3. Consumer proceeds with updated schema.

No coordination required. Zero downtime schema upgrades!

## Schema Registry vs. Schemaless

| Format     | With Schema Registry                                          | Without Schema Registry                                  |
|------------|---------------------------------------------------------------|-----------------------------------------------------------|
| JSON       | ❌ Schemaless, can't validate or evolve                       | ✅ Easy to debug, but lacks structure                     |
| Protobuf   | ✅ Strong schema + evolution support                         | ❌ Needs .proto file everywhere                          |
| Avro       | ✅ Compact, evolvable binary format                          | ❌ Schema must be embedded in each message               |

![](img/image-3.png)
![](img/image-4.png)

Using a Schema Registry with schema **ID** avoids inflating messages with repeated schema data. This helps keep message sizes small.

### Central Schema vs. Shared Code
- Registry = Central governance, live updates.
- Submodule `.proto` = Tight coupling, manual versioning.

## AWS Glue vs. Confluent Schema Registry

| Feature                    | AWS Glue Registry               | Confluent Schema Registry        |
|----------------------------|----------------------------------|----------------------------------|
| Schema versioning         | ✅ Supported                     | ✅ Supported                     |
| URL persistence           | ✅ ARN-based                     | ✅ REST endpoint-based            |
| Auto upgrade for consumer | ❌ Needs explicit fetch         | ✅ Auto fetch                     |
| Kafka support             | ✅ MSK                          | ✅ Confluent Kafka                |

## Why Use a Schema Registry?

1. **Guarantee Compatibility**: Prevent mismatched producer-consumer schemas.
2. **Support Evolution**: Add/remove fields without breaking clients.
3. **Centralized Governance**: No more shared `.proto` headaches.
4. **Smaller Messages**: Send schema ID, not full schema.
5. **Schema Validation**: Prevent invalid data from entering the stream.
6. **Dynamic Updates**: Auto-fetch new schemas at runtime.
7. **Compatibility Policies**: Enforce forward/backward rules.
8. **Schema Auditing**: View changes via REST API or UI.

## References
- [Confluent Blog: Schemas and Compatibility](https://www.confluent.io/blog/schemas-contracts-compatibility/)
- [Oliveyoung B2B MSK Case](https://oliveyoung.tech/2023-10-04/oliveyoung-b2b-msk-connect-introduction/)
- [Kafka Serialization Overview](https://www.linkedin.com/pulse/exploring-data-serialization-apache-kafka-json-protobuf-joe-z-tb2fc/)

