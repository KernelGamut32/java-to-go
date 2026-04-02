# Lab 5: Error Handling Patterns

**Course:** From Java to Go — Prepared for Mastercard
**Block:** 5 — Error Handling the Go Way
**Duration:** 45 minutes
**Difficulty:** Intermediate

---

## Objective

By the end of this lab, you will have translated a Java `try/catch/finally` pattern into idiomatic Go, defined sentinel errors, created a custom error type with structured fields, built an error wrapping chain across three application layers, used `defer` for resource cleanup, and identified a class of bugs that arise from silently ignoring errors. You will leave this lab able to write error handling code that any Go reviewer would accept without comment.

---

## Prerequisites

- Completed Labs 1–4 (comfortable with structs, methods, interfaces, and factory functions)
- Familiarity with Java's `try/catch/finally`, checked vs. unchecked exceptions, and custom exception classes

---

## Setup

```bash
mkdir -p ~/go-labs/lab05-errors
cd ~/go-labs/lab05-errors
go mod init github.com/mastercard/lab05-errors
code .
```

---

## Part 1: From try/catch/finally to if err != nil (8 minutes)

### Step 1 — Read the Java source (do not type this — just read it)

The following Java method reads a file, processes its contents, and handles errors with `try/catch/finally`:

```java
// Java — FileProcessor.java
public class FileProcessor {

    public String processFile(String path) throws ProcessingException {
        BufferedReader reader = null;
        try {
            reader = new BufferedReader(new FileReader(path));
            StringBuilder content = new StringBuilder();
            String line;
            while ((line = reader.readLine()) != null) {
                if (line.isBlank()) {
                    continue;
                }
                content.append(line.trim().toUpperCase()).append("\n");
            }
            if (content.isEmpty()) {
                throw new ProcessingException("file is empty: " + path);
            }
            return content.toString();

        } catch (FileNotFoundException e) {
            throw new ProcessingException("file not found: " + path, e);

        } catch (IOException e) {
            throw new ProcessingException("read error: " + path, e);

        } finally {
            if (reader != null) {
                try {
                    reader.close();
                } catch (IOException e) {
                    // swallowed — common Java anti-pattern
                }
            }
        }
    }
}
```

Notice the structure: nested `try/catch/finally`, a separate `catch` block per exception type, a `finally` block for cleanup with its own nested `try/catch`, and a swallowed close exception. This is standard Java — correct but verbose.

### Step 2 — Write the Go equivalent

Create a file called `fileprocessor.go`:

```go
package main

import (
	"bufio"
	"fmt"
	"os"
	"strings"
)

// ProcessFile reads a file, trims and uppercases each non-empty line,
// and returns the result. This is the Go equivalent of the Java method above.
func ProcessFile(path string) (string, error) {
	// Open the file. If this fails, return immediately with context.
	f, err := os.Open(path)
	if err != nil {
		return "", fmt.Errorf("process file %q: %w", path, err)
	}
	// defer replaces finally — f.Close() runs when this function returns,
	// regardless of whether we return normally or after an error.
	// No nested try/catch needed.
	defer f.Close()

	var result strings.Builder
	scanner := bufio.NewScanner(f)
	lineCount := 0

	for scanner.Scan() {
		line := strings.TrimSpace(scanner.Text())
		if line == "" {
			continue
		}
		result.WriteString(strings.ToUpper(line))
		result.WriteString("\n")
		lineCount++
	}

	// Check for scanner errors (e.g., read failure mid-file)
	if err := scanner.Err(); err != nil {
		return "", fmt.Errorf("process file %q: read error: %w", path, err)
	}

	if lineCount == 0 {
		return "", fmt.Errorf("process file %q: file is empty", path)
	}

	return result.String(), nil
}
```

Compare the two versions side by side in your mind:

| Aspect | Java | Go |
|---|---|---|
| **Error signal** | `throw new ProcessingException(...)` | `return "", fmt.Errorf(...)` |
| **Error check** | `catch (IOException e)` | `if err != nil` |
| **Cleanup** | `finally { reader.close() }` | `defer f.Close()` |
| **Close error** | Swallowed in nested try/catch | Handled by `defer` (or can be checked explicitly) |
| **Flow control** | Exceptions jump to catch blocks | Linear: check, handle, continue |
| **Nesting depth** | 3 levels (try, catch, finally) | 1 level throughout |

> **Key insight:** The Go version reads top to bottom. There are no jumps, no hidden control flow, no question about which `catch` block handles which error. Every error is checked at the point where it occurs. This is what Go means by "errors as values."

### Step 3 — Test the file processor

Create a file called `main.go`:

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	// === Part 1: File Processing ===
	fmt.Println("=== File Processing ===")

	// Create a test file
	testContent := "  hello world  \n\n  go is simple  \n  errors are values  \n"
	if err := os.WriteFile("test.txt", []byte(testContent), 0o644); err != nil {
		fmt.Fprintf(os.Stderr, "setup failed: %v\n", err)
		os.Exit(1)
	}
	defer os.Remove("test.txt")

	// Process a valid file
	result, err := ProcessFile("test.txt")
	if err != nil {
		fmt.Printf("Error: %v\n", err)
	} else {
		fmt.Printf("Result:\n%s", result)
	}

	// Process a non-existent file
	_, err = ProcessFile("nonexistent.txt")
	if err != nil {
		fmt.Printf("Expected error: %v\n", err)
	}

	// Process an empty file
	if err := os.WriteFile("empty.txt", []byte("   \n\n   \n"), 0o644); err != nil {
		fmt.Fprintf(os.Stderr, "setup failed: %v\n", err)
		os.Exit(1)
	}
	defer os.Remove("empty.txt")

	_, err = ProcessFile("empty.txt")
	if err != nil {
		fmt.Printf("Expected error: %v\n", err)
	}
}
```

Run:

```bash
go run .
```

Expected output:

```
=== File Processing ===
Result:
HELLO WORLD
GO IS SIMPLE
ERRORS ARE VALUES
Expected error: process file "nonexistent.txt": open nonexistent.txt: no such file or directory
Expected error: process file "empty.txt": file is empty
```

Notice how the non-existent file error includes the original OS error wrapped with `%w`. The full error chain reads naturally: what we were trying to do (`process file`) → what went wrong (`open nonexistent.txt`) → the OS error (`no such file or directory`).

---

## Part 2: Sentinel Errors and errors.Is() (8 minutes)

### Step 4 — Define sentinel errors for a user repository

Create a file called `errors.go`:

```go
package main

import "errors"

// Sentinel errors are package-level variables that represent specific,
// well-known error conditions. Callers check for them with errors.Is().
//
// Java equivalent: custom exception classes like UserNotFoundException.
// Go equivalent: a single error value that you compare against.
var (
	ErrNotFound  = errors.New("not found")
	ErrDuplicate = errors.New("already exists")
	ErrForbidden = errors.New("access denied")
)
```

These are simple, descriptive error values. They carry no context about *which* user was not found — that context is added by wrapping them with `fmt.Errorf` and `%w` at the call site.

### Step 5 — Build a user repository that returns sentinel errors

Create a file called `repository.go`:

```go
package main

import (
	"fmt"
	"sync"
)

// UserRecord represents a stored user.
type UserRecord struct {
	ID    int
	Name  string
	Email string
}

// UserRepository is an in-memory user store.
// It represents the data access layer in a typical layered architecture.
type UserRepository struct {
	mu    sync.RWMutex
	users map[int]UserRecord
	nextID int
}

// NewUserRepository creates an empty repository.
func NewUserRepository() *UserRepository {
	return &UserRepository{
		users:  make(map[int]UserRecord),
		nextID: 1,
	}
}

