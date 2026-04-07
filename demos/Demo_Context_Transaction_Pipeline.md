# Demo: Context in Go — A Transaction Enrichment Pipeline

**Audience:** Java developers transitioning to Go  
**Duration:** ~12–15 minutes live coding  
**Concepts:** `context.Background`, `context.WithTimeout`, `context.WithCancel`, `context.WithValue`, cancellation propagation, `ctx.Done()`, `ctx.Err()`, deadline budgeting  
**Java parallel:** `ExecutorService.invokeAll` with timeout, `Future.cancel`, `CompletableFuture` cancellation, `ThreadLocal` for request-scoped data, `@RequestScope` beans

---

## The Scenario

You are building a **transaction enrichment service** at a payment processor. When a transaction arrives, it must be enriched by calling three downstream services before a response can be sent:

1. **Fraud Check** — Is this transaction suspicious? (~200 ms)
2. **Currency Conversion** — What is the amount in the settlement currency? (~150 ms)
3. **Compliance Screening** — Is this entity on a sanctions list? (~300 ms)

The business rule: **the entire enrichment must complete within 2 seconds.** If it does not, return a partial result with whatever finished. If the client disconnects, stop all work immediately — do not waste resources on a response nobody will read.

In Java, you would use `ExecutorService.invokeAll(tasks, 2, TimeUnit.SECONDS)` or wire up `CompletableFuture` chains with `.orTimeout()`. In Go, you use `context.Context` — and every function in the chain receives the same cancellation signal automatically.

This demo progresses through four stages:

1. **`context.WithTimeout`** — Enforce a deadline on the entire pipeline
2. **Cascading cancellation** — Cancel one service and watch the others stop
3. **`context.WithValue`** — Thread a request ID through the pipeline without function parameters
4. **The full pipeline** — All three services racing against a deadline, with structured logging and graceful degradation

---

## Setup

```bash
mkdir -p context-demo && cd context-demo
go mod init context-demo
```

Everything runs in a single `main.go`.

---

## Part 0 — Simulate Downstream Services

> **Instructor talking point:** "Each service is a function that accepts a `context.Context` as its first parameter. This is not optional — it is the number one convention in Go. Every function that does I/O, every function that might block, every function that might be cancelled: `ctx context.Context` is the first parameter."

```go
package main

import (
	"context"
	"fmt"
	"math/rand/v2"
	"sync"
	"time"
)

// --- Simulated downstream services ---

// ServiceResult holds the outcome of a single service call.
type ServiceResult struct {
	Service string
	Data    string
	Err     error
	Elapsed time.Duration
}

// fraudCheck simulates a fraud detection call (100–400 ms).
func fraudCheck(ctx context.Context, txID string) ServiceResult {
	start := time.Now()
	delay := time.Duration(100+rand.IntN(300)) * time.Millisecond

	select {
	case <-time.After(delay):
		score := rand.Float64()
		return ServiceResult{
			Service: "fraud-check",
			Data:    fmt.Sprintf("tx %s: risk score %.2f — APPROVED", txID, score),
			Elapsed: time.Since(start),
		}
	case <-ctx.Done():
		return ServiceResult{
			Service: "fraud-check",
			Err:     fmt.Errorf("fraud check cancelled: %w", ctx.Err()),
			Elapsed: time.Since(start),
		}
	}
}

// currencyConvert simulates a currency conversion call (50–300 ms).
func currencyConvert(ctx context.Context, amount float64, from, to string) ServiceResult {
	start := time.Now()
	delay := time.Duration(50+rand.IntN(250)) * time.Millisecond

	select {
	case <-time.After(delay):
		rate := 0.85 + rand.Float64()*0.05
		converted := amount * rate
		return ServiceResult{
			Service: "currency-convert",
			Data:    fmt.Sprintf("%.2f %s → %.2f %s (rate: %.4f)", amount, from, converted, to, rate),
			Elapsed: time.Since(start),
		}
	case <-ctx.Done():
		return ServiceResult{
			Service: "currency-convert",
			Err:     fmt.Errorf("currency conversion cancelled: %w", ctx.Err()),
			Elapsed: time.Since(start),
		}
	}
}

// complianceScreen simulates a sanctions/compliance check (200–800 ms).
// This one is deliberately slow — it will sometimes exceed the deadline.
func complianceScreen(ctx context.Context, entityName string) ServiceResult {
	start := time.Now()
	delay := time.Duration(200+rand.IntN(600)) * time.Millisecond

	select {
	case <-time.After(delay):
		return ServiceResult{
			Service: "compliance-screen",
			Data:    fmt.Sprintf("entity %q: CLEAR — no sanctions match", entityName),
			Elapsed: time.Since(start),
		}
	case <-ctx.Done():
		return ServiceResult{
			Service: "compliance-screen",
			Err:     fmt.Errorf("compliance screening cancelled: %w", ctx.Err()),
			Elapsed: time.Since(start),
		}
	}
}
```

