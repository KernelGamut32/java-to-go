# Lab 7: Concurrency Patterns & Race Detection

**Block 7 — Advanced Concurrency Patterns**  
**Duration:** 60 minutes  
**Prerequisites:** Lab 6 completed, Go 1.25+ installed, VS Code with Go extension  
**Module:** `go mod init concurrency-patterns-lab`

---

## Learning Objectives

By the end of this lab you will be able to:

- Merge multiple channels into one using `select` (fan-in pattern)
- Enforce deadlines on operations using `select` and `time.After`
- Coordinate parallel work with `sync.WaitGroup`
- Detect data races using Go's built-in race detector (`-race`)
- Protect shared state with `sync.Mutex` and contrast it with a channel-based approach
- Propagate cancellation through goroutine chains using `context.Context`
- Identify goroutine leaks using `runtime.NumGoroutine()`

---

## Setup

```bash
mkdir -p concurrency-patterns-lab && cd concurrency-patterns-lab
go mod init concurrency-patterns-lab
```

Each exercise is a standalone program. Create a subdirectory per exercise or overwrite `main.go` between exercises.

---

## Exercise 1 — Fan-In: Merging Multiple Channels with `select` (10 min)

**Goal:** Multiple goroutines produce results on separate channels. Merge everything into a single output channel using `select`.

### Context

In Lab 6 you built a fan-out pattern — distributing work to many workers via a shared channel. Fan-in is the opposite: collecting results from many independent sources into one stream. The `select` statement is the key tool here — it lets you wait on multiple channel operations simultaneously, proceeding with whichever is ready first.

### Instructions

1. Create `ex1_fanin/main.go`.
2. Write a `source` function that takes a name string and returns a `<-chan string`. Inside, launch a goroutine that sends 5 messages (e.g., `"source-A: message 1"`) with a random delay between 100–500 ms, then closes the channel.
3. In `main`, create three sources: `"alpha"`, `"beta"`, `"gamma"`.
4. Write a `fanIn` function that accepts a variadic number of `<-chan string` channels and returns a single merged `<-chan string`. Inside, use a goroutine with a `select` loop to read from all input channels and forward each value to the output.
5. Range over the merged channel in `main` and print each message as it arrives.

### Solution

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"sync"
	"time"
)

// source returns a channel that emits 5 labeled messages at random intervals.
func source(name string) <-chan string {
	ch := make(chan string)
	go func() {
		defer close(ch)
		for i := 1; i <= 5; i++ {
			delay := time.Duration(100+rand.IntN(400)) * time.Millisecond
			time.Sleep(delay)
			ch <- fmt.Sprintf("%s: message %d", name, i)
		}
	}()
	return ch
}

// fanIn merges multiple input channels into a single output channel.
func fanIn(channels ...<-chan string) <-chan string {
	merged := make(chan string)
	var wg sync.WaitGroup

	for _, ch := range channels {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for msg := range ch {
				merged <- msg
			}
		}()
	}

	go func() {
		wg.Wait()
		close(merged)
	}()

	return merged
}

func main() {
	a := source("alpha")
	b := source("beta")
	c := source("gamma")

	for msg := range fanIn(a, b, c) {
		fmt.Println(msg)
	}

	fmt.Println("All sources exhausted.")
}
```

### What to Observe

- Messages arrive interleaved from all three sources. The order is nondeterministic and changes with each run.
- The `fanIn` function does not need to know how many messages each source will produce. It reads until every input channel is closed, then closes the output.
- The `sync.WaitGroup` inside `fanIn` tracks when all input channels are drained. Only then is the merged channel closed, which causes the `range` loop in `main` to terminate cleanly.
- This pattern is extremely common in real systems — for example, aggregating results from multiple microservice calls or database shards.

### Alternative: `select`-Based Fan-In (for exactly 2 channels)

If you have exactly two channels and want to see `select` in its raw form:

```go
func fanIn2(a, b <-chan string) <-chan string {
	merged := make(chan string)
	go func() {
		defer close(merged)
		for a != nil || b != nil {
			select {
			case msg, ok := <-a:
				if !ok {
					a = nil
					continue
				}
				merged <- msg
			case msg, ok := <-b:
				if !ok {
					b = nil
					continue
				}
				merged <- msg
			}
		}
	}()
	return merged
}
```

Setting a channel to `nil` disables that `select` case (a receive on a `nil` channel blocks forever, so `select` skips it). This is the idiomatic way to "remove" a channel from a `select` once it's closed.

---

## Exercise 2 — Timeout Pattern with `select` and `time.After` (8 min)

**Goal:** Simulate a slow operation and enforce a deadline using `select`.

### Instructions

1. Create `ex2_timeout/main.go`.
2. Write a `slowOperation` function that takes a `<-chan struct{}` (a "done" signal) and returns a `<-chan string`. Inside, launch a goroutine that sleeps for a random duration between 1–4 seconds, then sends a result — but only if the done channel has not been closed.
3. In `main`, call `slowOperation` and use `select` to wait for either the result or a 2-second timeout via `time.After`.
4. Print whether you got the result or timed out.

### Solution

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"time"
)

func slowOperation() <-chan string {
	result := make(chan string, 1)
	go func() {
		delay := time.Duration(1+rand.IntN(3)) * time.Second
		time.Sleep(delay)
		result <- fmt.Sprintf("completed after %v", delay)
	}()
	return result
}

func main() {
	fmt.Println("Starting operation with 2-second deadline...")

	result := slowOperation()

	select {
	case msg := <-result:
		fmt.Println("Success:", msg)
	case <-time.After(2 * time.Second):
		fmt.Println("Timeout: operation took too long")
	}
}
```