// Create adds a new user. Returns ErrDuplicate if the email already exists.
func (r *UserRepository) Create(name, email string) (UserRecord, error) {
	r.mu.Lock()
	defer r.mu.Unlock()

	// Check for duplicate email
	for _, u := range r.users {
		if u.Email == email {
			return UserRecord{}, fmt.Errorf("repo create user %q: %w", email, ErrDuplicate)
		}
	}

	user := UserRecord{
		ID:    r.nextID,
		Name:  name,
		Email: email,
	}
	r.users[user.ID] = user
	r.nextID++

	return user, nil
}

// GetByID retrieves a user by ID. Returns ErrNotFound if the user does not exist.
func (r *UserRepository) GetByID(id int) (UserRecord, error) {
	r.mu.RLock()
	defer r.mu.RUnlock()

	user, exists := r.users[id]
	if !exists {
		return UserRecord{}, fmt.Errorf("repo get user id=%d: %w", id, ErrNotFound)
	}

	return user, nil
}

// Delete removes a user by ID. Returns ErrNotFound if the user does not exist.
func (r *UserRepository) Delete(id int) error {
	r.mu.Lock()
	defer r.mu.Unlock()

	if _, exists := r.users[id]; !exists {
		return fmt.Errorf("repo delete user id=%d: %w", id, ErrNotFound)
	}

	delete(r.users, id)
	return nil
}
```

Every error returned by this repository wraps a sentinel error with `%w` and includes context (the operation and the relevant identifier). The caller never needs to parse the error string — they use `errors.Is()`.

### Step 6 — Handle sentinel errors with errors.Is()

Add to `main()`:

```go
	// === Part 2: Sentinel Errors ===
	fmt.Println("\n=== Sentinel Errors & errors.Is() ===")

	repo := NewUserRepository()

	// Create a user
	user, err := repo.Create("Ada Lovelace", "ada@example.com")
	if err != nil {
		fmt.Printf("Create error: %v\n", err)
	} else {
		fmt.Printf("Created: %+v\n", user)
	}

	// Try to create a duplicate
	_, err = repo.Create("Ada Clone", "ada@example.com")
	if err != nil {
		if errors.Is(err, ErrDuplicate) {
			fmt.Printf("Duplicate detected (errors.Is): %v\n", err)
		} else {
			fmt.Printf("Unexpected error: %v\n", err)
		}
	}

	// Look up a non-existent user
	_, err = repo.GetByID(999)
	if err != nil {
		if errors.Is(err, ErrNotFound) {
			fmt.Printf("Not found (errors.Is): %v\n", err)
		} else {
			fmt.Printf("Unexpected error: %v\n", err)
		}
	}

	// Delete a non-existent user
	err = repo.Delete(999)
	if errors.Is(err, ErrNotFound) {
		fmt.Printf("Delete not found: %v\n", err)
	}
```

Run:

```bash
go run .
```

Study the output. `errors.Is(err, ErrNotFound)` returns `true` even though `err` is `"repo get user id=999: not found"` — not equal to `ErrNotFound` by string comparison. The `%w` verb creates a wrapping chain, and `errors.Is()` walks that chain to find the sentinel.

> **Java parallel:** This is equivalent to `catch (UserNotFoundException e)` — you are matching on the *type* of problem, not the *message*. But in Go, you use value comparison (`errors.Is`) instead of type-based exception dispatch.

---

## Part 3: Custom Error Types and errors.As() (10 minutes)

### Step 7 — Define a custom ValidationError

Create a file called `validation.go`:

```go
package main

import (
	"fmt"
	"strings"
)

// ValidationError carries structured information about what failed validation.
//
// Java equivalent:
//   public class ValidationException extends Exception {
//       private final String field;
//       private final String reason;
//       private final Object rejectedValue;
//   }
//
// In Go, this is just a struct that implements the error interface.
type ValidationError struct {
	Field    string
	Message  string
	Rejected any // the value that failed validation (any = interface{})
}

