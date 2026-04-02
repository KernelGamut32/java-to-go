# Lab 4: Interface-Driven Design

**Course:** From Java to Go — Prepared for Mastercard
**Block:** 4 — Interfaces & Dependency Injection
**Duration:** 40 minutes
**Difficulty:** Intermediate

---

## Objective

By the end of this lab, you will have defined a Go interface, built two concrete implementations of it, injected those implementations into a service struct without any framework, swapped backends at runtime, written a mock for testing, and encountered the `nil` interface trap that catches even experienced Go developers. You will understand why Go interfaces are implicit, why they should be small, and how dependency injection works without Spring.

---

## Prerequisites

- Completed Labs 1–3 (comfortable with structs, methods, pointer receivers, and factory functions)
- Familiarity with Java interfaces, `implements`, and Spring's `@Autowired` / DI container

---

## Setup

```bash
mkdir -p ~/go-labs/lab04-interfaces
cd ~/go-labs/lab04-interfaces
go mod init github.com/mastercard/lab04-interfaces
code .
```

---

## Part 1: Define the Interface (5 minutes)

### Step 1 — Define the Storer interface

Create a file called `storer.go`:

```go
package main

import "errors"

// Storer defines the contract for a key-value storage backend.
//
// Notice:
// - The interface has exactly 2 methods. Small interfaces are idiomatic in Go.
// - There is no "implements" keyword anywhere in this file.
// - Any type that has these two methods automatically satisfies Storer.
//
// Java equivalent:
//   public interface Storer {
//       void save(String key, byte[] data) throws StorageException;
//       byte[] load(String key) throws StorageException;
//   }
type Storer interface {
	Save(key string, data []byte) error
	Load(key string) ([]byte, error)
}

// Sentinel errors for storage operations.
// These are package-level variables that callers can check with errors.Is().
var (
	ErrKeyNotFound = errors.New("key not found")
	ErrKeyExists   = errors.New("key already exists")
)
```

Study this file carefully. The entire interface definition is five lines. Compare this to a typical Java interface file — the structure is the same (a contract of method signatures), but there is no `public`, no `throws`, and crucially, no type will ever write `implements Storer`. Satisfaction is implicit.

> **Java parallel:** Java interfaces are explicit contracts — a class must declare `implements Storer`. Go interfaces are structural — if a type has `Save(string, []byte) error` and `Load(string) ([]byte, error)` methods, it satisfies `Storer` automatically, even if the type was written in a completely different codebase by a different team that has never seen the `Storer` interface.

### Step 2 — Understand the design principle

Before writing any implementation, note the Go proverb at work here:

> **"The bigger the interface, the weaker the abstraction."** — Rob Pike

Go's standard library demonstrates this everywhere: `io.Reader` has one method (`Read`), `io.Writer` has one method (`Write`), and `error` has one method (`Error`). Two methods is perfectly normal. Five methods is a code smell. Ten methods means you are almost certainly writing Java in Go.

---

## Part 2: MemoryStore Implementation (5 minutes)

### Step 3 — Build an in-memory backend

Create a file called `memory_store.go`:

```go
package main

import (
	"fmt"
	"sync"
)

// MemoryStore is a Storer backed by an in-memory map.
// It is safe for concurrent use.
//
// Notice: there is NO "implements Storer" declaration.
// MemoryStore satisfies Storer because it has Save() and Load()
// with the correct signatures. The compiler verifies this at usage.
type MemoryStore struct {
	mu   sync.RWMutex
	data map[string][]byte
}

// NewMemoryStore creates and returns a ready-to-use MemoryStore.
func NewMemoryStore() *MemoryStore {
	return &MemoryStore{
		data: make(map[string][]byte),
	}
}

// Save stores data under the given key.
func (m *MemoryStore) Save(key string, data []byte) error {
	if key == "" {
		return fmt.Errorf("save: key must not be empty")
	}

	m.mu.Lock()
	defer m.mu.Unlock()

	// Make a copy of the data to prevent aliasing
	// (remember Lab 2: slices share backing arrays)
	stored := make([]byte, len(data))
	copy(stored, data)

	m.data[key] = stored
	return nil
}

// Load retrieves data for the given key.
func (m *MemoryStore) Load(key string) ([]byte, error) {
	m.mu.RLock()
	defer m.mu.RUnlock()

	val, exists := m.data[key]
	if !exists {
		return nil, fmt.Errorf("load %q: %w", key, ErrKeyNotFound)
	}

	// Return a copy to prevent the caller from modifying our internal data
	result := make([]byte, len(val))
	copy(result, val)

	return result, nil
}
```

