# Lab 6: Goroutines & Channels

**Block 6 — Concurrency Foundations**  
**Duration:** 45 minutes  
**Prerequisites:** Go 1.25+ installed, VS Code with Go extension, JDK 23+ installed  
**Module:** `go mod init concurrency-lab`

---

## Learning Objectives

By the end of this lab you will be able to:

- Launch goroutines and collect results through channels
- Build producer/consumer pipelines using channels
- Distinguish buffered from unbuffered channel behavior
- Implement fan-out work distribution
- Recognize and interpret Go's runtime deadlock detection
- Compare Go's concurrency model to Java's `ExecutorService`

---

## Setup

Create your working directory and initialize the module:

```bash
mkdir -p concurrency-lab && cd concurrency-lab
go mod init concurrency-lab
```

Each exercise below is a self-contained `main.go` file. Create a separate subdirectory for each exercise, or overwrite `main.go` between exercises — whichever you prefer.

---

## Exercise 1 — Launch 10,000 Goroutines (10 min)

**Goal:** Launch 10,000 goroutines that each send their ID through a channel. Collect every result and verify that all 10,000 arrived.

### Instructions

1. Create a file `ex1_ten_thousand/main.go`.
2. In `main`, create an unbuffered `chan int`.
3. Launch 10,000 goroutines. Each goroutine receives its ID (0–9,999) and sends that ID into the channel.
4. In a separate goroutine (or after all sends), close the channel once all goroutines have sent their value. Use `sync.WaitGroup` to know when every goroutine has finished sending.
5. In `main`, range over the channel to collect results into a slice or simply count them.
6. Print the total count and verify it equals 10,000.

### Starter Skeleton

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	const numGoroutines = 10_000
	ch := make(chan int)

	var wg sync.WaitGroup
	for i := range numGoroutines {
		wg.Add(1)
		go func() {
			defer wg.Done()
			ch <- i
		}()
	}

	// Close the channel once all goroutines finish sending.
	go func() {
		wg.Wait()
		close(ch)
	}()

	count := 0
	for range ch {
		count++
	}

	fmt.Printf("Received %d results (expected %d)\n", count, numGoroutines)
}
```

### Run It

```bash
go run ex1_ten_thousand/main.go
```

### What to Observe

- The program completes almost instantly — 10,000 goroutines are trivially cheap.
- Each goroutine's stack starts at roughly 2–4 KB. Multiply that by 10,000 and you are using roughly 20–40 MB of stack space — well within reach. An equivalent program using OS threads would attempt to allocate ~10 GB of stack memory.
- Notice the use of `range numGoroutines` — this integer-range syntax (introduced in Go 1.22) is the idiomatic way to iterate `n` times.
- The loop variable `i` is scoped per iteration (Go 1.22+), so each goroutine captures its own copy. In Go versions before 1.22 you would need to pass `i` as a function parameter to avoid the classic closure-capture bug.

### Stretch Goal

Collect all IDs into a slice, sort them, and verify every integer from 0 to 9,999 is present. This demonstrates that no results are lost — channels provide safe, synchronized communication.

---

## Exercise 2 — Producer/Consumer Pipeline (10 min)

**Goal:** One goroutine produces data, another consumes and processes it via a channel.

### Instructions

1. Create a file `ex2_pipeline/main.go`.
2. Write a `produce` function that takes a send-only channel (`chan<- int`). It should generate integers 1 through 20 by sending each into the channel, then close the channel.
3. Write a `consume` function that takes a receive-only channel (`<-chan int`) and a `chan<- int` for results. For each value received, compute its square and send the result to the output channel.
4. In `main`, wire the pipeline: `produce → consume → main collects and prints`.

### Solution

```go
package main

import "fmt"

func produce(out chan<- int) {
	for i := 1; i <= 20; i++ {
		out <- i
	}
	close(out)
}

func consume(in <-chan int, out chan<- int) {
	for v := range in {
		out <- v * v
	}
	close(out)
}

func main() {
	raw := make(chan int)
	squared := make(chan int)

	go produce(raw)
	go consume(raw, squared)

	for result := range squared {
		fmt.Println(result)
	}
}
```

### What to Observe

- **Channel direction types** (`chan<-` and `<-chan`) enforce at compile time that `produce` can only send and `consume` can only receive on the appropriate channels. This is a safety feature — misuse is a compile error, not a runtime bug.
- The `range` loop over a channel automatically terminates when the channel is closed.
- The pipeline is fully synchronized: `produce` blocks on each send until `consume` is ready to receive (unbuffered channel), and `consume` blocks until `main` reads the result. This is the CSP (Communicating Sequential Processes) model in action.
- Only the **sender** closes a channel — never the receiver. Closing a channel you are receiving from is a programming error.

---

## Exercise 3 — Buffered vs. Unbuffered Channels (5 min)

**Goal:** Observe the difference in blocking behavior between buffered and unbuffered channels.

### Instructions

1. Create a file `ex3_buffering/main.go`.
2. First, create an **unbuffered** channel. Launch a goroutine that sends three values into it, printing a message before and after each send. In `main`, sleep for 500 ms between each receive so you can see the blocking pattern.
3. Then repeat the same experiment with a **buffered** channel of capacity 3. Observe how the send behavior changes.

### Solution

```go
package main