> **Instructor talking point:** "Look at the `select` statement in each service. It waits for *either* the work to finish (`time.After`) *or* the context to be cancelled (`ctx.Done()`). Whichever happens first wins. This is cooperative cancellation — Go cannot kill a goroutine from the outside, but the goroutine checks `ctx.Done()` and exits voluntarily. This is fundamentally different from Java's `Thread.interrupt()`, which is imperative and often ignored."

---

## Part 1 — `context.WithTimeout`: The Deadline

> **Instructor talking point:** "In Java, you'd pass a timeout to `Future.get(2, TimeUnit.SECONDS)`. In Go, you create a context with a deadline and pass it to every function. The context *is* the deadline — it flows through the entire call chain."

```go
func demoTimeout() {
	fmt.Println("================================================")
	fmt.Println("PART 1: context.WithTimeout — Enforcing a Deadline")
	fmt.Println("================================================")
	fmt.Println()

	// Create a context that will automatically cancel after 500ms.
	// This is deliberately short so that the compliance service
	// (200–800ms) sometimes times out.
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel() // ALWAYS defer cancel to release resources.

	// Show the deadline.
	deadline, _ := ctx.Deadline()
	fmt.Printf("  Deadline set: %v from now\n", time.Until(deadline).Round(time.Millisecond))
	fmt.Println("  Calling compliance screening (200–800ms range)...")
	fmt.Println()

	result := complianceScreen(ctx, "Acme Corp")

	if result.Err != nil {
		fmt.Printf("  ❌ %s FAILED after %v\n", result.Service, result.Elapsed.Round(time.Millisecond))
		fmt.Printf("     Error: %v\n", result.Err)
		fmt.Println()
		fmt.Println("  The context expired. ctx.Err() tells you WHY:")
		fmt.Printf("     ctx.Err() = %v\n", ctx.Err())
		fmt.Println()
		fmt.Println("  context.DeadlineExceeded means the timeout fired.")
		fmt.Println("  context.Canceled means someone called cancel() explicitly.")
	} else {
		fmt.Printf("  ✅ %s completed in %v\n", result.Service, result.Elapsed.Round(time.Millisecond))
		fmt.Printf("     Result: %s\n", result.Data)
	}
	fmt.Println()
}
```

> **Instructor talking point:** Run this 3–4 times. Sometimes compliance finishes in time, sometimes it times out. The nondeterminism is the point — real networks are unpredictable, and your code must handle both outcomes. Point out `defer cancel()` and explain: "Even if the timeout fires on its own, you must call cancel to release the internal timer. Forgetting this leaks resources on every request."

---

## Part 2 — Cascading Cancellation: One Signal, Many Goroutines

> **Instructor talking point:** "Here's where context gets powerful. You create ONE context, pass it to THREE goroutines, and when the deadline expires, ALL THREE receive the cancellation signal simultaneously. No bookkeeping, no `Future.cancel()` on each task, no `ExecutorService.shutdownNow()`. The context tree handles it."

