# Demo: Type Assertions in Go — A Payment Processing Error System

**Audience:** Java developers transitioning to Go  
**Duration:** ~10–12 minutes live coding  
**Concepts:** Type assertion (bare), type assertion (comma-ok), type switch  
**Java parallel:** `instanceof`, catch blocks with exception hierarchies, pattern matching (`switch` with `case Type t` in Java 21+)

---

## The Scenario

You are building a payment gateway. When a payment fails, the system returns an `error` — but not all errors are equal. Some are retryable (network timeouts), some require customer action (insufficient funds), and some are fatal (fraud detected). Your downstream code needs to inspect the error, extract structured details, and make routing decisions.

In Java, you would build an exception hierarchy (`PaymentException extends Exception`) and use `catch` blocks or `instanceof`. In Go, you build concrete error types that satisfy the `error` interface and use **type assertions** to extract the details.

This demo progresses through three stages:

1. **Bare type assertion** — fast and direct, but panics on the wrong type
2. **Comma-ok assertion** — safe, no panic, but verbose with multiple types
3. **Type switch** — the idiomatic Go solution for branching on multiple types

---

## Setup

Create the demo file. Everything runs in a single `main.go`:

```bash
mkdir -p type-assertions-demo && cd type-assertions-demo
go mod init type-assertions-demo
```

---

## Part 0 — Define the Error Types

> **Instructor talking point:** "In Java, you'd create a class hierarchy: `PaymentException` → `NetworkException`, `InsufficientFundsException`, `FraudException`. In Go, there's no `extends`. Instead, each error type is a struct that implements the `error` interface — which is just one method: `Error() string`."

```go
package main

import (
	"fmt"
	"time"
)

// --- Domain error types ---
// Each satisfies the built-in `error` interface by implementing Error() string.
// They also carry structured data that callers can extract via type assertions.

// NetworkError represents a transient failure (timeout, DNS, connection reset).
// These are retryable.
type NetworkError struct {
	Host       string
	StatusCode int
	RetryAfter time.Duration
}

func (e *NetworkError) Error() string {
	return fmt.Sprintf("network error: host %s returned %d (retry after %v)",
		e.Host, e.StatusCode, e.RetryAfter)
}

// InsufficientFundsError means the customer's account cannot cover the charge.
// Not retryable — requires customer action.
type InsufficientFundsError struct {
	AccountID string
	Requested float64
	Available float64
}

func (e *InsufficientFundsError) Error() string {
	return fmt.Sprintf("insufficient funds: account %s has $%.2f, needs $%.2f",
		e.AccountID, e.Available, e.Requested)
}

// FraudError means the payment was flagged by the fraud detection system.
// Not retryable — requires investigation.
type FraudError struct {
	TransactionID string
	Reason        string
	RiskScore     float64
}

func (e *FraudError) Error() string {
	return fmt.Sprintf("fraud detected: tx %s flagged for %q (risk score: %.1f)",
		e.TransactionID, e.Reason, e.RiskScore)
}
```

> **Instructor talking point:** "Notice — no base class, no hierarchy. Each struct is independent. The only contract is the `error` interface: implement `Error() string` and you are an error. Go's implicit interface satisfaction means these types don't even know the `error` interface exists — they just happen to have the right method."

---

## Part 1 — Simulate the Payment Processor

```go
// processPayment simulates a payment that can fail in different ways.
// It returns the built-in `error` interface — the caller doesn't know
// which concrete type is inside until they inspect it.
func processPayment(scenario string) error {
	switch scenario {
	case "network":
		return &NetworkError{
			Host:       "api.stripe.com",
			StatusCode: 503,
			RetryAfter: 5 * time.Second,
		}
	case "funds":
		return &InsufficientFundsError{
			AccountID: "acct-8842",
			Requested: 249.99,
			Available: 42.17,
		}
	case "fraud":
		return &FraudError{
			TransactionID: "tx-90210",
			Reason:        "velocity check failed",
			RiskScore:     0.94,
		}
	default:
		return nil // success
	}
}
```

> **Instructor talking point:** "The return type is `error` — an interface. The caller receives an opaque box. They can call `.Error()` to get a string, but they can't access `RetryAfter` or `RiskScore` without reaching into the box. That's what type assertions are for."

---

## Part 2 — Bare Type Assertion (The Dangerous Way)

