# Lab 10: Capstone — Concurrent Task Processor

**Block 10 — Wrap-Up, Common Pitfalls & Debugging**  
**Duration:** 35 minutes  
**Prerequisites:** Labs 6–9 completed, Go 1.25+ installed  
**Module:** `go mod init capstone`

---

## Learning Objectives

This capstone ties together every concept from Day 1 and Day 2 in a single cohesive project:

| Concept | Where It Appears |
|---|---|
| Structs & JSON struct tags | `BatchRequest`, `TaskItem`, `TaskResult` |
| Interfaces | `Processor` interface for testability |
| Error handling | Per-task errors without crashing the batch |
| Goroutines & channels | Worker pool with fan-out/fan-in |
| `sync.WaitGroup` | Coordinating workers |
| `context.WithTimeout` | Batch-level deadline enforcement |
| `net/http` + `ServeMux` | `POST /process` endpoint |
| `httptest` | Testing the endpoint |
| Table-driven tests | Testing processor logic |
| Race detector | Verifying concurrency safety |
| Project layout | `cmd/` + `internal/` structure |

---

## The Application

You will build an HTTP service with a single endpoint: `POST /process`. It accepts a JSON batch of tasks, processes each one concurrently through a worker pool, and returns the collected results — including per-task errors — as a JSON response. A batch-level timeout ensures the system never hangs indefinitely.

### Request Format

```json
{
  "tasks": [
    {"id": "t1", "type": "uppercase", "payload": "hello world"},
    {"id": "t2", "type": "count_words", "payload": "the quick brown fox"},
    {"id": "t3", "type": "reverse", "payload": "golang"},
    {"id": "t4", "type": "unknown_op", "payload": "will fail"}
  ]
}
```

### Response Format

```json
{
  "results": [
    {"id": "t1", "result": "HELLO WORLD", "error": ""},
    {"id": "t2", "result": "4", "error": ""},
    {"id": "t3", "result": "gnalog", "error": ""},
    {"id": "t4", "result": "", "error": "unsupported task type: unknown_op"}
  ],
  "total": 4,
  "succeeded": 3,
  "failed": 1
}
```

---

## Setup

```bash
mkdir -p capstone/cmd/server capstone/internal/processor capstone/internal/handler
cd capstone
go mod init capstone
```

---

## Step 1 — Define the Domain Types (3 min)

Create `internal/processor/types.go`:

```go
package processor

// TaskItem is a single unit of work in a batch.
type TaskItem struct {
	ID      string `json:"id"`
	Type    string `json:"type"`
	Payload string `json:"payload"`
}

// TaskResult is the outcome of processing a single TaskItem.
type TaskResult struct {
	ID     string `json:"id"`
	Result string `json:"result"`
	Error  string `json:"error"`
}

// BatchRequest is the JSON body for POST /process.
type BatchRequest struct {
	Tasks []TaskItem `json:"tasks"`
}

// BatchResponse is the JSON response from POST /process.
type BatchResponse struct {
	Results   []TaskResult `json:"results"`
	Total     int          `json:"total"`
	Succeeded int          `json:"succeeded"`
	Failed    int          `json:"failed"`
}
```

### What to Observe

- The `Error` field in `TaskResult` is a `string`, not an `error`. The `error` interface does not serialize to JSON cleanly (it would produce `{}`). Converting to a string at the boundary is idiomatic.
- `BatchResponse` includes summary counts (`Total`, `Succeeded`, `Failed`). The caller can check these without iterating the results array.

---

## Step 2 — Build the Processor with a Worker Pool (8 min)

Create `internal/processor/processor.go`:

