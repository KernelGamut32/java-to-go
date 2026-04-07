# Lab 9: Test Your REST API

**Block 9 — Testing & Project Layout**  
**Duration:** 50 minutes  
**Prerequisites:** Lab 8 completed (Task Manager REST API), Go 1.25+ installed  
**Starting point:** The `taskapi` project from Lab 8

---

## Learning Objectives

By the end of this lab you will be able to:

- Restructure a Go project into idiomatic `cmd/` and `internal/` layout
- Define a storage interface and implement a mock for isolated testing
- Write table-driven tests with `t.Run()` subtests
- Test HTTP handlers using `httptest.NewRecorder` and `httptest.NewRequest`
- Build reusable test helpers with `t.Helper()`
- Run tests with the race detector and interpret coverage output
- Identify and fix test pollution from shared mutable state

---

## Exercise 1 — Restructure into Idiomatic Project Layout (10 min)

**Goal:** Refactor the single-file REST API from Lab 8 into a production-style Go project layout.

### Target Structure

```
taskapi/
├── cmd/
│   └── server/
│       └── main.go            ← entry point (package main)
├── internal/
│   ├── model/
│   │   └── task.go            ← Task struct and JSON tags
│   ├── store/
│   │   └── store.go           ← TaskStore interface + in-memory implementation
│   └── handler/
│       ├── handler.go         ← HTTP handlers
│       └── handler_test.go    ← tests (you will write these)
├── go.mod
└── go.sum
```

### Why This Layout

- **`cmd/server/main.go`** — Contains only `func main()`. It wires together the components and starts the server. Nothing else lives here. If you add a CLI tool later, it goes in `cmd/cli/main.go`.
- **`internal/`** — The Go compiler enforces that packages under `internal/` cannot be imported by code outside this module. This is a compiler-enforced privacy boundary — not a convention, not a linter rule. Code in `internal/` is private to your module.
- **`internal/model/`** — Domain types. No business logic, no dependencies.
- **`internal/store/`** — Data access. Defines the `TaskStore` interface and provides the in-memory implementation.
- **`internal/handler/`** — HTTP handlers. Depends on the store interface, not the concrete implementation. This is where your tests will live.

### Instructions

1. From your `taskapi/` directory, create the directory structure:

```bash
mkdir -p cmd/server internal/model internal/store internal/handler
```

2. Create `internal/model/task.go`:

```go
package model

import "time"

// Task represents a to-do item.
type Task struct {
	ID        string    `json:"id"`
	Title     string    `json:"title"`
	Status    string    `json:"status"`
	CreatedAt time.Time `json:"created_at"`
}
```

3. Create `internal/store/store.go`:

```go
package store

import (
	"fmt"
	"sync"
	"time"

	"taskapi/internal/model"
)

// TaskStore defines the operations on tasks.
// Handlers depend on this interface, not the concrete implementation.
type TaskStore interface {
	Create(title string) model.Task
	All() []model.Task
	Get(id string) (model.Task, bool)
	Update(id, title, status string) (model.Task, bool)
	Delete(id string) bool
}

// MemoryStore is a concurrency-safe in-memory implementation of TaskStore.
type MemoryStore struct {
	mu     sync.RWMutex
	tasks  map[string]model.Task
	nextID int
}

// NewMemoryStore creates an initialized store.
func NewMemoryStore() *MemoryStore {
	return &MemoryStore{
		tasks: make(map[string]model.Task),
	}
}

func (s *MemoryStore) Create(title string) model.Task {
	s.mu.Lock()
	defer s.mu.Unlock()

	s.nextID++
	task := model.Task{
		ID:        fmt.Sprintf("task-%d", s.nextID),
		Title:     title,
		Status:    "pending",
		CreatedAt: time.Now().UTC(),
	}
	s.tasks[task.ID] = task
	return task
}

func (s *MemoryStore) All() []model.Task {
	s.mu.RLock()
	defer s.mu.RUnlock()

	result := make([]model.Task, 0, len(s.tasks))
	for _, t := range s.tasks {
		result = append(result, t)
	}
	return result
}

func (s *MemoryStore) Get(id string) (model.Task, bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	task, ok := s.tasks[id]
	return task, ok
}

func (s *MemoryStore) Update(id, title, status string) (model.Task, bool) {
	s.mu.Lock()
	defer s.mu.Unlock()

	task, ok := s.tasks[id]
	if !ok {
		return model.Task{}, false
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

func (s *MemoryStore) Delete(id string) bool {
	s.mu.Lock()
	defer s.mu.Unlock()

	_, ok := s.tasks[id]
	if ok {
		delete(s.tasks, id)
	}
	return ok
}
```

