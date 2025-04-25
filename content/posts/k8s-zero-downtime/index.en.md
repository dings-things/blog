---
title: "[Infra] Zero-Downtime Kubernetes Deployment Guide"
date: "2024-08-16"
draft: false
author: dingyu
categories: ["infra"]
tags: ["k8s", "zero-downtime"]
summary: How can we prevent elusive 502 and 504 errors during rolling updates in Kubernetes?
cover:
  image: img/kubernetes.png
  relative: true
comments: true
description: During our migration from IDC to EKS, we encountered numerous challenges—including security configurations, network settings, databases, and ultimately, application deployments. After each deployment, we frequently faced 502 and 504 errors without a clear solution. Since minimizing downtime was critical, I’ll share how we overcame these 502 and 504 issues.
---

## Background

- During load testing in a Kubernetes environment, intermittent 502/504 errors were observed.
- Pods were terminated before they could complete serving responses → 504 Gateway Timeout
- New Pods were created and started serving traffic before they were fully ready → 502 Bad Gateway

Rolling updates proceed smoothly **only if Readiness Probes are correctly configured**.

## Setup

### Load Testing Tools
- [bombardier](https://github.com/codesenberg/bombardier): Simple Go CLI load testing tool
- [vegeta](https://github.com/tsenart/vegeta): Flexible script-based HTTP load testing

---

### Readiness Probe
> Mechanism to determine if a Pod is ready to accept traffic

If a container is not ready (e.g. app not fully initialized), it should not receive traffic.

**Without a readiness probe**, incoming traffic may reach Pods before the application is ready, resulting in 502 errors.

![](img/1.png)

#### Example Deployment Snippet
```yaml
readinessProbe:
  httpGet:
    port: 8080
    path: /alive
    scheme: HTTP  
  initialDelaySeconds: 30
  periodSeconds: 30
```

#### Test Output
```bash
bombardier -c 200 -d 3m -l https://{endpoint}
```

Result (simplified):
```
HTTP codes:
  4xx - 753060, 5xx - 12
```
**5XX errors still present.**

---

### lifecycle & preStop Hook
> Used to execute a shutdown script **before** the container terminates.

This enables **graceful shutdown**: disconnect service → finish pending requests → terminate.

![](img/2.png)

#### Example Deployment Snippet
```yaml
lifecycle:
  preStop:
    exec:
      command:
        - /bin/sh
        - -c
        - sleep 40
```

This introduces a 40s delay before actual container shutdown.

#### Test Output
```bash
bombardier -c 200 -d 3m -l https://{endpoint}
```

Result (simplified):
```
HTTP codes:
  2xx - 751239, 5xx - 3
```
**Still not perfect.**

---

### terminationGracePeriodSeconds
> Time Kubernetes waits for a Pod to shut down **before** forcefully terminating with SIGKILL.

Default is **30 seconds**, which may be **shorter than your preStop delay**.

![](img/3.png)

#### Example Deployment Snippet
```yaml
terminationGracePeriodSeconds: 50
```

> Ensure the following relationship:
> `preStop (40s)` < `terminationGracePeriodSeconds (50s)` < `ALB timeout (60s)`

#### Test Output
```bash
bombardier -c 200 -d 3m -l https://{endpoint}
```

Result:
```
HTTP codes:
  2xx - 770240, 5xx - 0
```
🎉 **No more 5xx errors!**

---

### References
- [Kakao Tech - Zero Downtime Deployment in Kubernetes](https://tech.kakao.com/posts/360)
- [Kubernetes Pod Lifecycle Docs](https://kubernetes.io/ko/docs/concepts/workloads/pods/pod-lifecycle/)