Walk through the implementation and note the following:

- **`sync.RWMutex`** — protects the map for concurrent access. `RLock()` allows multiple simultaneous readers; `Lock()` is exclusive for writers. You will explore this further in Day 2.
- **`defer m.mu.Unlock()`** — the `defer` keyword ensures the lock is released when the function returns, even if an error occurs. This replaces Java's `try/finally` pattern.
- **Defensive copies** — both `Save` and `Load` copy the byte slices to prevent the slice aliasing bug from Lab 2. The caller cannot accidentally modify the store's internal data.
- **Error wrapping** — `fmt.Errorf("load %q: %w", key, ErrKeyNotFound)` wraps the sentinel error with context. Callers can use `errors.Is(err, ErrKeyNotFound)` to check the cause while still seeing which key was missing.
- **No `implements Storer`** — yet `MemoryStore` will satisfy the `Storer` interface when used.

---

## Part 3: FileStore Implementation (8 minutes)

### Step 4 — Build a filesystem backend

Create a file called `file_store.go`:

```go
package main

import (
	"fmt"
	"os"
	"path/filepath"
)

// FileStore is a Storer backed by the filesystem.
// Each key becomes a file in the base directory.
//
// Like MemoryStore, there is no "implements Storer" declaration.
// FileStore satisfies Storer because it has the right methods.
type FileStore struct {
	baseDir string
}

// NewFileStore creates a FileStore rooted at the given directory.
// It creates the directory if it does not exist.
func NewFileStore(baseDir string) (*FileStore, error) {
	if err := os.MkdirAll(baseDir, 0o755); err != nil {
		return nil, fmt.Errorf("create base dir %q: %w", baseDir, err)
	}
	return &FileStore{baseDir: baseDir}, nil
}

// Save writes data to a file named after the key.
func (f *FileStore) Save(key string, data []byte) error {
	if key == "" {
		return fmt.Errorf("save: key must not be empty")
	}

	path := filepath.Join(f.baseDir, key)

	if err := os.WriteFile(path, data, 0o644); err != nil {
		return fmt.Errorf("save %q: %w", key, err)
	}

	return nil
}

// Load reads data from the file named after the key.
func (f *FileStore) Load(key string) ([]byte, error) {
	path := filepath.Join(f.baseDir, key)

	data, err := os.ReadFile(path)
	if err != nil {
		if os.IsNotExist(err) {
			return nil, fmt.Errorf("load %q: %w", key, ErrKeyNotFound)
		}
		return nil, fmt.Errorf("load %q: %w", key, err)
	}

	return data, nil
}
```

Note how `FileStore` translates the `os.IsNotExist` error into the same `ErrKeyNotFound` sentinel that `MemoryStore` uses. This means callers can handle "key not found" uniformly regardless of which backend is active — they check `errors.Is(err, ErrKeyNotFound)` and it works for both.

> **Java parallel:** This is the same benefit you get from Java's interface-based programming: callers depend on the contract, not the implementation. In Java, you might throw a custom `KeyNotFoundException`. In Go, you wrap the sentinel error with context.

---

## Part 4: The Service — Dependency Injection Without a Framework (8 minutes)

### Step 5 — Build a service that accepts a Storer

Create a file called `service.go`:

```go
package main

import (
	"fmt"
)

// Service is a business logic layer that depends on a Storer.
//
// The Storer is injected via the constructor — no framework needed.
// Compare this to Spring's @Autowired or @Inject:
//
//   Java:
//   @Service
//   public class DocumentService {
//       @Autowired
//       private Storer storer;  // Spring resolves this at runtime
//   }
//
//   Go:
//   svc := NewService(memoryStore)  // You resolve it yourself. Explicitly.
type Service struct {
	store Storer
}

// NewService creates a Service with the given storage backend.
// This is constructor injection — the dependency is required, explicit,
// and impossible to forget.
func NewService(store Storer) *Service {
	return &Service{store: store}
}

// SaveDocument stores a document under the given key.
func (s *Service) SaveDocument(key string, content string) error {
	data := []byte(content)

	if err := s.store.Save(key, data); err != nil {
		return fmt.Errorf("save document %q: %w", key, err)
	}

	fmt.Printf("[Service] Saved document %q (%d bytes)\n", key, len(data))
	return nil
}

// LoadDocument retrieves a document by key.
func (s *Service) LoadDocument(key string) (string, error) {
	data, err := s.store.Load(key)
	if err != nil {
		return "", fmt.Errorf("load document %q: %w", key, err)
	}

	fmt.Printf("[Service] Loaded document %q (%d bytes)\n", key, len(data))
	return string(data), nil
}
```

