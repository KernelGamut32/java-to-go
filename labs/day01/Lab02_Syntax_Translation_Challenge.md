# Lab 2: Syntax Translation Challenge

**Course:** From Java to Go — Prepared for Mastercard
**Block:** 2 — Go Syntax Fundamentals
**Duration:** 65 minutes
**Difficulty:** Introductory to Intermediate

---

## Objective

By the end of this lab, you will be comfortable declaring variables with both `var` and `:=`, working with Go's core collection types (slices and maps), writing functions with multiple return values, using higher-order functions, and organizing code across multiple files with Go's visibility rules. You will also encounter your first Go "gotcha" — slice aliasing — so you can recognize it before it bites you in production.

---

## Prerequisites

- Completed Lab 1 (Go 1.25+ installed, VS Code configured with the Go extension)
- Familiarity with Java's `ArrayList`, `HashMap`, lambdas, and access modifiers

---

## Part 1: Variables, Type Inference & Zero Values (10 minutes)

### Step 1 — Create the lab project

```bash
mkdir -p ~/go-labs/lab02-syntax
cd ~/go-labs/lab02-syntax
go mod init github.com/mastercard/lab02-syntax
code .
```

### Step 2 — Explore variable declarations and zero values

Create a file called `main.go` and type the following:

```go
package main

import "fmt"

func main() {
	// === var declarations with explicit types ===
	var count int
	var name string
	var active bool
	var rate float64
	var data []int       // slice
	var lookup map[string]int // map

	fmt.Println("=== Zero Values ===")
	fmt.Printf("int:     %d\n", count)
	fmt.Printf("string:  %q\n", name)   // %q shows quotes around the empty string
	fmt.Printf("bool:    %t\n", active)
	fmt.Printf("float64: %f\n", rate)
	fmt.Printf("slice:   %v (nil? %t, len: %d)\n", data, data == nil, len(data))
	fmt.Printf("map:     %v (nil? %t)\n", lookup, lookup == nil)
}
```

Run it:

```bash
go run main.go
```

Expected output:

```
=== Zero Values ===
int:     0
string:  ""
bool:    false
float64: 0.000000
slice:   [] (nil? true, len: 0)
map:     map[] (nil? true)
```

Study the output carefully. Every type in Go has a **zero value** — there is no `null` in the way Java uses it. An uninitialized `int` is `0`, not `null`. An uninitialized `string` is `""`, not `null`. Slices and maps are `nil` when uninitialized, but a `nil` slice still has a length of `0` and you can safely call `len()` on it without a panic.

> **Java parallel:** In Java, uninitialized local variables cause a compile error. Uninitialized object fields default to `null`, `0`, or `false`. Go's zero values are similar to Java's field defaults, but they apply everywhere — locals included — and are always safe to use.

### Step 3 — Short variable declarations

Add the following below the existing code inside `main()`:

```go
	// === Short variable declaration (:=) ===
	fmt.Println("\n=== Short Declarations ===")

	city := "Kansas City"          // inferred as string
	population := 508_090          // inferred as int (underscores for readability)
	latitude := 39.0997            // inferred as float64
	growing := true                // inferred as bool

	fmt.Printf("city:       %s (%T)\n", city, city)
	fmt.Printf("population: %d (%T)\n", population, population)
	fmt.Printf("latitude:   %f (%T)\n", latitude, latitude)
	fmt.Printf("growing:    %t (%T)\n", growing, growing)
```

Run again and observe the `%T` verb printing each variable's inferred type.

> **Key rule:** Use `:=` inside functions for concise declarations. Use `var` when you want an explicit type, when you want the zero value without assigning, or at the package level (`:=` is not allowed outside functions).

### Step 4 — Constants and iota

Add the following below your existing code inside `main()`:

```go
	// === Constants and iota ===
	fmt.Println("\n=== Constants & Iota ===")

	const maxRetries = 3
	const apiURL = "https://api.mastercard.com"

	fmt.Printf("maxRetries: %d\n", maxRetries)
	fmt.Printf("apiURL:     %s\n", apiURL)
```

Now create a new file called `status.go` in the same directory:

```go
package main

import "fmt"

// Status represents an HTTP-like status category.
// iota auto-increments starting from 0.
type Status int

const (
	StatusPending    Status = iota // 0
	StatusProcessing               // 1
	StatusCompleted                // 2
	StatusFailed                   // 3
)

func (s Status) String() string {
	switch s {
	case StatusPending:
		return "PENDING"
	case StatusProcessing:
		return "PROCESSING"
	case StatusCompleted:
		return "COMPLETED"
	case StatusFailed:
		return "FAILED"
	default:
		return fmt.Sprintf("UNKNOWN(%d)", s)
	}
}
```

