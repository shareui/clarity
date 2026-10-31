# Clarity Schema Language (.cty)

Files with the `.cty` extension provide a declarative description of a command-line interface using a specialized Domain Specific Language (DSL). The schema file is processed by a procedural macro during compilation, establishing a strictly-typed grammar without manual string inspection.

---

## 1. Schema Directives Reference

| Directive | Scope | Status | Purpose |
| :--- | :--- | :--- | :--- |
| `app "<name>"` | Schema Header | **Required** | Executable name used in `USAGE:` help lines. If omitted, the compiler raises error 105. |
| `version "<version>"` | Schema Header | Optional | Program version displayed for `--version`, `-V`, and `-v`. |
| `author "<author>"` | Schema Header | Optional | Author or organization information. |
| `about "<description>"` | Schema Header | Optional | High-level summary of the program shown in root `--help`. |
| `default_flags true/false` | Schema Header | Optional | Automatically includes `-h, --help` and `-V, --version` (defaults to `true`). |
| `config "<path>"` | Schema Header | Optional | Relative path to TOML configuration file (stored in user directory `~/.config/`). |
| `define * { ... }` | Schema Body | Optional | Block of **common (global) flags** inherited across all subcommands. |
| `define { ... }` | Schema Body | Optional | **Root command** (executed when the application is invoked without subcommands). |
| `define <name> { ... }` | Schema Body | Optional | Named **subcommand** with its own isolated flags and positional parameters. |
| `uncommon define <name>` | Schema Body | Optional | Subcommand **isolated** from common flags declared in `define *`. |
| `config define <name>` | Schema Body | Optional | Built-in subcommand for reading and modifying configuration keys. |

---

## 2. Understanding `define *`, Root `define`, and Subcommands

Distinguishing between these command constructs is essential when designing your CLI:

```
+---------------------------------------------------------------+
|                       Clarity Schema (.cty)                   |
|                                                               |
|  1. define * { ... }             -> Common flags (--verbose)  |
|                                     (inherited by subcommands)|
|                                                               |
|  2. define "about" { ... }       -> Root command              |
|                                     (invoked: ./app --flag)   |
|                                                               |
|  3. define build { ... }         -> Named subcommand          |
|                                     (invoked: ./app build ...) |
+---------------------------------------------------------------+
```

### 2.1 Common Flags: `define * { ... }`

The `define *` block declares flags that should be available across **every subcommand** in your program (such as `--verbose`, `--quiet`, or `--json`).

Example schema:

```cty
app "cloudcli"
version "1.0.0"

// Common flags for all subcommands
define * {
    --verbose, -v: bool = false "Enable detailed log output";
    --json: bool = false "Format output as JSON";
}

define deploy {
    param env: string "Target deployment environment";
}

define status {
    --timeout: int32 = 30 "Request timeout in seconds";
}
```

#### What gets compiled behind the scenes:

```yukino
// Dedicated struct for shared flags
public struct CommonFlags {
    verbose: bool
    json: bool
}

public struct DeployArgs {
    env: string
}

public struct StatusArgs {
    timeout: int32
}

// Every command variant bundles both common and local options
public enum Command {
    Deploy(common: CommonFlags, args: DeployArgs),
    Status(common: CommonFlags, args: StatusArgs),
}
```

#### Expected Terminal Invocations:

```sh
# Deploying with a long global flag
./cloudcli deploy production --verbose
# In application code:
# common.verbose == true, common.json == false
# args.env == "production"

# Using the short flag -v
./cloudcli deploy staging -v
# In application code:
# common.verbose == true, common.json == false
# args.env == "staging"

# Checking status with custom timeout and JSON output
./cloudcli status --timeout 60 --json
# In application code:
# common.verbose == false, common.json == true
# args.timeout == 60
```

> If a specific command should **not** inherit common flags, prepend the `uncommon` keyword:
> ```cty
> uncommon define ping {
>     param host: string;
> }
> ```
> In the generated code, `Command::Ping(args: PingArgs)` will not contain a `common` field.

---

### 2.2 Root Command: `define { ... }`

A root command is declared with `define` immediately followed by an optional description string or directly by an opening brace `{` (without any command name).

Use a root command when your program does not have subcommands (like `cat`, `grep`, or `curl`), or when it can be called directly with flags at the top level:

Example schema:

```cty
app "minicat"
version "0.1.0"
about "Concatenate and display files"

define "Print file contents to standard output" {
    --numbers, -n: bool = false "Number all output lines";
    param file: string "Source file path";
}
```

#### What gets compiled behind the scenes:

```yukino
public struct RootArgs {
    numbers: bool
    file: string
}

public enum Command {
    Root(args: RootArgs),
}
```

#### Expected Terminal Invocations:

```sh
# Running with the short flag
./minicat -n document.txt
# In application code: args.numbers == true, args.file == "document.txt"

# Running with the long flag
./minicat --numbers /etc/hosts
# In application code: args.numbers == true, args.file == "/etc/hosts"

# Running without optional flags (defaults to false)
./minicat readme.md
# In application code: args.numbers == false, args.file == "readme.md"
```

---

### 2.3 Named Subcommands: `define <name> { ... }`

Used for multi-purpose CLI utilities with distinct subcommands (like `git commit`, `docker run`, or `cargo build`):

```cty
app "pkgtool"
version "2.0.0"

define init "Initialize a new project" {
    --name: string = "app" "Project package name";
}

define publish "Publish package to registry" {
    --dry-run: bool = false "Validate package without uploading";
    param registry: string = "default" "Target registry";
}
```

#### What gets compiled behind the scenes:

```yukino
public struct InitArgs {
    name: string
}

public struct PublishArgs {
    dry_run: bool
    registry: string
}

public enum Command {
    Init(args: InitArgs),
    Publish(args: PublishArgs),
}
```

#### Expected Terminal Invocations:

```sh
# Initialize with default package name
./pkgtool init
# In application code: args.name == "app"

# Initialize with explicit name
./pkgtool init --name my_app
# In application code: args.name == "my_app"

# Publish with positional registry argument
./pkgtool publish crates-io
# In application code: args.dry_run == false, args.registry == "crates-io"

# Publish in dry-run mode
./pkgtool publish internal --dry-run
# In application code: args.dry_run == true, args.registry == "internal"
```

---

### 2.4 Nested Subcommands: define inside define

To structure multi-level CLI applications (such as `git remote add`, `npm config set`, or `docker container run`), the `clarity` library supports nesting `define` blocks inside parent commands to arbitrary depths.

Example schema with nested subcommands:

```cty
app "mytool"
version "1.0.0"

define config "Manage application configuration" {
    --global, -g: bool = false "Operate on system-wide configuration";

    define set "Set configuration key value" {
        param key: string "Configuration key name";
        param val: string "New key value";
    }

    define get "Retrieve configuration key value" {
        param key: string "Configuration key name";
    }

    define list "List all stored configuration settings" {
        --json: bool = false "Format output as JSON";
    }
}
```

#### What gets compiled behind the scenes:

A `<Parent>Subcommand` enum is generated for the parent command, and an optional `sub` field is added to the parent argument struct:

```yukino
// Nested subcommand argument structs
public struct ConfigSetArgs {
    key: string
    val: string
}

public struct ConfigGetArgs {
    key: string
}

public struct ConfigListArgs {
    json: bool
}

// Subcommand variant enum
public enum ConfigSubcommand {
    Set(args: ConfigSetArgs),
    Get(args: ConfigGetArgs),
    List(args: ConfigListArgs),
}

// Parent command argument struct
public struct ConfigArgs {
    global: bool
    sub: ConfigSubcommand?
}

// Main application command enum
public enum Command {
    Config(args: ConfigArgs),
}
```

#### Usage in Application Code:

```yukino
switch cmd {
    Command::Config(cfg) -> {
        if cfg.global {
            println("Using global configuration scope")
        }
        switch cfg.sub {
            ConfigSubcommand::Set(args) -> {
                println("Setting: " + args.key + " = " + args.val)
            }
            ConfigSubcommand::Get(args) -> {
                println("Retrieving: " + args.key)
            }
            ConfigSubcommand::List(args) -> {
                if args.json {
                    println("{\"status\": \"all_configs\"}")
                } else {
                    println("Listing all configuration keys")
                }
            }
            null -> {
                println("No action specified. Try 'mytool config --help' for available actions.")
            }
        }
    }
}
```

#### Expected Terminal Invocations:

```sh
# Setting a configuration value
./mytool config set user.theme dark
# In application code:
# cfg.global == false
# args.key == "user.theme", args.val == "dark"

# Reading a configuration value with parent flag
./mytool config --global get user.theme
# In application code:
# cfg.global == true
# args.key == "user.theme"

# Listing all settings as JSON
./mytool config list --json
# In application code:
# cfg.global == false
# args.json == true

# Viewing parent command help (lists all available subcommands)
./mytool config --help

# Viewing specific nested subcommand help
./mytool config set --help
```

---

## 3. Defining Options and Flags

Flags are declared inside command blocks with a double-dash prefix `--`.

### Flag Declaration Syntax:

```cty
--<long_name> [, -<short_name>] : <type> [?] [ = <default_value> ] [modifiers...] ["about"];
```

Component requirements:
- Long name `--name`: **Required**.
- Short name `,-n`: **Optional** (single ASCII letter).
- Data type `: <type>`: **Required**.
- Nullable modifier `?`: **Optional** (generates `T?` in Yukino).
- Default value `= <val>`: **Optional**. For `bool`, defaults to `false`; for `count`, defaults to `0`. If a non-nullable option has **no default value**, it is treated as **[required]**, and the program exits with code 2 if omitted.
- Help description `"text"`: **Optional**.

### Supported Data Types:

| Schema Type | Yukino Type | Default Behavior | Description |
| :--- | :--- | :--- | :--- |
| `bool` | `bool` | `false` | Boolean toggle flag (`--release`) |
| `count` | `uint8` | `0` | Multi-occurrence counter (`-v` -> 1, `-vv` -> 2) |
| `int32` | `int32` | required or default | 32-bit signed integer (`--port 8080`) |
| `int` / `int64` | `int64` | required or default | 64-bit signed integer (`--limit 100000`) |
| `float` / `float64` | `float64` | required or default | 64-bit floating point number (`--ratio 0.75`) |
| `string` / `str` / `path` | `string` | required or default | String or file path (`--output file.txt`) |
| `string[]` / `path[]` | `string[]` | empty slice `[]` | Repeated options (`-i a.txt -i b.txt`) |
| `int[]` | `int32[]` | empty slice `[]` | Repeated integer parameters |
| `enum { A, B, C }` | `string` | required or default | Choice restricted to listed variants |

### Example with Diverse Flag Types:

```cty
define server "Configure server runtime" {
    --daemon, -d: bool = false "Run in background as daemon";
    --workers, -w: int32 = 4 "Worker thread count";
    --host: string = "127.0.0.1" "Network interface binding";
    --log-level: enum { debug, info, warn, error } = "info" "Logging verbosity";
    --cert: path? "Optional TLS certificate path";
    --tags: string[] "Process tags for monitoring";
}
```

#### What gets compiled behind the scenes:

```yukino
public struct ServerArgs {
    daemon: bool
    workers: int32
    host: string
    log_level: string
    cert: string?
    tags: string[]
}
```

#### Expected Terminal Invocations:

```sh
# Running server with default settings
./server_app server
# In application code:
# args.daemon == false
# args.workers == 4
# args.host == "127.0.0.1"
# args.log_level == "info"
# args.cert == null
# args.tags == []

# Running with custom workers, host, debug log level, certificate, and tags
./server_app server -d -w 8 --host 0.0.0.0 --log-level debug --cert /etc/ssl/cert.pem --tags web --tags api
# In application code:
# args.daemon == true
# args.workers == 8
# args.host == "0.0.0.0"
# args.log_level == "debug"
# args.cert == "/etc/ssl/cert.pem"
# args.tags == ["web", "api"]
```

---

### 3.4 Overriding and Disabling Standard Help and Version Flags

By default, the compiler automatically injects standard help and version flags:
- `-h`, `--help`: Prints command help and exits with status code `0`.
- `-V`, `-v`, `--version`: Prints application version and exits with status code `0` (if `version` is set).

The library provides two flexible mechanisms to customize or disable this behavior:

#### 1. Flag Overrides

If you define a custom flag with the name `help`, `h`, `version`, `V`, or `v`, the runtime **automatically disables** the standard interceptor for that flag and passes execution directly to your application code.

The most common practical example is reusing `-v` for verbosity levels rather than version output:

```cty
app "logger"
version "1.0.0"

define * {
    // Override the short flag -v for logging verbosity:
    --verbose, -v: count = 0 "Logging verbosity (-v, -vv, -vvv)";
}

define run {
    param target: string "Target file to run";
}
```