// Error implements the error interface.
// This single method is all that is needed to make ValidationError an error.
func (ve *ValidationError) Error() string {
	return fmt.Sprintf("validation failed on field %q: %s (got %v)",
		ve.Field, ve.Message, ve.Rejected)
}

// ValidateUserInput checks user input and returns a ValidationError
// with structured data if anything is wrong.
func ValidateUserInput(name, email string, age int) error {
	if strings.TrimSpace(name) == "" {
		return &ValidationError{
			Field:    "name",
			Message:  "must not be empty",
			Rejected: name,
		}
	}

	if len(name) < 2 {
		return &ValidationError{
			Field:    "name",
			Message:  "must be at least 2 characters",
			Rejected: name,
		}
	}

	if !strings.Contains(email, "@") || !strings.Contains(email, ".") {
		return &ValidationError{
			Field:    "email",
			Message:  "must be a valid email address",
			Rejected: email,
		}
	}

	if age < 0 || age > 150 {
		return &ValidationError{
			Field:    "age",
			Message:  "must be between 0 and 150",
			Rejected: age,
		}
	}

	return nil
}
```

Notice that `ValidationError` is a plain struct that implements the `error` interface by having an `Error() string` method. That is the entire contract. No extending `Exception`, no `throws` declaration, no checked/unchecked distinction.

### Step 8 — Extract structured data with errors.As()

Add to `main()`:

```go
	// === Part 3: Custom Error Types & errors.As() ===
	fmt.Println("\n=== Custom Error Types & errors.As() ===")

	// Test various invalid inputs
	testCases := []struct {
		name  string
		email string
		age   int
	}{
		{"", "ada@example.com", 30},           // empty name
		{"A", "ada@example.com", 30},          // name too short
		{"Ada", "not-an-email", 30},           // bad email
		{"Ada", "ada@example.com", -5},        // negative age
		{"Ada", "ada@example.com", 30},        // valid — no error
	}

	for _, tc := range testCases {
		err := ValidateUserInput(tc.name, tc.email, tc.age)
		if err != nil {
			// Use errors.As to extract the ValidationError
			var ve *ValidationError
			if errors.As(err, &ve) {
				fmt.Printf("Validation failed:\n")
				fmt.Printf("  Field:    %s\n", ve.Field)
				fmt.Printf("  Message:  %s\n", ve.Message)
				fmt.Printf("  Rejected: %v\n", ve.Rejected)
			} else {
				fmt.Printf("Non-validation error: %v\n", err)
			}
		} else {
			fmt.Printf("Input (%s, %s, %d) is valid\n", tc.name, tc.email, tc.age)
		}
	}
```

Run:

```bash
go run .
```

`errors.As(err, &ve)` does two things: it walks the error wrapping chain looking for a `*ValidationError`, and if found, it assigns the value to `ve` so you can access the structured fields. This is the Go equivalent of `catch (ValidationException e)` followed by `e.getField()`.

> **Java parallel:** `errors.Is()` is like `catch (SpecificException e)` — matching on a known value. `errors.As()` is like `catch` combined with `instanceof` and a cast — matching on a type and extracting data. The critical difference: Java dispatches by exception type in the `catch` clause; Go dispatches by calling `errors.Is()` or `errors.As()` in an `if` statement. Same power, explicit control flow.

---

## Part 4: Error Wrapping Across Layers (10 minutes)

### Step 9 — Build a three-layer error chain

In a real application, errors pass through layers: **repository → service → handler**. Each layer adds context. The original error is preserved for programmatic checking.

Create a file called `service.go`:

```go
package main

import (
	"fmt"
)

// UserService is the business logic layer.
// It depends on *UserRepository and calls ValidateUserInput.
type UserService struct {
	repo *UserRepository
}

// NewUserService creates a UserService.
func NewUserService(repo *UserRepository) *UserService {
	return &UserService{repo: repo}
}