import (
	"fmt"
	"time"
)

func sender(ch chan<- int, label string) {
	for i := 1; i <= 3; i++ {
		fmt.Printf("[%s] about to send %d\n", label, i)
		ch <- i
		fmt.Printf("[%s] sent %d\n", label, i)
	}
	close(ch)
}

func receiver(ch <-chan int, label string) {
	for v := range ch {
		fmt.Printf("[%s] received %d\n", label, v)
		time.Sleep(500 * time.Millisecond)
	}
}

func main() {
	fmt.Println("=== Unbuffered Channel ===")
	unbuf := make(chan int)
	go sender(unbuf, "unbuf")
	receiver(unbuf, "unbuf")

	fmt.Println()

	fmt.Println("=== Buffered Channel (cap 3) ===")
	buf := make(chan int, 3)
	go sender(buf, "buf")
	// Small delay to let the sender fill the buffer before we start receiving.
	time.Sleep(100 * time.Millisecond)
	receiver(buf, "buf")
}
```

### What to Observe

- **Unbuffered:** Each "about to send" is followed by a pause — the sender blocks until the receiver calls receive. Sends and receives are interleaved one-for-one.
- **Buffered (capacity 3):** All three sends complete immediately (the buffer absorbs them). The receiver then drains the buffer at its own pace. The "about to send" and "sent" messages appear in rapid succession before any "received" messages.
- Buffered channels decouple the speed of the sender and receiver — up to the buffer capacity. Once the buffer is full, the sender blocks just like an unbuffered channel.

---

## Exercise 4 — Fan-Out Pattern (8 min)

**Goal:** Distribute work to N worker goroutines via a shared channel.

### Instructions

1. Create a file `ex4_fanout/main.go`.
2. Define a `worker` function that takes a worker ID, a receive-only `<-chan int` for jobs, and a send-only `chan<- string` for results. Each worker reads a job from the channel, simulates processing with a 100 ms sleep, and sends back a result string.
3. In `main`, create a jobs channel and a results channel. Launch 5 workers.
4. Send 20 jobs into the jobs channel, then close it.
5. Collect all 20 results and print them.

### Solution

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func worker(id int, jobs <-chan int, results chan<- string, wg *sync.WaitGroup) {
	defer wg.Done()
	for job := range jobs {
		time.Sleep(100 * time.Millisecond) // simulate work
		results <- fmt.Sprintf("worker %d processed job %d", id, job)
	}
}

func main() {
	const numWorkers = 5
	const numJobs = 20

	jobs := make(chan int, numJobs)
	results := make(chan string, numJobs)

	var wg sync.WaitGroup
	for w := 1; w <= numWorkers; w++ {
		wg.Add(1)
		go worker(w, jobs, results, &wg)
	}

	for j := 1; j <= numJobs; j++ {
		jobs <- j
	}
	close(jobs)

	// Close results channel once all workers finish.
	go func() {
		wg.Wait()
		close(results)
	}()

	for r := range results {
		fmt.Println(r)
	}
}
```

### What to Observe

- Jobs are distributed across workers automatically — whichever worker is free picks the next job from the shared channel. This is sometimes called "competing consumers."
- The output order is nondeterministic. Different workers process different jobs on each run. This is concurrency in action — the Go scheduler decides which goroutine runs next.
- The buffered jobs channel allows all 20 jobs to be enqueued before any worker reads them. This keeps the sender from blocking.
- This pattern scales trivially: change `numWorkers` to 50 or 500 and observe the wall-clock time drop proportionally (down to the overhead of goroutine scheduling).

### Stretch Goal (Go 1.25+)

Go 1.25 introduced `sync.WaitGroup.Go`, which combines `wg.Add(1)` and `go func()` into a single call. Refactor the worker launch loop to use it:

```go
for w := 1; w <= numWorkers; w++ {
    wg.Go(func() {
        // Note: 'w' is captured per-iteration (Go 1.22+ loop scoping).
        for job := range jobs {
            time.Sleep(100 * time.Millisecond)
            results <- fmt.Sprintf("worker %d processed job %d", w, job)
        }
    })
}
```

This eliminates the manual `Add`/`Done` bookkeeping and is now the idiomatic pattern in Go 1.25+.

---

## Exercise 5 — Deadlock Detection (4 min)

**Goal:** Intentionally create a deadlock and learn to read Go's runtime error message.

### Instructions

1. Create a file `ex5_deadlock/main.go`.
2. Write the shortest program you can that deadlocks. The simplest way: create an unbuffered channel and try to send on it in `main` with no other goroutine to receive.
3. Run it and read the error output carefully.

