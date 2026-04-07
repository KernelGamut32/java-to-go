# Lab 8: Build a REST API

**Block 8 — Building HTTP Services (The Standard Library Way)**  
**Duration:** 65 minutes  
**Prerequisites:** Labs 6–7 completed, Go 1.25+ installed, VS Code with Go extension  
**Module:** `go mod init taskapi`

---

## Learning Objectives

By the end of this lab you will be able to:

- Stand up an HTTP server using only the Go standard library
- Register routes with method matching and path parameters (Go 1.22+ `ServeMux`)
- Define domain structs with JSON struct tags
- Implement full CRUD operations with proper HTTP status codes
- Protect shared state with `sync.RWMutex`
- Parse JSON request bodies and return structured error responses
- Build composable middleware using the `http.Handler` interface
- Log requests with structured logging via `log/slog`

---

## The Application

You will build a **Task Manager API** — a complete CRUD service with zero external dependencies. Every import comes from the Go standard library.

### Endpoints

| Method | Path | Description | Success Code |
|--------|------|-------------|--------------|
| `GET` | `/health` | Health check | 200 |
| `POST` | `/tasks` | Create a task | 201 |
| `GET` | `/tasks` | List all tasks | 200 |
| `GET` | `/tasks/{id}` | Get a task by ID | 200 |
| `PUT` | `/tasks/{id}` | Update a task | 200 |
| `DELETE` | `/tasks/{id}` | Delete a task | 204 |

---

## Setup

```bash
mkdir -p taskapi && cd taskapi
go mod init taskapi
```

You will build the entire application in a single file (`main.go`) for simplicity. In a production codebase you would split this into packages — but the goal here is to see how little Go code is needed.

---

## Step 1 — Health Check and Server Scaffold (5 min)

**Goal:** Get a server running with a single endpoint.

### Instructions

1. Create `main.go`.
2. Create an `http.ServeMux`, register a `GET /health` handler, and start the server on port `8080`.

### Code

```go
package main

import (
	"encoding/json"
	"log/slog"
	"net/http"
	"os"
)

func main() {
	logger := slog.New(slog.NewTextHandler(os.Stdout, nil))

	mux := http.NewServeMux()

	mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
	})

	logger.Info("server starting", "addr", ":8080")
	if err := http.ListenAndServe(":8080", mux); err != nil {
		logger.Error("server failed", "error", err)
		os.Exit(1)
	}
}
```

### Test It

In a second terminal:

```bash
curl -s http://localhost:8080/health | jq .
```

Expected output:

```json
{
  "status": "ok"
}
```

### What to Observe

- The route pattern `"GET /health"` includes the HTTP method. This is the Go 1.22+ enhanced `ServeMux` — no third-party router needed. A `POST /health` request will automatically receive a `405 Method Not Allowed` response.
- `json.NewEncoder(w).Encode(...)` streams JSON directly into the `http.ResponseWriter` — no intermediate byte buffer.
- `slog.New(slog.NewTextHandler(...))` creates a structured logger. You will use this throughout the lab.

---

## Step 2 — Define the Domain Model (5 min)

**Goal:** Define the `Task` struct and the in-memory store.

### Instructions

Add the following above your `main` function:

```go
import (
	"encoding/json"
	"log/slog"
	"net/http"
	"os"
	"sync"
	"time"
)

// Task represents a to-do item.
type Task struct {
	ID        string    `json:"id"`
	Title     string    `json:"title"`
	Status    string    `json:"status"`
	CreatedAt time.Time `json:"created_at"`
}

// TaskStore is a concurrency-safe in-memory store.
type TaskStore struct {
	mu    sync.RWMutex
	tasks map[string]Task
	nextID int
}

// NewTaskStore creates an initialized store.
func NewTaskStore() *TaskStore {
	return &TaskStore{
		tasks: make(map[string]Task),
	}
}
```

### What to Observe

- **JSON struct tags** (`json:"id"`) control the field names in JSON output. Without tags, Go uses the uppercase field name (`ID`, `Title`), which doesn't match typical JSON conventions.
- `sync.RWMutex` allows multiple concurrent readers (`RLock`) but exclusive writers (`Lock`). This is the right choice for a read-heavy data store.
- `nextID` is a simple auto-incrementing counter. In production you would use UUIDs, but sequential IDs make the lab easier to follow.