// RegisterUser validates input, then creates a user in the repository.
// Errors from both validation and the repository are wrapped with service-level context.
func (s *UserService) RegisterUser(name, email string, age int) (UserRecord, error) {
	// Validate input — returns *ValidationError on failure
	if err := ValidateUserInput(name, email, age); err != nil {
		return UserRecord{}, fmt.Errorf("service register user: %w", err)
	}

	// Create in repository — may return ErrDuplicate
	user, err := s.repo.Create(name, email)
	if err != nil {
		return UserRecord{}, fmt.Errorf("service register user: %w", err)
	}

	return user, nil
}

// GetUser retrieves a user by ID.
// Wraps repository errors with service context.
func (s *UserService) GetUser(id int) (UserRecord, error) {
	user, err := s.repo.GetByID(id)
	if err != nil {
		return UserRecord{}, fmt.Errorf("service get user: %w", err)
	}
	return user, nil
}
```

Now create `handler.go` — the outermost layer:

```go
package main

import (
	"errors"
	"fmt"
)

// HandleRegistration simulates an HTTP handler that registers a user.
// This is the outermost layer — it decides how to respond based on the error type.
func HandleRegistration(svc *UserService, name, email string, age int) {
	user, err := svc.RegisterUser(name, email, age)
	if err != nil {
		// Check for validation errors — respond with 400-like message
		var ve *ValidationError
		if errors.As(err, &ve) {
			fmt.Printf("[400 Bad Request] Field %q: %s\n", ve.Field, ve.Message)
			return
		}

		// Check for duplicate — respond with 409-like message
		if errors.Is(err, ErrDuplicate) {
			fmt.Printf("[409 Conflict] User with this email already exists\n")
			return
		}

		// Unknown error — respond with 500-like message
		fmt.Printf("[500 Internal Error] %v\n", err)
		return
	}

	fmt.Printf("[201 Created] User: %+v\n", user)
}

// HandleGetUser simulates an HTTP handler that retrieves a user.
func HandleGetUser(svc *UserService, id int) {
	user, err := svc.GetUser(id)
	if err != nil {
		if errors.Is(err, ErrNotFound) {
			fmt.Printf("[404 Not Found] User id=%d does not exist\n", id)
			return
		}
		fmt.Printf("[500 Internal Error] %v\n", err)
		return
	}

	fmt.Printf("[200 OK] User: %+v\n", user)
}
```

### Step 10 — Exercise the full error chain

Add to `main()`:

```go
	// === Part 4: Error Wrapping Chain ===
	fmt.Println("\n=== Error Wrapping Chain: Handler → Service → Repository ===")

	svc := NewUserService(repo) // repo was created in Part 2

	// Successful registration
	HandleRegistration(svc, "Grace Hopper", "grace@example.com", 85)

	// Validation failure (bad email) — errors.As extracts ValidationError
	HandleRegistration(svc, "Alan Turing", "not-an-email", 41)

	// Duplicate email — errors.Is finds ErrDuplicate through the wrapping chain
	HandleRegistration(svc, "Ada Again", "ada@example.com", 30)

	// Successful lookup
	HandleGetUser(svc, 1)

	// Not found
	HandleGetUser(svc, 999)
