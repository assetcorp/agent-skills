# Clap (Rust)

Framework-specific patterns for applying the agent-friendly CLI checklist in Rust. Clap has 302M+ crate downloads and is the standard for Rust CLIs.

## Setup

```toml
# Cargo.toml
[dependencies]
clap = { version = "4", features = ["derive"] }
serde_json = "1"
```

## Non-interactive flags (Checklist 1)

Using Clap's derive API:

```rust
use clap::{Parser, ValueEnum};

#[derive(ValueEnum, Clone)]
enum Environment {
    Staging,
    Production,
}

#[derive(Parser)]
#[command(name = "mycli")]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Parser)]
enum Commands {
    Deploy(DeployArgs),
}

#[derive(Parser)]
struct DeployArgs {
    #[arg(long, value_enum)]
    env: Environment,

    #[arg(long, default_value = "latest")]
    tag: String,

    #[arg(long, help = "Skip confirmation")]
    force: bool,
}
```

## Structured output (Checklist 2)

Add a global output format flag:

```rust
use clap::ValueEnum;
use serde::Serialize;

#[derive(ValueEnum, Clone)]
enum OutputFormat {
    Text,
    Json,
}

#[derive(Parser)]
struct Cli {
    #[arg(long, value_enum, default_value = "text", global = true)]
    output: OutputFormat,

    #[command(subcommand)]
    command: Commands,
}

#[derive(Serialize)]
struct ApiResult<T: Serialize> {
    status: String,
    data: T,
    error: Option<ApiError>,
}

fn print_result<T: Serialize + std::fmt::Display>(data: T, format: &OutputFormat) {
    match format {
        OutputFormat::Json => {
            let result = ApiResult {
                status: "success".into(),
                data,
                error: None,
            };
            println!("{}", serde_json::to_string_pretty(&result).unwrap());
        }
        OutputFormat::Text => println!("{data}"),
    }
}
```

## Help with examples (Checklist 3)

Clap supports `after_help` for examples:

```rust
#[derive(Parser)]
#[command(
    about = "Deploy an application to a target environment",
    after_help = "\
Examples:
  mycli deploy --env staging
  mycli deploy --env production --tag v1.2.3
  mycli deploy --env staging --force"
)]
struct DeployArgs {
    #[arg(long, value_enum)]
    env: Environment,
}
```

## Actionable errors (Checklist 4)

Use `clap::Error` for argument errors and `anyhow` or custom errors for runtime failures:

```rust
use std::process::ExitCode;

fn main() -> ExitCode {
    let cli = Cli::parse();

    match run(cli) {
        Ok(()) => ExitCode::SUCCESS,
        Err(e) => {
            eprintln!("Error: {e}");
            if let Some(suggestion) = e.suggestion() {
                eprintln!("  {suggestion}");
            }
            e.exit_code()
        }
    }
}
```

## Exit codes (Checklist 8)

```rust
use std::process::ExitCode;

enum AppExitCode {
    Success = 0,
    General = 1,
    Usage = 2,
    Config = 3,
    NotFound = 4,
    Permission = 5,
    Network = 10,
}

impl From<AppExitCode> for ExitCode {
    fn from(code: AppExitCode) -> Self {
        ExitCode::from(code as u8)
    }
}

impl AppError {
    fn exit_code(&self) -> ExitCode {
        match self {
            AppError::Config(_) => AppExitCode::Config.into(),
            AppError::NotFound(_) => AppExitCode::NotFound.into(),
            AppError::Permission(_) => AppExitCode::Permission.into(),
            _ => AppExitCode::General.into(),
        }
    }
}
```

## Stdin support (Checklist 9)

```rust
use std::io::{self, Read};

#[derive(Parser)]
struct ImportArgs {
    #[arg(long, help = "Read from stdin")]
    stdin: bool,

    #[arg(long, help = "Read from file")]
    file: Option<String>,
}

fn read_input(args: &ImportArgs) -> io::Result<String> {
    if args.stdin {
        let mut buf = String::new();
        io::stdin().read_to_string(&mut buf)?;
        Ok(buf)
    } else if let Some(path) = &args.file {
        std::fs::read_to_string(path)
    } else {
        Err(io::Error::new(
            io::ErrorKind::InvalidInput,
            "provide --stdin or --file <path>",
        ))
    }
}
```

## Env var support (Checklist 10)

Clap supports env vars natively per argument:

```rust
#[derive(Parser)]
struct DeployArgs {
    #[arg(long, env = "MYCLI_TOKEN", help = "API token [$MYCLI_TOKEN]")]
    token: String,

    #[arg(long, env = "MYCLI_ENV", help = "Target environment [$MYCLI_ENV]")]
    env: Environment,
}
```

## Subcommands (Checklist 11)

```rust
#[derive(Parser)]
enum Commands {
    #[command(about = "Manage services")]
    Service(ServiceArgs),
    #[command(about = "Manage configuration")]
    Config(ConfigArgs),
}

#[derive(Parser)]
struct ServiceArgs {
    #[command(subcommand)]
    command: ServiceCommands,
}

#[derive(Parser)]
enum ServiceCommands {
    Create(CreateArgs),
    List(ListArgs),
    Delete(DeleteArgs),
}
```