4. Create `internal/handler/handler.go`:

```go
package handler

import (
	"encoding/json"
	"log/slog"
	"net/http"
	"time"

	"taskapi/internal/store"
)

// Server holds dependencies for HTTP handlers.
type Server struct {
	Store  store.TaskStore
	Logger *slog.Logger
}

// NewServer creates a Server with the given dependencies.
func NewServer(s store.TaskStore, logger *slog.Logger) *Server {
	return &Server{Store: s, Logger: logger}
}

// Routes returns an http.Handler with all routes registered.
func (s *Server) Routes() http.Handler {
	mux := http.NewServeMux()

	mux.HandleFunc("GET /health", s.handleHealth)
	mux.HandleFunc("POST /tasks", s.handleCreateTask)
	mux.HandleFunc("GET /tasks", s.handleListTasks)
	mux.HandleFunc("GET /tasks/{id}", s.handleGetTask)
	mux.HandleFunc("PUT /tasks/{id}", s.handleUpdateTask)
	mux.HandleFunc("DELETE /tasks/{id}", s.handleDeleteTask)

	return s.loggingMiddleware(mux)
}

func (s *Server) handleHealth(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusOK, map[string]string{"status": "ok"})
}

func (s *Server) handleCreateTask(w http.ResponseWriter, r *http.Request) {
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
	task := s.Store.Create(input.Title)
	writeJSON(w, http.StatusCreated, task)
}

func (s *Server) handleListTasks(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusOK, s.Store.All())
}

func (s *Server) handleGetTask(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")
	task, ok := s.Store.Get(id)
	if !ok {
		writeError(w, http.StatusNotFound, "task not found: "+id)
		return
	}
	writeJSON(w, http.StatusOK, task)
}

func (s *Server) handleUpdateTask(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")
	var input struct {
		Title  string `json:"title"`
		Status string `json:"status"`
	}
	if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
		writeError(w, http.StatusBadRequest, "invalid JSON: "+err.Error())
		return
	}
	task, ok := s.Store.Update(id, input.Title, input.Status)
	if !ok {
		writeError(w, http.StatusNotFound, "task not found: "+id)
		return
	}
	writeJSON(w, http.StatusOK, task)
}

func (s *Server) handleDeleteTask(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")
	if !s.Store.Delete(id) {
		writeError(w, http.StatusNotFound, "task not found: "+id)
		return
	}
	w.WriteHeader(http.StatusNoContent)
}

func (s *Server) loggingMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		wrapped := &statusRecorder{ResponseWriter: w, statusCode: http.StatusOK}
		next.ServeHTTP(wrapped, r)
		s.Logger.Info("request",
			"method", r.Method,
			"path", r.URL.Path,
			"status", wrapped.statusCode,
			"duration_ms", time.Since(start).Milliseconds(),
		)
	})
}

type statusRecorder struct {
	http.ResponseWriter
	statusCode int
}

func (sr *statusRecorder) WriteHeader(code int) {
	sr.statusCode = code
	sr.ResponseWriter.WriteHeader(code)
}

func writeJSON(w http.ResponseWriter, status int, data any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(data)
}

func writeError(w http.ResponseWriter, status int, message string) {
	writeJSON(w, status, map[string]string{"error": message})
}
```

5. Create `cmd/server/main.go`:

```go
package main

import (
	"log/slog"
	"net/http"
	"os"

	"taskapi/internal/handler"
	"taskapi/internal/store"
)

func main() {
	logger := slog.New(slog.NewTextHandler(os.Stdout, nil))

	s := handler.NewServer(store.NewMemoryStore(), logger)

	logger.Info("server starting", "addr", ":8080")
	if err := http.ListenAndServe(":8080", s.Routes()); err != nil {
		logger.Error("server failed", "error", err)
		os.Exit(1)
	}
}
```

### Verify It Compiles and Runs

```bash
go run ./cmd/server
```

Test with curl to confirm everything still works:

```bash
curl -s http://localhost:8080/health | jq .
```

### What to Observe