---

## Step 3 — Implement Store Methods (10 min)

**Goal:** Build the data access layer on `TaskStore`.

### Instructions

Add these methods to `TaskStore`:

```go
import (
	"fmt"
	"encoding/json"
	"log/slog"
	"net/http"
	"os"
	"sync"
	"time"
)

// Create adds a new task and returns it with a generated ID.
func (s *TaskStore) Create(title string) Task {
	s.mu.Lock()
	defer s.mu.Unlock()

	s.nextID++
	task := Task{
		ID:        fmt.Sprintf("task-%d", s.nextID),
		Title:     title,
		Status:    "pending",
		CreatedAt: time.Now().UTC(),
	}
	s.tasks[task.ID] = task
	return task
}

// All returns every task as a slice.
func (s *TaskStore) All() []Task {
	s.mu.RLock()
	defer s.mu.RUnlock()

	result := make([]Task, 0, len(s.tasks))
	for _, t := range s.tasks {
		result = append(result, t)
	}
	return result
}

// Get returns a single task by ID.
func (s *TaskStore) Get(id string) (Task, bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	task, ok := s.tasks[id]
	return task, ok
}

// Update replaces the title and/or status of an existing task.
func (s *TaskStore) Update(id, title, status string) (Task, bool) {
	s.mu.Lock()
	defer s.mu.Unlock()

	task, ok := s.tasks[id]
	if !ok {
		return Task{}, false
	}
	if title != "" {
		task.Title = title
	}
	if status != "" {
		task.Status = status
	}
	s.tasks[id] = task
	return task, true
}

// Delete removes a task by ID. Returns true if it existed.
func (s *TaskStore) Delete(id string) bool {
	s.mu.Lock()
	defer s.mu.Unlock()

	_, ok := s.tasks[id]
	if ok {
		delete(s.tasks, id)
	}
	return ok
}
```

### What to Observe

- Every public method acquires the appropriate lock (`RLock` for reads, `Lock` for writes) and defers the unlock. The `defer` guarantees the lock is released even if the method panics — this is a Go best practice.
- `All()` pre-allocates the result slice with `make([]Task, 0, len(s.tasks))`. This avoids repeated allocations as `append` grows the slice.
- `Get` returns `(Task, bool)` — the standard Go pattern for "found or not found." No exceptions, no `Optional<T>`, no null. Java developers: this is Go's `Map.getOrDefault` without the ceremony.

---

## Step 4 — Implement CRUD Handlers (15 min)

**Goal:** Wire up all five CRUD endpoints.

### Instructions

Add a helper function for writing JSON responses and JSON error responses, then implement each handler. Replace your `main` function with the expanded version below.

### Helper Functions

```go
// writeJSON writes a JSON response with the given status code.
func writeJSON(w http.ResponseWriter, status int, data any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(data)
}

// writeError writes a JSON error response.
func writeError(w http.ResponseWriter, status int, message string) {
	writeJSON(w, status, map[string]string{"error": message})
}
```

### Handlers