### Run It Multiple Times

```bash
for i in $(seq 1 5); do go run ex2_timeout/main.go; done
```

### What to Observe

- Roughly half the runs succeed and half time out, depending on the random delay.
- `select` blocks until **one** of its cases is ready, then executes that case. The other case is abandoned.
- `time.After` returns a channel that receives a value after the specified duration. It is a clean, composable way to express deadlines without callbacks or timers.
- **Important:** The goroutine inside `slowOperation` continues running even after the timeout. It will eventually send its result into the buffered channel (capacity 1), which will be garbage collected. In production code you would use `context.Context` to actually cancel the work — you will do exactly that in Exercise 6.

---

## Exercise 3 — Parallel File Processor with `sync.WaitGroup` (8 min)

**Goal:** Simulate processing a batch of files in parallel, waiting for all workers to complete.

### Instructions

1. Create `ex3_waitgroup/main.go`.
2. Define a list of 8 simulated file names (e.g., `"report-Q1.csv"`, `"data-2024.json"`, etc.).
3. Write a `processFile` function that takes a file name, prints a "processing" message, sleeps for a random duration (200–800 ms) to simulate work, then prints a "done" message.
4. In `main`, launch one goroutine per file using `sync.WaitGroup` to track completion.
5. After `wg.Wait()` returns, print a summary line.

### Solution

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"sync"
	"time"
)

func processFile(name string) {
	fmt.Printf("  ⏳ processing %s\n", name)
	delay := time.Duration(200+rand.IntN(600)) * time.Millisecond
	time.Sleep(delay)
	fmt.Printf("  ✅ finished %s (%v)\n", name, delay)
}

func main() {
	files := []string{
		"report-Q1.csv", "report-Q2.csv", "report-Q3.csv", "report-Q4.csv",
		"data-2024.json", "users-export.xml", "config-prod.yaml", "audit-log.txt",
	}

	fmt.Printf("Processing %d files in parallel...\n\n", len(files))

	var wg sync.WaitGroup
	for _, f := range files {
		wg.Add(1)
		go func() {
			defer wg.Done()
			processFile(f)
		}()
	}

	wg.Wait()
	fmt.Printf("\nAll %d files processed.\n", len(files))
}
```

### Go 1.25+ Variant Using `WaitGroup.Go`

```go
var wg sync.WaitGroup
for _, f := range files {
	wg.Go(func() {
		processFile(f)
	})
}
wg.Wait()
```

`wg.Go` handles `Add(1)` before the goroutine launches and `Done()` when the function returns. This eliminates the most common `WaitGroup` mistake — calling `Add` inside the goroutine instead of before it.

### What to Observe

- All files start processing nearly simultaneously. The total wall-clock time is roughly equal to the slowest file, not the sum of all files.
- The "finished" messages arrive in a nondeterministic order — whichever file's simulated work completes first prints first.
- `wg.Wait()` blocks `main` until every goroutine has called `wg.Done()`. Without the `WaitGroup`, `main` would exit immediately and kill all goroutines.

### Common Mistake to Avoid

Never call `wg.Add(1)` inside the goroutine. There is a race condition: `wg.Wait()` might execute before the goroutine starts and calls `Add`, causing `Wait` to return prematurely. Always call `Add` before launching the goroutine — or use `wg.Go` which handles this for you.

---

## Exercise 4 — Data Race: Detect It (8 min)

**Goal:** Write a program with a deliberate data race. Use the race detector to find it.

### Instructions

1. Create `ex4_race/main.go`.
2. Declare a global `counter` variable (an `int`).
3. Launch 1,000 goroutines that each increment `counter` 1,000 times.
4. Wait for all goroutines to finish and print the final counter value.
5. First, run normally: `go run ex4_race/main.go`. Note the final value — it will likely be wrong (less than 1,000,000).
6. Then run with the race detector: `go run -race ex4_race/main.go`. Read the output carefully.

### The Buggy Program

```go
package main