```go
func demoCascadingCancellation() {
	fmt.Println("================================================")
	fmt.Println("PART 2: Cascading Cancellation — One Signal, All Stop")
	fmt.Println("================================================")
	fmt.Println()

	// 400ms deadline — some services will finish, some may not.
	ctx, cancel := context.WithTimeout(context.Background(), 400*time.Millisecond)
	defer cancel()

	fmt.Println("  Launching 3 services with a 400ms deadline...")
	fmt.Println()

	results := make(chan ServiceResult, 3)

	// Launch all three concurrently. They all share the same context.
	go func() { results <- fraudCheck(ctx, "tx-7890") }()
	go func() { results <- currencyConvert(ctx, 1250.00, "USD", "EUR") }()
	go func() { results <- complianceScreen(ctx, "GlobalTrade Ltd") }()

	// Collect all three results.
	for range 3 {
		r := <-results
		if r.Err != nil {
			fmt.Printf("  ❌ %-20s CANCELLED after %v — %v\n",
				r.Service, r.Elapsed.Round(time.Millisecond), r.Err)
		} else {
			fmt.Printf("  ✅ %-20s completed in %v — %s\n",
				r.Service, r.Elapsed.Round(time.Millisecond), r.Data)
		}
	}

	fmt.Println()
	fmt.Println("  Key insight: you created ONE context and passed it to THREE goroutines.")
	fmt.Println("  When the deadline fired, all three received the signal via ctx.Done().")
	fmt.Println("  No manual tracking. No thread interrupts. The context IS the coordination.")
	fmt.Println()
}
```

> **Instructor talking point:** Run this a few times. The fast services (fraud, currency) usually succeed. The slow one (compliance) often gets cancelled. Point out that the cancelled service returns *immediately* — it does not wait for its simulated delay to expire. That's because `select` picks whichever case is ready first.

---

## Part 3 — `context.WithValue`: Request-Scoped Data

> **Instructor talking point:** "In Java, you'd use `ThreadLocal` or Spring's `@RequestScope` to carry a request ID through the call chain. In Go, you attach it to the context. Every function already receives the context — so no additional plumbing is needed. But there's a rule: context values are for *request-scoped* metadata only — not for passing business data. Think trace IDs, auth tokens, correlation IDs."

```go
// contextKey is an unexported type to prevent key collisions.
// NEVER use built-in types (string, int) as context keys — they collide
// across packages. This pattern guarantees uniqueness.
type contextKey string

const requestIDKey contextKey = "requestID"

// withRequestID attaches a request ID to the context.
func withRequestID(ctx context.Context, id string) context.Context {
	return context.WithValue(ctx, requestIDKey, id)
}

// getRequestID extracts the request ID from the context.
func getRequestID(ctx context.Context) string {
	if id, ok := ctx.Value(requestIDKey).(string); ok {
		return id
	}
	return "unknown"
}

func demoContextValues() {
	fmt.Println("================================================")
	fmt.Println("PART 3: context.WithValue — Request-Scoped Data")
	fmt.Println("================================================")
	fmt.Println()

	// Simulate an incoming HTTP request: create context with timeout AND request ID.
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	ctx = withRequestID(ctx, "req-abc-42")

	// The request ID flows through the entire pipeline without
	// adding a parameter to every function signature.
	fmt.Printf("  Request ID from context: %s\n", getRequestID(ctx))
	fmt.Println()

	// Simulate a service call that logs the request ID from the context.
	enrichedLog := func(ctx context.Context, service, message string) {
		fmt.Printf("  [%s] [%s] %s\n", getRequestID(ctx), service, message)
	}

	enrichedLog(ctx, "fraud-check", "starting risk assessment")
	enrichedLog(ctx, "currency-convert", "fetching EUR/USD rate")
	enrichedLog(ctx, "compliance-screen", "screening against OFAC list")

	fmt.Println()
	fmt.Println("  The request ID was attached ONCE at the top of the call chain.")
	fmt.Println("  Every function extracted it from ctx — no extra parameter needed.")
	fmt.Println()
	fmt.Println("  Java equivalent: ThreadLocal<String> or MDC.put(\"requestId\", id)")
	fmt.Println("  Go advantage:    context values are immutable and propagate through")
	fmt.Println("                   goroutines — ThreadLocal does NOT cross thread boundaries.")
	fmt.Println()
}
```