```go
func main() {
	logger := slog.New(slog.NewTextHandler(os.Stdout, nil))
	store := NewTaskStore()
	mux := http.NewServeMux()

	// Health check
	mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
		writeJSON(w, http.StatusOK, map[string]string{"status": "ok"})
	})

	// POST /tasks — Create a task
	mux.HandleFunc("POST /tasks", func(w http.ResponseWriter, r *http.Request) {
		var input struct {
			Title string `json:"title"`
		}
		if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
			writeError(w, http.StatusBadRequest, "invalid JSON: "+err.Error())
			return
		}
		if input.Title == "" {
			writeError(w, http.StatusBadRequest, "title is required")
			return
		}

		task := store.Create(input.Title)
		writeJSON(w, http.StatusCreated, task)
	})

	// GET /tasks — List all tasks
	mux.HandleFunc("GET /tasks", func(w http.ResponseWriter, r *http.Request) {
		tasks := store.All()
		writeJSON(w, http.StatusOK, tasks)
	})

	// GET /tasks/{id} — Get a task by ID
	mux.HandleFunc("GET /tasks/{id}", func(w http.ResponseWriter, r *http.Request) {
		id := r.PathValue("id")
		task, ok := store.Get(id)
		if !ok {
			writeError(w, http.StatusNotFound, "task not found: "+id)
			return
		}
		writeJSON(w, http.StatusOK, task)
	})

	// PUT /tasks/{id} — Update a task
	mux.HandleFunc("PUT /tasks/{id}", func(w http.ResponseWriter, r *http.Request) {
		id := r.PathValue("id")

		var input struct {
			Title  string `json:"title"`
			Status string `json:"status"`
		}
		if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
			writeError(w, http.StatusBadRequest, "invalid JSON: "+err.Error())
			return
		}

		task, ok := store.Update(id, input.Title, input.Status)
		if !ok {
			writeError(w, http.StatusNotFound, "task not found: "+id)
			return
		}
		writeJSON(w, http.StatusOK, task)
	})

	// DELETE /tasks/{id} — Delete a task
	mux.HandleFunc("DELETE /tasks/{id}", func(w http.ResponseWriter, r *http.Request) {
		id := r.PathValue("id")
		if !store.Delete(id) {
			writeError(w, http.StatusNotFound, "task not found: "+id)
			return
		}
		w.WriteHeader(http.StatusNoContent)
	})

	logger.Info("server starting", "addr", ":8080")
	if err := http.ListenAndServe(":8080", mux); err != nil {
		logger.Error("server failed", "error", err)
		os.Exit(1)
	}
}
```

### What to Observe

- **`r.PathValue("id")`** extracts the `{id}` wildcard from the URL. This is the Go 1.22+ path parameter API — no third-party router required.
- **Method matching** is in the pattern string: `"POST /tasks"`, `"GET /tasks/{id}"`, etc. If someone sends a `PATCH /tasks/123` request, the server automatically returns `405 Method Not Allowed` with an `Allow` header listing the valid methods.
- **`json.NewDecoder(r.Body).Decode(&input)`** streams the request body directly into a struct. This is more efficient than reading the entire body into memory first with `io.ReadAll`. For small payloads the difference is negligible, but the habit matters.
- Anonymous structs (`var input struct { Title string }`) are used for request parsing. They are scoped to the handler and avoid polluting the package namespace with single-use types.
- **`w.WriteHeader` must be called before writing the body.** Once you call `w.Write()` or `json.NewEncoder(w).Encode(...)`, Go automatically sends a `200 OK` status. The `writeJSON` helper calls `WriteHeader` first to set the correct code.

---

## Step 5 — Test Your API with curl (5 min)

**Goal:** Verify every endpoint works correctly.

Restart your server (`go run main.go`), then run each command:

### Create Tasks

```bash
curl -s -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Learn goroutines"}' | jq .

curl -s -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Build a REST API"}' | jq .

curl -s -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Deploy to production"}' | jq .
```

### List All Tasks

```bash
curl -s http://localhost:8080/tasks | jq .
```

### Get a Single Task

```bash
curl -s http://localhost:8080/tasks/task-1 | jq .
```

### Update a Task

```bash
curl -s -X PUT http://localhost:8080/tasks/task-1 \
  -H "Content-Type: application/json" \
  -d '{"status": "completed"}' | jq .
```

### Delete a Task

```bash
curl -s -X DELETE http://localhost:8080/tasks/task-2 -w "\nHTTP Status: %{http_code}\n"
```

### Error Cases

```bash
# Missing title (400 Bad Request)
curl -s -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{}' | jq .

# Invalid JSON (400 Bad Request)
curl -s -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d 'not json' | jq .

# Task not found (404 Not Found)
curl -s http://localhost:8080/tasks/task-999 | jq .

# Wrong method (405 Method Not Allowed)
curl -s -X PATCH http://localhost:8080/tasks/task-1 -w "\nHTTP Status: %{http_code}\n"
```

---

## Step 6 — Logging Middleware with `slog` (10 min)

**Goal:** Build a middleware that logs every request with method, path, and response time using structured logging.

### Context

