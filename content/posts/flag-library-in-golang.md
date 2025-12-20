---
title: "Flag library in Golang"
date: 2025-12-17T17:17:07+01:00
type: "post"
tags: ["golang"]
---

In Go, the `flag` package is the standard way to handle command-line arguments. It allows you to define options like `-port=8080` or `-verbose` that users can pass when running your program.

### The Three-Step Workflow

To use flags correctly, you must follow this specific order:

1. **Define:** Tell Go what flags to look for (names, types, and defaults).
2. **Parse:** Call `flag.Parse()`. This is the most important step; without it, your variables will never receive the values from the command line.
3. **Access:** Use the values in your logic.



### Defining Flags (Two Ways)

There are two common ways to define a flag: returning a **pointer** or binding to an **existing variable**.

#### Method A: Returning a Pointer

This is the most common method. The function returns a pointer to the value.

```go
// Returns a *string
namePtr := flag.String("name", "Guest", "Name to greet")

// Returns an *int
portPtr := flag.Int("port", 8080, "Port number")

```

#### Method B: Binding to a Variable (`Var` suffix)

Use this if you already have a variable declared (common when using structs).

```go
var mode string
flag.StringVar(&mode, "mode", "fast", "Operation mode")

```


### Common Flag Types & Examples

The package supports all basic Go types and even time durations.

| Type | Function Example | How to use in Terminal |
| --- | --- | --- |
| **String** | `flag.String("s", "val", "desc")` | `-s="hello"` or `-s hello` |
| **Integer** | `flag.Int("n", 10, "desc")` | `-n 50` |
| **Boolean** | `flag.Bool("v", false, "desc")` | `-v` (sets to true) or `-v=false` |
| **Duration** | `flag.Duration("d", 5*time.Second, "desc")` | `-d 10m` or `-d 1h30m` |



Here is a script that combines these concepts into a working tool.

```go
package main

import (
	"flag"
	"fmt"
	"time"
)

func main() {
	// 1. Define flags
	user := flag.String("user", "admin", "The username to log in as")
	retries := flag.Int("retries", 3, "Number of connection attempts")
	debug := flag.Bool("debug", false, "Enable debug logging")
	timeout := flag.Duration("timeout", 5*time.Second, "Max wait time")

	// 2. PARSE (CRITICAL: Must be called before accessing flags)
	flag.Parse()

	// 3. Use values (Note the '*' for pointers)
	fmt.Printf("Logging in as: %s\n", *user)
	fmt.Printf("Retries: %d\n", *retries)
	fmt.Printf("Timeout: %v\n", *timeout)

	if *debug {
		fmt.Println("DEBUG: Detailed logs enabled.")
	}
}

```

#### Good-To-Remember

* **Automatic Help:** Run your program with `-h` or `--help`. The `flag` package automatically generates a clean usage menu based on your descriptions.
* **Boolean Values:** Unlike other types, a boolean flag doesn't need a value. Just saying `-debug` sets it to `true`. If you want to set it to `false` explicitly, you *must* use an equals sign: `-debug=false`.
* **Positional Args:** Anything provided after the flags (like file names) can be accessed using `flag.Args()`.



### Subcommands
In Go, a **Subcommand** is handled by creating independent sets of flags using `flag.NewFlagSet`. This allows your program to have different options depending on which "action" the user chooses.
When you use subcommands, the standard `flag.Parse()` **isn't enough** because the flags change based on the first argument. Instead, you check the first argument (the command) and then parse _only_ the flags associated with that command.


Here is how you can create a tool with two subcommands: `fetch` and `send`.