The crucial line is the `NewService` signature:

```go
func NewService(store Storer) *Service
```

The parameter type is **`Storer` (the interface)**, not `*MemoryStore` or `*FileStore`. This is the Go principle of **"accept interfaces, return structs."** The service does not know or care which backend it is using. It only knows that the backend can `Save` and `Load`.

> **Java parallel:** In Spring, you write `@Autowired Storer storer` and the framework resolves the bean at startup using classpath scanning, component registration, and a dependency injection container. In Go, you write `NewService(myStore)` and pass the dependency yourself. It is one line of code instead of an entire framework. The trade-off: you wire things manually, but the dependency graph is explicit, visible, and has no magic.

### Step 6 — Wire it up and swap implementations

Create `main.go`:

```go
package main

import (
	"errors"
	"fmt"
	"os"
)

func main() {
	// ============================================================
	// Backend 1: MemoryStore
	// ============================================================
	fmt.Println("=== Using MemoryStore ===")

	memStore := NewMemoryStore()
	svc := NewService(memStore) // inject MemoryStore as a Storer

	runDemo(svc)

	// ============================================================
	// Backend 2: FileStore (swap with zero code changes to Service)
	// ============================================================
	fmt.Println("\n=== Using FileStore ===")

	// Create a temp directory for the file store
	tmpDir, err := os.MkdirTemp("", "lab04-*")
	if err != nil {
		fmt.Fprintf(os.Stderr, "failed to create temp dir: %v\n", err)
		os.Exit(1)
	}
	defer os.RemoveAll(tmpDir) // clean up when main exits

	fileStore, err := NewFileStore(tmpDir)
	if err != nil {
		fmt.Fprintf(os.Stderr, "failed to create file store: %v\n", err)
		os.Exit(1)
	}

	svc = NewService(fileStore) // inject FileStore as a Storer — same Service

	runDemo(svc)
}

// runDemo exercises the Service with save, load, and error handling.
// It accepts *Service, which internally uses whatever Storer was injected.
func runDemo(svc *Service) {
	// Save a document
	err := svc.SaveDocument("readme.txt", "This is the README content.")
	if err != nil {
		fmt.Printf("Error saving: %v\n", err)
		return
	}

	// Load the document
	content, err := svc.LoadDocument("readme.txt")
	if err != nil {
		fmt.Printf("Error loading: %v\n", err)
		return
	}
	fmt.Printf("Content: %s\n", content)

	// Try to load a non-existent document
	_, err = svc.LoadDocument("nonexistent.txt")
	if err != nil {
		// Check if it's a "key not found" error using errors.Is
		if errors.Is(err, ErrKeyNotFound) {
			fmt.Printf("Expected error: %v\n", err)
		} else {
			fmt.Printf("Unexpected error: %v\n", err)
		}
	}
}
```

Run:

```bash
go run .
```

Expected output:

```
=== Using MemoryStore ===
[Service] Saved document "readme.txt" (27 bytes)
[Service] Loaded document "readme.txt" (27 bytes)
Content: This is the README content.
Expected error: load document "nonexistent.txt": load "nonexistent.txt": key not found

=== Using FileStore ===
[Service] Saved document "readme.txt" (27 bytes)
[Service] Loaded document "readme.txt" (27 bytes)
Content: This is the README content.
Expected error: load document "nonexistent.txt": load "nonexistent.txt": key not found
```

Both backends produce identical behavior from the `Service`'s perspective. The `Service` code did not change at all between backends — only the value passed to `NewService()` changed. That is dependency injection. No annotations. No container. No XML. One line of wiring code.

### Step 7 — Verify implicit interface satisfaction

You never wrote `implements Storer` anywhere. To prove that the compiler is checking this for you, add a compile-time assertion at the bottom of `memory_store.go`:

```go
// Compile-time check: *MemoryStore must satisfy Storer.
// This line produces no runtime code — it is a zero-cost assertion.
var _ Storer = (*MemoryStore)(nil)
```

And at the bottom of `file_store.go`:

```go
// Compile-time check: *FileStore must satisfy Storer.
var _ Storer = (*FileStore)(nil)
```