> **Instructor talking point:** Emphasize the custom key type (`contextKey`). Using a bare `string` as a key is a bug waiting to happen — two unrelated packages might both use `"requestID"` and silently overwrite each other. The unexported custom type makes collisions impossible.

---

## Part 4 — The Full Pipeline: Graceful Degradation Under Pressure

> **Instructor talking point:** "This is the production pattern. An HTTP request arrives. We create a context with a deadline. We fan out to three services. We collect whatever finishes in time, and we return a partial result if the deadline fires. The client gets the best answer we can give within the time budget."

```go
// EnrichmentResult is the aggregated response from all services.
type EnrichmentResult struct {
	RequestID string
	Succeeded []ServiceResult
	Failed    []ServiceResult
	TimedOut  bool
	TotalTime time.Duration
}

func enrichTransaction(ctx context.Context) EnrichmentResult {
	start := time.Now()
	reqID := getRequestID(ctx)

	results := make(chan ServiceResult, 3)

	// Fan out: launch all three services with the same context.
	go func() { results <- fraudCheck(ctx, "tx-"+reqID) }()
	go func() { results <- currencyConvert(ctx, 1250.00, "USD", "EUR") }()
	go func() { results <- complianceScreen(ctx, "GlobalTrade Ltd") }()

	var succeeded, failed []ServiceResult
	collected := 0

	// Collect results until all three finish or the context expires.
	for collected < 3 {
		select {
		case r := <-results:
			collected++
			if r.Err != nil {
				failed = append(failed, r)
			} else {
				succeeded = append(succeeded, r)
			}

		case <-ctx.Done():
			// Deadline fired. Drain whatever is already in the channel.
			draining := true
			for draining {
				select {
				case r := <-results:
					collected++
					if r.Err != nil {
						failed = append(failed, r)
					} else {
						succeeded = append(succeeded, r)
					}
				default:
					draining = false
				}
			}

			// Mark remaining uncollected services as timed out.
			for collected < 3 {
				failed = append(failed, ServiceResult{
					Service: "(uncollected)",
					Err:     ctx.Err(),
				})
				collected++
			}

			return EnrichmentResult{
				RequestID: reqID,
				Succeeded: succeeded,
				Failed:    failed,
				TimedOut:  true,
				TotalTime: time.Since(start),
			}
		}
	}

	return EnrichmentResult{
		RequestID: reqID,
		Succeeded: succeeded,
		Failed:    failed,
		TimedOut:  false,
		TotalTime: time.Since(start),
	}
}

func demoFullPipeline() {
	fmt.Println("================================================")
	fmt.Println("PART 4: Full Pipeline — Graceful Degradation")
	fmt.Println("================================================")
	fmt.Println()

	// Run the pipeline 3 times with different deadlines.
	deadlines := []struct {
		label   string
		timeout time.Duration
	}{
		{"generous (2s)", 2 * time.Second},
		{"tight (300ms)", 300 * time.Millisecond},
		{"impossible (50ms)", 50 * time.Millisecond},
	}

	for _, d := range deadlines {
		fmt.Printf("  --- Deadline: %s ---\n", d.label)

		ctx, cancel := context.WithTimeout(context.Background(), d.timeout)
		ctx = withRequestID(ctx, fmt.Sprintf("req-%d", time.Now().UnixMilli()%10000))

		result := enrichTransaction(ctx)
		cancel()

		fmt.Printf("  Request: %s | Total: %v | Timed out: %t\n",
			result.RequestID,
			result.TotalTime.Round(time.Millisecond),
			result.TimedOut)

		for _, r := range result.Succeeded {
			fmt.Printf("    ✅ %-20s %v — %s\n",
				r.Service, r.Elapsed.Round(time.Millisecond), r.Data)
		}
		for _, r := range result.Failed {
			fmt.Printf("    ❌ %-20s %v — %v\n",
				r.Service, r.Elapsed.Round(time.Millisecond), r.Err)
		}
		fmt.Println()
	}
}
```

---

## Part 5 — Bonus: `context.WithCancel` — Simulating Client Disconnect

> **Instructor talking point:** "What happens when a user closes their browser mid-request? In Go's `net/http`, the request's context is cancelled automatically. Let's simulate that — we'll cancel the context manually after 200ms, as if the client hung up."

