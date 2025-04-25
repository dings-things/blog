---
date: "2025-01-12"
draft: false
author: dingyu
categories: ["convention"]
tags:
- go
- convention
title: '[Go] Go Convention'
summary: Efficient Go Project Structure Guide
cover:
    image: img/go-thumbnail.png
    relative: true
comments: true
description: There aren’t many Go developers in Korea, and major IT companies using Go are as rare as unicorns. At my company, we have accumulated our own know-how over the years while developing with Go. Here, I’d like to suggest some key conventions that I believe are useful for maintaining services in the long run.
---

# Background
- Inconsistent project structures per service made code hard to follow.
- No shared conventions made common module/CI reuse difficult, reducing productivity.
- Onboarding was challenging, especially for Go newcomers.
- Our team has now matured in using Go and has built internal best practices.
- We want to establish conventions for better maintainability and clarity across services.

# Proposed Project Structure
> Sections marked with * are mandatory.
> This structure assumes MSA-style projects rather than large monoliths.

## Example Structure (Domain: match sampling)
```
.
├── *docs
│   ├── *swagger.yaml
│   ├── sequence.md
│   └── architecture.md
├── *cmd
│   └── *main.go
├── pkg
│   ├── file_parser.go
│   └── time_convertor.go
└── *internal
    ├── *handler
    │   ├── *v1/sampling_handler.go
    │   ├── v2/sampling_handler.go
    │   ├── server.go
    │   ├── health_handler.go
    │   ├── swagger_handler.go
    │   └── auth_middleware.go
    ├── data
    │   ├── mysqldb.go
    │   ├── redis.go
    │   ├── feature_event_producer.go
    │   ├── match_repository.go
    │   └── nass_api.go
    ├── *service
    │   ├── kda_sampler.go
    │   ├── match_sampling_usecase.go
    │   └── kda_sampler_test.go
    ├── logger.go
    ├── constants.go
    └── *config.go
├── *gitlab-ci.yml
├── *go.mod
├── *go.sum
└── *README.md
```

![](layered-architecture.png)

| Section | Required | Description |
|--------|----------|-------------|
| **docs** | ✔ | Project-level diagrams and specs |
| **cmd** | ✔ | Entry point for DI & execution |
| **pkg** | optional | Utility modules safe for external reuse |
| **internal** | ✔ | Core domain logic, hidden from outside |
| **handler** | ✔ | HTTP/gRPC/Kafka handlers (versioned) |
| **data** | optional | Database, external API, Kafka interactions |
| **service** | ✔ | Business logic per SRP, unit tested |
| **root files** | ✔ | CI, readme, mod/sum files |

# Constant Convention
- Use `PascalCase` for constants shared across packages.
- Use `camelCase` for internal/private constants.
- If a private constant must be exposed, wrap it via a public method.

# Data Model Convention
- Define models close to their use (not in shared folders).
- Use DTOs between layers, and convert as needed.
- For 2+ arguments, use structs.
- Keep validation logic in methods for readability and testability.

```go
// AS-IS
if strings.HasPrefix(match.Version, "rc") && match.detail == "test" { ... }

// TO-BE
if match.IsTest() { ... }
```

## Test Conventions

Testing business logic is **mandatory**, not optional.
You should write test cases for already-defined errors in a **concise yet detailed** manner.

### Deterministic Asynchronous Unit Testing
Avoid relying on `time.Sleep` or blindly logging after async execution without assertions.

Prevent **flaky tests** by leveraging **Dependency Injection (DI)** and `assert.Eventually`.

#### 1. Injecting a Logger
```go
// NewQueue creates a Queue instance responsible for business logic
func NewQueue(
	config Config,
	httpClient *http.Client,
	logger *zerolog.Logger,
) (queue Queue, err error) {
	// The queue will execute the thread executor when Start() is called.
	queue = Queue{
		config:   config,
		client:   httpClient,
		logger:   logger,
		quitChan: make(chan struct{}),
	}
	return
}
```

#### 2. Testing Output

- **Test case for queue failure logging**
```go
t.Run("Logs failure correctly when queue processing fails", func(t *testing.T) {
	// given
	var buffer bytes.Buffer
	... inject logger with buffer as output

	// when
	... execute async task
	event1, err := queue.Push([]byte(validJSON1))
	assert.NoError(t, err)
	event2, err := queue.Push([]byte(validJSON2))
	assert.NoError(t, err)

	// then
	assert.Eventually(t, func() bool {
		output := buffer.String()
		return strings.Contains(output, event1.TraceID().String()) &&
			strings.Contains(output, event2.TraceID().String()) &&
			strings.Contains(output, `"success":false`)
	}, 1*time.Second, 10*time.Millisecond)
})
```

- **Test case for queue success logging**
```go
t.Run("Logs success correctly when queue processes successfully", func(t *testing.T) {
	// given
	var buffer bytes.Buffer
	... inject logger with buffer as output

	// when
	... execute async task
	event1, err := queue.Push([]byte(validJSON1))
	assert.NoError(t, err)
	event2, err := queue.Push([]byte(validJSON2))
	assert.NoError(t, err)

	// then
	assert.Eventually(t, func() bool {
		output := buffer.String()
		return strings.Contains(output, event1.TraceID().String()) &&
			strings.Contains(output, event2.TraceID().String()) &&
			strings.Contains(output, `"success":true`)
	}, 1*time.Second, 10*time.Millisecond)
})
```