```go
package processor

import (
	"context"
	"fmt"
	"strings"
	"sync"
)

// Processor defines the interface for processing task batches.
// Handlers depend on this interface, tests can substitute a mock.
type Processor interface {
	ProcessBatch(ctx context.Context, tasks []TaskItem) BatchResponse
}

// WorkerPoolProcessor processes tasks concurrently using a fixed worker pool.
type WorkerPoolProcessor struct {
	NumWorkers int
}

// NewWorkerPoolProcessor creates a processor with the given worker count.
func NewWorkerPoolProcessor(numWorkers int) *WorkerPoolProcessor {
	return &WorkerPoolProcessor{NumWorkers: numWorkers}
}

// ProcessBatch distributes tasks across workers and collects results.
// It respects the context deadline — if the context is cancelled, in-flight
// work finishes but no new tasks are started.
func (p *WorkerPoolProcessor) ProcessBatch(ctx context.Context, tasks []TaskItem) BatchResponse {
	if len(tasks) == 0 {
		return BatchResponse{Results: []TaskResult{}}
	}

	jobs := make(chan TaskItem, len(tasks))
	results := make(chan TaskResult, len(tasks))

	// Launch workers.
	var wg sync.WaitGroup
	for range p.NumWorkers {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for task := range jobs {
				select {
				case <-ctx.Done():
					results <- TaskResult{
						ID:    task.ID,
						Error: "cancelled: " + ctx.Err().Error(),
					}
				default:
					results <- processOne(task)
				}
			}
		}()
	}

	// Enqueue all tasks and close the jobs channel.
	for _, task := range tasks {
		jobs <- task
	}
	close(jobs)

	// Close results channel once all workers finish.
	go func() {
		wg.Wait()
		close(results)
	}()

	// Collect results.
	resp := BatchResponse{
		Results: make([]TaskResult, 0, len(tasks)),
		Total:   len(tasks),
	}
	for r := range results {
		if r.Error == "" {
			resp.Succeeded++
		} else {
			resp.Failed++
		}
		resp.Results = append(resp.Results, r)
	}

	return resp
}

// processOne handles a single task. It returns a result — never panics.
func processOne(task TaskItem) TaskResult {
	var result string
	var err error

	switch task.Type {
	case "uppercase":
		result = strings.ToUpper(task.Payload)
	case "reverse":
		result = reverseString(task.Payload)
	case "count_words":
		words := strings.Fields(task.Payload)
		result = fmt.Sprintf("%d", len(words))
	default:
		err = fmt.Errorf("unsupported task type: %s", task.Type)
	}

	tr := TaskResult{ID: task.ID, Result: result}
	if err != nil {
		tr.Error = err.Error()
	}
	return tr
}

// reverseString reverses a string by runes (handles multi-byte UTF-8 correctly).
func reverseString(s string) string {
	runes := []rune(s)
	for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
		runes[i], runes[j] = runes[j], runes[i]
	}
	return string(runes)
}
```

### What to Observe

- **Worker pool pattern:** A fixed number of goroutines read from a shared `jobs` channel. This is the fan-out pattern from Lab 6, now applied to real work.
- **Context checking:** Each worker checks `ctx.Done()` before processing. If the batch deadline has passed, the worker marks the task as cancelled instead of doing unnecessary work. In-flight work is not interrupted — Go does not have thread interruption. The context check happens between tasks, not during them.
- **`processOne` never panics.** Unknown task types return an error, not a crash. This is idiomatic Go error handling — errors are values, not exceptions. A single bad task does not take down the batch.
- **`reverseString` operates on runes**, not bytes. This correctly handles multi-byte UTF-8 characters. Reversing a `[]byte` would split multi-byte characters and produce garbage.
- **The `Processor` interface** has a single method. This follows the Go principle of small interfaces — and makes the handler testable with a mock.

---

## Step 3 — Build the HTTP Handler (5 min)

Create `internal/handler/handler.go`:

```go
package handler

import (
	"context"
	"encoding/json"
	"log/slog"
	"net/http"
	"time"

	"capstone/internal/processor"
)

// Server holds dependencies for the HTTP handler.
type Server struct {
	Proc    processor.Processor
	Logger  *slog.Logger
	Timeout time.Duration
}

// NewServer creates a Server with the given dependencies.
func NewServer(proc processor.Processor, logger *slog.Logger, timeout time.Duration) *Server {
	return &Server{Proc: proc, Logger: logger, Timeout: timeout}
}

// Routes returns the HTTP handler with all routes registered.
func (s *Server) Routes() http.Handler {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /health", s.handleHealth)
	mux.HandleFunc("POST /process", s.handleProcess)
	return mux
}

func (s *Server) handleHealth(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusOK, map[string]string{"status": "ok"})
}

func (s *Server) handleProcess(w http.ResponseWriter, r *http.Request) {
	var req processor.BatchRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		writeError(w, http.StatusBadRequest, "invalid JSON: "+err.Error())
		return
	}

	if len(req.Tasks) == 0 {
		writeError(w, http.StatusBadRequest, "tasks array is required and must not be empty")
		return
	}

	// Enforce a batch-level timeout.
	ctx, cancel := context.WithTimeout(r.Context(), s.Timeout)
	defer cancel()

	s.Logger.Info("processing batch",
		"task_count", len(req.Tasks),
		"timeout", s.Timeout.String(),
	)

	resp := s.Proc.ProcessBatch(ctx, req.Tasks)

	s.Logger.Info("batch complete",
		"total", resp.Total,
		"succeeded", resp.Succeeded,
		"failed", resp.Failed,
	)

	writeJSON(w, http.StatusOK, resp)
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

### What to Observe

- **`context.WithTimeout(r.Context(), s.Timeout)`** derives a child context from the request's context with an added deadline. If the HTTP client disconnects, `r.Context()` is cancelled automatically — and the timeout ensures the batch does not run forever regardless.
- **`defer cancel()`** is required. Even if the timeout fires on its own, `cancel` must be called to release resources. The Go vet tool will warn you if you forget.
- The handler is clean: parse input → validate → call processor → return output. No concurrency logic leaks into the handler — that is the processor's responsibility.

---

## Step 4 — Wire It Up (3 min)

Create `cmd/server/main.go`:

```go
package main

import (
	"log/slog"
	"net/http"
	"os"
	"time"

	"capstone/internal/handler"
	"capstone/internal/processor"
)

func main() {
	logger := slog.New(slog.NewTextHandler(os.Stdout, nil))

	proc := processor.NewWorkerPoolProcessor(5)
	srv := handler.NewServer(proc, logger, 10*time.Second)

	logger.Info("capstone server starting", "addr", ":8080", "workers", 5)
	if err := http.ListenAndServe(":8080", srv.Routes()); err != nil {
		logger.Error("server failed", "error", err)
		os.Exit(1)
	}
}
```

### Verify

```bash
go run ./cmd/server
```

In another terminal:

```bash
curl -s -X POST http://localhost:8080/process \
  -H "Content-Type: application/json" \
  -d '{
    "tasks": [
      {"id": "t1", "type": "uppercase", "payload": "hello world"},
      {"id": "t2", "type": "count_words", "payload": "the quick brown fox"},
      {"id": "t3", "type": "reverse", "payload": "golang"},
      {"id": "t4", "type": "unknown_op", "payload": "will fail"}
    ]
  }' | jq .
```

Expected output:

```json
{
  "results": [
    {"id": "t1", "result": "HELLO WORLD", "error": ""},
    {"id": "t2", "result": "4", "error": ""},
    {"id": "t3", "result": "gnalog", "error": ""},
    {"id": "t4", "result": "", "error": "unsupported task type: unknown_op"}
  ],
  "total": 4,
  "succeeded": 3,
  "failed": 1
}
```

Note that result ordering may vary between runs — the worker pool processes tasks concurrently, so whichever worker finishes first sends its result first. The IDs let the caller correlate results to requests regardless of order.

---

## Step 5 — Write Table-Driven Tests (8 min)

Create `internal/processor/processor_test.go`:

```go
package processor_test

import (
	"context"
	"testing"
	"time"

	"capstone/internal/processor"
)

