# Lab 3: From Classes to Structs

**Course:** From Java to Go — Prepared for Mastercard
**Block:** 3 — Structs, Methods & Composition
**Duration:** 60 minutes
**Difficulty:** Intermediate

---

## Objective

By the end of this lab, you will have replaced Java-style classes, constructors, and inheritance with Go structs, factory functions, methods with receivers, and struct embedding. You will understand the difference between value and pointer receivers, know when to use each, and be able to spot the method shadowing pitfall that trips up Go newcomers. You will also use Go's escape analysis tooling to see how the compiler decides whether data lives on the stack or the heap.

---

## Prerequisites

- Completed Labs 1 and 2 (Go 1.25+ installed, comfortable with variables, slices, maps, and packages)
- Familiarity with Java classes, constructors, inheritance (`extends`), method overriding, and `NullPointerException`

---

## Setup

```bash
mkdir -p ~/go-labs/lab03-structs
cd ~/go-labs/lab03-structs
go mod init github.com/mastercard/lab03-structs
code .
```

---

## Part 1: Defining Structs & Factory Functions (10 minutes)

### Step 1 — Define a User struct

Create a file called `user.go`:

```go
package main

import (
	"fmt"
	"strings"
	"time"
)

// User represents a system user.
// In Java, this would be a class with private fields and getters/setters.
// In Go, fields are just fields — no getters/setters by default.
type User struct {
	ID        int
	FirstName string
	LastName  string
	Email     string
	Active    bool
	CreatedAt time.Time
}
```

Notice the differences from a Java class immediately:

- **No access modifiers on fields.** Uppercase = exported, lowercase = unexported. That is the entire visibility system.
- **No getter/setter boilerplate.** Fields are accessed directly. You only write accessor methods when you need computed values or controlled access.
- **No constructor.** Go structs do not have constructors. You will write a factory function instead.

### Step 2 — Write a factory function

Add the following to `user.go`, below the struct definition:

```go
// NewUser creates and returns a pointer to a new User.
// This is Go's replacement for Java constructors.
// By convention, factory functions are named New<Type> or New.
func NewUser(id int, firstName, lastName, email string) *User {
	return &User{
		ID:        id,
		FirstName: firstName,
		LastName:  lastName,
		Email:     strings.ToLower(strings.TrimSpace(email)),
		Active:    true,
		CreatedAt: time.Now(),
	}
}
```

Study this carefully:

- **`*User`** — the function returns a pointer to a `User`. This is idiomatic when the caller needs to mutate the struct or when the struct is large enough that copying it would be wasteful.
- **`&User{...}`** — the `&` takes the address of the struct literal. Go's compiler performs escape analysis and allocates this on the heap automatically because the pointer escapes the function.
- **Named field initialization** — `ID: id` is explicit. Unlike Java constructors, there is no `this.id = id` pattern. The struct literal syntax is self-documenting.
- **Validation on construction** — the email is trimmed and lowercased. Just as in Java, you validate inputs in the factory function.

### Step 3 — Test the factory function

Create `main.go`:

```go
package main

import "fmt"

func main() {
	// === Struct Creation ===
	fmt.Println("=== Struct Creation ===")

	// Using the factory function (preferred)
	user1 := NewUser(1, "Ada", "Lovelace", "  Ada.Lovelace@Mastercard.com  ")
	fmt.Printf("user1: %+v\n", user1)

	// Direct struct literal (no validation, useful for tests)
	user2 := User{
		ID:        2,
		FirstName: "Alan",
		LastName:  "Turing",
		Email:     "alan@example.com",
		Active:    true,
	}
	fmt.Printf("user2: %+v\n", user2)

	// Zero-value struct (all fields at their zero values)
	var user3 User
	fmt.Printf("user3 (zero): %+v\n", user3)
	fmt.Printf("  user3.Active is %t (not nil, not null — just false)\n", user3.Active)
	fmt.Printf("  user3.CreatedAt is %v (zero time, not null)\n", user3.CreatedAt)
}
```

Run:

```bash
go run .
```

Note that `user1` prints with a `&` prefix because it is a pointer, and that `user3` has zero values for every field — `""` for strings, `0` for int, `false` for bool, and the zero `time.Time`. Nothing is `null`.

