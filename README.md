# clarity

A declarative, schema-driven CLI library for the [Yukino](https://github.com/yukitsuri/yukino) programming language.

`clarity` allows developers to define command-line interfaces in clean `.cty` schema files. At compile time, a procedural macro compiles the schema into strictly-typed Yukino structures, enums, and an efficient runtime parsing engine. It delivers full feature parity with standard CLI conventions: zero boilerplate, type-safe pattern matching, colored help, versioning, typo suggestions, and automatic validation.

---

## Features

- **Declarative Schema Grammar (`.cty`)**: Define applications, subcommands, nested subcommands (`define` inside `define`), flags, positional arguments, and groups cleanly without writing manual parsing code.
- **Compile-Time Code Generation**: The `#[clarity::scheme("...")]` procedural macro generates native Yukino argument structs and a `Command` enum for exhaustive `switch` pattern matching.
- **High-Performance Bytecode Engine**: Schemas are compiled into compact bytecode executed by the lightweight `clarity_rt` runtime virtual machine.
- **Auto-Generated Colored Help**: Generates formatted help with standard sections (`USAGE:`, `FLAGS:`, `OPTIONS:`, `ARGS:`, `SUBCOMMANDS:`) styled with ANSI colors via `std.outf`.
- **Auto-Version and Flag Overrides**: Automatically responds to `--version`, `-V`, and `-v` flags, with support for custom flag overrides (such as `-v` for verbosity count) or disabling via `default_flags false`.
- **Typo Correction and Suggestions**: Built-in Levenshtein distance matcher suggests similar subcommands or options on typos (*"tip: a similar subcommand exists: 'build'"*).
- **Flag Relationships and Groups**: Support for `requires`, `conflicts`, mutually exclusive (`or`), and co-dependent (`and`) option groups.
- **Config and Environment Integration**: Automatically falls back to environment variables (`env`) or configuration file keys (`config`) when options are omitted.
- **Rich Value Validation**: Enforce ranges, regex patterns, string length, choices (`one_of`), and path existence (`exists`, `parent_exists`).
- **Shell Completions**: Built-in generators for Bash, Zsh, and Fish completion scripts.

---

## Installation

Add `clarity` to your project manifest under the `[libs.runtime]` section in `yuki.toml`:

```toml
[libs.runtime]
clarity = { git = "https://github.com/shareui/clarity", tag = "0.1.0", package = "clarity" }
```

---

## Quick Example

### 1. Define your CLI schema (`cli.cty`)

Create a `cli.cty` file in your application root:

```cty
app "demo"
version "1.0.0"
about "Demo CLI application with Clarity"

define build "Build the project" {
    --release, -r: bool = false "Build in release mode";
    --output, -o: string = "a.out" "Output file";
    param target: string = "host" "Target architecture";
}

define run "Run target binary" {
    --port, -p: int32 = 8080 "Server port";
    param file: string = "main.bin" "File to run";
}
```

### What the macro generates behind the scenes

The `#[clarity::scheme("cli.cty")]` macro transforms the `.cty` file into native Yukino code:

```yukino
public struct BuildArgs {
    release: bool
    output: string
    target: string
}

public struct RunArgs {
    port: int32
    file: string
}

public enum Command {
    Build(args: BuildArgs),
    Run(args: RunArgs),
}

public fun parse() -> Command { ... }
```

### 2. Implement your entrypoint (`src/main.ykn`)

```yukino
load std.io: println
load clarity: *

#[clarity::scheme("cli.cty")]
fun clarity_init() { space }

fun main() {
    set cmd = parse()
    switch cmd {
        Command::Build(build_args) -> {
            println("Building for target: " + build_args.target)
            if build_args.release {
                println("Optimization: release")
            }
            println("Output: " + build_args.output)
        }
        Command::Run(run_args) -> {
            println("Starting server on port: " + cast(run_args.port, "string"))
            println("Loading file: " + run_args.file)
        }
    }
}
```

### 3. Run and test

```sh
# Display auto-generated colored help
./demo --help

# Run a subcommand with options and positionals
./demo build --release -o myapp x86_64

# Typo correction in action
./demo bulid
# error: unrecognized subcommand 'bulid'
#   tip: a similar subcommand exists: 'build'
#
# For more information, try '--help'.
```

---

## Documentation

Full documentation is available in English and Russian:

- **English Documentation**:
  - [Overview and Quick Start](docs/en/0-overview.md)
  - [Clarity Schema Language (`.cty`)](docs/en/1-schema.md)
  - [Runtime API and Features](docs/en/2-runtime.md)

- **Russian Documentation**:
  - [Обзор и быстрый старт](docs/ru/0-overview.md)
  - [Язык схем Clarity (`.cty`)](docs/ru/1-schema.md)
  - [Рантайм API и возможности](docs/ru/2-runtime.md)

---

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

Copyright (c) 2026 shareui.