> **Instructor talking point:** "If you *know* the type, you can assert directly. This is the equivalent of a Java cast: `(NetworkError) err`. If you're right, you get the value. If you're wrong, your program crashes."

```go
func demoBareAssertion() {
	fmt.Println("========================================")
	fmt.Println("PART 1: Bare Type Assertion (dangerous)")
	fmt.Println("========================================")

	err := processPayment("network")

	// We KNOW this is a *NetworkError — assert directly.
	netErr := err.(*NetworkError)

	// Now we have access to the structured fields.
	fmt.Printf("  Host:        %s\n", netErr.Host)
	fmt.Printf("  Status:      %d\n", netErr.StatusCode)
	fmt.Printf("  Retry after: %v\n", netErr.RetryAfter)
	fmt.Println()

	// --- Now the dangerous part ---
	// What if we guess wrong?
	fmt.Println("  Now asserting a *NetworkError as *FraudError...")
	fmt.Println("  This will PANIC. Watch the output:")
	fmt.Println()

	// Uncomment the line below to see the panic:
	// _ = err.(*FraudError) // PANIC: interface conversion: *main.NetworkError is *main.NetworkError, not *main.FraudError

	fmt.Println("  (Line is commented out to keep the demo running.)")
	fmt.Println("  In Java, this is like an unchecked ClassCastException.")
	fmt.Println("  Rule: NEVER use bare assertions unless you are 100% certain of the type.")
	fmt.Println()
}
```

> **Instructor talking point:** Uncomment the panic line live. Let them see the crash. Then re-comment it. The visceral moment of watching a panic teaches more than any slide.

---

## Part 3 — Comma-Ok Assertion (The Safe Way)

> **Instructor talking point:** "Go gives you a safe alternative: the comma-ok idiom. Instead of panicking, it returns a boolean. This is like Java's `instanceof` check before casting — but combined into a single expression."

```go
func demoCommaOkAssertion() {
	fmt.Println("========================================")
	fmt.Println("PART 2: Comma-Ok Assertion (safe)")
	fmt.Println("========================================")

	err := processPayment("funds")

	// Try *NetworkError — this will fail safely.
	if netErr, ok := err.(*NetworkError); ok {
		fmt.Printf("  Network error — retry after %v\n", netErr.RetryAfter)
	} else {
		fmt.Println("  Not a NetworkError (ok = false, no panic)")
	}

	// Try *InsufficientFundsError — this will succeed.
	if fundsErr, ok := err.(*InsufficientFundsError); ok {
		fmt.Println("  Insufficient funds detected!")
		fmt.Printf("    Account:   %s\n", fundsErr.AccountID)
		fmt.Printf("    Requested: $%.2f\n", fundsErr.Requested)
		fmt.Printf("    Available: $%.2f\n", fundsErr.Available)
		fmt.Printf("    Shortfall: $%.2f\n", fundsErr.Requested-fundsErr.Available)
	}

	fmt.Println()
	fmt.Println("  This is safe but verbose — imagine checking 5 or 10 error types")
	fmt.Println("  with chained if/else blocks. There's a better way...")
	fmt.Println()
}
```

> **Instructor talking point:** "This works perfectly. But look at the shape of the code: if/else, if/else, if/else. Now imagine you have ten error types. In Java, you'd write a chain of `catch` blocks. Go has something better: the type switch."

---

## Part 4 — Type Switch (The Idiomatic Way)

> **Instructor talking point:** "This is the payoff. A type switch is like Java 21's pattern matching in `switch`, but Go has had it since version 1.0. Inside each case, the variable is *automatically* narrowed to the matched type — no explicit cast, no `(Type) var`, no `instanceof` check."