- `cmd/server/main.go` is 20 lines. It creates dependencies, wires them, and starts the server. That is the only job of a `main` function.
- The `handler` package depends on `store.TaskStore` (an interface), not `store.MemoryStore` (a concrete type). This is the key to testability — you can substitute a mock store in tests.
- The `internal/` directory is enforced by the Go compiler. Try importing `taskapi/internal/handler` from a separate module — the compiler will refuse.

---

## Exercise 2 — Build a Mock Store (5 min)

**Goal:** Create a mock implementation of `TaskStore` for isolated handler testing.

### Instructions

Create `internal/handler/handler_test.go` and start with the mock:

```go
package handler_test

import (
	"bytes"
	"encoding/json"
	"io"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"testing"

	"taskapi/internal/handler"
	"taskapi/internal/model"
)

// mockStore is a test-only implementation of store.TaskStore.
// Each method is a function field that the test can set.
type mockStore struct {
	createFn func(title string) model.Task
	allFn    func() []model.Task
	getFn    func(id string) (model.Task, bool)
	updateFn func(id, title, status string) (model.Task, bool)
	deleteFn func(id string) bool
}

func (m *mockStore) Create(title string) model.Task          { return m.createFn(title) }
func (m *mockStore) All() []model.Task                       { return m.allFn() }
func (m *mockStore) Get(id string) (model.Task, bool)        { return m.getFn(id) }
func (m *mockStore) Update(id, t, s string) (model.Task, bool) { return m.updateFn(id, t, s) }
func (m *mockStore) Delete(id string) bool                   { return m.deleteFn(id) }
```

### What to Observe

- The mock uses **function fields** — each test sets only the functions it needs. This is the idiomatic Go approach to mocking. No mock framework, no code generation, no `@Mock` annotations.
- The test file is in the `handler_test` package (note the `_test` suffix). This means it can only access exported symbols from the `handler` package, just like any external consumer. This enforces that you test the public API, not implementation details.
- `mockStore` satisfies `store.TaskStore` because it implements all five methods. Go's implicit interface satisfaction means you never write `implements TaskStore`.

---

## Exercise 3 — Test Helper Function (3 min)

**Goal:** Create a reusable helper for constructing test HTTP requests with JSON bodies.

### Instructions

Add this helper to `handler_test.go`:

```go
// newTestServer creates a Server backed by the given mock store.
// It uses a no-op logger to keep test output clean.
func newTestServer(t *testing.T, ms *mockStore) *handler.Server {
	t.Helper()
	logger := slog.New(slog.NewHandler(io.Discard))
	return handler.NewServer(ms, logger)
}

// jsonRequest creates an HTTP request with a JSON body.
func jsonRequest(t *testing.T, method, path string, body any) *http.Request {
	t.Helper()
	var buf bytes.Buffer
	if body != nil {
		if err := json.NewEncoder(&buf).Encode(body); err != nil {
			t.Fatalf("failed to encode request body: %v", err)
		}
	}
	req := httptest.NewRequest(method, path, &buf)
	req.Header.Set("Content-Type", "application/json")
	return req
}

// decodeResponse reads the response body into the given target.
func decodeResponse(t *testing.T, resp *http.Response, target any) {
	t.Helper()
	defer resp.Body.Close()
	if err := json.NewDecoder(resp.Body).Decode(target); err != nil {
		t.Fatalf("failed to decode response: %v", err)
	}
}
```

> **Note:** `slog.NewHandler` is not actually exported. Use this instead for the discarding logger:
```go
logger := slog.New(slog.NewTextHandler(io.Discard, nil))
```

### What to Observe

- **`t.Helper()`** marks these functions as test helpers. When a test fails inside a helper, Go reports the line number of the *calling test*, not the helper — just like JUnit's assertion library. Without `t.Helper()`, the error points to the helper function, which is useless for debugging.
- **`t.Fatalf`** immediately stops the current test. Use it for setup failures where continuing would be meaningless (can't encode a request body = can't run the test).
- These helpers eliminate repetitive boilerplate. Every handler test will use `jsonRequest` and `decodeResponse`.

---

## Exercise 4 — Table-Driven Tests for Task JSON Serialization (5 min)

**Goal:** Verify that the `Task` struct serializes to and from JSON correctly.

### Instructions

Create `internal/model/task_test.go`:

```go
package model_test

import (
	"encoding/json"
	"testing"
	"time"

	"taskapi/internal/model"
)

func TestTaskJSON(t *testing.T) {
	fixedTime := time.Date(2026, 4, 6, 12, 0, 0, 0, time.UTC)

	tests := []struct {
		name     string
		task     model.Task
		wantJSON string
	}{
		{
			name: "full task",
			task: model.Task{
				ID:        "task-1",
				Title:     "Write tests",
				Status:    "pending",
				CreatedAt: fixedTime,
			},
			wantJSON: `{"id":"task-1","title":"Write tests","status":"pending","created_at":"2026-04-06T12:00:00Z"}`,
		},
		{
			name: "empty fields serialize as zero values",
			task: model.Task{},
			wantJSON: `{"id":"","title":"","status":"","created_at":"0001-01-01T00:00:00Z"}`,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name+"/marshal", func(t *testing.T) {
			got, err := json.Marshal(tt.task)
			if err != nil {
				t.Fatalf("Marshal() error: %v", err)
			}
			if string(got) != tt.wantJSON {
				t.Errorf("Marshal()\n  got:  %s\n  want: %s", got, tt.wantJSON)
			}
		})

		t.Run(tt.name+"/unmarshal", func(t *testing.T) {
			var got model.Task
			if err := json.Unmarshal([]byte(tt.wantJSON), &got); err != nil {
				t.Fatalf("Unmarshal() error: %v", err)
			}
			if got != tt.task {
				t.Errorf("Unmarshal()\n  got:  %+v\n  want: %+v", got, tt.task)
			}
		})
	}
}
```

### Run It

```bash
go test -v ./internal/model/
```

### What to Observe

- **Table-driven tests** define all cases in a `[]struct` slice. Adding a new case is adding a struct literal — no new function, no copy-paste. This is the Go equivalent of JUnit's `@ParameterizedTest`, but it uses plain data structures instead of annotations.
- Each test case runs as a named **subtest** via `t.Run()`. You can run a single case: `go test -run TestTaskJSON/full_task/marshal ./internal/model/`.
- The marshal and unmarshal tests are paired: what you encode must decode back to the original value. This round-trip pattern catches struct tag typos and serialization mismatches.

---

## Exercise 5 — Table-Driven Handler Tests with httptest (15 min)

**Goal:** Test the HTTP handlers in isolation using `httptest.NewRecorder`, the mock store, and table-driven tests.

### Instructions

Add the following tests to `internal/handler/handler_test.go`:

### Create Task Handler Tests

```go
func TestHandleCreateTask(t *testing.T) {
	tests := []struct {
		name       string
		body       any
		wantStatus int
		wantError  string
	}{
		{
			name:       "valid task",
			body:       map[string]string{"title": "Test task"},
			wantStatus: http.StatusCreated,
		},
		{
			name:       "missing title",
			body:       map[string]string{},
			wantStatus: http.StatusBadRequest,
			wantError:  "title is required",
		},
		{
			name:       "invalid JSON",
			body:       nil, // we will send raw invalid bytes below
			wantStatus: http.StatusBadRequest,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			ms := &mockStore{
				createFn: func(title string) model.Task {
					return model.Task{
						ID:     "task-1",
						Title:  title,
						Status: "pending",
					}
				},
			}
			srv := handler.NewServer(ms, slog.New(slog.NewTextHandler(io.Discard, nil)))

			var req *http.Request
			if tt.name == "invalid JSON" {
				req = httptest.NewRequest("POST", "/tasks", bytes.NewBufferString("not json"))
				req.Header.Set("Content-Type", "application/json")
			} else {
				req = jsonRequest(t, "POST", "/tasks", tt.body)
			}

			w := httptest.NewRecorder()
			srv.Routes().ServeHTTP(w, req)

			if w.Code != tt.wantStatus {
				t.Errorf("status = %d, want %d", w.Code, tt.wantStatus)
			}

			if tt.wantError != "" {
				var resp map[string]string
				json.NewDecoder(w.Body).Decode(&resp)
				if resp["error"] != tt.wantError {
					t.Errorf("error = %q, want %q", resp["error"], tt.wantError)
				}
			}
		})
	}
}
```

### Get Task Handler Tests