func TestProcessOne(t *testing.T) {
	tests := []struct {
		name       string
		task       processor.TaskItem
		wantResult string
		wantError  string
	}{
		{
			name:       "uppercase",
			task:       processor.TaskItem{ID: "1", Type: "uppercase", Payload: "hello"},
			wantResult: "HELLO",
		},
		{
			name:       "uppercase with spaces",
			task:       processor.TaskItem{ID: "2", Type: "uppercase", Payload: "go is fun"},
			wantResult: "GO IS FUN",
		},
		{
			name:       "reverse ASCII",
			task:       processor.TaskItem{ID: "3", Type: "reverse", Payload: "abcdef"},
			wantResult: "fedcba",
		},
		{
			name:       "reverse UTF-8",
			task:       processor.TaskItem{ID: "4", Type: "reverse", Payload: "héllo"},
			wantResult: "olléh",
		},
		{
			name:       "count words",
			task:       processor.TaskItem{ID: "5", Type: "count_words", Payload: "one two three"},
			wantResult: "3",
		},
		{
			name:       "count words with extra whitespace",
			task:       processor.TaskItem{ID: "6", Type: "count_words", Payload: "  spaced   out  "},
			wantResult: "2",
		},
		{
			name:       "empty payload",
			task:       processor.TaskItem{ID: "7", Type: "count_words", Payload: ""},
			wantResult: "0",
		},
		{
			name:      "unsupported type returns error",
			task:      processor.TaskItem{ID: "8", Type: "bad_type", Payload: "data"},
			wantError: "unsupported task type: bad_type",
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			proc := processor.NewWorkerPoolProcessor(1)
			resp := proc.ProcessBatch(
				context.Background(),
				[]processor.TaskItem{tt.task},
			)

			if len(resp.Results) != 1 {
				t.Fatalf("expected 1 result, got %d", len(resp.Results))
			}

			got := resp.Results[0]
			if got.Result != tt.wantResult {
				t.Errorf("result = %q, want %q", got.Result, tt.wantResult)
			}
			if got.Error != tt.wantError {
				t.Errorf("error = %q, want %q", got.Error, tt.wantError)
			}
		})
	}
}

func TestProcessBatchCounts(t *testing.T) {
	tasks := []processor.TaskItem{
		{ID: "1", Type: "uppercase", Payload: "hello"},
		{ID: "2", Type: "reverse", Payload: "world"},
		{ID: "3", Type: "invalid", Payload: "x"},
		{ID: "4", Type: "count_words", Payload: "a b c"},
	}

	proc := processor.NewWorkerPoolProcessor(2)
	resp := proc.ProcessBatch(context.Background(), tasks)

	if resp.Total != 4 {
		t.Errorf("total = %d, want 4", resp.Total)
	}
	if resp.Succeeded != 3 {
		t.Errorf("succeeded = %d, want 3", resp.Succeeded)
	}
	if resp.Failed != 1 {
		t.Errorf("failed = %d, want 1", resp.Failed)
	}
}

func TestProcessBatchTimeout(t *testing.T) {
	// Create a context that is already expired.
	ctx, cancel := context.WithTimeout(context.Background(), 1*time.Nanosecond)
	defer cancel()

	// Give it a moment to actually expire.
	time.Sleep(1 * time.Millisecond)

	tasks := []processor.TaskItem{
		{ID: "1", Type: "uppercase", Payload: "hello"},
		{ID: "2", Type: "reverse", Payload: "world"},
	}

	proc := processor.NewWorkerPoolProcessor(2)
	resp := proc.ProcessBatch(ctx, tasks)

	// All tasks should report cancellation.
	for _, r := range resp.Results {
		if r.Error == "" {
			t.Errorf("task %s: expected cancellation error, got success", r.ID)
		}
	}
	if resp.Failed != 2 {
		t.Errorf("failed = %d, want 2", resp.Failed)
	}
}

func TestProcessBatchEmpty(t *testing.T) {
	proc := processor.NewWorkerPoolProcessor(2)
	resp := proc.ProcessBatch(context.Background(), []processor.TaskItem{})

	if resp.Total != 0 {
		t.Errorf("total = %d, want 0", resp.Total)
	}
	if resp.Results == nil {
		t.Error("results should be non-nil empty slice, got nil")
	}
}
```

### What to Observe

- **`TestProcessOne`** is a classic table-driven test. Eight cases, one loop, zero copy-paste. Adding a new task type means adding one struct literal.
- **`TestProcessBatchCounts`** tests the batch-level aggregation — a mixed batch with 3 successes and 1 failure should produce the correct summary counts.
- **`TestProcessBatchTimeout`** creates an already-expired context to verify that the processor respects deadlines. This is a deterministic test — no flaky timing dependencies.
- **`TestProcessBatchEmpty`** verifies the edge case: an empty batch returns an empty (but non-nil) results slice. `json.Marshal` of a `nil` slice produces `null`; a non-nil empty slice produces `[]`. This distinction matters for API consumers.
- The `processOne` function is unexported (lowercase). We test it indirectly through `ProcessBatch`. This is deliberate: we test the public API, not internal implementation. If `processOne` is refactored or renamed, no tests break.

---

## Step 6 — Test the HTTP Endpoint (5 min)

Create `internal/handler/handler_test.go`:

```go
package handler_test