> **Java parallel:** In Java, `new User()` allocates on the heap and returns a reference. In Go, `&User{}` does the same, but `User{}` (without `&`) creates a value that may live on the stack. The compiler decides — you do not.

---

## Part 2: Methods and Receivers (12 minutes)

### Step 4 — Add methods with pointer receivers

Add the following methods to `user.go`:

```go
// String returns a human-readable representation of the User.
// This is a VALUE receiver — it does not modify the User.
// Implementing String() is like overriding toString() in Java.
func (u User) String() string {
	status := "active"
	if !u.Active {
		status = "inactive"
	}
	return fmt.Sprintf("%s %s <%s> [%s]", u.FirstName, u.LastName, u.Email, status)
}

// FullName returns the user's full name.
// Value receiver — read-only access is all that is needed.
func (u User) FullName() string {
	return u.FirstName + " " + u.LastName
}

// Validate checks whether the User has valid data.
// Value receiver — it reads fields but does not modify them.
// Returns an error (preview of Block 5 error handling).
func (u User) Validate() error {
	if u.FirstName == "" {
		return fmt.Errorf("first name is required")
	}
	if u.LastName == "" {
		return fmt.Errorf("last name is required")
	}
	if !strings.Contains(u.Email, "@") {
		return fmt.Errorf("invalid email: %s", u.Email)
	}
	return nil
}

// UpdateEmail changes the user's email.
// POINTER receiver — this method modifies the User.
// Without the pointer, changes would apply to a copy and be lost.
func (u *User) UpdateEmail(newEmail string) error {
	cleaned := strings.ToLower(strings.TrimSpace(newEmail))
	if !strings.Contains(cleaned, "@") {
		return fmt.Errorf("invalid email: %s", newEmail)
	}
	u.Email = cleaned
	return nil
}

// Deactivate marks the user as inactive.
// Pointer receiver — modifies the Active field.
func (u *User) Deactivate() {
	u.Active = false
}
```

Read the comments on each method carefully. The receiver type — `(u User)` vs `(u *User)` — is the most important decision you make when writing Go methods.

**The rule of thumb:**

| Situation | Receiver type | Reason |
|---|---|---|
| Method reads fields only | `(u User)` — value | Safe, no side effects |
| Method modifies fields | `(u *User)` — pointer | Changes persist after the call |
| Struct is large (many fields) | `(u *User)` — pointer | Avoids copying the entire struct |
| Consistency within a type | pointer if **any** method needs pointer | Keeps the method set uniform |

In practice, **most methods use pointer receivers** because at least one method usually needs to mutate state, and mixing receiver types on the same struct is confusing.

### Step 5 — Call the methods

Add to `main()`:

```go
	// === Methods ===
	fmt.Println("\n=== Methods ===")

	fmt.Printf("user1.String():   %s\n", user1.String())
	fmt.Printf("user1.FullName(): %s\n", user1.FullName())

	// Validate a good user
	if err := user1.Validate(); err != nil {
		fmt.Printf("Validation failed: %s\n", err)
	} else {
		fmt.Println("user1 is valid")
	}

	// Validate a bad user
	badUser := User{ID: 99, Email: "not-an-email"}
	if err := badUser.Validate(); err != nil {
		fmt.Printf("badUser validation: %s\n", err)
	}

	// Mutating methods (pointer receivers)
	fmt.Printf("\nBefore email update: %s\n", user1.Email)
	err := user1.UpdateEmail("ada.new@mastercard.com")
	if err != nil {
		fmt.Printf("Update failed: %s\n", err)
	}
	fmt.Printf("After email update:  %s\n", user1.Email)

	user1.Deactivate()
	fmt.Printf("After deactivate:    %s\n", user1)
```

Run:

```bash
go run .
```

Notice that `fmt.Printf` with `%s` automatically calls `user1.String()`. This is because Go's `fmt` package checks for the `fmt.Stringer` interface — if a type has a `String() string` method, `fmt` uses it. This is Go's equivalent of Java's `toString()`.

> **Java parallel:** In Java, every method is implicitly an instance method with access to `this`. In Go, the receiver is an explicit parameter. Writing `func (u *User) Deactivate()` is like writing a static method `static void deactivate(User u)` in Java — except Go's syntax makes it look like an instance method when called.