#### Expected Terminal Invocations:

```sh
# Flag -v is evaluated as a custom counter, not as version!
./logger run -vvv app.bin
# In application code: common.verbose == 3, args.target == "app.bin"

# Version flags -V and --version remain functional
./logger --version
# Output: logger 1.0.0
```

If custom help logic is needed:

```cty
define export {
    --help, -h: bool = false "Display extended export manual";
}
```

When invoking `./app export --help`, the runtime does not terminate the process, but sets `args.help = true` inside `main`.

#### 2. Disabling Completely via `default_flags false`

If your CLI tool should not provide automatic help or version flags (for example, strict protocol tools or non-interactive binaries), specify `default_flags false` in the schema header:

```cty
app "raw_service"
default_flags false

define {
    --port: int32 = 8080 "Service network port";
}
```

When `default_flags false` is enabled:
- The runtime does not intercept `-h`, `--help`, `-v`, `-V`, or `--version`.
- Standard help lines are omitted from `FLAGS:`.
- If a user passes `--help`, the CLI reports an unexpected option error (`error: unexpected argument '--help' found`) with exit code `2`, unless explicitly defined by you.

---

## 4. Defining Positional Parameters (`param`)

Positional parameters are supplied without flag names and match according to their sequential appearance on the command line.

### Parameter Declaration Syntax:

```cty
param <name> : <type> [?] [ = <default_value> ] [modifiers...] ["about"];
```

Component requirements:
- `param <name>`: **Required**.
- Data type `: <type>`: **Required**.
- Nullable modifier `?`: **Optional**.
- Array modifier `[]` (variadic): **Optional**. Matches one or more values into a slice.
- Default value `= <val>`: **Optional**. Non-nullable parameters without defaults are marked **[required]**.

> [!IMPORTANT]
> A variadic parameter (`param files: string[]`) **must be the last positional parameter** in the command definition. Declaring additional positional parameters after a variadic parameter results in compile error 103.

### Example:

```cty
define copy "Copy files between paths" {
    --force, -f: bool = false "Overwrite existing destination";
    param source: string "Source file path";
    param destination: string "Destination path";
}

define archive "Compress multiple files into an archive" {
    param target_archive: string "Output archive name";
    param inputs: string[] "Input files to compress";
}
```

#### What gets compiled behind the scenes:

```yukino
public struct CopyArgs {
    force: bool
    source: string
    destination: string
}

public struct ArchiveArgs {
    target_archive: string
    inputs: string[]
}
```

#### Expected Terminal Invocations:

```sh
# Standard file copy
./fileutil copy src/main.ykn dist/main.ykn
# In application code:
# args.force == false
# args.source == "src/main.ykn"
# args.destination == "dist/main.ykn"

# File copy with force flag
./fileutil copy -f old.txt new.txt
# In application code:
# args.force == true
# args.source == "old.txt"
# args.destination == "new.txt"

# Archiving a variable number of files
./fileutil archive bundle.tar a.txt b.txt c.txt
# In application code:
# args.target_archive == "bundle.tar"
# args.inputs == ["a.txt", "b.txt", "c.txt"]
```

---

## 5. Flag Modifiers and Constraints

Modifiers specify relationships between parameters, fallback data sources, and custom diagnostics:

### 5.1 Environment Variable Fallback: `env(...)`

Reads the value from an environment variable when the option is omitted from CLI arguments:

```cty
define connect {
    --token: string env(API_TOKEN) "API authentication token";
    --host: string = "localhost" env(SERVICE_HOST) "Service hostname";
}
```

#### Expected Terminal Invocations:

```sh
# Providing token explicitly via CLI
./connect --token "secret-key-123"

# Relying on environment variable fallback
export API_TOKEN="secret-key-123"
./connect
# In application code: args.token == "secret-key-123", args.host == "localhost"
```

### 5.2 Configuration File Fallback: `config(...)`

Reads the value from the application's TOML configuration file when CLI flags and environment variables are absent:

```cty
define build {
    --output: string config(build.output) = "dist/app" "Build destination";
}
```

### 5.3 Co-Dependency: `requires`

Enforces that an option can only be supplied when another option is also present:

```cty
define publish {
    --signed: bool = false "Enable digital signature";
    --key: path requires signed "Private key path for signature";
}
```