import (
	"fmt"
	"sync"
)

var counter int

func main() {
	var wg sync.WaitGroup

	for range 1000 {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for range 1000 {
				counter++ // DATA RACE: unsynchronized read-modify-write
			}
		}()
	}

	wg.Wait()
	fmt.Printf("Final counter: %d (expected 1,000,000)\n", counter)
}
```

### Run Without Race Detector

```bash
go run ex4_race/main.go
```

You will see output like:

```
Final counter: 547382 (expected 1,000,000)
```

The value is wrong and different every time. The increment operation (`counter++`) is not atomic — it is a read, add, and write. Multiple goroutines read the same value, add 1, and write back, overwriting each other's increments.

### Run With Race Detector

```bash
go run -race ex4_race/main.go
```

The race detector prints detailed output showing exactly where the race occurs:

```
==================
WARNING: DATA RACE
Read at 0x... by goroutine 8:
  main.main.func1()
      /path/to/ex4_race/main.go:16 +0x...

Previous write at 0x... by goroutine 7:
  main.main.func1()
      /path/to/ex4_race/main.go:16 +0x...
...
==================
```

### What to Observe

- The race detector tells you **which variable** is being raced on, **which goroutines** are involved, and **which lines of code** perform the conflicting accesses.
- It identifies both the "Read" and "Previous write" — this is the classic read-modify-write race.
- The race detector adds significant overhead (2–10x slower, 5–10x more memory). It is a development and CI tool, not a production flag.
- **Best practice:** Add `go test -race ./...` to your CI pipeline. Every test run should use the race detector. Data races in Go are undefined behavior — the program can do anything, including appearing to work correctly.

---

## Exercise 5 — Fix the Race: Mutex vs. Channel (10 min)

**Goal:** Fix the data race from Exercise 4 two different ways: first with `sync.Mutex`, then with a channel-based approach. Compare them.

### Part A: Fix with `sync.Mutex`

1. Create `ex5_mutex/main.go`.
2. Copy the buggy program from Exercise 4.
3. Add a `sync.Mutex`. Lock it before incrementing and unlock after.

```go
package main

import (
	"fmt"
	"sync"
)

var (
	counter int
	mu      sync.Mutex
)

func main() {
	var wg sync.WaitGroup

	for range 1000 {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for range 1000 {
				mu.Lock()
				counter++
				mu.Unlock()
			}
		}()
	}

	wg.Wait()
	fmt.Printf("Final counter: %d (expected 1,000,000)\n", counter)
}
```

### Verify: Run with Race Detector

```bash
go run -race ex5_mutex/main.go
```

No warnings. The counter should be exactly 1,000,000.

### Part B: Fix with a Channel

1. Create `ex5_channel/main.go`.
2. Instead of a shared variable, use a channel. Each goroutine sends its increment count to a dedicated "aggregator" goroutine that owns the counter.

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	increments := make(chan int, 256)

	var wg sync.WaitGroup
	for range 1000 {
		wg.Add(1)
		go func() {
			defer wg.Done()
			localCount := 0
			for range 1000 {
				localCount++
			}
			increments <- localCount
		}()
	}

	// Close the channel once all goroutines finish.
	go func() {
		wg.Wait()
		close(increments)
	}()

	// Single aggregator: only this goroutine touches 'counter'.
	counter := 0
	for inc := range increments {
		counter += inc
	}

	fmt.Printf("Final counter: %d (expected 1,000,000)\n", counter)
}
```

### Verify

```bash
go run -race ex5_channel/main.go
```

No warnings. Counter is exactly 1,000,000.

### Compare: Mutex vs. Channel