---

## Part 3: Pointers — Value vs. Reference Semantics (8 minutes)

### Step 6 — Observe the difference between pointer and value parameters

Create a file called `pointers.go`:

```go
package main

import "fmt"

// TryUpdateByValue receives a COPY of the User.
// Changes are lost when the function returns.
func TryUpdateByValue(u User, newName string) {
	u.FirstName = newName
	fmt.Printf("  Inside TryUpdateByValue: FirstName = %s\n", u.FirstName)
}

// UpdateByPointer receives a POINTER to the User.
// Changes persist because we are modifying the original.
func UpdateByPointer(u *User, newName string) {
	u.FirstName = newName
	fmt.Printf("  Inside UpdateByPointer:  FirstName = %s\n", u.FirstName)
}
```

### Step 7 — Demonstrate the difference

Add to `main()`:

```go
	// === Pointer vs Value ===
	fmt.Println("\n=== Pointer vs Value ===")

	subject := NewUser(10, "Grace", "Hopper", "grace@example.com")
	fmt.Printf("Before:  FirstName = %s\n", subject.FirstName)

	// Pass by value — the function gets a copy
	TryUpdateByValue(*subject, "CHANGED")
	fmt.Printf("After TryUpdateByValue:  FirstName = %s  ← unchanged!\n", subject.FirstName)

	// Pass by pointer — the function modifies the original
	UpdateByPointer(subject, "CHANGED")
	fmt.Printf("After UpdateByPointer:   FirstName = %s  ← changed!\n", subject.FirstName)
```

Run:

```bash
go run .
```

Expected output:

```
Before:  FirstName = Grace
  Inside TryUpdateByValue: FirstName = CHANGED
After TryUpdateByValue:  FirstName = Grace  ← unchanged!
  Inside UpdateByPointer:  FirstName = CHANGED
After UpdateByPointer:   FirstName = CHANGED  ← changed!
```

This is the single most important concept for Java developers to internalize. In Java, all objects are passed by reference (via reference value). In Go, **structs are values by default** — passing a struct to a function copies it. You must explicitly use a pointer (`*User`) when you want reference semantics.

### Step 8 — Nil pointer safety

Add to `main()`:

```go
	// === Nil Pointers ===
	fmt.Println("\n=== Nil Pointers ===")

	var nilUser *User // nil pointer — similar to Java's null reference
	fmt.Printf("nilUser == nil: %t\n", nilUser == nil)

	// Calling a method on a nil pointer PANICS (like Java's NullPointerException)
	// Uncomment the next line to see the panic:
	// fmt.Println(nilUser.FullName())

	// You can guard against nil explicitly:
	if nilUser != nil {
		fmt.Println(nilUser.FullName())
	} else {
		fmt.Println("nilUser is nil — skipped method call")
	}
```

