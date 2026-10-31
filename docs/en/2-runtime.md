# Clarity Runtime API and Features (clarity_rt)

The `clarity_rt` module executes the bytecode compiled from `.cty` schemas, handles command-line argument parsing, formats colored help and version information, provides typo suggestions, performs runtime validation, generates shell completion scripts, and interfaces with configuration files.

---

## 1. Import: load clarity

```yukino
load clarity: parse, parse_args, try_parse, try_parse_args, Command, ClarityRuntimeError, Config, config_open, config_open_file, generate_bash, generate_zsh, generate_fish
```

---

## 2. Data Structures and Error Codes

### 2.1 ClarityRuntimeError

A runtime error structure thrown via `panic` during critical validation failures or corrupted bytecode input:

```yukino
public struct ClarityRuntimeError {
    message: string
    code: int32
}
```

Runtime validator error codes:
- `101`: Integer value outside permitted `range(min, max)` (`validate_range_int`).
- `102`: String does not match regular expression `pattern("...")` (`validate_pattern`).
- `103`: Target path does not exist on disk `exists` (`validate_exists`).
- `104`: Parent directory of path does not exist `parent_exists` (`validate_parent_exists`).
- `105`: String length is less than minimum requirement `min_len(n)` (`validate_min_len`).
- `106`: String length exceeds permitted maximum `max_len(n)` (`validate_max_len`).
- `107`: Supplied value is not in permitted list of variants `one_of(...)` (`validate_one_of`).

### 2.2 CLI Process Exit Codes

`clarity` complies with standard Unix CLI conventions:
- **Code `0` (Success)**: Clean execution, or after displaying built-in help (`--help`, `-h`, `help <cmd>`) or version (`--version`, `-V`, `-v`).
- **Code `2` (CLI Syntax Error)**: Unrecognized option, missing required positional argument, missing value for an option, mutual exclusivity conflict, or misspelled command.

---

## 3. Parsing Functions Reference

### 3.1 parse() -> Command

Parses the real command-line arguments of the current process obtained from `args()`.

```yukino
public fun parse() -> Command
```

- **Returns:** The generated `Command` enum variant holding populated argument structs.
- **Help and Version Behavior:** Automatically prints ANSI-styled help or version and terminates with exit code `0`.
- **Error Behavior:** Prints formatted error messages with typo suggestions and terminates with exit code `2`.

Example usage:

```yukino
load std.io: println
load clarity: *

#[clarity::scheme("cli.cty")]
fun clarity_init() { space }

fun main() {
    set cmd = parse()
    switch cmd {
        Command::Build(args) -> println("Output: " + args.output)
        Command::Run(args) -> println("File: " + args.file)
    }
}
```

---

### 3.2 parse_args(cli_args: string[]) -> Command

Parses a custom slice of argument strings rather than the current process arguments.

```yukino
public fun parse_args(cli_args: string[]) -> Command
```

- **Arguments:** `cli_args` — array of argument strings without the binary name (e.g., `["build", "--release", "x86_64"]`).
- **Purpose:** Ideal for automated unit and integration testing without launching new OS processes.
- **Error Behavior:** Matches `parse()` behavior (terminates with exit code `2` on syntax error).

Unit test example:

```yukino
#[test]
fun test_cli_parsing() {
    set cmd = parse_args(["build", "--release", "arm64"])
    switch cmd {
        Command::Build(args) -> {
            assert(args.release == true)
            assert(args.target == "arm64")
        }
        else -> panic("expected build command")
    }
}
```

---

### 3.3 try_parse() -> Command?

Safe variant of `parse()` that returns `null` instead of terminating the process on error.

```yukino
public fun try_parse() -> Command?
```

- **Returns:** The parsed `Command` variant, or `null` if parsing fails.

---

### 3.4 try_parse_args(cli_args: string[]) -> Command?

Safe variant of `parse_args()` that returns `null` on parsing error.

```yukino
public fun try_parse_args(cli_args: string[]) -> Command?
```

- **Returns:** The parsed `Command` variant, or `null` if parsing fails.

---

## 4. Configuration File Management: Config Class

The `Config` class reads and writes persistent application settings in TOML format.

### 4.1 Factory Functions

#### config_open(app_name: string) -> Config

Opens or creates a configuration file in the user's config directory: `~/.config/clarity-cli/<app_name>/config.toml`.

```yukino
set cfg = config_open("mytool")
```

#### config_open_file(file_path: string) -> Config

Opens or creates a configuration file at an explicit path. Parent folders are created automatically if missing.

```yukino
set cfg = config_open_file("/etc/mytool/settings.toml")
```

---

### 4.2 Config Class Methods

#### get_str(key: string, fallback: string) -> string
Returns the string value for `key`, or `fallback` if absent.

#### get_int(key: string, fallback: int64) -> int64
Returns the 64-bit integer value for `key`, or `fallback`.

#### get_float(key: string, fallback: float64) -> float64
Returns the float value for `key`, or `fallback`.

#### get_bool(key: string, fallback: bool) -> bool
Returns the boolean value for `key`, or `fallback`.

#### get_str_array(key: string) -> string[]
Returns an array of strings for `key`, or an empty slice if absent.

#### set_str(key: string, val: string)
Sets a string value in the configuration document.

#### set_int(key: string, val: int64)
Sets an integer value.

#### set_bool(key: string, val: bool)
Sets a boolean value.

#### has(key: string) -> bool
Checks if the specified key exists.

#### keys() -> string[]
Returns a list of all root-level keys.

#### remove(key: string)
Removes a key from the document.

#### save() -> bool
Writes the configuration document to disk. Returns `true` on success.

Example usage:

```yukino
load clarity: config_open

fun save_settings() {
    set cfg = config_open("mytool")
    cfg.set_str("theme", "dracula")
    cfg.set_int("max_connections", 100)
    cfg.set_bool("auto_update", true)
    cfg.save()
}
```

---

## 5. Shell Completion Script Generators

The library includes generators for Bash, Zsh, and Fish shell completions:

### 5.1 generate_bash(ref schema: RtSchema) -> string

Generates a Bash completion script compatible with `bash-completion`.

### 5.2 generate_zsh(ref schema: RtSchema) -> string

Generates a Zsh completion script using the standard `_describe` format.

### 5.3 generate_fish(ref schema: RtSchema) -> string

Generates a Fish shell completion script with descriptions.

Example command exposing completion generation:

```yukino
// In cli.cty:
// define completions { param shell: string "Target shell (bash, zsh, fish)"; }

Command::Completions(args) -> {
    if args.shell == "bash" {
        println(generate_bash(ref vm.schema))
    } elif args.shell == "zsh" {
        println(generate_zsh(ref vm.schema))
    } elif args.shell == "fish" {
        println(generate_fish(ref vm.schema))
    }
}
```

---

[Back: 1. Clarity Schema Language (`.cty`)](1-schema.md) | [Top: 0. Overview and Quick Start](0-overview.md)