#### Expected Terminal Invocations:

```sh
# Valid invocation (both required flags present)
./pkg publish --signed --key cert.pem

# Invalid invocation (missing dependency)
./pkg publish --key cert.pem
# error: the argument '--key' requires '--signed'
```

### 5.4 Mutual Exclusivity: `conflicts`

Prevents two incompatible flags from being used together:

```cty
define test {
    --all: bool = false conflicts target "Run all tests";
    --target: string conflicts all "Run a single test target";
}
```

#### Expected Terminal Invocations:

```sh
# Valid invocations
./test_runner test --all
./test_runner test --target auth_tests

# Invalid invocation (mutually exclusive options)
./test_runner test --all --target auth_tests
# error: the argument '--all' cannot be used with '--target'
```

### 5.5 Custom Error Messages: `error "..."`

Overrides standard error messages with domain-specific explanations:

```cty
define run {
    --port: int32 error "Port must be a valid number between 1 and 65535" range(1, 65535);
}
```

---

## 6. Value Validators

Validators verify user input before execution passes to application code:

| Validator | Applicable Types | Purpose |
| :--- | :--- | :--- |
| `range(min, max)` | Numeric types (`int`, `float`) | Checks that value is within `[min, max]` inclusive. |
| `pattern("<regex>")` | Text types | Matches string against regular expression via `std.regex`. |
| `min_len(n)` | String types | Enforces minimum string length in characters. |
| `max_len(n)` | String types | Enforces maximum string length in characters. |
| `one_of("a", "b", ...)` | String types | Restricts value to listed string variants. |
| `exists` | `path`, `string` | Verifies that file or directory exists on disk. |
| `parent_exists` | `path`, `string` | Verifies that the parent directory of path exists. |

### Example Schema with Validators:

```cty
define deploy {
    --workers: int32 = 4 range(1, 64) "Worker count between 1 and 64";
    --version-tag: string pattern("^v[0-9]+\\.[0-9]+\\.[0-9]+$") "Semver tag like v1.0.0";
    --config-file: path exists "Existing configuration file";
    --report-file: path parent_exists "Report file (parent folder must exist)";
}
```

#### Expected Terminal Invocations:

```sh
# Valid invocation
./deployer deploy --workers 8 --version-tag v2.1.0 --config-file app.toml --output-file dist/log.txt

# Invalid invocation (range violation)
./deployer deploy --workers 128
# panic: flag '--workers' value 128 out of range [1, 64]
```

---

## 7. Option Groups (`or`, `and`)

Groups enforce logical constraints across sets of flags.

### 7.1 Mutually Exclusive Group: `or`

Permits at most one option from the group:

```cty
define format "Format output stream" {
    or output_mode ? {
        --json: bool = false "Format as JSON";
        --yaml: bool = false "Format as YAML";
        --xml: bool = false "Format as XML";
    }
}
```

- `or <name> ?`: The question mark makes the group **optional** (0 or 1 flag may be given).
- `or <name>`: Without the question mark, the group is **required** (exactly 1 flag must be given).

#### What gets compiled behind the scenes:

```yukino
public struct FormatArgs {
    json: bool
    yaml: bool
    xml: bool
}
```

#### Expected Terminal Invocations:

```sh
# Valid invocation
./formatter format --json
# In application code: args.json == true, args.yaml == false, args.xml == false

# Invalid invocation (multiple conflicting options)
./formatter format --json --yaml
# error: the argument '--json' cannot be used with '--yaml'
```

### 7.2 Co-Dependent Group: `and`

Requires all options in the group to be provided together if any one is passed:

```cty
define login "Authenticate against remote service" {
    and credentials ? {
        --username: string "User account name";
        --password: string "Account secret key";
    }
}
```

#### What gets compiled behind the scenes:

```yukino
public struct LoginArgs {
    username: string
    password: string
}
```

#### Expected Terminal Invocations:

```sh
# Valid invocation (both options supplied together)
./auth login --username admin --password secret
# In application code: args.username == "admin", args.password == "secret"

# Invalid invocation (missing co-dependent option)
./auth login --username admin
# error: the following required flag was not provided: '--password'
```

---

[Back: 0. Overview and Quick Start](0-overview.md) | [Next: 2. Runtime API and Features](2-runtime.md)