> **Java parallel:** A `nil` pointer in Go behaves like `null` in Java. Calling a method on it causes a panic (Go's equivalent of a `NullPointerException`). The key difference: Go value types (non-pointer structs) can **never** be nil, which eliminates an entire category of null-safety bugs at the type level.

---

## Part 4: Composition with Struct Embedding (12 minutes)

### Step 9 — Define an Address struct and embed it in User

Create a file called `address.go`:

```go
package main

import "fmt"

// Address represents a mailing address.
type Address struct {
	Street  string
	City    string
	State   string
	ZipCode string
	Country string
}

// String returns a formatted address.
func (a Address) String() string {
	return fmt.Sprintf("%s, %s, %s %s, %s", a.Street, a.City, a.State, a.ZipCode, a.Country)
}

// IsComplete reports whether all required address fields are filled.
func (a Address) IsComplete() bool {
	return a.Street != "" && a.City != "" && a.State != "" && a.ZipCode != "" && a.Country != ""
}
```

### Step 10 — Create an enhanced user with embedded Address

Create a file called `enhanced_user.go`:

```go
package main

import (
	"fmt"
	"strings"
	"time"
)

// EnhancedUser demonstrates struct embedding (composition).
// The Address struct is EMBEDDED — its fields and methods are promoted
// to EnhancedUser, as if they were declared directly on it.
type EnhancedUser struct {
	ID        int
	FirstName string
	LastName  string
	Email     string
	Active    bool
	CreatedAt time.Time
	Address                    // embedded struct — note: no field name
	Phone     string           // regular field
}

// NewEnhancedUser creates an EnhancedUser with the given details.
func NewEnhancedUser(id int, firstName, lastName, email string) *EnhancedUser {
	return &EnhancedUser{
		ID:        id,
		FirstName: firstName,
		LastName:  lastName,
		Email:     strings.ToLower(strings.TrimSpace(email)),
		Active:    true,
		CreatedAt: time.Now(),
	}
}

// FullProfile returns a summary including the promoted address fields.
func (eu *EnhancedUser) FullProfile() string {
	addr := "no address on file"
	if eu.Address.IsComplete() {
		addr = eu.Address.String()
	}
	return fmt.Sprintf("%s %s <%s> — %s", eu.FirstName, eu.LastName, eu.Email, addr)
}
```

### Step 11 — Access promoted fields and methods

Add to `main()`:

```go
	// === Composition with Embedding ===
	fmt.Println("\n=== Composition with Embedding ===")

	eu := NewEnhancedUser(100, "Linus", "Torvalds", "linus@example.com")

	// Set address fields DIRECTLY on EnhancedUser — they are promoted
	eu.Street = "123 Kernel Ave"
	eu.City = "Portland"
	eu.State = "OR"
	eu.ZipCode = "97201"
	eu.Country = "US"

	// You can also access them via the embedded struct name
	fmt.Printf("Via promoted field:  eu.City    = %s\n", eu.City)
	fmt.Printf("Via explicit path:   eu.Address.City = %s\n", eu.Address.City)

	// Promoted methods work the same way
	fmt.Printf("eu.IsComplete(): %t\n", eu.IsComplete())
	fmt.Printf("eu.FullProfile(): %s\n", eu.FullProfile())
```

Run:

```bash
go run .
```

The key insight: `eu.City` and `eu.Address.City` refer to the same field. Embedding **promotes** the fields and methods of the inner struct to the outer struct. The inner struct is still there — you can access it by name (`eu.Address`) — but its members are also available as if they were declared directly on `EnhancedUser`.

> **Java parallel:** This is the Go equivalent of `class EnhancedUser extends User` — except there is no inheritance chain, no polymorphism through `super`, and no `@Override`. The embedded struct is a field, not a parent class. The outer struct "has an" address; it does not "extend" address.

---

## Part 5: Refactoring a Java Inheritance Chain (10 minutes)

This exercise gives you a Java class hierarchy and asks you to translate it into Go. Read the Java code first, then build the Go equivalent.

### Step 12 — Read the Java source (do not type this — just read it)

The following Java code models an `Animal → Dog → GuideDog` inheritance chain:

```java
// Java — Animal.java
public abstract class Animal {
    private String name;
    private int age;

    public Animal(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() { return name; }
    public int getAge() { return age; }

    public abstract String speak();

    public String describe() {
        return name + " (age " + age + ") says: " + speak();
    }
}

// Java — Dog.java
public class Dog extends Animal {
    private String breed;

    public Dog(String name, int age, String breed) {
        super(name, age);
        this.breed = breed;
    }

    public String getBreed() { return breed; }

    @Override
    public String speak() { return "Woof!"; }

    public String fetch(String item) {
        return getName() + " fetches the " + item;
    }
}

// Java — GuideDog.java
public class GuideDog extends Dog {
    private String handler;

    public GuideDog(String name, int age, String breed, String handler) {
        super(name, age, breed);
        this.handler = handler;
    }

    @Override
    public String speak() { return "Woof! (quietly)"; }

    public String guide() {
        return getName() + " guides " + handler + " safely";
    }
}
```

Notice the pattern: three levels of inheritance, abstract method, method overriding, `super()` calls, and a deepening constructor chain. This is typical enterprise Java.

### Step 13 — Build the Go equivalent using composition

Create a file called `animals.go`:

```go
package main

import "fmt"

// Animal holds the base data that all animals share.
// This is NOT a base class — it is a data container that will be embedded.
type Animal struct {
	Name string
	Age  int
}

// Describe returns a description using the animal's own data.
// Note: this method cannot call a "subclass" speak() method because
// Go has no virtual dispatch. We will handle this differently.
func (a Animal) Describe() string {
	return fmt.Sprintf("%s (age %d)", a.Name, a.Age)
}

// Dog composes Animal by embedding it, and adds dog-specific data.
type Dog struct {
	Animal         // embedded — promotes Name, Age, Describe()
	Breed  string
}

// NewDog is the factory function for Dog.
func NewDog(name string, age int, breed string) *Dog {
	return &Dog{
		Animal: Animal{Name: name, Age: age},
		Breed:  breed,
	}
}

// Speak returns what a dog says.
func (d Dog) Speak() string {
	return "Woof!"
}

// Fetch returns a description of the dog fetching an item.
func (d Dog) Fetch(item string) string {
	return fmt.Sprintf("%s fetches the %s", d.Name, item)
}

// FullDescribe provides a complete description including the sound.
// Since Go has no virtual dispatch, we call Speak() explicitly
// on the concrete type.
func (d Dog) FullDescribe() string {
	return fmt.Sprintf("%s says: %s", d.Describe(), d.Speak())
}

// GuideDog composes Dog (which already embeds Animal).
// This gives GuideDog access to Name, Age, Breed, Describe(),
// Speak(), Fetch(), and FullDescribe() — all promoted.
type GuideDog struct {
	Dog             // embedded — promotes everything from Dog AND Animal
	Handler string
}

// NewGuideDog is the factory function for GuideDog.
func NewGuideDog(name string, age int, breed string, handler string) *GuideDog {
	return &GuideDog{
		Dog:     Dog{Animal: Animal{Name: name, Age: age}, Breed: breed},
		Handler: handler,
	}
}

// Speak overrides (shadows) Dog's Speak method.
func (g GuideDog) Speak() string {
	return "Woof! (quietly)"
}

// Guide returns a description of the guide dog at work.
func (g GuideDog) Guide() string {
	return fmt.Sprintf("%s guides %s safely", g.Name, g.Handler)
}

// FullDescribe is redefined here because GuideDog.Speak() returns
// different text than Dog.Speak(). Without this, the promoted
// Dog.FullDescribe() would call Dog.Speak(), not GuideDog.Speak().
// This is the "no virtual dispatch" trade-off in Go.
func (g GuideDog) FullDescribe() string {
	return fmt.Sprintf("%s says: %s", g.Describe(), g.Speak())
}
```

### Step 14 — Exercise the composition hierarchy

Add to `main()`:

```go
	// === Composition Hierarchy: Animal → Dog → GuideDog ===
	fmt.Println("\n=== Animal → Dog → GuideDog ===")

	dog := NewDog("Rex", 5, "German Shepherd")
	fmt.Printf("Dog: %s\n", dog.FullDescribe())
	fmt.Printf("  Breed: %s\n", dog.Breed)
	fmt.Printf("  Fetch: %s\n", dog.Fetch("ball"))

	gd := NewGuideDog("Buddy", 3, "Labrador", "Alice")
	fmt.Printf("\nGuideDog: %s\n", gd.FullDescribe())
	fmt.Printf("  Breed:   %s\n", gd.Breed)       // promoted from Dog
	fmt.Printf("  Handler: %s\n", gd.Handler)
	fmt.Printf("  Guide:   %s\n", gd.Guide())
	fmt.Printf("  Fetch:   %s\n", gd.Fetch("toy")) // promoted from Dog

	// Accessing the embedded types explicitly
	fmt.Printf("\n  gd.Dog.Speak():    %s\n", gd.Dog.Speak())    // Dog's version
	fmt.Printf("  gd.Speak():        %s\n", gd.Speak())          // GuideDog's version (shadows Dog)
	fmt.Printf("  gd.Animal.Name:    %s\n", gd.Animal.Name)      // two levels deep
```

Run:

```bash
go run .
```

Pay close attention to the last three lines. `gd.Speak()` calls `GuideDog.Speak()` (the shadowing method), but `gd.Dog.Speak()` calls the original `Dog.Speak()`. You can always reach the embedded struct's methods by qualifying the call with the type name.

> **Critical difference from Java:** In Java, `GuideDog.speak()` overrides `Dog.speak()` through virtual dispatch — even if you have a `Dog` reference pointing to a `GuideDog`, calling `speak()` calls the `GuideDog` version. In Go, there is **no virtual dispatch**. If you call `Dog.Speak()` on a `Dog` value, you always get `Dog`'s implementation, even if that `Dog` is embedded inside a `GuideDog`. This is why `GuideDog` redefines `FullDescribe()` — the promoted `Dog.FullDescribe()` would call `Dog.Speak()`, not `GuideDog.Speak()`.

---

## Part 6: The Method Shadowing Pitfall (3 minutes)

### Step 15 — Demonstrate method collision with embedding

Create a file called `shadowing.go`:

```go
package main

import "fmt"

// Logger provides logging functionality.
type Logger struct {
	Prefix string
}

func (l Logger) String() string {
	return fmt.Sprintf("Logger[%s]", l.Prefix)
}

// Auditor provides audit trail functionality.
type Auditor struct {
	AuditLevel int
}

func (a Auditor) String() string {
	return fmt.Sprintf("Auditor[level=%d]", a.AuditLevel)
}

// Service embeds BOTH Logger and Auditor.
// Both have a String() method — this creates ambiguity.
type Service struct {
	Logger
	Auditor
	Name string
}

// DemoShadowing shows what happens with ambiguous promoted methods.
func DemoShadowing() {
	fmt.Println("\n=== Method Shadowing Pitfall ===")

	svc := Service{
		Logger:  Logger{Prefix: "SVC"},
		Auditor: Auditor{AuditLevel: 3},
		Name:    "PaymentService",
	}

	// These work fine — accessing each embedded type's method explicitly
	fmt.Printf("svc.Logger.String():  %s\n", svc.Logger.String())
	fmt.Printf("svc.Auditor.String(): %s\n", svc.Auditor.String())

	// This will NOT compile — ambiguous selector:
	// fmt.Println(svc.String())
	// ERROR: ambiguous selector svc.String

	// The fix: define String() on Service itself to resolve the ambiguity
	fmt.Printf("svc.Name: %s\n", svc.Name)
}
```

Add to `main()`:

```go
	DemoShadowing()
```

Run:

```bash
go run .
```

Now uncomment the `fmt.Println(svc.String())` line. Run again and read the compile error. It tells you that `String` is ambiguous because both `Logger` and `Auditor` promote it. Comment it back out.

The fix is to define a `String()` method directly on `Service`, which resolves the ambiguity by shadowing both promoted versions. This is Go's way of handling the "diamond problem" — it simply does not compile until you resolve the conflict explicitly.

> **Java parallel:** Java solves the diamond problem with interfaces and `default` methods — the compiler forces you to provide an implementation when two interfaces have conflicting defaults. Go's approach is the same principle: you must resolve the ambiguity yourself.

---

## Part 7: Escape Analysis (5 minutes)

### Step 16 — Observe stack vs. heap allocation decisions

Create a file called `escape.go`:

```go
package main

import "fmt"

// createOnStack returns a User by value.
// The compiler MAY keep this on the stack.
func createOnStack() User {
	u := User{ID: 1, FirstName: "Stack", LastName: "User", Email: "stack@test.com"}
	return u
}

// createOnHeap returns a pointer to a User.
// The User MUST escape to the heap because a pointer to it leaves the function.
func createOnHeap() *User {
	u := User{ID: 2, FirstName: "Heap", LastName: "User", Email: "heap@test.com"}
	return &u
}

// processLocally creates a User and never lets it leave.
// The compiler should keep this entirely on the stack.
func processLocally() string {
	u := User{ID: 3, FirstName: "Local", LastName: "User", Email: "local@test.com"}
	return u.FullName()
}

// DemoEscape calls the functions above. Escape analysis results
// are not visible at runtime — you need the compiler flag to see them.
func DemoEscape() {
	fmt.Println("\n=== Escape Analysis ===")
	v := createOnStack()
	p := createOnHeap()
	n := processLocally()
	fmt.Printf("Stack user: %s\n", v.FullName())
	fmt.Printf("Heap user:  %s\n", p.FullName())
	fmt.Printf("Local call: %s\n", n)
}
```

Add to `main()`:

```go
	DemoEscape()
```

### Step 17 — Run the escape analysis tool

Build with the escape analysis flag:

```bash
go build -gcflags='-m' . 2>&1 | grep escape
```

You will see output similar to:

```
./escape.go:14:9: &u escapes to heap
```

The `&u escapes to heap` line tells you that the `User` in `createOnHeap()` is allocated on the heap because its address is returned to the caller. The `User` in `createOnStack()` and `processLocally()` may remain on the stack because their addresses do not escape.

For more detail, add a second `-m`:

```bash
go build -gcflags='-m -m' . 2>&1 | grep -A2 escape.go
```

This shows the compiler's reasoning for each allocation decision.

> **Java parallel:** In Java, all objects are heap-allocated (the JIT compiler may perform scalar replacement, but this is opaque). In Go, the compiler's escape analysis is transparent and deterministic — you can see exactly what goes on the heap and why. Fewer heap allocations means less GC pressure, which is one reason Go services tend to have lower tail latency than equivalent Java services.

---

## Lab Checklist

Before you move on, verify you have completed the following:

- [ ] Defined a `User` struct with typed fields and created instances using a `NewUser()` factory function, struct literals, and zero-value initialization
- [ ] Implemented methods with value receivers (`String()`, `FullName()`, `Validate()`) and pointer receivers (`UpdateEmail()`, `Deactivate()`) and can explain when to use each
- [ ] Demonstrated that passing a struct by value creates a copy (changes are lost) and passing by pointer allows mutation
- [ ] Handled a nil pointer check and understand that value-type structs (non-pointer) can never be nil
- [ ] Built a composition hierarchy: `Address` embedded in `EnhancedUser`, with promoted field and method access
- [ ] Refactored the Java `Animal → Dog → GuideDog` inheritance chain into Go using struct embedding
- [ ] Understand that Go has **no virtual dispatch** — a shadowing method on an outer struct does not automatically replace the embedded struct's method in other promoted calls
- [ ] Demonstrated the method shadowing pitfall when two embedded structs promote a method with the same name
- [ ] Used `go build -gcflags='-m'` to observe escape analysis and identified which allocations go to the heap

---

## Key Takeaways

1. **Structs are not classes.** They have no constructors, no inheritance, no access modifiers beyond exported/unexported. Factory functions (`New*`) and method receivers replace Java's constructors and instance methods.

2. **Pointer receivers for mutation, value receivers for reads.** If you need to change a field, use `*Type`. If you only read, use `Type`. When in doubt, use a pointer receiver — it is always safe and avoids unnecessary copies.

3. **Composition replaces inheritance.** Struct embedding gives you field and method promotion — the outer struct "has a" inner struct. But there is no polymorphic dispatch through the embedding chain. If `GuideDog` needs different behavior from `Dog.Speak()`, `GuideDog` must define its own version.

4. **Escape analysis is your friend.** Go's compiler is transparent about what goes on the stack vs. the heap. Use `-gcflags='-m'` in performance-sensitive code to verify allocation behavior. Fewer heap allocations = less GC overhead.

5. **The shadowing pitfall is real.** When two embedded structs promote methods with the same name, the compiler rejects the ambiguous call. You must resolve it by defining the method on the outer struct. This is Go's version of the diamond problem.

---

## Troubleshooting

**`go run main.go` fails with "undefined: NewUser":**
You have multiple `.go` files in `package main`. Use `go run .` (with a dot) to compile all files in the directory.

**"ambiguous selector" compile error:**
This is expected in Step 15 when two embedded structs promote the same method. Either access the method through the specific embedded type (`svc.Logger.String()`) or define the method on the outer struct to resolve it.

**No output from escape analysis:**
Make sure you are using `2>&1` to capture stderr (Go's compiler diagnostics go to stderr, not stdout). Also ensure you are running `go build -gcflags='-m' .` with the trailing dot.

**"cannot use u (variable of type User) as *User" error:**
You are passing a `User` value where a `*User` pointer is expected. Use `&u` to take the address, or change the function signature to accept `User` by value.