### Deadlock Program

```go
package main

func main() {
	ch := make(chan int)
	ch <- 42 // blocks forever: no goroutine is receiving
}
```

### Run It

```bash
go run ex5_deadlock/main.go
```

### Expected Output

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
        /path/to/ex5_deadlock/main.go:4 +0x...
exit status 2
```

### What to Observe

- Go's runtime detects when **all** goroutines are blocked and none can make progress. It prints `all goroutines are asleep - deadlock!` and terminates the program.
- The stack trace tells you exactly which line caused the block and what kind of operation it was (`chan send` in this case).
- This detection only works when **every** goroutine is blocked. If even one goroutine is running (for example, a `time.Sleep` loop), the runtime will not detect the deadlock — your program will simply hang.

### Follow-Up: A Subtler Deadlock

Try this variant where two goroutines deadlock each other:

```go
package main

func main() {
	ch1 := make(chan int)
	ch2 := make(chan int)

	go func() {
		val := <-ch1   // waits for ch1
		ch2 <- val     // then sends to ch2
	}()

	val := <-ch2       // main waits for ch2
	ch1 <- val         // then sends to ch1
}
```

Both goroutines are waiting on each other — a classic circular dependency. Go detects this and prints the same fatal error, this time showing two blocked goroutines in the stack trace.

---

## Exercise 6 — Java Comparison: Producer/Consumer (8 min)

**Goal:** Implement the same producer/consumer pipeline from Exercise 2 using Java's `ExecutorService`. Feel the difference in boilerplate and mental model.

### Instructions

1. Create a file `ProducerConsumer.java`.
2. Use a `BlockingQueue` as the communication channel between a producer task and a consumer task.
3. Submit both tasks to an `ExecutorService`.
4. The producer generates integers 1–20; the consumer reads and prints their squares.
5. Use a sentinel value (e.g., `-1`) or a separate completion signal to tell the consumer to stop.

### Java Solution

```java
import java.util.concurrent.*;

public class ProducerConsumer {

    // Sentinel value signaling that production is complete.
    private static final int DONE = -1;

    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<Integer> queue = new LinkedBlockingQueue<>();

        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {

            // Producer: generate values 1–20, then send sentinel.
            executor.submit(() -> {
                try {
                    for (int i = 1; i <= 20; i++) {
                        queue.put(i);
                    }
                    queue.put(DONE);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });

            // Consumer: read values, compute squares, stop on sentinel.
            Future<?> consumer = executor.submit(() -> {
                try {
                    while (true) {
                        int value = queue.take();
                        if (value == DONE) {
                            break;
                        }
                        System.out.println(value * value);
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });

            // Wait for the consumer to finish.
            consumer.get();

        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```

### Run It

```bash
javac ProducerConsumer.java
java ProducerConsumer
```

### Compare: Go vs. Java

| Aspect | Go | Java |
|---|---|---|
| **Channel / queue** | Built-in `chan` type | `BlockingQueue` (imported) |
| **Termination signal** | `close(ch)` + `range` loop | Sentinel value or external flag |
| **Launching concurrent work** | `go` keyword (one word) | `executor.submit(...)` with lambda |
| **Error handling in tasks** | Errors sent through channels | `try/catch InterruptedException` in every task |
| **Boilerplate** | ~15 lines | ~35 lines |
| **Checked exceptions** | N/A — Go has no checked exceptions | Must handle `InterruptedException`, `ExecutionException` |
| **Thread cost** | Goroutines: ~2–4 KB stack, M:N scheduled | Virtual threads (JDK 21+): lightweight, but still heavier API surface |

### What to Observe

- Java's virtual threads (JDK 21+) solve the same *performance* problem as goroutines — lightweight, M:N scheduled threads. But the *programming model* is still more verbose: you need `BlockingQueue`, sentinel values or flags, checked exception handling, and explicit lifecycle management (`try-with-resources` on the executor).
- Go's `close(ch)` + `range` pattern eliminates the need for sentinel values entirely. The language gives you a first-class way to signal "no more data."
- The `go` keyword makes launching concurrent work feel as natural as calling a function. In Java, even with virtual threads, you are always aware that you are interacting with a thread management API.
- Neither approach is "wrong" — but Go's concurrency primitives are baked into the language, while Java's are library features layered on top.

---

## Wrap-Up Checklist

Before you move on, confirm you can answer these questions:

- [ ] What is the approximate memory cost of a goroutine vs. an OS thread?
- [ ] What happens when you send on an unbuffered channel with no receiver?
- [ ] Why should only the **sender** close a channel?
- [ ] How does Go's runtime detect a deadlock, and when does it fail to detect one?
- [ ] What is the fan-out pattern, and why is it useful?
- [ ] How does `sync.WaitGroup.Go` (Go 1.25) simplify goroutine management?
- [ ] How does Go's channel-based concurrency differ from Java's `BlockingQueue` approach?

---

**End of Lab 6**