```go
func demoTypeSwitch() {
	fmt.Println("========================================")
	fmt.Println("PART 3: Type Switch (idiomatic Go)")
	fmt.Println("========================================")

	// Process three different failure scenarios.
	scenarios := []string{"network", "funds", "fraud", "success"}

	for _, scenario := range scenarios {
		err := processPayment(scenario)
		fmt.Printf("  Scenario: %-10s → ", scenario)

		if err == nil {
			fmt.Println("payment succeeded!")
			continue
		}

		// The type switch: Go's answer to Java's multi-catch and instanceof chains.
		//
		// switch v := err.(type) — the special .(type) syntax is ONLY valid inside
		// a switch statement. Inside each case, `v` is automatically the concrete
		// type — no further assertion needed.
		switch v := err.(type) {

		case *NetworkError:
			// v is *NetworkError here — access all fields directly.
			fmt.Printf("RETRYABLE — %s returned %d, retry in %v\n",
				v.Host, v.StatusCode, v.RetryAfter)

		case *InsufficientFundsError:
			// v is *InsufficientFundsError here.
			fmt.Printf("CUSTOMER ACTION — account %s short by $%.2f\n",
				v.AccountID, v.Requested-v.Available)

		case *FraudError:
			// v is *FraudError here.
			fmt.Printf("BLOCKED — tx %s, risk score %.0f%%, reason: %s\n",
				v.TransactionID, v.RiskScore*100, v.Reason)

		default:
			// Catch-all for error types we don't know about.
			fmt.Printf("UNKNOWN ERROR — %v\n", v)
		}
	}
	fmt.Println()
}
```

---

## Part 5 — Putting It All Together: The Error Router

> **Instructor talking point:** "Here's where it gets real. In production, you'd build a function like this — an error router that decides what to do based on the error type. Retry network errors, notify the customer for funds errors, alert the security team for fraud. One function, clean branching, no panic risk."

```go
// handlePaymentError routes an error to the appropriate response strategy.
// This is the kind of function you'd find in a real payment gateway.
func handlePaymentError(err error) (action string, retryable bool) {
	switch v := err.(type) {

	case *NetworkError:
		return fmt.Sprintf("schedule retry after %v against %s",
			v.RetryAfter, v.Host), true

	case *InsufficientFundsError:
		return fmt.Sprintf("notify customer: account %s needs $%.2f more",
			v.AccountID, v.Requested-v.Available), false

	case *FraudError:
		return fmt.Sprintf("alert security team: tx %s flagged (score: %.0f%%)",
			v.TransactionID, v.RiskScore*100), false

	default:
		return fmt.Sprintf("log unknown error: %v", err), false
	}
}

func demoErrorRouter() {
	fmt.Println("========================================")
	fmt.Println("BONUS: Production Error Router")
	fmt.Println("========================================")

	scenarios := []string{"network", "funds", "fraud"}

	for _, scenario := range scenarios {
		err := processPayment(scenario)
		action, retryable := handlePaymentError(err)
		fmt.Printf("  %-10s → retryable: %-5t → %s\n", scenario, retryable, action)
	}
	fmt.Println()
}
```

---

## Part 6 — The `main` Function

```go
func main() {
	demoBareAssertion()
	demoCommaOkAssertion()
	demoTypeSwitch()
	demoErrorRouter()

	fmt.Println("========================================")
	fmt.Println("KEY TAKEAWAYS")
	fmt.Println("========================================")
	fmt.Println("  1. Bare assertion:  v := err.(Type)        → panics if wrong")
	fmt.Println("  2. Comma-ok:        v, ok := err.(Type)    → safe, but verbose")
	fmt.Println("  3. Type switch:     switch v := err.(type) → idiomatic, clean, safe")
	fmt.Println()
	fmt.Println("  Java parallel:")
	fmt.Println("    - Bare assertion    ≈  (Type) obj          — ClassCastException risk")
	fmt.Println("    - Comma-ok          ≈  if (obj instanceof Type t) { ... }")
	fmt.Println("    - Type switch       ≈  switch (obj) { case Type t -> ... }  (Java 21+)")
	fmt.Println()
	fmt.Println("  Go proverb: \"Errors are values.\"")
	fmt.Println("  Type assertions let you treat them as RICH, STRUCTURED values —")
	fmt.Println("  not just strings.")
}
```

---

## Run the Demo

```bash
go run main.go
```

### Expected Output