```go
func TestHandleGetTask(t *testing.T) {
	tests := []struct {
		name       string
		taskID     string
		found      bool
		wantStatus int
	}{
		{
			name:       "existing task",
			taskID:     "task-1",
			found:      true,
			wantStatus: http.StatusOK,
		},
		{
			name:       "missing task",
			taskID:     "task-999",
			found:      false,
			wantStatus: http.StatusNotFound,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			ms := &mockStore{
				getFn: func(id string) (model.Task, bool) {
					if !tt.found {
						return model.Task{}, false
					}
					return model.Task{ID: id, Title: "Test", Status: "pending"}, true
				},
			}
			srv := handler.NewServer(ms, slog.New(slog.NewTextHandler(io.Discard, nil)))

			req := httptest.NewRequest("GET", "/tasks/"+tt.taskID, nil)
			w := httptest.NewRecorder()
			srv.Routes().ServeHTTP(w, req)

			if w.Code != tt.wantStatus {
				t.Errorf("status = %d, want %d", w.Code, tt.wantStatus)
			}
		})
	}
}
```

### Delete Task Handler Tests

```go
func TestHandleDeleteTask(t *testing.T) {
	tests := []struct {
		name       string
		taskID     string
		exists     bool
		wantStatus int
	}{
		{
			name:       "delete existing task",
			taskID:     "task-1",
			exists:     true,
			wantStatus: http.StatusNoContent,
		},
		{
			name:       "delete missing task",
			taskID:     "task-999",
			exists:     false,
			wantStatus: http.StatusNotFound,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			ms := &mockStore{
				deleteFn: func(id string) bool {
					return tt.exists
				},
			}
			srv := handler.NewServer(ms, slog.New(slog.NewTextHandler(io.Discard, nil)))

			req := httptest.NewRequest("DELETE", "/tasks/"+tt.taskID, nil)
			w := httptest.NewRecorder()
			srv.Routes().ServeHTTP(w, req)

			if w.Code != tt.wantStatus {
				t.Errorf("status = %d, want %d", w.Code, tt.wantStatus)
			}
		})
	}
}
```

### Run All Tests

```bash
go test -v ./internal/handler/
```

### What to Observe

- **`httptest.NewRecorder()`** creates a fake `http.ResponseWriter` that captures the status code, headers, and body. No real HTTP server is started, no port is opened, no network I/O happens. Tests run in microseconds.
- **`srv.Routes().ServeHTTP(w, req)`** calls the full router stack, including middleware. You are testing the handler exactly as it runs in production — with method matching, path parameter extraction, and logging — not just a bare function.
- Each test creates its **own mock store** with only the methods that test needs. The mock is scoped to the test — no shared mutable state between cases.
- The Java equivalent would require `MockMvc` from Spring Test or `RestAssured`, along with `@MockBean`, `@Autowired`, and a test application context. In Go, it is plain function calls.

---

## Exercise 6 — Test Pollution: Shared State Pitfall (5 min)

**Goal:** Demonstrate how sharing a store between test cases causes tests to depend on execution order.

### Instructions

Add this test to `handler_test.go`:

```go
func TestSharedStatePollution(t *testing.T) {
	// BAD: All subtests share the same store. Order matters.
	sharedStore := &mockStore{
		data: make(map[string]model.Task),
	}

	// To illustrate, let's use a simple inline mock with actual state.
	// This shows why you should NEVER do this.

	store := struct {
		tasks map[string]model.Task
	}{
		tasks: make(map[string]model.Task),
	}

	t.Run("create adds a task", func(t *testing.T) {
		store.tasks["task-1"] = model.Task{ID: "task-1", Title: "First"}
		if len(store.tasks) != 1 {
			t.Errorf("expected 1 task, got %d", len(store.tasks))
		}
	})

	t.Run("list expects empty store but finds leftover data", func(t *testing.T) {
		// This test ASSUMES a clean store, but the previous test mutated it.
		if len(store.tasks) != 0 {
			t.Errorf("POLLUTION: expected 0 tasks, got %d — data leaked from a previous test", len(store.tasks))
		}
	})
}
```

### Run It

```bash
go test -v -run TestSharedStatePollution ./internal/handler/
```

### Expected Output

```
=== RUN   TestSharedStatePollution
=== RUN   TestSharedStatePollution/create_adds_a_task
=== RUN   TestSharedStatePollution/list_expects_empty_store_but_finds_leftover_data
    handler_test.go:XX: POLLUTION: expected 0 tasks, got 1 — data leaked from a previous test
--- FAIL: TestSharedStatePollution (0.00s)
```

