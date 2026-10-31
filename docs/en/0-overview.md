# Clarity Library

The `clarity` library is a declarative, schema-driven command-line interface (CLI) library for the Yukino programming language. Instead of writing verbose and error-prone argument parsing loops by hand, `clarity` separates interface declaration from application logic into a specialized `.cty` schema file. At compile time, a procedural macro compiles the schema into strictly-typed argument structs, a `Command` enum, and a compact runtime bytecode engine.

The library delivers high ergonomics and safety: exhaustive subcommand pattern matching via `switch`, auto-generated ANSI-colored help, typo suggestions via Levenshtein distance matching, runtime value validators, environment variable and configuration file integration, and shell completion scripts for Bash, Zsh, and Fish.

---

## 1. Introduction to CLI for Beginners

A Command Line Interface (CLI) is a mechanism for interacting with computer programs by typing textual commands into a terminal shell.

### Anatomy of a CLI Invocation

Consider a typical terminal command:

```sh
mygit commit -m "initial commit" --verbose src/main.ykn
```

This invocation consists of the following components:
1. **Application binary (`mygit`)**: The executable name (declared in the schema using the `app` directive).
2. **Subcommand (`commit`)**: The specific task or mode of execution. Subcommands can have their own isolated flags and arguments (declared using `define commit`).
3. **Option with value (`-m "initial commit"`)**: A parameter followed by a value. Can take a short single-dash form (`-m`) or a long double-dash form (`--message`).
4. **Boolean switch flag (`--verbose`)**: A true/false flag. Its presence enables the mode (`true`), while its absence leaves it disabled (`false`).
5. **Positional parameter (`src/main.ykn`)**: An argument passed without a flag name, identified purely by its positional order (declared using `param`).

### Why a Declarative Schema Beats Manual Parsing

When parsing arguments manually with `args()`:
- You must write repetitive loops, nested `if / else` chains, and convert strings into numbers or booleans manually.
- You must format `--help` and `-h` messages by hand and keep them synchronized whenever options change.
- Handling typos, missing arguments, or invalid ranges requires extensive boilerplate code.

With `clarity`, you declare the interface specification once in a `.cty` schema file. The compiler macro verifies the schema, builds runtime bytecode, and generates native Yukino structs, guaranteeing type safety and structured error reporting.

---

## 2. Installation: load clarity

Add `clarity` to your project manifest under the `[libs.runtime]` section in `yuki.toml`:

```toml
[libs.runtime]
clarity = { git = "https://github.com/shareui/clarity", tag = "0.1.0", path = "libs/clarity" }
clarity_rt = { git = "https://github.com/shareui/clarity", tag = "0.1.0", path = "libs/rt" }
scheme = { git = "https://github.com/shareui/clarity", tag = "0.1.0", path = "libs/macro" }
```

Library components:
- `clarity`: The unified public library facade exporting types and functions.
- `clarity_rt`: The runtime virtual machine and parsing engine.
- `scheme`: The compile-time procedural macro attribute `#[clarity::scheme]`.

### Importing in Application Code

In your entry point (`src/main.ykn`):

```yukino
load clarity: *
```

Available entities:
- **Macro Attribute:** `#[clarity::scheme("path/to/schema.cty")]`.
- **Generated Entities:** `parse()`, `parse_args(cli_args)`, `try_parse()`, `try_parse_args(cli_args)`, `Command` enum, `<CommandName>Args` structs.
- **Runtime Classes and Functions:** `ClarityVm`, `VmResult`, `run_vm`, `run_vm_safe`, `Config`, `config_open`, `config_open_file`.
- **Completion Generators:** `generate_bash`, `generate_zsh`, `generate_fish`.
- **Help and Version Renderers:** `render_command_help`, `render_app_help`, `render_version`.
- **Error Structure:** `ClarityRuntimeError`.

---

## 3. Architecture and Compilation Workflow

Clarity operates across three cohesive layers:

1. **Declarative Schema (`.cty`)**: The developer writes high-level definitions of application metadata, subcommands, flags, types, default values, positional parameters, and validators.
2. **Procedural Macro (`scheme`)**: When the compiler processes `#[clarity::scheme("cli.cty")]`, it parses the schema, validates semantic correctness, emits compact bytecode, and injects native Yukino code:
   - Typed `<Command>Args` structures with proper field types (`bool`, `string`, `int32`, `float64`, etc.).
   - An exhaustive `Command` enum (`Command::Build(BuildArgs)`).
   - The `parse() -> Command` bridge connecting system `args()` to the runtime VM.
3. **Runtime Virtual Machine (`clarity_rt`)**: Deserializes the bytecode, processes terminal tokens, handles automatic `--help` and `--version`, evaluates Levenshtein suggestions for typos, and enforces validators.

---

## 4. Quick Example

### Step 1. Define your CLI schema (`cli.cty`)

Create a `cli.cty` file in your application root:

```cty
app "mytool"
version "1.0.0"
about "Example console application"

define build "Build the target package" {
    --release, -r: bool = false "Optimize build artifact";
    --output, -o: string = "app.bin" "Output binary path";
    param target: string = "host" "Target platform architecture";
}

define run "Execute built binary" {
    --port, -p: int32 = 8080 "Server network port";
    param file: string "Path to binary file";
}
```

### What the macro generates behind the scenes

At compile time, the `#[clarity::scheme]` macro expands the `.cty` file into native Yukino code:

```yukino
// Strongly-typed argument structs
public struct BuildArgs {
    release: bool
    output: string
    target: string
}

public struct RunArgs {
    port: int32
    file: string
}

// Exhaustive command enum
public enum Command {
    Build(args: BuildArgs),
    Run(args: RunArgs),
}

// Entrypoint parsing function
public fun parse() -> Command {
    // Delegates to clarity_rt VM
    ...
}
```

### Step 2. Use it in `src/main.ykn`

```yukino
load std.io: println
load clarity: *

#[clarity::scheme("cli.cty")]
fun clarity_init() { space }

fun main() {
    set cmd = parse()
    switch cmd {
        Command::Build(build_args) -> {
            println("Command: build")
            if build_args.release {
                println("Mode: release")
            } else {
                println("Mode: debug")
            }
            println("Output: " + build_args.output)
            println("Target: " + build_args.target)
        }
        Command::Run(run_args) -> {
            println("Command: run")
            println("Port: " + cast(run_args.port, "string"))
            println("File: " + run_args.file)
        }
    }
}
```

### Step 3. Test in the terminal

```sh
# Display auto-generated colored help
./mytool --help

# Run a subcommand with options and positionals
./mytool build --release -o server x86_64

# Typo suggestion in action
./mytool bulid
# error: unrecognized subcommand 'bulid'
#   tip: a similar subcommand exists: 'build'
#
# For more information, try '--help'.
```

---

## 5. Documentation Sections

- [1. Clarity Schema Language (`.cty`)](1-schema.md) — Comprehensive guide to `.cty` syntax: required vs optional directives, common flags (`define *`), root command (`define`), subcommands and nested subcommands (`define` inside `define`), help/version flag overrides, validators, groups, and generated code blocks.
- [2. Runtime API and Features](2-runtime.md) — Reference for `clarity_rt` runtime functions: safe parsing `try_parse()`, Bash/Zsh/Fish completions, TOML configuration store, and error codes.

---

[Next: 1. Clarity Schema Language (`.cty`)](1-schema.md)