```

Run:

```bash
go run .
```

Expected output:

```
=== Error Wrapping Chain: Handler → Service → Repository ===
[201 Created] User: {ID:2 Name:Grace Hopper Email:grace@example.com}
[400 Bad Request] Field "email": must be a valid email address
[409 Conflict] User with this email already exists
[200 OK] User: {ID:1 Name:Ada Lovelace Email:ada@example.com}
[404 Not Found] User id=999 does not exist
```

Trace the duplicate email error mentally:

1. `repo.Create` returns `fmt.Errorf("repo create user %q: %w", email, ErrDuplicate)`
2. `svc.RegisterUser` wraps it: `fmt.Errorf("service register user: %w", err)`
3. `HandleRegistration` calls `errors.Is(err, ErrDuplicate)` — this walks the chain: service wrapper → repo wrapper → `ErrDuplicate`. Match found.

The full error string would be `"service register user: repo create user "ada@example.com": already exists"`, but the handler never prints it — it translates the error into a user-facing message. Each layer adds context; the outermost layer decides the response.

> **Java parallel:** This is equivalent to a Java exception propagating through layers with `catch` at each boundary. The difference: in Java, the exception flies up the stack automatically. In Go, each layer explicitly returns the error with added context. This makes the error path visible in the code — you can read exactly what happens when something fails.

---

## Part 5: Defer for Cleanup (4 minutes)

### Step 11 — Write a multi-resource cleanup function

Create a file called `cleanup.go`:

```go
package main

import (
	"fmt"
	"os"
	"strings"
)

// CopyFileUppercase reads a source file, uppercases the content,
// and writes it to a destination file.
// It demonstrates defer for managing multiple resources.
func CopyFileUppercase(srcPath, dstPath string) error {
	// Open source file
	src, err := os.Open(srcPath)
	if err != nil {
		return fmt.Errorf("copy uppercase: open source: %w", err)
	}
	defer src.Close() // first defer — will run last (LIFO order)

	// Create destination file
	dst, err := os.Create(dstPath)
	if err != nil {
		return fmt.Errorf("copy uppercase: create dest: %w", err)
	}
	defer dst.Close() // second defer — will run first (LIFO order)

	// Read all content from source
	buf := make([]byte, 1024)
	var content strings.Builder

	for {
		n, err := src.Read(buf)
		if n > 0 {
			content.Write(buf[:n])
		}
		if err != nil {
			if err.Error() == "EOF" {
				break
			}
			return fmt.Errorf("copy uppercase: read: %w", err)
		}
	}

	// Write uppercased content to destination
	upper := strings.ToUpper(content.String())
	if _, err := dst.WriteString(upper); err != nil {
		return fmt.Errorf("copy uppercase: write: %w", err)
	}

	fmt.Printf("[CopyFileUppercase] %s → %s (%d bytes)\n", srcPath, dstPath, len(upper))
	return nil
}
```

Note the **LIFO** (last-in, first-out) order of `defer` execution. If you open `src` first and `dst` second, the defers execute in reverse: `dst.Close()` runs before `src.Close()`. This is the same guarantee that Java's try-with-resources provides through its declaration order.

### Step 12 — Test the cleanup function

Add to `main()`:

```go
	// === Part 5: Defer for Cleanup ===
	fmt.Println("\n=== Defer for Cleanup ===")

	// Create a source file
	os.WriteFile("source.txt", []byte("defer replaces finally\ngo is explicit\n"), 0o644)
	defer os.Remove("source.txt")

	err = CopyFileUppercase("source.txt", "output.txt")
	if err != nil {
		fmt.Printf("Error: %v\n", err)
	} else {
		data, _ := os.ReadFile("output.txt")
		fmt.Printf("Output content:\n%s", string(data))
	}
	defer os.Remove("output.txt")

	// Test with non-existent source
	err = CopyFileUppercase("nope.txt", "output2.txt")
	if err != nil {
		fmt.Printf("Expected error: %v\n", err)
	}
```

Run:

```bash
go run .
```

> **Java parallel:** `defer f.Close()` replaces the entire `finally { if (reader != null) { try { reader.close(); } catch (...) {} } }` block. In Java 7+, try-with-resources (`try (var r = ...)`) achieves the same thing more concisely, but Go's `defer` is even simpler: one line, placed right after the resource is opened, guaranteed to execute on function exit.

---

## Part 6: Spot the Bug — Silent Error Ignoring (5 minutes)

### Step 13 — Read the buggy code

Create a file called `buggy.go`:

```go
package main

import (
	"fmt"
	"os"
	"strings"
)

