# Lab 1: Environment Setup & Hello Go

**Course:** From Java to Go — Prepared for Mastercard
**Block:** 1 — The Go Philosophy & Environment Setup
**Duration:** 45 minutes
**Difficulty:** Introductory

---

## Objective

By the end of this lab, you will have a working Go development environment, you will have written and run your first Go program, and you will have explored the core toolchain that makes Go opinionated by design. You will also create a minimal Java equivalent to feel the structural difference firsthand.

---

## Prerequisites

- A workstation with administrator/sudo access
- Java JDK 23+ already installed (verify with `java --version`)
- Maven 3.9+ already installed (verify with `mvn --version`)
- Internet access for downloading Go and VS Code extensions
- Familiarity with your terminal (bash, zsh, PowerShell, or Windows Terminal)

---

## Part 1: Install Go (5 minutes)

### Step 1 — Download and install Go 1.25+

Go to [https://go.dev/dl/](https://go.dev/dl/) and download the installer for your operating system.

**macOS (Apple Silicon or Intel):**

```bash
# After downloading the .pkg file, run it — or use Homebrew:
brew install go
```

**Linux (amd64):**

```bash
# Replace the version number with the latest 1.25.x release
wget https://go.dev/dl/go1.25.0.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.25.0.linux-amd64.tar.gz
```

Then add Go to your PATH. Append these lines to your `~/.bashrc`, `~/.zshrc`, or equivalent:

```bash
export PATH=$PATH:/usr/local/go/bin
export PATH=$PATH:$(go env GOPATH)/bin
```

Reload your shell:

```bash
source ~/.bashrc   # or source ~/.zshrc
```

**Windows:**

Run the `.msi` installer from the download page. It adds Go to your PATH automatically. Restart your terminal after installation.

### Step 2 — Verify the installation

Open a **new** terminal window and run:

```bash
go version
```

You should see output similar to:

```
go version go1.25.0 darwin/arm64
```

> **If you see `command not found`:** Your PATH is not configured correctly. Revisit the PATH export step above and make sure you opened a new terminal window.

### Step 3 — Confirm the Go environment

Run:

```bash
go env GOPATH
go env GOROOT
```

- `GOROOT` is where Go itself is installed (you should not modify files here).
- `GOPATH` is your workspace for downloaded modules (defaults to `~/go`). In modern Go (1.11+), you do **not** need to work inside GOPATH — modules let you work from any directory.

> **Java parallel:** Think of `GOROOT` like your JDK installation directory and `GOPATH/pkg/mod` like your local Maven repository (`~/.m2/repository`).

---

## Part 2: Configure VS Code (5 minutes)

### Step 4 — Install the Go extension

1. Open VS Code.
2. Open the Extensions panel (`Ctrl+Shift+X` / `Cmd+Shift+X`).
3. Search for **"Go"** and install the extension published by **the Go team at Google** (publisher ID: `golang.go`).
4. After installation, VS Code will show a prompt in the bottom-right corner: **"Analysis tools missing."** Click **Install All**. This installs `gopls` (the Go language server), `dlv` (the debugger), and other tools.

Wait for all tools to finish installing. You will see confirmation messages in the VS Code output panel.

### Step 5 — Install gofumpt

`gofumpt` is a stricter formatter than the standard `gofmt`. It enforces additional formatting rules adopted by most Go projects.

In your terminal:

```bash
go install mvdan.cc/gofumpt@latest
```

Then configure VS Code to use it:

1. Open VS Code Settings (`Ctrl+,` / `Cmd+,`).
2. Search for **"go format tool"**.
3. Set **Go: Format Tool** to `gofumpt`.
4. Verify that **Editor: Format On Save** is enabled (search for "format on save").

> **Java parallel:** This is the equivalent of setting up your IntelliJ code style — except in Go, there is exactly one style. No team debates. No config files to share.

---

## Part 3: Your First Go Program (10 minutes)

### Step 6 — Create a project directory and initialize a module

Open your terminal and run:

```bash
mkdir -p ~/go-labs/lab01-hello
cd ~/go-labs/lab01-hello
```

Initialize a Go module:

```bash
go mod init github.com/mastercard/lab01-hello
```

This creates a `go.mod` file. Open it and inspect its contents:

```bash
cat go.mod
```

You should see:

```
module github.com/mastercard/lab01-hello

go 1.25
```

> **Java parallel:** This single file replaces your entire `pom.xml` or `build.gradle`. No plugins section, no dependency management block, no build configuration. The module path (`github.com/mastercard/lab01-hello`) serves the same purpose as a Maven `groupId` + `artifactId`.

### Step 7 — Write Hello, World

Open the project in VS Code:

```bash
code .
```

Create a new file called `main.go` and type the following **exactly** (do not copy-paste — typing it will help you notice Go's syntax differences):

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, Go! Welcome from the Java side.")
}
```

Save the file. Notice a few things immediately:

- **`package main`** — every executable must be in package `main`. There is no `public class` wrapper.
- **`func main()`** — the entry point. No `public static void main(String[] args)`. No return type, no parameters.
- **No semicolons** — the Go compiler inserts them automatically based on line endings.
- **`fmt.Println`** — the `fmt` package is part of the standard library. The capital `P` means `Println` is an exported (public) function.

### Step 8 — Run the program

From your terminal, inside the `lab01-hello` directory:

```bash
go run main.go
```

Expected output:

```
Hello, Go! Welcome from the Java side.
```

`go run` compiles and executes in one step — it does **not** produce a binary on disk.

### Step 9 — Build a compiled binary

```bash
go build -o hello
```

This produces a standalone binary called `hello` (or `hello.exe` on Windows). Run it:

```bash
./hello
```

Check the file size:

```bash
ls -lh hello
```

You will see a binary in the range of 1–2 MB. This is a **fully statically linked executable** — no JVM, no runtime dependencies, no classpath. You can copy this single file to any machine with the same OS/architecture and it will run.

> **Java parallel:** To achieve something similar in Java, you would need to build a native image with GraalVM or ship a JRE alongside your JAR. In Go, this is the default.

### Step 10 — Cross-compile (bonus observation)

Try this — but do not worry about running the result:

```bash
GOOS=linux GOARCH=amd64 go build -o hello-linux
```

You just compiled a Linux binary from your current machine. Cross-compilation is built into the Go toolchain with zero additional setup.

---

## Part 4: Explore the Formatting Tools (8 minutes)

### Step 11 — Break the formatting intentionally

Open `main.go` and deliberately introduce bad formatting. Replace the contents with:

```go
package main

import "fmt"

func    main(  )    {
fmt.Println(   "Hello, Go! Welcome from the Java side."  )
  if true {
 fmt.Println( "Formatting is wrong on purpose."  )
    }
}
```

Save the file. If you configured format-on-save in Step 5, VS Code will automatically fix the formatting when you save. If it does, undo the formatting (`Ctrl+Z` / `Cmd+Z`) to see the broken version again, then observe what changes when you save.

### Step 12 — Run gofmt from the terminal

To see the formatted output without modifying the file:

```bash
gofmt main.go
```

To format the file in place:

```bash
gofmt -w main.go
```

Now try `gofumpt`:

```bash
gofumpt -w main.go
```

Open the file and verify it is now cleanly formatted. Notice that `gofumpt` may apply additional rules beyond `gofmt` — for example, removing unnecessary blank lines inside function bodies.

> **Key takeaway:** In Go, formatting is not a preference. It is a **solved problem**. Every Go file in every project on earth looks the same. This eliminates an entire category of code review comments.

### Step 13 — Watch gofmt reject invalid syntax

Create a file with a syntax error:

```bash
cat > broken.go << 'EOF'
package main

func main() {
    fmt.Println("missing import")
}
EOF
```

Run:

```bash
gofmt broken.go
```

You will see an error because `fmt` is not imported. Note that `gofmt` does **not** fix logic errors — it only formats valid Go code. Clean up:

```bash
rm broken.go
```

---

## Part 5: Static Analysis with go vet (5 minutes)

### Step 14 — Introduce a deliberate bug

Replace the contents of `main.go` with:

```go
package main

import "fmt"

func main() {
	name := "Mastercard"
	age := 58

	// This line has a bug: %d expects an integer, but we are passing a string.
	fmt.Printf("Company: %d, Founded: %s years ago\n", name, age)
}
```

Save the file. This code **will compile and run** — but it will produce garbage output because the format verbs are swapped (`%d` is for integers, `%s` is for strings).

### Step 15 — Run go vet

```bash
go vet ./...
```

You will see output similar to:

```
./main.go:10:2: fmt.Printf format %d has arg name of wrong type string
./main.go:10:2: fmt.Printf format %s has arg age of wrong type int
```

`go vet` caught the mismatch at analysis time — no unit test required.

### Step 16 — Fix the bug

Correct the format string:

```go
fmt.Printf("Company: %s, Founded: %d years ago\n", name, age)
```

Run `go vet ./...` again and confirm it passes cleanly.

> **Java parallel:** Java's `String.format()` does not catch type mismatches at compile time either. You would need a linter like SpotBugs or Error Prone for equivalent static analysis. In Go, `go vet` is built in and runs in milliseconds.

---

## Part 6: Explore go doc (3 minutes)

### Step 17 — Read documentation from the terminal

Run:

```bash
go doc fmt
```

This prints a summary of the `fmt` package — its purpose and its exported functions.

Now drill into a specific function:

```bash
go doc fmt.Println
```

You will see the function signature and a brief description. Try one more:

```bash
go doc fmt.Fprintf
```

> **Java parallel:** This is similar to browsing Javadoc, except you do not need a browser. The documentation lives alongside the source code and is always available offline via `go doc`.

---

## Part 7: The Java Comparison (9 minutes)

This section is intentionally brief. The goal is not to write Java — you already know Java. The goal is to **feel** the difference in project weight.

### Step 18 — Create an equivalent Maven project

Open a **separate** terminal (keep your Go project open). Navigate to a temporary directory:

```bash
mkdir -p ~/go-labs/lab01-java-comparison
cd ~/go-labs/lab01-java-comparison
```

Generate a minimal Maven project:

```bash
mvn archetype:generate \
  -DgroupId=com.mastercard.lab01 \
  -DartifactId=hello-java \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DarchetypeVersion=1.5 \
  -DinteractiveMode=false
```

### Step 19 — Inspect the generated structure

```bash
find hello-java -type f
```

You will see something like:

```
hello-java/pom.xml
hello-java/src/main/java/com/mastercard/lab01/App.java
hello-java/src/test/java/com/mastercard/lab01/AppTest.java
```

Now open `hello-java/pom.xml` and look at its size. Even this minimal project has a multi-line XML configuration file.

Open `hello-java/src/main/java/com/mastercard/lab01/App.java`:

```java
package com.mastercard.lab01;

public class App {
    public static void main(String[] args) {
        System.out.println("Hello, Java!");
    }
}
```

### Step 20 — Compare side by side

| Aspect | Go | Java |
|---|---|---|
| **Files to print "Hello"** | 2 (`main.go`, `go.mod`) | 3+ (`App.java`, `pom.xml`, directory tree) |
| **Project config** | 2 lines in `go.mod` | ~30 lines in `pom.xml` |
| **Entry point** | `func main()` | `public static void main(String[] args)` |
| **Build command** | `go build -o hello` | `mvn package` (downloads the internet first) |
| **Output** | Single static binary (~1–2 MB) | JAR file (requires JVM to run) |
| **Formatting** | `gofmt` — universal, built-in | Checkstyle/Spotless — must configure, must agree |
| **Static analysis** | `go vet` — built-in | SpotBugs/Error Prone — separate tools to install |

> **This is not about Java being bad.** Java is a powerful, mature ecosystem. The point is that Go makes different trade-offs: it sacrifices flexibility and framework richness for simplicity, speed, and a minimal surface area. As a Java developer, you are trading a deep toolbox for a sharp, small one.

### Step 21 — Clean up the Java project

```bash
cd ~/go-labs
rm -rf lab01-java-comparison
```

---

## Lab Checklist

Before you move on, verify you have completed the following:

- [ ] Go 1.25+ is installed and `go version` prints correctly
- [ ] VS Code has the Go extension installed with all analysis tools
- [ ] `gofumpt` is installed and configured as the default formatter
- [ ] You created a module with `go mod init` and understand what `go.mod` contains
- [ ] You wrote, ran (`go run`), and built (`go build`) a Hello World program
- [ ] You intentionally broke formatting and fixed it with `gofmt` / `gofumpt`
- [ ] You introduced a `fmt.Printf` bug and detected it with `go vet`
- [ ] You used `go doc` to read standard library documentation from the terminal
- [ ] You created a Maven project and compared the structural overhead to Go

---

## Key Takeaways

1. **Go's toolchain is integrated.** Formatting (`gofmt`), static analysis (`go vet`), documentation (`go doc`), dependency management (`go mod`), and testing (`go test`) are all built in. In Java, each of these is a separate tool you must choose, configure, and maintain.

2. **There is one way to format Go code.** This is not a limitation — it is a feature. It eliminates bikeshedding and makes every Go codebase instantly readable.

3. **Go produces standalone binaries.** No JVM, no classpath, no runtime dependencies. A Go binary is a single file you can deploy anywhere.

4. **Simplicity is intentional.** Go's lack of certain features (no inheritance, no exceptions, no annotations) is a deliberate design choice — not a shortcoming. The rest of this course will show you what Go offers instead.

---

## Troubleshooting

**`gopls` is not starting in VS Code:**
Open the VS Code command palette (`Ctrl+Shift+P` / `Cmd+Shift+P`), search for "Go: Install/Update Tools", select all tools, and click OK. Restart VS Code after installation completes.

**`gofumpt` command not found:**
Make sure `$(go env GOPATH)/bin` is in your PATH. Run `echo $PATH` to verify. The `go install` command places binaries in `$(go env GOPATH)/bin`.

**`go vet` reports no output (no errors found):**
Double-check that you swapped the format verbs correctly in Step 14. The format string must have `%d` where a `string` is passed and `%s` where an `int` is passed for `go vet` to flag it.

**Maven archetype generation fails:**
Make sure Maven 3.9+ is installed and you have internet access. If Maven is not available, you can skip Part 7 — the comparison is illustrative, not essential to the Go learning objectives.