Back in `main.go`, add inside `main()`:

```go
	// Using iota-based constants
	status := StatusCompleted
	fmt.Printf("status:     %s (underlying: %d)\n", status, int(status))
```

Run with:

```bash
go run .
```

Note the use of `go run .` (with a dot) instead of `go run main.go`. The dot tells Go to compile **all** `.go` files in the current directory. This is necessary now that you have two source files in the same package.

> **Java parallel:** `iota` serves a similar purpose to Java enums, but it is simpler — it is just an auto-incrementing integer within a `const` block. There is no `enum` keyword in Go.

---

## Part 2: Multiple Return Values (10 minutes)

### Step 5 — Write a function with multiple return values

Create a new file called `mathutil.go`:

```go
package main

// SumAndAverage takes a slice of integers and returns the sum and the average.
// If the slice is empty, it returns 0, 0.
func SumAndAverage(numbers []int) (int, float64) {
	if len(numbers) == 0 {
		return 0, 0.0
	}

	sum := 0
	for _, n := range numbers {
		sum += n
	}

	avg := float64(sum) / float64(len(numbers))
	return sum, avg
}
```

Observe the following:

- **`(int, float64)`** — the function returns two values. No wrapper class needed, no `Pair<Integer, Double>`, no custom result object.
- **`float64(sum)`** — explicit type conversion. Go has no implicit numeric conversions. You must convert `int` to `float64` yourself.
- **`for _, n := range numbers`** — `range` iterates over the slice. The `_` discards the index (similar to Java's enhanced for-each loop).
- **`len(numbers)`** — a built-in function. No `.size()` method call.

### Step 6 — Call the function from main

Add the following to `main()` in `main.go`:

```go
	// === Multiple Return Values ===
	fmt.Println("\n=== Multiple Return Values ===")

	scores := []int{92, 87, 76, 95, 88}
	sum, avg := SumAndAverage(scores)
	fmt.Printf("Scores: %v\n", scores)
	fmt.Printf("Sum: %d, Average: %.2f\n", sum, avg)

	// Discarding a return value with _
	_, avgOnly := SumAndAverage(scores)
	fmt.Printf("Average only: %.2f\n", avgOnly)

	// Empty slice case
	emptySum, emptyAvg := SumAndAverage([]int{})
	fmt.Printf("Empty: Sum=%d, Average=%.2f\n", emptySum, emptyAvg)
```

Run:

```bash
go run .
```

> **Java parallel:** In Java 23, returning multiple values requires either a record, a custom class, or an array. Go makes this a first-class language feature. The `result, err` convention (which you will use constantly starting in Block 5) is built on this.

### Step 7 — Named return values

Create a file called `namedreturns.go`:

```go
package main

// MinMax returns the minimum and maximum values from a slice.
// Named return values document what the function returns.
func MinMax(numbers []int) (min int, max int) {
	if len(numbers) == 0 {
		return 0, 0
	}

	min = numbers[0]
	max = numbers[0]

	for _, n := range numbers[1:] {
		if n < min {
			min = n
		}
		if n > max {
			max = n
		}
	}

	return min, max
}
```

Add to `main()`:

```go
	// Named return values
	lo, hi := MinMax(scores)
	fmt.Printf("Min: %d, Max: %d\n", lo, hi)
```

Run and verify. Named return values act as documentation — they tell the caller what each position means. In this lab, we use explicit `return min, max` rather than a bare `return` for clarity.

> **Best practice:** Named return values are useful for documentation, but avoid bare `return` statements (returning without listing the values). Bare returns reduce readability, especially in longer functions.

---

## Part 3: Word Frequency Counter with Maps (10 minutes)

### Step 8 — Build a word frequency counter

Create a file called `wordcount.go`:

```go
package main

import "strings"

// CountWords takes a block of text and returns a map of word frequencies.
// Words are normalized to lowercase.
func CountWords(text string) map[string]int {
	freq := make(map[string]int)

	words := strings.Fields(text) // splits on any whitespace
	for _, word := range words {
		normalized := strings.ToLower(word)
		freq[normalized]++
	}

	return freq
}
```

Notice:

- **`make(map[string]int)`** — you must initialize a map with `make` before writing to it. A `nil` map can be read (returns zero values) but **panics on write**.
- **`freq[normalized]++`** — if the key does not exist, the zero value for `int` (which is `0`) is returned, then incremented to `1`. No `containsKey()` check needed. No `getOrDefault()`.
- **`strings.Fields(text)`** — splits on any whitespace and trims automatically. Cleaner than Java's `split("\\s+")`.

### Step 9 — Use the counter and iterate the map

Add to `main()`:

```go
	// === Maps: Word Frequency Counter ===
	fmt.Println("\n=== Word Frequency Counter ===")

	text := "Go is simple Go is fast Go is not Java but Java developers love Go"
	freq := CountWords(text)

	fmt.Printf("Word frequencies for: %q\n", text)
	for word, count := range freq {
		fmt.Printf("  %-12s → %d\n", word, count)
	}

	// Checking if a key exists (the "comma ok" idiom)
	count, exists := freq["go"]
	fmt.Printf("\n\"go\" appears %d times (exists: %t)\n", count, exists)

	count, exists = freq["rust"]
	fmt.Printf("\"rust\" appears %d times (exists: %t)\n", count, exists)

	// Deleting a key
	delete(freq, "is")
	fmt.Printf("After deleting \"is\": %v\n", freq)
```

Run:

```bash
go run .
```

Note that **map iteration order is randomized** in Go. If you run the program multiple times, the words may print in a different order each time. This is intentional — it prevents developers from accidentally depending on insertion order.

> **Java parallel:** Go's `map[string]int` replaces Java's `HashMap<String, Integer>`. The "comma ok" idiom (`count, exists := freq["go"]`) replaces Java's `containsKey()` + `get()`. The built-in `delete()` function replaces Java's `remove()`.

---

## Part 4: Slices Deep Dive (12 minutes)

### Step 10 — Slice basics: append and slice expressions

Create a file called `slicedemo.go`:

```go
package main

import "fmt"

// DemoSlices demonstrates core slice operations.
func DemoSlices() {
	fmt.Println("=== Slice Basics ===")

	// Creating a slice with a literal
	fruits := []string{"apple", "banana", "cherry", "date", "elderberry"}
	fmt.Printf("fruits:  %v (len=%d, cap=%d)\n", fruits, len(fruits), cap(fruits))

	// Slice expression [low:high] — creates a view, NOT a copy
	firstThree := fruits[0:3]
	lastTwo := fruits[3:]
	fmt.Printf("firstThree: %v\n", firstThree)
	fmt.Printf("lastTwo:    %v\n", lastTwo)

	// Appending to a slice
	fruits = append(fruits, "fig", "grape")
	fmt.Printf("after append: %v (len=%d, cap=%d)\n", fruits, len(fruits), cap(fruits))

	// Observing capacity growth
	fmt.Println("\n=== Capacity Growth ===")
	var numbers []int
	prevCap := 0
	for i := 0; i < 20; i++ {
		numbers = append(numbers, i)
		if cap(numbers) != prevCap {
			fmt.Printf("  len=%-3d cap changed: %d → %d\n", len(numbers), prevCap, cap(numbers))
			prevCap = cap(numbers)
		}
	}
}
```

Add to `main()`:

```go
	// === Slices Deep Dive ===
	fmt.Println()
	DemoSlices()
```

Run and observe how capacity grows. When a slice's length exceeds its capacity, Go allocates a new, larger backing array and copies the elements. The growth factor is implementation-defined but generally doubles for small slices and grows by about 25% for larger ones.

> **Java parallel:** This is similar to `ArrayList`'s internal array resizing, but in Go, the mechanism is visible and controllable. You can pre-allocate capacity with `make([]int, 0, expectedSize)` — similar to `new ArrayList<>(expectedSize)`.

### Step 11 — Pre-allocating slices with make

Add the following function to `slicedemo.go`:

```go
// DemoMakeSlice shows how to pre-allocate slice capacity.
func DemoMakeSlice() {
	fmt.Println("\n=== Pre-allocating with make ===")

	// make([]T, length, capacity)
	// length = number of elements (initialized to zero values)
	// capacity = size of backing array

	// Common mistake: make([]int, 10) creates 10 zero-valued elements
	wrong := make([]int, 10)
	wrong = append(wrong, 42)
	fmt.Printf("wrong: %v (len=%d) — 42 is at index 10, not 0!\n", wrong, len(wrong))

	// Correct: make([]int, 0, 10) creates an empty slice with room for 10
	right := make([]int, 0, 10)
	right = append(right, 42)
	fmt.Printf("right: %v (len=%d, cap=%d)\n", right, len(right), cap(right))
}
```

Add to `main()`:

```go
	DemoMakeSlice()
```

Run and study the difference. The `make([]int, 10)` mistake is one of the most common Go beginner errors — it creates a slice with 10 zeros and then appends after them instead of into them.

### Step 12 — The slice aliasing pitfall

This is the most important part of the slice section. Add to `slicedemo.go`:

```go
// DemoSliceAliasing demonstrates that slices share backing arrays.
func DemoSliceAliasing() {
	fmt.Println("\n=== ⚠️  Slice Aliasing Pitfall ===")

	original := []string{"Go", "Java", "Python", "Rust"}
	fmt.Printf("original: %v\n", original)

	// Create a sub-slice — this is a VIEW into the same backing array
	sub := original[1:3]
	fmt.Printf("sub (original[1:3]): %v\n", sub)

	// Modify the sub-slice
	sub[0] = "CHANGED"
	fmt.Printf("\nAfter sub[0] = \"CHANGED\":\n")
	fmt.Printf("  sub:      %v\n", sub)
	fmt.Printf("  original: %v  ← original[1] changed too!\n", original)

	// To avoid aliasing, make an explicit copy
	fmt.Println("\n=== Safe Copy ===")
	original2 := []string{"Go", "Java", "Python", "Rust"}
	safeCopy := make([]string, 2)
	copy(safeCopy, original2[1:3])

	safeCopy[0] = "SAFE"
	fmt.Printf("safeCopy:  %v\n", safeCopy)
	fmt.Printf("original2: %v  ← unchanged\n", original2)
}
```

Add to `main()`:

```go
	DemoSliceAliasing()
```

Run and study the output carefully. When you create a sub-slice with `original[1:3]`, you are **not** creating a new array. You are creating a new slice header that points to the same underlying memory. Modifying the sub-slice modifies the original.

> **Java parallel:** Java's `List.subList()` has the same aliasing behavior — changes to the sublist are reflected in the original list. But Java's `Arrays.copyOfRange()` creates an independent copy. In Go, you must use the built-in `copy()` function to achieve the same independence. This is a critical distinction that causes real bugs in production code.

---

## Part 5: Higher-Order Functions (8 minutes)

### Step 13 — Write a generic filter function

Create a file called `functional.go`:

```go
package main

// Filter returns a new slice containing only the elements
// for which the predicate function returns true.
func Filter(numbers []int, predicate func(int) bool) []int {
	var result []int
	for _, n := range numbers {
		if predicate(n) {
			result = append(result, n)
		}
	}
	return result
}

// Map applies a transform function to each element and returns a new slice.
func Map(numbers []int, transform func(int) int) []int {
	result := make([]int, len(numbers))
	for i, n := range numbers {
		result[i] = transform(n)
	}
	return result
}

// Reduce folds a slice into a single value using an accumulator function.
func Reduce(numbers []int, initial int, accumulate func(int, int) int) int {
	result := initial
	for _, n := range numbers {
		result = accumulate(result, n)
	}
	return result
}
```

### Step 14 — Use the higher-order functions

Add to `main()`:

```go
	// === Higher-Order Functions ===
	fmt.Println("\n=== Higher-Order Functions ===")

	numbers := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

	// Filter: keep only even numbers
	evens := Filter(numbers, func(n int) bool {
		return n%2 == 0
	})
	fmt.Printf("Evens:   %v\n", evens)

	// Filter: keep numbers greater than 5
	gtFive := Filter(numbers, func(n int) bool {
		return n > 5
	})
	fmt.Printf("GT 5:    %v\n", gtFive)

	// Map: square each number
	squared := Map(numbers, func(n int) int {
		return n * n
	})
	fmt.Printf("Squared: %v\n", squared)

	// Reduce: sum all numbers
	total := Reduce(numbers, 0, func(acc, n int) int {
		return acc + n
	})
	fmt.Printf("Sum:     %d\n", total)

	// Chaining: sum of squares of even numbers
	result := Reduce(
		Map(
			Filter(numbers, func(n int) bool { return n%2 == 0 }),
			func(n int) int { return n * n },
		),
		0,
		func(acc, n int) int { return acc + n },
	)
	fmt.Printf("Sum of squares of evens: %d\n", result)
```

Run:

```bash
go run .
```

> **Java parallel:** Go's anonymous functions (`func(n int) bool { ... }`) serve the same role as Java lambdas (`n -> n % 2 == 0`). The syntax is more verbose, but the behavior is identical. Note that Go does not have a `Stream` API — these patterns are built with plain functions and slices. Starting with Go 1.23, the standard library includes `slices` and `maps` packages with helper functions, but most Go developers still write explicit loops for clarity.

### Step 15 — Closures: functions that capture state

Add to `main()`:

```go
	// === Closures ===
	fmt.Println("\n=== Closures ===")

	// A closure that captures and modifies an outer variable
	counter := 0
	increment := func() int {
		counter++
		return counter
	}

	fmt.Printf("Call 1: %d\n", increment())
	fmt.Printf("Call 2: %d\n", increment())
	fmt.Printf("Call 3: %d\n", increment())
	fmt.Printf("counter variable: %d\n", counter) // counter was modified by the closure
```

The closure `increment` captures `counter` by reference — calling `increment()` modifies the same `counter` variable in the outer scope. This is the same behavior as Java lambdas capturing effectively-final variables — except in Go, the captured variable **can** be mutated.

---

## Part 6: Packages & Visibility (10 minutes)

### Step 16 — Create a multi-file package structure

You will now create a separate package to experience Go's visibility rules. From your terminal:

```bash
mkdir -p ~/go-labs/lab02-syntax/stringutil
```

Create the file `stringutil/reverse.go`:

```go
package stringutil

// Reverse returns the reverse of a string.
// This function is exported (starts with uppercase R).
func Reverse(s string) string {
	runes := []rune(s)
	reverseRunes(runes)
	return string(runes)
}

// reverseRunes reverses a slice of runes in place.
// This function is unexported (starts with lowercase r).
// It can only be called from within the stringutil package.
func reverseRunes(runes []rune) {
	for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
		runes[i], runes[j] = runes[j], runes[i]
	}
}
```

Now create a second file in the same package, `stringutil/analyze.go`:

```go
package stringutil

import "unicode"

// WordCount is an exported type — visible outside this package.
type WordCount struct {
	Word  string // exported field
	Count int    // exported field
	source string // unexported field — only accessible within stringutil
}

// NewWordCount creates a WordCount and records the source.
// Factory functions replace Java constructors.
func NewWordCount(word string, count int, source string) WordCount {
	return WordCount{
		Word:   word,
		Count:  count,
		source: source,
	}
}

// Source returns the unexported source field.
// This is the accessor pattern when you need controlled read access.
func (wc WordCount) Source() string {
	return wc.source
}

// IsTitle reports whether the word starts with an uppercase letter.
func IsTitle(s string) bool {
	if len(s) == 0 {
		return false
	}
	runes := []rune(s)
	return unicode.IsUpper(runes[0])
}
```

### Step 17 — Use the package from main

Update `main.go` — add `"github.com/mastercard/lab02-syntax/stringutil"` to your imports, then add to `main()`:

```go
	// === Packages & Visibility ===
	fmt.Println("\n=== Packages & Visibility ===")

	// Using exported functions
	reversed := stringutil.Reverse("Mastercard")
	fmt.Printf("Reverse(\"Mastercard\"): %s\n", reversed)

	isTitle := stringutil.IsTitle("Go")
	fmt.Printf("IsTitle(\"Go\"): %t\n", isTitle)

	// Using the factory function and exported fields
	wc := stringutil.NewWordCount("go", 42, "readme.md")
	fmt.Printf("WordCount: word=%s, count=%d\n", wc.Word, wc.Count)
	fmt.Printf("Source (via accessor): %s\n", wc.Source())
```

### Step 18 — Verify that unexported identifiers are inaccessible

Add the following lines one at a time. Each one should produce a **compile error**. Verify each, then comment it out or delete it before proceeding:

```go
	// Uncomment each line one at a time to see the compile error:

	// ERROR: cannot refer to unexported name stringutil.reverseRunes
	// stringutil.reverseRunes([]rune("test"))

	// ERROR: unknown field source in struct literal
	// wc2 := stringutil.WordCount{Word: "test", Count: 1, source: "x"}

	// ERROR: wc.source undefined (cannot refer to unexported field)
	// fmt.Println(wc.source)
```

Uncomment the first line. Run `go run .` and read the error message. Comment it out. Repeat for each line.

> **Java parallel:** In Java, you have four levels of access: `public`, `protected`, package-private (default), and `private`. In Go, there are exactly two: **exported** (uppercase first letter) and **unexported** (lowercase first letter). There is no `protected` equivalent. Unexported identifiers are visible within their own package only — not in subpackages, not in test files in external packages.

### Step 19 — Run the complete program

Make sure all compile errors from Step 18 are commented out, then:

```bash
go run .
```

Verify that all sections produce correct output. If you see import errors, run:

```bash
go mod tidy
```

This updates `go.mod` to reference any internal packages correctly.

---

## Part 7: Putting It All Together (5 minutes)

### Step 20 — Review your project structure

Run:

```bash
find . -name "*.go" -o -name "go.mod" | sort
```

You should see:

```
./go.mod
./functional.go
./main.go
./mathutil.go
./namedreturns.go
./slicedemo.go
./status.go
./stringutil/analyze.go
./stringutil/reverse.go
./wordcount.go
```

This is a valid Go project with multiple files in the `main` package and a separate `stringutil` package. Every `.go` file in the root directory belongs to `package main` and compiles together. The `stringutil` directory is its own package.

### Step 21 — Final verification

Run the complete suite one last time:

```bash
go vet ./...
go run .
```

`go vet ./...` checks all packages (including `stringutil`). If no issues are reported, your code is clean.

---

## Lab Checklist

Before you move on, verify you have completed the following:

- [ ] Declared variables with `var` and `:=`; observed zero values for `int`, `string`, `bool`, `float64`, `slice`, and `map`
- [ ] Used constants and `iota` to create an enumeration-like type
- [ ] Wrote `SumAndAverage()` with multiple return values and used it with `:=` destructuring
- [ ] Wrote `MinMax()` with named return values
- [ ] Built a `CountWords()` function using a `map[string]int`; used the "comma ok" idiom and `delete()`
- [ ] Explored slice append, capacity growth, and pre-allocation with `make`
- [ ] **Demonstrated and understood the slice aliasing pitfall** — modifying a sub-slice modifies the original
- [ ] Used `copy()` to create an independent slice
- [ ] Wrote `Filter`, `Map`, and `Reduce` higher-order functions with `func` parameters
- [ ] Used anonymous functions (closures) and observed captured variable mutation
- [ ] Created a `stringutil` package with exported and unexported functions, types, and fields
- [ ] Verified that unexported identifiers produce compile errors when accessed from outside the package

---

## Key Takeaways

1. **Zero values eliminate null.** Every type has a usable default. You will never see a `NullPointerException` for an uninitialized `int` or `string`. Slices and maps are `nil` when uninitialized, but `nil` slices are safe to read — you just cannot write to a `nil` map.

2. **Multiple return values are idiomatic.** The `result, err` pattern you will use in Block 5 is built on this. Get comfortable returning two or more values — it is the Go way.

3. **Slices are views, not copies.** This is the single most common source of subtle bugs for Go newcomers. When in doubt, use `copy()` to make an independent slice.

4. **Maps are unordered.** If you need ordered iteration, sort the keys first. Never depend on map iteration order.

5. **Visibility is binary.** Uppercase first letter = exported. Lowercase = unexported. No `public`, `private`, `protected` keywords. This simplicity is intentional.

6. **Functions are values.** You can pass them as arguments, return them, and store them in variables — just like Java lambdas, but with explicit type signatures.

---

## Troubleshooting

**`go run main.go` fails with "undefined: SumAndAverage":**
You have multiple `.go` files in `package main`. Use `go run .` (with a dot) to compile all files in the directory, not just `main.go`.

**`imported and not used` error:**
Go does not allow unused imports. If you added `"github.com/mastercard/lab02-syntax/stringutil"` but have not used it yet, the compiler will reject it. Add the usage code first, or temporarily comment out the import.

**`cannot use ... as type ...` errors:**
Go has no implicit type conversions. If you are passing an `int` where a `float64` is expected, you must explicitly convert with `float64(myInt)`.

**Map panic: `assignment to entry in nil map`:**
You tried to write to a map that was declared but not initialized. Always use `make(map[K]V)` or a map literal (`map[string]int{"a": 1}`) before writing.

**Slice aliasing confusion:**
If modifying a sub-slice changed your original, this is expected behavior — not a bug. Use `copy()` when you need independence. Re-read Step 12 if this is unclear.