// ProcessTransactions reads a file of transactions and returns the total.
// THIS CODE HAS MULTIPLE BUGS related to ignored errors.
// Your job: find them all.
func ProcessTransactions(path string) float64 {
	// BUG 1: Where is it?
	data, _ := os.ReadFile(path)

	lines := strings.Split(string(data), "\n")
	total := 0.0

	for _, line := range lines {
		line = strings.TrimSpace(line)
		if line == "" {
			continue
		}

		// BUG 2: Where is it?
		var amount float64
		fmt.Sscanf(line, "%f", &amount)

		total += amount
	}

	// BUG 3: Where is it?
	os.WriteFile("transactions_total.txt", []byte(fmt.Sprintf("%.2f", total)), 0o644)

	return total
}
```

### Step 14 — Find all three bugs

Study the code above. Each line marked with a `BUG` comment has an error return value that is silently discarded using `_` or simply not captured. Add the following to `main()`:

```go
	// === Part 6: Spot the Bug ===
	fmt.Println("\n=== Spot the Bug: Silent Error Ignoring ===")

	// This will "succeed" silently with a bogus result
	total := ProcessTransactions("nonexistent_transactions.txt")
	fmt.Printf("Buggy total from non-existent file: %.2f (should have been an error!)\n", total)
```

Run:

```bash
go run .
```

The function returns `0.00` without any indication that the file does not exist. In production, this would mean silently processing zero transactions and reporting a zero balance — a data integrity disaster.

### Step 15 — Write the fixed version

Create a file called `buggy_fixed.go`:

```go
package main

import (
	"fmt"
	"os"
	"strconv"
	"strings"
)

// ProcessTransactionsFixed is the corrected version with proper error handling.
func ProcessTransactionsFixed(path string) (float64, error) {
	// FIX 1: Check the error from os.ReadFile
	data, err := os.ReadFile(path)
	if err != nil {
		return 0, fmt.Errorf("process transactions: %w", err)
	}

	lines := strings.Split(string(data), "\n")
	total := 0.0
	processed := 0

	for i, line := range lines {
		line = strings.TrimSpace(line)
		if line == "" {
			continue
		}

		// FIX 2: Use strconv.ParseFloat and check the error
		amount, err := strconv.ParseFloat(line, 64)
		if err != nil {
			return 0, fmt.Errorf("process transactions: line %d: invalid amount %q: %w",
				i+1, line, err)
		}

		total += amount
		processed++
	}

	// FIX 3: Check the error from os.WriteFile
	outputPath := "transactions_total.txt"
	if err := os.WriteFile(outputPath, []byte(fmt.Sprintf("%.2f", total)), 0o644); err != nil {
		return 0, fmt.Errorf("process transactions: write output: %w", err)
	}
	defer os.Remove(outputPath)

	fmt.Printf("[ProcessTransactionsFixed] Processed %d transactions, total: %.2f\n",
		processed, total)
	return total, nil
}
```

Add to `main()`:

```go
	// Test the fixed version with a non-existent file
	_, err = ProcessTransactionsFixed("nonexistent_transactions.txt")
	if err != nil {
		fmt.Printf("Fixed version caught error: %v\n", err)
	}

	// Test the fixed version with valid data
	os.WriteFile("transactions.txt", []byte("100.50\n200.75\n-50.25\n"), 0o644)
	defer os.Remove("transactions.txt")

	total2, err := ProcessTransactionsFixed("transactions.txt")
	if err != nil {
		fmt.Printf("Error: %v\n", err)
	} else {
		fmt.Printf("Fixed total: %.2f\n", total2)
	}

	// Test the fixed version with corrupt data
	os.WriteFile("bad_transactions.txt", []byte("100.50\nNOT_A_NUMBER\n50.00\n"), 0o644)
	defer os.Remove("bad_transactions.txt")

	_, err = ProcessTransactionsFixed("bad_transactions.txt")
	if err != nil {
		fmt.Printf("Fixed version caught bad data: %v\n", err)
	}