```go
func demoClientDisconnect() {
	fmt.Println("================================================")
	fmt.Println("BONUS: Client Disconnect — context.WithCancel")
	fmt.Println("================================================")
	fmt.Println()

	ctx, cancel := context.WithCancel(context.Background())
	ctx = withRequestID(ctx, "req-disconnect")

	// Simulate client hanging up after 200ms.
	var wg sync.WaitGroup
	wg.Add(1)
	go func() {
		defer wg.Done()
		time.Sleep(200 * time.Millisecond)
		fmt.Println("  📱 Client disconnected! Calling cancel()...")
		cancel()
	}()

	fmt.Println("  Starting enrichment (client will disconnect in 200ms)...")
	fmt.Println()

	result := enrichTransaction(ctx)

	wg.Wait()

	fmt.Println()
	fmt.Printf("  Request: %s | Total: %v | Timed out: %t\n",
		result.RequestID,
		result.TotalTime.Round(time.Millisecond),
		result.TimedOut)
	fmt.Printf("  ctx.Err() = %v\n", ctx.Err())
	fmt.Println()
	fmt.Println("  Notice: ctx.Err() is context.Canceled, not DeadlineExceeded.")
	fmt.Println("  In production, you'd check this to distinguish between:")
	fmt.Println("    - context.DeadlineExceeded → log as timeout, maybe retry")
	fmt.Println("    - context.Canceled         → client left, don't waste resources")
	fmt.Println()
}
```

---

## Part 6 — The `main` Function

```go
func main() {
	demoTimeout()
	demoCascadingCancellation()
	demoContextValues()
	demoFullPipeline()
	demoClientDisconnect()

	fmt.Println("================================================")
	fmt.Println("KEY TAKEAWAYS")
	fmt.Println("================================================")
	fmt.Println()
	fmt.Println("  context.Context is Go's answer to four Java problems at once:")
	fmt.Println()
	fmt.Println("    1. TIMEOUTS")
	fmt.Println("       Java:  Future.get(2, TimeUnit.SECONDS)")
	fmt.Println("       Go:    ctx, cancel := context.WithTimeout(ctx, 2*time.Second)")
	fmt.Println()
	fmt.Println("    2. CANCELLATION")
	fmt.Println("       Java:  future.cancel(true) + Thread.interrupt()")
	fmt.Println("       Go:    cancel()  →  every goroutine sees <-ctx.Done()")
	fmt.Println()
	fmt.Println("    3. REQUEST-SCOPED DATA")
	fmt.Println("       Java:  ThreadLocal / MDC / @RequestScope")
	fmt.Println("       Go:    context.WithValue(ctx, key, value)")
	fmt.Println()
	fmt.Println("    4. CASCADING SHUTDOWN")
	fmt.Println("       Java:  ExecutorService.shutdownNow() (best effort)")
	fmt.Println("       Go:    Parent cancel → all children cancelled automatically")
	fmt.Println()
	fmt.Println("  The Golden Rules:")
	fmt.Println("    • ctx is ALWAYS the first parameter")
	fmt.Println("    • ALWAYS defer cancel()")
	fmt.Println("    • Check ctx.Done() in any long-running loop or select")
	fmt.Println("    • Use ctx.Err() to distinguish timeout from cancellation")
	fmt.Println("    • NEVER store context in a struct — pass it explicitly")
	fmt.Println()
}
```

---

## Run the Demo

```bash
go run main.go
```

### Expected Output (varies due to randomized delays)

