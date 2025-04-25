---
title: "[Third-Party] Integrating Sentry in Go"
date: "2024-09-11"
draft: false
author: dingyu
categories: ["third-party"]
tags: ["go", "sentry"]
summary: A tool that supports both APM and error traceability?
cover:
  image: img/sentry.png
  relative: true
comments: true
description: This post explores how to effectively integrate Sentry into Go projects to enhance error tracking, performance monitoring, and issue management. It covers the importance of structured logging, setting up Sentry alerts, and implementing error capturing using Go's pkg/errors for better stack trace visibility. Additionally, it introduces best practices such as singleton-based initialization, contextual error handling, and panic recovery middleware, ensuring stable and efficient service operations.
---

## Why Sentry?

- Traditional log monitoring has limitations such as:
  - Elasticsearch field mapping errors causing dropped log events
  - Need to request system logs from SEs to retrieve stack traces

Sentry provides a centralized, visual, and automated error tracking and performance monitoring solution.

## What Sentry Provides

1. **Exception Capture**: Automatically detects and logs application errors with stack trace and context.
2. **Transaction Monitoring**: Tracks API/web request performance metrics.
3. **Tracing**: Visualizes end-to-end latency across services and components.
4. **Alerting**: Integrates with tools like Slack, email to notify devs of issues.
5. **Release Tracking**: Monitors how errors correlate to new application releases.

![](img/1.png)

### Alert Rule Examples

- **Issues**: Trigger alerts based on type of exception or service.
- **Number of Errors**: Trigger when a particular error exceeds a threshold.
- **Users Experiencing Errors**: Notify when errors affect multiple users, e.g. 100 users hitting login errors.

![](img/2.png)
![](img/3.png)
![](img/4.png)
![](img/5.png)

## Client Configuration

- **DSN**: Unique key per project for event ingestion.
  - [Settings → Projects → Client Keys]
  - ![](img/6.png)
- **Environment**: Differentiate between `production`, `staging`, etc.
- **Release**: Tag errors by release version.
- **SampleRate / TracesSampleRate**: Control event sampling to optimize cost and performance.
- **BeforeSend**: Hook to redact sensitive data or conditionally send events.
- **AttachStacktrace**: Attach stack traces even for non-errors.
- **ServerName**: Identify source server.
- **Transport**: Custom HTTP config for timeouts and retries.

### Init Best Practice
```go
err := sentry.Init(sentry.ClientOptions{
  Dsn:              config.Sentry.DSN,
  SampleRate:       config.Sentry.SampleRate,
  EnableTracing:    config.Sentry.EnableTrace,
  Debug:            config.Sentry.Debug,
  TracesSampleRate: config.Sentry.TracesSampleRate,
  Environment:      config.Sentry.Environment,
  AttachStacktrace: true,
  Transport: &sentry.HTTPSyncTransport{ Timeout: config.Sentry.Timeout },
})
```

## Error Capturing

### Capture with Context
```go
hub := sentry.GetHubFromContext(ctx)
if hub == nil {
  hub = sentry.CurrentHub().Clone()
  ctx = sentry.SetHubOnContext(ctx, hub)
}
hub.CaptureException(err)
```

### Standard vs. pkg/errors

- `errors.New`: no stack trace
- `pkg/errors.Wrap`: includes full stack trace

![](img/7.png)

## Contextual Scopes
```go
hub := sentry.GetHubFromContext(r.Context())
if hub == nil {
  hub = sentry.CurrentHub().Clone()
  r = r.WithContext(sentry.SetHubOnContext(r.Context(), hub))
}
hub.Scope().SetRequest(r)
```

## Singleton Initialization
```go
type SentryInitializer struct {
  conf *Config
  enabled bool
  mutex sync.RWMutex
}

func (si *SentryInitializer) Init() error {
  si.mutex.Lock()
  defer si.mutex.Unlock()
  err := sentry.Init(...)
  si.enabled = true
  return err
}
```

## Capturing via Interface
![](img/8.png)
```go
type ErrorCapturer interface {
  CaptureError(ctx context.Context, err error)
}

func (ec *sentryErrorCapturer) CaptureError(ctx context.Context, err error) {
  if err == nil { return }
  if !ec.Enabled() { ec.Init() }
  hub := sentry.GetHubFromContext(ctx)
  if hub == nil {
    hub = sentry.CurrentHub().Clone()
    ctx = sentry.SetHubOnContext(ctx, hub)
  }
  hub.CaptureException(err)
}
```

## Panic Recovery Middleware
![](img/9.png)
```go
func (rm *RecoverMiddleware) Register(h http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    hub := sentry.GetHubFromContext(r.Context())
    if hub == nil {
      hub = sentry.CurrentHub().Clone()
      r = r.WithContext(sentry.SetHubOnContext(r.Context(), hub))
    }
    hub.Scope().SetRequest(r)

    defer func() {
      if err := recover(); err != nil {
        hub.RecoverWithContext(r.Context(), err)
        // log stack trace, respond 500
      }
    }()

    h.ServeHTTP(w, r)
  })
}
```

## Summary

Sentry simplifies APM and error tracking for Go applications:
- Automatic stack trace
- Release correlation
- Slack/email alerts
- Low overhead

It’s a powerful ally for ensuring observability and reliability in production.