```

Run:

```bash
go run .
```

The fixed version catches all three bugs: the missing file, the unparseable line, and the write failure. Each error includes context about where and why it failed.

**Summary of the three bugs:**

| Bug | Buggy code | Problem | Fix |
|---|---|---|---|
| 1 | `data, _ := os.ReadFile(path)` | File read error silently discarded; `data` is nil, produces empty result | Check `err` and return it |
| 2 | `fmt.Sscanf(line, "%f", &amount)` | Parse errors silently ignored; corrupt data treated as `0.0` | Use `strconv.ParseFloat` and check `err` |
| 3 | `os.WriteFile(...)` (no error check) | Write failure silently ignored; output file may not exist | Capture and check the error |

> **Key takeaway:** The `_` on an error return value is the most dangerous single character in Go. It tells the compiler "I know there might be an error, and I am choosing to ignore it." In code review, `_, _ = someFunc()` or `_ = os.WriteFile(...)` should always trigger a question: "Why are we ignoring this error?"

---

## Lab Checklist

Before you move on, verify you have completed the following:

- [ ] Translated a Java `try/catch/finally` file processor into Go using `if err != nil` and `defer`
- [ ] Defined sentinel errors (`ErrNotFound`, `ErrDuplicate`, `ErrForbidden`) as package-level variables
- [ ] Built a `UserRepository` that wraps sentinel errors with `fmt.Errorf` and `%w`
- [ ] Used `errors.Is()` to check for sentinel errors through a wrapping chain
- [ ] Created a `ValidationError` custom type that implements the `error` interface
- [ ] Used `errors.As()` to extract structured field data from a wrapped `ValidationError`
- [ ] Built a three-layer error chain (repository → service → handler) where each layer adds context
- [ ] Used `defer` for multi-resource cleanup in `CopyFileUppercase`
- [ ] **Found all three silent error bugs** in `ProcessTransactions` and understand why each is dangerous
- [ ] Wrote a fixed version that handles all errors explicitly

---

## Key Takeaways

1. **Errors are values, not exceptions.** The `error` interface is one method: `Error() string`. Any type can be an error. There is no `throw`, no `catch`, no `finally`. You check `if err != nil` and decide what to do.

2. **Wrap errors with context, not noise.** Use `fmt.Errorf("operation: %w", err)` to add context at each layer. Do not add context that duplicates what the underlying error already says.

3. **`errors.Is()` walks the chain.** It finds sentinel errors through any number of wrapping layers. Use it instead of `==` comparison, which only matches the outermost error.

4. **`errors.As()` extracts structured errors.** It finds custom error types through wrapping layers and lets you access their fields. This replaces Java's `catch (SpecificException e)` with field access.

5. **`defer` replaces `finally`.** Place it immediately after opening a resource. It runs in LIFO order on function exit. It is one line where Java needs three or more.

6. **Never ignore errors with `_`.** If you see `_, _ = riskyFunc()`, that is a bug waiting to happen. Always handle or explicitly document why an error can be safely ignored.

---

## Troubleshooting

**"undefined: errors" compile error:**
Add `"errors"` to your import block. This is the standard library `errors` package — no external dependency.

**`errors.Is()` returns false when you expect true:**
Make sure you are wrapping with `%w` (not `%v`). The `%v` verb formats the error as a string and breaks the wrapping chain. Only `%w` preserves the chain for `errors.Is()` and `errors.As()`.

**`errors.As()` not matching your custom error type:**
Make sure you are passing a pointer to a pointer: `var ve *ValidationError; errors.As(err, &ve)`. The `&ve` is critical — `errors.As` needs a `**ValidationError` to assign into.

**Defer running in unexpected order:**
Defers execute in LIFO order (last defer runs first). If you open file A then file B, file B closes before file A. This is by design and matches the natural cleanup order.