```
========================================
PART 1: Bare Type Assertion (dangerous)
========================================
  Host:        api.stripe.com
  Status:      503
  Retry after: 5s

  Now asserting a *NetworkError as *FraudError...
  This will PANIC. Watch the output:

  (Line is commented out to keep the demo running.)
  In Java, this is like an unchecked ClassCastException.
  Rule: NEVER use bare assertions unless you are 100% certain of the type.

========================================
PART 2: Comma-Ok Assertion (safe)
========================================
  Not a NetworkError (ok = false, no panic)
  Insufficient funds detected!
    Account:   acct-8842
    Requested: $249.99
    Available: $42.17
    Shortfall: $207.82

  This is safe but verbose — imagine checking 5 or 10 error types
  with chained if/else blocks. There's a better way...

========================================
PART 3: Type Switch (idiomatic Go)
========================================
  Scenario: network    → RETRYABLE — api.stripe.com returned 503, retry in 5s
  Scenario: funds      → CUSTOMER ACTION — account acct-8842 short by $207.82
  Scenario: fraud      → BLOCKED — tx tx-90210, risk score 94%, reason: velocity check failed
  Scenario: success    → payment succeeded!

========================================
BONUS: Production Error Router
========================================
  network    → retryable: true  → schedule retry after 5s against api.stripe.com
  funds      → retryable: false → notify customer: account acct-8842 needs $207.82 more
  fraud      → retryable: false → alert security team: tx tx-90210 flagged (score: 94%)

========================================
KEY TAKEAWAYS
========================================
  1. Bare assertion:  v := err.(Type)        → panics if wrong
  2. Comma-ok:        v, ok := err.(Type)    → safe, but verbose
  3. Type switch:     switch v := err.(type) → idiomatic, clean, safe

  Java parallel:
    - Bare assertion    ≈  (Type) obj          — ClassCastException risk
    - Comma-ok          ≈  if (obj instanceof Type t) { ... }
    - Type switch       ≈  switch (obj) { case Type t -> ... }  (Java 21+)

  Go proverb: "Errors are values."
  Type assertions let you treat them as RICH, STRUCTURED values —
  not just strings.
```

---

## Instructor Notes

### The Narrative Arc

This demo is structured as a deliberate progression — each part solves a problem created by the previous one:

1. **Bare assertion** is fast but *dangerous*. You show the panic (or tease it). Students feel the risk.
2. **Comma-ok** is *safe* but verbose. You show the if/else chain growing. Students feel the friction.
3. **Type switch** resolves both problems: safe *and* clean. The automatic type narrowing inside each case is the "wow" moment — no cast, no `ok` check, the variable just *is* the right type.
4. **Error router** shows the pattern applied to a real production function. Students see how they would actually use this tomorrow.

### Live Coding Tips

- **Uncomment the panic line in Part 1** during the demo. Let the program crash. Show the error message. Then re-comment and re-run. The crash makes the lesson stick.
- **Ask the class** after Part 2: "What happens when you have ten error types?" Let them feel the pain of chained if/else before revealing the type switch.
- **Highlight the automatic type narrowing** in Part 3: inside `case *NetworkError:`, the variable `v` is already `*NetworkError` — you can access `v.Host` directly. No second assertion, no cast. This is the moment Java developers say "oh, that's nice."

### Java Comparison Summary

| Concept | Java | Go |
|---|---|---|
| Error hierarchy | `class NetworkException extends PaymentException` | Independent structs, each implementing `error` |
| Type check | `if (e instanceof NetworkException ne)` | `if ne, ok := err.(*NetworkError); ok` |
| Multi-type branch | `catch (NetworkException e) { } catch (FraudException e) { }` | `switch v := err.(type) { case *NetworkError: ... case *FraudError: ... }` |
| Panic on wrong type | `(NetworkException) fraudErr` → `ClassCastException` | `err.(*NetworkError)` → `panic` |
| No hierarchy needed | Requires `extends` chain | Each type is independent; implicit interface satisfaction |

### Common Student Questions

**Q: "Can I use `errors.As` instead of a type switch?"**  
A: Yes — and you should when working with wrapped errors (from `fmt.Errorf("%w", err)`). `errors.As` unwraps the chain and finds the target type. Type switches work on the *outermost* type only. Mention `errors.As` and note that it is covered in the error handling block — but for direct, unwrapped errors like this demo, a type switch is the cleanest tool.

**Q: "Why pointer receivers (`*NetworkError`) in the assertions?"**  
A: Because the `Error()` method is defined on `*NetworkError` (pointer receiver), only the pointer type satisfies the `error` interface. Asserting to `NetworkError` (value type) would fail. This is a subtle but critical point — the method set of a value type does not include pointer-receiver methods.

**Q: "Why not just use `.Error()` and parse the string?"**  
A: Because strings are unstructured. You cannot reliably extract `RetryAfter` from a free-form string. Type assertions give you typed, structured access — the same reason Java uses exception fields instead of parsing `getMessage()`.

---

**End of Demo**