### What to Observe

- The second subtest fails because it inherits the state mutated by the first. Subtests within a `t.Run` share the same parent scope — if that scope holds mutable state, tests become order-dependent.
- **The fix:** Create a fresh store (or mock) inside each subtest. This is exactly what our handler tests in Exercise 5 do — each `t.Run` creates its own `mockStore`.
- This is the Go equivalent of JUnit's `@BeforeEach` — except in Go, you just construct a new value. No lifecycle annotations needed.

---

## Exercise 7 — Run with Race Detector, Coverage, and Verbose Output (7 min)

**Goal:** Run the full test suite with all diagnostic flags and interpret the output.

### Run the Full Suite

```bash
go test -v -race -cover ./...
```

This command:
- **`-v`** — Verbose: prints each test name and result.
- **`-race`** — Enables the race detector. Any unsynchronized access to shared memory will be flagged.
- **`-cover`** — Reports code coverage percentage per package.
- **`./...`** — Runs tests in all packages recursively.

### Expected Output (example)

```
=== RUN   TestTaskJSON
=== RUN   TestTaskJSON/full_task/marshal
=== RUN   TestTaskJSON/full_task/unmarshal
=== RUN   TestTaskJSON/empty_fields_serialize_as_zero_values/marshal
=== RUN   TestTaskJSON/empty_fields_serialize_as_zero_values/unmarshal
--- PASS: TestTaskJSON (0.00s)
ok      taskapi/internal/model      0.003s  coverage: 100.0% of statements
=== RUN   TestHandleCreateTask
=== RUN   TestHandleCreateTask/valid_task
=== RUN   TestHandleCreateTask/missing_title
=== RUN   TestHandleCreateTask/invalid_JSON
--- PASS: TestHandleCreateTask (0.00s)
=== RUN   TestHandleGetTask
...
ok      taskapi/internal/handler    0.005s  coverage: 78.4% of statements
?       taskapi/cmd/server          [no test files]
```

### Generate a Coverage Profile

```bash
go test -coverprofile=coverage.out ./...
go tool cover -func=coverage.out
```

This prints line-by-line coverage for every function:

```
taskapi/internal/handler/handler.go:25:   handleHealth        100.0%
taskapi/internal/handler/handler.go:30:   handleCreateTask    100.0%
taskapi/internal/handler/handler.go:48:   handleListTasks     0.0%
taskapi/internal/handler/handler.go:53:   handleGetTask       100.0%
...
total:                                    (statements)        78.4%
```

### View Coverage in the Browser

```bash
go tool cover -html=coverage.out
```

This opens an HTML report with green (covered) and red (uncovered) lines. It makes gaps obvious at a glance.

### What to Observe

- The race detector adds significant overhead but catches concurrency bugs that are invisible without it. In Labs 6–7 you saw it catch a data race on a shared counter — the same tool works on your HTTP handlers. **Add `-race` to your CI pipeline and never remove it.**
- Coverage is a useful metric for finding untested code paths, but 100% coverage does not mean correct code. A handler can have 100% line coverage and still have a logic bug. Coverage tells you what you *haven't* tested, not that what you have tested is right.
- `cmd/server` reports `[no test files]` — that is expected. The main function is a thin wiring layer; test the components it wires, not the wiring itself.

### Stretch Goal: Add Tests for Uncovered Handlers

Check the coverage report. If `handleListTasks` or `handleUpdateTask` show 0% coverage, write table-driven tests for them following the same pattern as Exercises 5. Aim for at least 85% handler coverage.

---

## Wrap-Up Checklist

Before you move on, confirm you can answer these questions:

- [ ] What does the `internal/` directory enforce, and how is it different from Java's `private` modifier?
- [ ] Why should handlers depend on an interface (`TaskStore`) rather than a concrete type (`MemoryStore`)?
- [ ] What is the structure of a table-driven test, and how does it compare to JUnit's `@ParameterizedTest`?
- [ ] What does `t.Helper()` do, and why is it important for test output readability?
- [ ] What is `httptest.NewRecorder()` and how does it differ from starting a real HTTP server?
- [ ] What causes test pollution, and how do you prevent it?
- [ ] What do `-race`, `-cover`, and `-coverprofile` flags do?
- [ ] How does Go's testing approach (no assertion library, no annotations, no test runner configuration) compare to JUnit + Mockito?

---

**End of Lab 9**