Middleware in Go is simply a function that takes an `http.Handler` and returns a new `http.Handler`. The wrapper calls the original handler and adds behavior around it — logging, authentication, CORS headers, etc. There is no framework annotation, no `@Filter`, no `web.xml`. It is a plain function.

### Instructions

1. Add the middleware function below.
2. Wrap your mux with the middleware in `main`.

### The Middleware

```go
// loggingMiddleware logs every request with method, path, status code, and duration.
func loggingMiddleware(logger *slog.Logger, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()

		// Wrap the ResponseWriter to capture the status code.
		wrapped := &statusRecorder{ResponseWriter: w, statusCode: http.StatusOK}

		next.ServeHTTP(wrapped, r)

		logger.Info("request",
			"method", r.Method,
			"path", r.URL.Path,
			"status", wrapped.statusCode,
			"duration_ms", time.Since(start).Milliseconds(),
		)
	})
}

// statusRecorder wraps http.ResponseWriter to capture the status code.
type statusRecorder struct {
	http.ResponseWriter
	statusCode int
}

func (sr *statusRecorder) WriteHeader(code int) {
	sr.statusCode = code
	sr.ResponseWriter.WriteHeader(code)
}
```

### Update `main` — Wrap the Mux

Replace the `http.ListenAndServe` line:

```go
	handler := loggingMiddleware(logger, mux)

	logger.Info("server starting", "addr", ":8080")
	if err := http.ListenAndServe(":8080", handler); err != nil {
		logger.Error("server failed", "error", err)
		os.Exit(1)
	}
```

### Test It

Make a few curl requests and observe the server's stdout:

```
time=2026-04-06T13:05:22.000Z level=INFO msg=request method=GET path=/health status=200 duration_ms=0
time=2026-04-06T13:05:23.000Z level=INFO msg=request method=POST path=/tasks status=201 duration_ms=0
time=2026-04-06T13:05:24.000Z level=INFO msg=request method=GET path=/tasks/task-999 status=404 duration_ms=0
```

### What to Observe

- **Structured logging** with `slog` produces key-value pairs, not free-form strings. This is parseable by log aggregation tools (Datadog, Splunk, CloudWatch) without regex.
- The `statusRecorder` pattern is necessary because `http.ResponseWriter` does not expose the status code after it is written. By embedding the original writer and overriding `WriteHeader`, we capture the code while preserving all other behavior.
- Middleware composes by nesting: `loggingMiddleware(logger, authMiddleware(apiKey, mux))`. Each layer wraps the next. This is identical in spirit to Java's servlet filters or Spring interceptors, but expressed as plain functions.

### Switching to JSON Log Output

For production, replace the handler in the logger:

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
```

Output becomes:

```json
{"time":"2026-04-06T13:05:22Z","level":"INFO","msg":"request","method":"GET","path":"/health","status":200,"duration_ms":0}
```

This is the Go equivalent of configuring SLF4J + Logback with a JSON encoder — except it is built in and requires no dependencies.

---

## Step 7 — Test the Complete API (5 min)

**Goal:** Run through a full workflow to verify everything works together.

Restart the server and execute this script:

```bash
echo "=== Health Check ==="
curl -s http://localhost:8080/health | jq .

echo -e "\n=== Create 3 Tasks ==="
curl -s -X POST http://localhost:8080/tasks -H "Content-Type: application/json" \
  -d '{"title":"Write unit tests"}' | jq .
curl -s -X POST http://localhost:8080/tasks -H "Content-Type: application/json" \
  -d '{"title":"Review pull request"}' | jq .
curl -s -X POST http://localhost:8080/tasks -H "Content-Type: application/json" \
  -d '{"title":"Update documentation"}' | jq .

echo -e "\n=== List All Tasks ==="
curl -s http://localhost:8080/tasks | jq .

echo -e "\n=== Update Task 1 ==="
curl -s -X PUT http://localhost:8080/tasks/task-1 -H "Content-Type: application/json" \
  -d '{"status":"in-progress"}' | jq .

echo -e "\n=== Get Task 1 ==="
curl -s http://localhost:8080/tasks/task-1 | jq .