```
================================================
PART 1: context.WithTimeout — Enforcing a Deadline
================================================

  Deadline set: 500ms from now
  Calling compliance screening (200–800ms range)...

  ❌ compliance-screen   CANCELLED after 500ms
     Error: compliance screening cancelled: context deadline exceeded

  The context expired. ctx.Err() tells you WHY:
     ctx.Err() = context deadline exceeded

  context.DeadlineExceeded means the timeout fired.
  context.Canceled means someone called cancel() explicitly.

================================================
PART 2: Cascading Cancellation — One Signal, All Stop
================================================

  Launching 3 services with a 400ms deadline...

  ✅ currency-convert     completed in 187ms — 1250.00 USD → 1093.75 EUR (rate: 0.8750)
  ✅ fraud-check          completed in 243ms — tx tx-7890: risk score 0.42 — APPROVED
  ❌ compliance-screen    CANCELLED after 400ms — compliance screening cancelled: context deadline exceeded

  Key insight: you created ONE context and passed it to THREE goroutines.
  When the deadline fired, all three received the signal via ctx.Done().
  No manual tracking. No thread interrupts. The context IS the coordination.

================================================
PART 3: context.WithValue — Request-Scoped Data
================================================

  Request ID from context: req-abc-42

  [req-abc-42] [fraud-check] starting risk assessment
  [req-abc-42] [currency-convert] fetching EUR/USD rate
  [req-abc-42] [compliance-screen] screening against OFAC list

  The request ID was attached ONCE at the top of the call chain.
  Every function extracted it from ctx — no extra parameter needed.

  Java equivalent: ThreadLocal<String> or MDC.put("requestId", id)
  Go advantage:    context values are immutable and propagate through
                   goroutines — ThreadLocal does NOT cross thread boundaries.

================================================
PART 4: Full Pipeline — Graceful Degradation
================================================

  --- Deadline: generous (2s) ---
  Request: req-4217 | Total: 614ms | Timed out: false
    ✅ currency-convert     92ms — 1250.00 USD → 1106.25 EUR (rate: 0.8850)
    ✅ fraud-check          203ms — tx tx-req-4217: risk score 0.71 — APPROVED
    ✅ compliance-screen    614ms — entity "GlobalTrade Ltd": CLEAR — no sanctions match

  --- Deadline: tight (300ms) ---
  Request: req-4218 | Total: 300ms | Timed out: true
    ✅ currency-convert     128ms — 1250.00 USD → 1100.00 EUR (rate: 0.8800)
    ✅ fraud-check          247ms — tx tx-req-4218: risk score 0.33 — APPROVED
    ❌ compliance-screen    300ms — compliance screening cancelled: context deadline exceeded

  --- Deadline: impossible (50ms) ---
  Request: req-4219 | Total: 50ms | Timed out: true
    ❌ fraud-check          50ms — fraud check cancelled: context deadline exceeded
    ❌ currency-convert     50ms — currency conversion cancelled: context deadline exceeded
    ❌ compliance-screen    0s — context deadline exceeded

================================================
BONUS: Client Disconnect — context.WithCancel
================================================

  Starting enrichment (client will disconnect in 200ms)...

  📱 Client disconnected! Calling cancel()...

  Request: req-disconnect | Total: 200ms | Timed out: true
  ctx.Err() = context canceled

  Notice: ctx.Err() is context.Canceled, not DeadlineExceeded.
  In production, you'd check this to distinguish between:
    - context.DeadlineExceeded → log as timeout, maybe retry
    - context.Canceled         → client left, don't waste resources

================================================
KEY TAKEAWAYS
================================================

  context.Context is Go's answer to four Java problems at once:

    1. TIMEOUTS
       Java:  Future.get(2, TimeUnit.SECONDS)
       Go:    ctx, cancel := context.WithTimeout(ctx, 2*time.Second)

    2. CANCELLATION
       Java:  future.cancel(true) + Thread.interrupt()
       Go:    cancel()  →  every goroutine sees <-ctx.Done()

    3. REQUEST-SCOPED DATA
       Java:  ThreadLocal / MDC / @RequestScope
       Go:    context.WithValue(ctx, key, value)

    4. CASCADING SHUTDOWN
       Java:  ExecutorService.shutdownNow() (best effort)
       Go:    Parent cancel → all children cancelled automatically

  The Golden Rules:
    • ctx is ALWAYS the first parameter
    • ALWAYS defer cancel()
    • Check ctx.Done() in any long-running loop or select
    • Use ctx.Err() to distinguish timeout from cancellation
    • NEVER store context in a struct — pass it explicitly
```

---

## Instructor Notes

### The Narrative Arc

This demo builds pressure deliberately:

1. **Part 1** introduces the concept gently — one service, one timeout. Students see what happens when the clock runs out.
2. **Part 2** scales to three concurrent services sharing one context. The "one signal, all stop" moment is when Java developers say "wait, they ALL got cancelled from one call?"
3. **Part 3** pivots to `WithValue` — a different use of context. This addresses the "how do I pass a trace ID through goroutines?" question before anyone asks it. The `ThreadLocal` comparison lands hard because Java developers know that `ThreadLocal` does not propagate across threads — but context values propagate across goroutines naturally.
4. **Part 4** is the crescendo. Three deadline scenarios show the spectrum: generous (everything succeeds), tight (partial success), impossible (everything fails). Graceful degradation is visible in the output. This is production behavior.
5. **Part 5** shows `WithCancel` — the manual signal. The client disconnect scenario is immediately relatable. The `ctx.Err()` distinction (`Canceled` vs. `DeadlineExceeded`) is a real production debugging skill.

### Live Coding Tips

- **Run Part 2 three times.** The output changes each run because the delays are randomized. This viscerally demonstrates that your code *must* handle both success and cancellation for every service.
- **In Part 4, pause on the "impossible" deadline** (50ms). Everything fails. Ask the class: "Is this a bug?" No — it is the system behaving correctly. The client asked for enrichment, we tried our best within the budget, and we returned what we could. That is graceful degradation.
- **In Part 5, ask:** "What is the difference between `DeadlineExceeded` and `Canceled`?" The first means time ran out. The second means someone actively called `cancel()`. In production, you'd handle them differently — retry on timeout, abandon on cancel.

### Java Comparison Summary

| Concern | Java | Go |
|---|---|---|
| Timeout on a task | `future.get(2, SECONDS)` | `context.WithTimeout(ctx, 2*time.Second)` |
| Cancel running work | `future.cancel(true)` + cooperative interrupt check | `cancel()` + `<-ctx.Done()` in select |
| Cancel all subtasks | Loop over futures, cancel each | Cancel parent context → all children cancelled |
| Request-scoped data | `ThreadLocal` / MDC / `@RequestScope` | `context.WithValue` |
| Crosses thread boundaries? | `ThreadLocal`: **NO** | Context values: **YES** |
| Deadline inspection | Not built in — track manually | `ctx.Deadline()` returns `(time.Time, bool)` |
| Distinguish timeout vs cancel | `TimeoutException` vs `CancellationException` | `ctx.Err() == context.DeadlineExceeded` vs `context.Canceled` |

### Common Student Questions

**Q: "Can I use context to pass the database connection or logger?"**  
A: No. Context values are for *request-scoped metadata* — trace IDs, auth tokens, correlation IDs. Pass the database and logger as explicit function parameters or struct fields. Using context for dependency injection makes your code opaque and untestable. The Go documentation is explicit: "Use context Values only for request-scoped data that transits processes and APIs, not for passing optional parameters to functions."

**Q: "Why `defer cancel()` even if the timeout will fire on its own?"**  
A: `WithTimeout` creates an internal timer goroutine. If your function returns before the timeout fires (because the work finished early), calling `cancel()` stops the timer and frees the goroutine immediately. Without `defer cancel()`, that goroutine lingers until the timer expires — a resource leak. Multiply by thousands of requests per second and you have a real problem.

**Q: "How does this work with `net/http`?"**  
A: Every `*http.Request` has a `Context()` method that returns a context. When the HTTP client disconnects, this context is cancelled automatically. Your handlers should use `r.Context()` as the parent context for all downstream calls. This is why `context.WithTimeout(r.Context(), 2*time.Second)` works — it creates a child of the request context, so it is cancelled if either the timeout fires OR the client disconnects.

**Q: "Is the custom key type (`contextKey`) really necessary?"**  
A: Yes. If two packages both use `context.WithValue(ctx, "requestID", ...)`, they silently overwrite each other. An unexported type like `type contextKey string` is scoped to your package — no other package can create a matching key, so collisions are impossible. This is not optional; it is a best practice enforced by linters like `golangci-lint`.

---

**End of Demo**