Run:

```bash
go build .
```

No errors means the assertion passed. Now break it intentionally — rename `Save` to `SaveData` in `MemoryStore` (just temporarily) and run `go build .` again. You will see an error like:

```
cannot use (*MemoryStore)(nil) (value of type *MemoryStore) as Storer value in variable declaration:
    *MemoryStore does not implement Storer (missing method Save)
```

Restore the method name to `Save` before continuing.

> **Best practice:** The `var _ Storer = (*MemoryStore)(nil)` pattern is idiomatic Go. It catches interface compliance errors at compile time rather than at the point of use. Many Go projects include these assertions at the bottom of implementation files.

---

## Part 5: Mock Implementation for Testing (7 minutes)

### Step 8 — Build a mock Storer

Create a file called `mock_store.go`:

```go
package main

import "fmt"

// MockStore is a test double that records calls and returns preconfigured responses.
// In Go, you write mocks by hand — no Mockito, no framework needed.
//
// This is a preview of Day 2 testing. For now, focus on how the
// interface makes this possible without modifying Service.
type MockStore struct {
	// SaveFunc and LoadFunc let each test configure custom behavior.
	// If nil, the mock uses default behavior.
	SaveFunc func(key string, data []byte) error
	LoadFunc func(key string) ([]byte, error)

	// SaveCalls records every call to Save for later assertions.
	SaveCalls []MockSaveCall
	LoadCalls []string
}

// MockSaveCall records the arguments of a single Save call.
type MockSaveCall struct {
	Key  string
	Data []byte
}

// Save delegates to SaveFunc if set, otherwise succeeds silently.
func (m *MockStore) Save(key string, data []byte) error {
	m.SaveCalls = append(m.SaveCalls, MockSaveCall{Key: key, Data: data})

	if m.SaveFunc != nil {
		return m.SaveFunc(key, data)
	}
	return nil
}

// Load delegates to LoadFunc if set, otherwise returns ErrKeyNotFound.
func (m *MockStore) Load(key string) ([]byte, error) {
	m.LoadCalls = append(m.LoadCalls, key)

	if m.LoadFunc != nil {
		return m.LoadFunc(key, data)
	}
	return nil, ErrKeyNotFound
}

// Compile-time check: *MockStore must satisfy Storer.
var _ Storer = (*MockStore)(nil)
```

**Wait — there is a deliberate bug in this file.** Before running, read the `Load` method carefully. The `LoadFunc` call passes `data` as a second argument, but `data` is not defined in `Load`'s scope. This will not compile.

### Step 9 — Find and fix the compile error

Run:

```bash
go build .
```

You will see an error on the `LoadFunc` call. Fix it by removing the undefined `data` argument:

```go
// Load delegates to LoadFunc if set, otherwise returns ErrKeyNotFound.
func (m *MockStore) Load(key string) ([]byte, error) {
	m.LoadCalls = append(m.LoadCalls, key)

	if m.LoadFunc != nil {
		return m.LoadFunc(key)
	}
	return nil, ErrKeyNotFound
}
```

Run `go build .` again to confirm it compiles.

### Step 10 — Use the mock to test Service behavior

Add the following to `main()`, after the FileStore demo:

```go
	// ============================================================
	// Backend 3: MockStore (for testing)
	// ============================================================
	fmt.Println("\n=== Using MockStore ===")

	mock := &MockStore{
		// Configure Load to return specific data for "config.json"
		LoadFunc: func(key string) ([]byte, error) {
			if key == "config.json" {
				return []byte(`{"env": "test", "debug": true}`), nil
			}
			return nil, ErrKeyNotFound
		},
	}

	svc = NewService(mock) // inject the mock — Service doesn't know the difference

	// Save something — the mock records the call but doesn't persist
	_ = svc.SaveDocument("audit.log", "user logged in")

	// Load the preconfigured response
	content, err = svc.LoadDocument("config.json")
	if err != nil {
		fmt.Printf("Error: %v\n", err)
	} else {
		fmt.Printf("Config: %s\n", content)
	}

	// Verify that Save was called with the right arguments
	fmt.Printf("\nMock recorded %d Save call(s):\n", len(mock.SaveCalls))
	for i, call := range mock.SaveCalls {
		fmt.Printf("  [%d] key=%q data=%q\n", i, call.Key, string(call.Data))
	}

	fmt.Printf("Mock recorded %d Load call(s): %v\n", len(mock.LoadCalls), mock.LoadCalls)
```