| Aspect | `sync.Mutex` | Channel-based |
|---|---|---|
| **Shared state** | Yes — `counter` is accessed by all goroutines | No — each goroutine works on a local variable |
| **Synchronization** | Explicit lock/unlock around critical section | Implicit via channel send/receive |
| **Risk of forgetting** | Forgetting to unlock causes deadlock; forgetting to lock causes a race | Harder to misuse — the compiler enforces the communication |
| **Performance** | Fine-grained locking can be faster for simple operations | Channel overhead is higher per operation, but design is cleaner |
| **Best for** | Protecting a simple shared variable (counters, caches) | Coordinating work, pipelines, fan-in/fan-out |

### When to Use Which

- **Mutex:** When you have a simple, small critical section — a counter, a cache, a map that multiple goroutines read and write. This is a pragmatic choice and perfectly idiomatic Go.
- **Channel:** When goroutines need to coordinate work, pass data, or signal completion. The Go proverb applies: *"Don't communicate by sharing memory; share memory by communicating."* Channels make ownership transfer explicit.
- In practice, most Go programs use both. Use the right tool for the right job.

---

## Exercise 6 — Context Cancellation Chain (8 min)

**Goal:** Propagate cancellation through a 3-layer goroutine chain using `context.WithCancel`.

### Instructions

1. Create `ex6_context/main.go`.
2. Build three "layers" of goroutines: `layer1` launches `layer2`, which launches `layer3`. Each layer does simulated work in a loop, checking `ctx.Done()` to know when to stop.
3. In `main`, create a cancellable context. Launch `layer1`. After 2 seconds, call `cancel()`.
4. Observe all three layers shutting down promptly.

### Solution

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

func layer3(ctx context.Context, wg *sync.WaitGroup) {
	defer wg.Done()
	for {
		select {
		case <-ctx.Done():
			fmt.Println("  [layer3] cancelled, shutting down")
			return
		case <-time.After(300 * time.Millisecond):
			fmt.Println("  [layer3] doing deep work...")
		}
	}
}

func layer2(ctx context.Context, wg *sync.WaitGroup) {
	defer wg.Done()

	var innerWg sync.WaitGroup
	innerWg.Add(1)
	go layer3(ctx, &innerWg)

	for {
		select {
		case <-ctx.Done():
			fmt.Println(" [layer2] cancelled, waiting for layer3...")
			innerWg.Wait()
			fmt.Println(" [layer2] shut down")
			return
		case <-time.After(500 * time.Millisecond):
			fmt.Println(" [layer2] doing middle work...")
		}
	}
}

func layer1(ctx context.Context, wg *sync.WaitGroup) {
	defer wg.Done()

	var innerWg sync.WaitGroup
	innerWg.Add(1)
	go layer2(ctx, &innerWg)

	for {
		select {
		case <-ctx.Done():
			fmt.Println("[layer1] cancelled, waiting for layer2...")
			innerWg.Wait()
			fmt.Println("[layer1] shut down")
			return
		case <-time.After(700 * time.Millisecond):
			fmt.Println("[layer1] doing top-level work...")
		}
	}
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())

	var wg sync.WaitGroup
	wg.Add(1)
	go layer1(ctx, &wg)

	fmt.Println("System running. Will cancel in 2 seconds...\n")
	time.Sleep(2 * time.Second)

	fmt.Println("\n--- Cancelling context ---\n")
	cancel()

	wg.Wait()
	fmt.Println("\nAll layers shut down cleanly.")
}
```

### What to Observe

- When `cancel()` is called, the `ctx.Done()` channel is closed. This signal propagates instantly to all three layers — they don't need to know about each other. The context is the cancellation tree.
- Each layer waits for its child to finish before reporting itself as shut down. This ensures orderly, cascading shutdown — no goroutine is orphaned.
- `context.WithCancel` is the foundation. In production you will also use `context.WithTimeout` and `context.WithDeadline` for automatic cancellation after a duration or at a specific time.
- Every function that might run for a non-trivial amount of time should accept a `context.Context` as its first parameter. This is an enforced convention in the Go standard library and a community best practice.

### Stretch Goal: Replace with `context.WithTimeout`

Replace the manual `cancel()` after 2 seconds with:

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()
```

The behavior is identical, but the timeout is expressed declaratively. `defer cancel()` is required to release resources even if the context times out on its own.

---

## Exercise 7 — Goroutine Leak Detection (8 min)