echo -e "\n=== Delete Task 2 ==="
curl -s -X DELETE http://localhost:8080/tasks/task-2 -w "HTTP Status: %{http_code}\n"

echo -e "\n=== Verify Task 2 Deleted ==="
curl -s http://localhost:8080/tasks/task-2 | jq .

echo -e "\n=== Final Task List ==="
curl -s http://localhost:8080/tasks | jq .
```

Check your server's terminal — you should see a structured log line for every request with method, path, status, and duration.

---

## Step 8 — Stretch Goal: API Key Authentication Middleware (10 min)

**Goal:** Add a middleware that rejects requests without a valid `X-API-Key` header.

### Instructions

1. Add the `authMiddleware` function.
2. Stack it with the logging middleware so both run on every request.
3. The `/health` endpoint should remain public (unauthenticated).

### The Middleware

```go
// authMiddleware rejects requests without a valid API key.
func authMiddleware(apiKey string, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Allow health checks without authentication.
		if r.URL.Path == "/health" {
			next.ServeHTTP(w, r)
			return
		}

		key := r.Header.Get("X-API-Key")
		if key == "" {
			writeError(w, http.StatusUnauthorized, "missing X-API-Key header")
			return
		}
		if key != apiKey {
			writeError(w, http.StatusForbidden, "invalid API key")
			return
		}

		next.ServeHTTP(w, r)
	})
}
```

### Update `main` — Stack Middleware

```go
	const apiKey = "my-secret-key-12345"

	handler := loggingMiddleware(logger, authMiddleware(apiKey, mux))

	logger.Info("server starting", "addr", ":8080", "auth", "enabled")
	if err := http.ListenAndServe(":8080", handler); err != nil {
		logger.Error("server failed", "error", err)
		os.Exit(1)
	}
```

### Test It

```bash
# Health check — no key required
curl -s http://localhost:8080/health | jq .

# Missing key — 401
curl -s http://localhost:8080/tasks | jq .

# Wrong key — 403
curl -s -H "X-API-Key: wrong" http://localhost:8080/tasks | jq .

# Correct key — 200
curl -s -H "X-API-Key: my-secret-key-12345" http://localhost:8080/tasks | jq .

# Create with key
curl -s -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -H "X-API-Key: my-secret-key-12345" \
  -d '{"title":"Authenticated task"}' | jq .
```

### What to Observe

- Middleware stacks execute outside-in: logging runs first (wrapping everything), then auth, then the route handler. The log captures the status code even for rejected requests.
- The `/health` exemption is a simple path check. In a production app you might use a separate `ServeMux` for public routes or a more sophisticated allowlist.
- The API key is hardcoded here for simplicity. In production, read it from an environment variable or a secrets manager.

---

## Complete File Reference

At this point your `main.go` should contain approximately 180–200 lines of code with zero external dependencies. For reference, an equivalent Spring Boot application with the same functionality would require a `pom.xml` or `build.gradle`, a main class with `@SpringBootApplication`, a `Task` entity class, a `TaskController` with `@RestController` / `@RequestMapping` annotations, a `TaskService`, exception handler classes, `application.yml` for configuration, and logging configuration — typically 300–500 lines across 5–8 files, plus framework dependency downloads.

Go's standard library gives you the router, the JSON codec, the logger, and the HTTP server. No framework needed.

---

## Wrap-Up Checklist

Before you move on, confirm you can answer these questions:

- [ ] How does Go 1.22+ `ServeMux` method matching work, and what happens when a request uses an unregistered method?
- [ ] What does `r.PathValue("id")` return, and where is `{id}` defined?
- [ ] Why use `json.NewDecoder(r.Body)` instead of `io.ReadAll` + `json.Unmarshal`?
- [ ] What is the purpose of JSON struct tags like `json:"created_at"`?
- [ ] How does `sync.RWMutex` differ from `sync.Mutex`, and why is it appropriate here?
- [ ] How does Go middleware composition compare to Spring's `@Filter` / `HandlerInterceptor`?
- [ ] What is `slog` and how does it compare to SLF4J + Logback?
- [ ] Why does the `statusRecorder` type embed `http.ResponseWriter`?

---

**End of Lab 8**
