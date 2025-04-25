---
date: "2024-11-05"
draft: false
author: dingyu
categories: ["conference"]
tags:
- go
- kubernetes
- good practice
title: '[Go] Gophercon 2024 - Kubernetes Platform Programming'
summary: How to use the Kubernetes API with Go… and deploy applications using Operators
cover:
    image: img/go-thumbnail.png
    relative: true
comments: true
description: 
---

## Kubernetes API

- An interface to interact with the Kubernetes platform.
- Used by users, administrators, and applications to manage clusters and resources.
- Enables a series of operations such as listing or creating resources.
- Supports application deployment and status monitoring.
- Provided as an HTTP API, compatible with various languages and tools.

> `https://<APISERVER>/<GROUP>/<VERSION>/<RESOURCE>/<NAME>`

- **API SERVER**: The address of the API server
- **GROUP**:
  - `/api`: Core API group (e.g., Pod, Service)
  - `/apis/*`: Extended API group for deployment and management features
- **VERSION**: e.g., v1 (stable), v1beta, v1alpha1
- **RESOURCE**: The type of resource (e.g., /pods, /services)
  - `<NAMESPACE>/<RESOURCE>`: Namespace-specific queries
- **NAME**: The specific name of the target resource

## client-go Library

- Abstracts Kubernetes API calls into convenient Go functions.
- Access both core and extended resource groups.
- Supports initialization inside/outside the cluster.
- Provides Informers to cache resource changes.

- Querying `Pods("")` allows access to all namespaces (within permission scope).

### When Updating a Deployment Concurrently

- The first request that reaches the server is accepted, others fail.
  - Similar to MVCC: version conflict detection prevents lost updates.

### Simultaneous Image Change from Different Servers

- One request will succeed; the rest will fail with a `Conflict` error.

### Optimistic Concurrency

| Step 1 | Step 2 | Step 3 |
|--------|--------|--------|
| Submit update with resource version | Apply if version matches stored version | Fail on version mismatch |

Use `retry.RetryOnConflict()` to automatically retry on version conflicts.

## Controllers and Operators in Kubernetes

**Controller**:
- Manages native Kubernetes resources (Pods, Deployments, Services).

**Operator**:
- Manages complex state via custom resources.
- Automates application deployment and management.

## kubebuilder Framework

- Go-based framework to develop Kubernetes controllers/operators.
- Generates boilerplate for CRD management and reconciliation.
- Helps focus only on core business logic.
- Alternative to client-go (which is more complex).

**Component Overview:**
- **Process**: Application configuration/init
- **Manager**: API server communication, event caching, metrics
- **Webhook**: Initialization and validation logic
- **Controller**: Filters resource events and triggers Reconciler
- **Reconciler**: Implements logic for managing custom resources

- **SPEC**: Desired state definition
- **STATUS**: Current state reporting

## Controller Option Configuration

- Settings for the `Reconcile()` function triggered on resource changes.
- Includes filters, performance settings, resource scopes.

- Generation field can be used to only trigger reconciliation on spec changes.
- Configure max concurrent reconciliations via controller options.

### When is Reconcile() Triggered?

- On operator startup (traverses all resources)
- On resource updates
  - Default: any field change
  - Customizable to spec-only changes

## Takeaways

- While CPU and memory-based autoscaling is standard, customizing based on application nature can improve performance.
  - e.g., for queue-based workers, scale via queue length.
- Conflict errors during concurrent deployment can be mitigated using custom controllers.
- Instead of manually triggering deployment via Rundeck, a DaemonSet-based operator could handle automation (with caution).