import (
	"bytes"
	"context"
	"encoding/json"
	"io"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"testing"
	"time"

	"capstone/internal/handler"
	"capstone/internal/processor"
)

// mockProcessor is a test double for processor.Processor.
type mockProcessor struct {
	processBatchFn func(ctx context.Context, tasks []processor.TaskItem) processor.BatchResponse
}

func (m *mockProcessor) ProcessBatch(ctx context.Context, tasks []processor.TaskItem) processor.BatchResponse {
	return m.processBatchFn(ctx, tasks)
}

func TestHandleProcess(t *testing.T) {
	tests := []struct {
		name       string
		body       string
		wantStatus int
		wantTotal  int
	}{
		{
			name:       "valid batch",
			body:       `{"tasks":[{"id":"1","type":"uppercase","payload":"hi"}]}`,
			wantStatus: http.StatusOK,
			wantTotal:  1,
		},
		{
			name:       "invalid JSON",
			body:       `not json`,
			wantStatus: http.StatusBadRequest,
		},
		{
			name:       "empty tasks array",
			body:       `{"tasks":[]}`,
			wantStatus: http.StatusBadRequest,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			mock := &mockProcessor{
				processBatchFn: func(_ context.Context, tasks []processor.TaskItem) processor.BatchResponse {
					results := make([]processor.TaskResult, len(tasks))
					for i, task := range tasks {
						results[i] = processor.TaskResult{
							ID:     task.ID,
							Result: "processed",
						}
					}
					return processor.BatchResponse{
						Results:   results,
						Total:     len(tasks),
						Succeeded: len(tasks),
					}
				},
			}

			logger := slog.New(slog.NewTextHandler(io.Discard, nil))
			srv := handler.NewServer(mock, logger, 5*time.Second)

			req := httptest.NewRequest("POST", "/process", bytes.NewBufferString(tt.body))
			req.Header.Set("Content-Type", "application/json")
			w := httptest.NewRecorder()

			srv.Routes().ServeHTTP(w, req)

			if w.Code != tt.wantStatus {
				t.Errorf("status = %d, want %d", w.Code, tt.wantStatus)
			}

			if tt.wantStatus == http.StatusOK {
				var resp processor.BatchResponse
				json.NewDecoder(w.Body).Decode(&resp)
				if resp.Total != tt.wantTotal {
					t.Errorf("total = %d, want %d", resp.Total, tt.wantTotal)
				}
			}
		})
	}
}
```

---

## Step 7 — Run with Race Detector (3 min)

This is the final verification. The race detector confirms that your concurrent code has no data races.

```bash
go test -v -race -cover ./...
```

### Expected Output

```
=== RUN   TestProcessOne
=== RUN   TestProcessOne/uppercase
=== RUN   TestProcessOne/uppercase_with_spaces
=== RUN   TestProcessOne/reverse_ASCII
=== RUN   TestProcessOne/reverse_UTF-8
=== RUN   TestProcessOne/count_words
=== RUN   TestProcessOne/count_words_with_extra_whitespace
=== RUN   TestProcessOne/empty_payload
=== RUN   TestProcessOne/unsupported_type_returns_error
--- PASS: TestProcessOne (0.00s)
=== RUN   TestProcessBatchCounts
--- PASS: TestProcessBatchCounts (0.00s)
=== RUN   TestProcessBatchTimeout
--- PASS: TestProcessBatchTimeout (0.00s)
=== RUN   TestProcessBatchEmpty
--- PASS: TestProcessBatchEmpty (0.00s)
ok      capstone/internal/processor    0.005s  coverage: 95.0% of statements
=== RUN   TestHandleProcess
=== RUN   TestHandleProcess/valid_batch
=== RUN   TestHandleProcess/invalid_JSON
=== RUN   TestHandleProcess/empty_tasks_array
--- PASS: TestHandleProcess (0.00s)
ok      capstone/internal/handler      0.003s  coverage: 82.0% of statements
```

**No race conditions detected.** Your concurrent task processor is safe.

### What the Race Detector Verifies Here

- Multiple worker goroutines writing to the `results` channel concurrently
- The main goroutine reading from `results` while workers are still sending
- The `WaitGroup` coordination between workers and the channel closer
- Context cancellation racing with task processing

If any of these interactions had a data race, `-race` would catch it. This is why every CI pipeline for Go should include `-race`.

---

## Final Project Structure

```
capstone/
├── cmd/
│   └── server/
│       └── main.go                  (15 lines)
├── internal/
│   ├── processor/
│   │   ├── types.go                 (domain types)
│   │   ├── processor.go             (worker pool logic)
│   │   └── processor_test.go        (table-driven tests)
│   └── handler/
│       ├── handler.go               (HTTP handler)
│       └── handler_test.go          (endpoint tests)
├── go.mod
└── go.sum
```

**Total lines of code:** approximately 350, including tests.  
**External dependencies:** zero.  
**Concepts used:** structs, interfaces, JSON encoding, error handling, goroutines, channels, `WaitGroup`, `context`, `net/http`, `ServeMux` routing, `httptest`, `slog`, table-driven tests, race detection.

---

## Concepts Recap — Everything You Used

Take a moment to reflect on how far you have come from "Hello, World" in Lab 1:

1. **Structs & struct tags** (Lab 3) — `TaskItem`, `TaskResult`, `BatchRequest`, `BatchResponse` with JSON tags
2. **Interfaces** (Lab 4) — `Processor` interface for testability; mock in tests
3. **Error handling** (Lab 5) — per-task errors as values, no panics, no exceptions
4. **Goroutines & channels** (Lab 6) — worker pool, fan-out/fan-in, channel closing
5. **`sync.WaitGroup`** (Lab 7) — coordinating worker completion
6. **`context.Context`** (Lab 7) — batch-level deadline, cancellation propagation
7. **`net/http` + `ServeMux`** (Lab 8) — method-matched routing, JSON parsing, error responses
8. **`slog`** (Lab 8) — structured logging with key-value pairs
9. **`httptest`** (Lab 9) — testing handlers without a real server
10. **Table-driven tests** (Lab 9) — eight test cases, one loop
11. **Project layout** (Lab 9) — `cmd/`, `internal/`, interface boundaries
12. **Race detector** (Lab 7, Lab 9) — verifying concurrency safety

Every one of these is a standard library feature or a built-in Go tool. You imported zero frameworks and zero third-party dependencies to build a concurrent, tested, production-style HTTP service.

---

## Where to Go Next

Now that you have the foundations, here are recommended next steps:

- **Read** [Effective Go](https://go.dev/doc/effective_go) — the definitive style guide
- **Bookmark** [Go Proverbs](https://go-proverbs.github.io) — design wisdom in one-liners
- **Follow** [The Go Blog](https://go.dev/blog) — especially the posts on slices, error handling, and concurrency
- **Explore next:** generics (type parameters), `database/sql` for database access, gRPC with Protocol Buffers, advanced testing with fuzz testing (`testing.F`), and the `context` package in depth

---

## Wrap-Up Checklist

- [ ] Can you explain how the worker pool pattern distributes tasks across goroutines?
- [ ] Why does `processOne` return an error string instead of panicking?
- [ ] How does `context.WithTimeout` prevent the batch from running forever?
- [ ] Why is `Processor` an interface with a single method?
- [ ] How do the handler tests use a mock processor to avoid testing concurrency logic in the HTTP layer?
- [ ] What does `go test -race` verify in this project?
- [ ] Could you add a new task type (e.g., `"sha256"`) and a corresponding test case without modifying any existing test code?

---

**End of Lab 10 — End of Course**