Run:

```bash
go run .
```

The mock lets you control exactly what `Load` returns and verify exactly what `Save` received — without touching the filesystem or an in-memory map. The `Service` has no idea it is talking to a mock. This is the power of interface-based design.

> **Java parallel:** In Java, you would use Mockito: `when(storer.load("config.json")).thenReturn(...)` and `verify(storer).save(eq("audit.log"), any())`. In Go, you write a small struct with function fields. It is more verbose but has zero dependencies and is completely transparent — no reflection, no proxies, no bytecode manipulation.

---

## Part 6: The Nil Interface Trap (7 minutes)

This is the most subtle bug in Go's interface system. Every Go developer encounters it eventually. Understanding it now will save you hours of debugging later.

### Step 11 — Understand how interfaces are represented internally

A Go interface value is stored as a pair of pointers internally:

```
┌──────────────────────────┐
│  Interface Value         │
│  ┌────────┬───────────┐  │
│  │  type  │  pointer  │  │
│  └────────┴───────────┘  │
└──────────────────────────┘
```

- **type** — which concrete type is stored (e.g., `*MemoryStore`)
- **pointer** — the actual data (e.g., the address of the `MemoryStore`)

An interface is `nil` **only when both fields are nil** — no type and no pointer. If a type is set but the pointer is nil, the interface is **not nil**.

### Step 12 — Demonstrate the trap

Create a file called `nil_trap.go`:

```go
package main

import "fmt"

// getStore simulates a function that conditionally returns a Storer.
// THIS FUNCTION HAS A SUBTLE BUG.
func getStore(useFile bool) Storer {
	if useFile {
		// Pretend we failed to create a file store.
		// A Java developer would return null here.
		var fs *FileStore // fs is a nil *FileStore pointer
		return fs         // BUG: returns a non-nil interface holding a nil pointer
	}
	return NewMemoryStore()
}

// getStoreFixed is the correct version.
func getStoreFixed(useFile bool) Storer {
	if useFile {
		// Return nil explicitly — this produces a truly nil interface
		return nil
	}
	return NewMemoryStore()
}

// DemoNilTrap shows the difference between a nil interface and
// an interface holding a nil concrete value.
func DemoNilTrap() {
	fmt.Println("\n=== The Nil Interface Trap ===")

	// --- The buggy version ---
	store := getStore(true)

	// This check PASSES even though the underlying pointer is nil!
	if store != nil {
		fmt.Println("Buggy:  store != nil is TRUE (surprise!)")
		fmt.Printf("        Type: %T, Value: %v\n", store, store)

		// This will PANIC at runtime because the concrete pointer is nil
		// Uncomment to see the panic:
		// store.Save("key", []byte("data"))
	}

	// --- The fixed version ---
	storeFixed := getStoreFixed(true)

	if storeFixed != nil {
		fmt.Println("Fixed:  store != nil is TRUE")
	} else {
		fmt.Println("Fixed:  store == nil is TRUE (correct!)")
	}

	// --- Show the internal representation ---
	fmt.Println("\n--- Why This Happens ---")

	var nilInterface Storer           // truly nil: type=nil, pointer=nil
	var nilPointer *MemoryStore       // nil pointer to a concrete type
	var wrappedNil Storer = nilPointer // non-nil interface: type=*MemoryStore, pointer=nil

	fmt.Printf("nilInterface == nil: %t  (type=%-20T value=%v)\n",
		nilInterface == nil, nilInterface, nilInterface)
	fmt.Printf("wrappedNil   == nil: %t  (type=%-20T value=%v)\n",
		wrappedNil == nil, wrappedNil, wrappedNil)
	fmt.Printf("nilPointer   == nil: %t  (type=%-20T value=%v)\n",
		nilPointer == nil, nilPointer, nilPointer)
}
```

### Step 13 — Run and study the output

Add to `main()`:

```go
	DemoNilTrap()
```

Run:

```bash
go run .
```

Expected output:

```
=== The Nil Interface Trap ===
Buggy:  store != nil is TRUE (surprise!)
        Type: *main.FileStore, Value: <nil>
Fixed:  store == nil is TRUE (correct!)

--- Why This Happens ---
nilInterface == nil: true   (type=<nil>                value=<nil>)
wrappedNil   == nil: false  (type=*main.MemoryStore    value=<nil>)
nilPointer   == nil: true   (type=*main.MemoryStore    value=<nil>)
```