```go
package main

import (
	"flag"
	"fmt"
	"os"
)

func main() {
	// 1. Define the subcommands
	fetchCmd := flag.NewFlagSet("fetch", flag.ExitOnError)
	sendCmd := flag.NewFlagSet("send", flag.ExitOnError)

	// 2. Define flags unique to 'fetch'
	url := fetchCmd.String("url", "", "The URL to fetch from")
	
	// 3. Define flags unique to 'send'
	dest := sendCmd.String("dest", "localhost", "The destination address")
	secure := sendCmd.Bool("secure", false, "Use SSL/TLS")

	// 4. Check if a subcommand was even provided
	if len(os.Args) < 2 {
		fmt.Println("expected 'fetch' or 'send' subcommands")
		os.Exit(1)
	}

	// 5. Switch on the subcommand and parse accordingly
	switch os.Args[1] {
	case "fetch":
		fetchCmd.Parse(os.Args[2:]) // Parse everything AFTER the 'fetch' word
		fmt.Printf("Fetching from URL: %s\n", *url)
	case "send":
		sendCmd.Parse(os.Args[2:]) // Parse everything AFTER the 'send' word
		fmt.Printf("Sending to: %s (Secure: %v)\n", *dest, *secure)
	default:
		fmt.Println("Unknown command")
		os.Exit(1)
	}
}
```

***

#### Items to remember


* **Independent Help Menus**: If you run `program fetch -h`, it will only show the help for the fetch command. If you run `program send -h`, it shows the send options.

* **`flag.ExitOnError`**: This tells the program to automatically print the usage and exit if the user provides an invalid flag or types `-h`. This is the practice that I do all the time.

* When using `NewFlagSet`, the global `flag.Parse()` (from my previous example) is **not used**. Each sub-command has its own `.Parse()` method. If you call the global `flag.Parse()` at the top of your `main`, it will consume your subcommand names as if they were positional arguments, and your `switch` statement might fail.


### Command Handlers
As your CLI tool grows, putting everything in `main.go` creates a "giant switch statement" that becomes hard to maintain. To keep things clean, the standard practice is to move each command's logic into its own function (or even its own file).

Here is the professional way to structure subcommands using a **functional approach**.

Instead of defining flags in `main`, you define a function for each command that sets up its own flags, parses them, and executes the logic.

```go
package main

import (
	"flag"
	"fmt"
	"os"
)

func main() {
	if len(os.Args) < 2 {
		fmt.Println("Usage: mytool <command> [arguments]")
		fmt.Println("Commands: upload, download")
		os.Exit(1)
	}

	// Route to the correct function
	switch os.Args[1] {
	case "upload":
		handleUpload(os.Args[2:])
	case "download":
		handleDownload(os.Args[2:])
	default:
		fmt.Printf("Unknown command: %s\n", os.Args[1])
		os.Exit(1)
	}
}

// --- Command Handlers ---

func handleUpload(args []string) {
	fs := flag.NewFlagSet("upload", flag.ExitOnError)
	file := fs.String("file", "", "Path to file")
	
	fs.Parse(args)

	if *file == "" {
		fmt.Println("Error: --file is required")
		fs.Usage()
		os.Exit(1)
	}
	fmt.Printf("Uploading %s...\n", *file)
}

func handleDownload(args []string) {
	fs := flag.NewFlagSet("download", flag.ExitOnError)
	id := fs.Int("id", 0, "ID of the item to download")
	
	fs.Parse(args)
	fmt.Printf("Downloading item %d...\n", *id)
}
```


### Benefits of this Structure

1. **Scope Isolation**: The flags for `upload` don't exist in the `download` function. This prevents "variable name collisions."

2. **File Separation**: You can easily move `handleUpload` into a file named `upload.go` and `handleDownload` into `download.go`. As long as they are in the same `package main`, it works perfectly.

3. **Validation**: You can perform custom validation (like checking if a string is empty) immediately after parsing within the function.

### When to move beyond the `flag` package?

While the standard `flag` library is powerful, it has some limitations:

* It doesn't support "Short" vs "Long" flags (e.g., `-v` and `--verbose` as the same thing).

* It doesn't automatically handle nested subcommands (e.g., `git remote add origin`).

If you find yourself needing those features, you can use libraries like **Kong** or **Cobra**. 


---

For comments, please send me 📧 [*<span style="border-bottom: 1px dashed #666;">an email</span>*](mailto:vahid.rafiei@gmail.com).