**Goal:** Create a goroutine leak by forgetting to close a channel. Detect it with `runtime.NumGoroutine()`.

### Instructions

1. Create `ex7_leak/main.go`.
2. Write a `leakyProducer` function that returns a `<-chan int`. Inside, launch a goroutine that generates values in an infinite loop, sending each to the channel. Do **not** provide any way to stop it.
3. In `main`, call `leakyProducer` three times. From each returned channel, read only 5 values and then move on (stop reading).
4. After all reads, print `runtime.NumGoroutine()`. You should see leaked goroutines.
5. Then write a `cleanProducer` that takes a `context.Context` and stops when the context is cancelled. Demonstrate that it does not leak.

### Solution

```go
package main

import (
	"context"
	"fmt"
	"runtime"
	"time"
)

// leakyProducer starts a goroutine that sends values forever.
// If the consumer stops reading, this goroutine blocks on the send and leaks.
func leakyProducer() <-chan int {
	ch := make(chan int)
	go func() {
		i := 0
		for {
			ch <- i // blocks forever once consumer stops reading
			i++
		}
	}()
	return ch
}

// cleanProducer respects cancellation via context.
func cleanProducer(ctx context.Context) <-chan int {
	ch := make(chan int)
	go func() {
		defer close(ch)
		i := 0
		for {
			select {
			case <-ctx.Done():
				return
			case ch <- i:
				i++
			}
		}
	}()
	return ch
}

func main() {
	fmt.Printf("Goroutines at start: %d\n\n", runtime.NumGoroutine())

	// --- Leaky version ---
	fmt.Println("=== Leaky Producers ===")
	for p := range 3 {
		ch := leakyProducer()
		for range 5 {
			<-ch // read only 5 values
		}
		fmt.Printf("  Producer %d: read 5 values, stopped reading\n", p)
	}

	// Give goroutines a moment to settle.
	time.Sleep(100 * time.Millisecond)
	fmt.Printf("\nGoroutines after leaky producers: %d (3 are leaked!)\n\n", runtime.NumGoroutine())

	// --- Clean version ---
	fmt.Println("=== Clean Producers ===")
	for p := range 3 {
		ctx, cancel := context.WithCancel(context.Background())
		ch := cleanProducer(ctx)
		for range 5 {
			<-ch
		}
		cancel() // signal the producer to stop
		fmt.Printf("  Producer %d: read 5 values, cancelled context\n", p)
	}

	// Give goroutines a moment to shut down.
	time.Sleep(100 * time.Millisecond)
	fmt.Printf("\nGoroutines after clean producers: %d (no leaks)\n", runtime.NumGoroutine())
}
```

### What to Observe

- After the leaky producers, `runtime.NumGoroutine()` reports 4 (main + 3 leaked goroutines). Each leaked goroutine is permanently blocked on `ch <- i`, waiting for a receiver that will never come. It can never be garbage collected because the channel reference keeps it alive.
- After the clean producers, the goroutine count drops back to 1 (just main). Calling `cancel()` closes the `ctx.Done()` channel, which triggers the `select` case in the producer, causing it to return and be cleaned up.
- **Goroutine leaks are the Go equivalent of memory leaks.** They consume memory and scheduling resources silently. The fix is always the same: give every goroutine a way to be told to stop — either by closing a channel or cancelling a context.

### Production Detection

In production, you can expose `runtime.NumGoroutine()` as a metric (via Prometheus, for example) and alert when it grows unboundedly. The `net/http/pprof` package provides a `/debug/pprof/goroutine` endpoint that shows a full stack trace of every live goroutine — invaluable for diagnosing leaks.

---

## Wrap-Up Checklist

Before you move on, confirm you can answer these questions:

- [ ] How does `select` choose between multiple ready channels?
- [ ] What happens when you set a channel variable to `nil` inside a `select`?
- [ ] Why does `time.After` work as a timeout mechanism in a `select`?
- [ ] What is the difference between `sync.Mutex` and a channel for protecting shared state? When would you choose each?
- [ ] What does the `-race` flag do, and why should it be in your CI pipeline?
- [ ] How does `context.WithCancel` propagate cancellation to child goroutines?
- [ ] How do you detect goroutine leaks, and what is the most common cause?
- [ ] What does `sync.WaitGroup.Go` (Go 1.25) do, and why is it safer than manual `Add`/`Done`?

---

**End of Lab 7**