Read the `wrappedNil` line carefully. The interface is **not nil** because it has type information (`*MemoryStore`) even though the pointer itself is nil. The `!= nil` check passes, and calling any method on it will panic.

### Step 14 — Internalize the rule

The rule is simple:

> **Never return a typed nil pointer as an interface.** Always return the bare `nil` when you mean "no value."

```go
// ✗ WRONG — returns a non-nil interface holding a nil pointer
var p *FileStore
return p

// ✓ CORRECT — returns a truly nil interface
return nil
```

This trap does not exist in Java because Java's `null` is untyped — `null instanceof Storer` is always `false`, and a null check always works. Go's interface nil check is more nuanced because interfaces carry type information.

> **Java parallel:** Imagine if Java had a `null` that knew its type — `null` as a `FileStore` reference would still pass `if (storer != null)`. That is exactly what Go does when you return a typed nil through an interface. The fix is always to return the interface's own `nil`, not a concrete type's nil.

---

## Lab Checklist

Before you move on, verify you have completed the following:

- [ ] Defined a `Storer` interface with two methods (`Save` and `Load`) — no `implements` keyword anywhere
- [ ] Built `MemoryStore` with a `sync.RWMutex`-protected map that satisfies `Storer` implicitly
- [ ] Built `FileStore` backed by the filesystem that satisfies `Storer` implicitly
- [ ] Added compile-time interface satisfaction checks (`var _ Storer = (*MemoryStore)(nil)`)
- [ ] Created `Service` with constructor injection — `NewService(store Storer)` — and swapped backends by passing different implementations
- [ ] Verified that the same `runDemo()` function works identically with `MemoryStore`, `FileStore`, and `MockStore`
- [ ] Built a `MockStore` with configurable function fields and call recording
- [ ] **Demonstrated and understood the nil interface trap** — an interface holding a nil concrete pointer is not nil
- [ ] Know the fix: return bare `nil`, never a typed nil pointer, when returning an interface

---

## Key Takeaways

1. **Interfaces are implicit.** A type satisfies an interface by having the right methods — no `implements` keyword. This means you can define an interface *after* the concrete types exist, and you can define interfaces in the consumer's package rather than the provider's.

2. **Small interfaces are powerful.** `Storer` has two methods. Go's standard library interfaces are often one method: `io.Reader`, `io.Writer`, `error`, `fmt.Stringer`. Resist the urge to build large interfaces — break them into composable pieces.

3. **"Accept interfaces, return structs."** `NewService` accepts `Storer` (interface) but returns `*Service` (concrete struct). `NewMemoryStore` returns `*MemoryStore` (concrete struct), not `Storer`. This pattern keeps your API flexible for callers while keeping return types concrete and inspectable.

4. **Dependency injection needs no framework.** Pass the dependency through the constructor. This makes the dependency graph explicit, testable, and visible in the code. No scanning, no annotations, no container startup time.

5. **The nil interface trap is real.** An interface is nil only when both its type and value are nil. Returning a typed nil pointer through an interface produces a non-nil interface that will panic when used. Always return bare `nil` for "no value."

---

## Troubleshooting

**"missing method Save" compile error:**
Check that your method signatures exactly match the interface: `Save(key string, data []byte) error`. Parameter names do not need to match, but types and order must be identical.

**"cannot use (*MemoryStore)(nil) as Storer" error:**
This means `MemoryStore` does not satisfy the `Storer` interface. Check your method signatures — the most common issue is a value receiver (`func (m MemoryStore)`) when the interface requires methods on the pointer type. If any method uses a pointer receiver, the compile-time check must use `(*MemoryStore)(nil)`, not `(MemoryStore)(nil)`.

**FileStore creates files in unexpected locations:**
Make sure you are using `os.MkdirTemp` to create a temporary directory as shown in Step 6. The `defer os.RemoveAll(tmpDir)` cleans up when `main()` exits.

**MockStore panic: "invalid memory address":**
Check that `LoadFunc` is set before calling `LoadDocument`. If `LoadFunc` is nil, the default behavior returns `ErrKeyNotFound` without panicking. If you see a panic, you may have an uninitialized function field that was called directly.

**Nil interface trap confusion:**
Re-read Step 11 slowly. The key mental model: an interface value is a `(type, pointer)` pair. `nil` means `(nil, nil)`. A typed nil is `(*FileStore, nil)` — the type slot is filled, so the interface is not nil.
