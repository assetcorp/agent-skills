# Click / Typer (Python)

Framework-specific patterns for applying the agent-friendly CLI checklist in Python. Click is the foundation; Typer adds type-hint convenience on top.

## Setup

```bash
pip install click
# or for Typer (includes Click)
pip install typer[all]
```

## Non-interactive flags (Checklist 1)

Click uses decorators for arguments. Every parameter is a flag by default:

```python
import click

@click.command()
@click.option("--env", required=True, type=click.Choice(["staging", "production"]))
@click.option("--tag", default="latest", help="Image tag")
@click.option("--force", is_flag=True, help="Skip confirmation")
def deploy(env, tag, force):
    if not force:
        click.confirm(f"Deploy {tag} to {env}?", abort=True)
    run_deploy(env, tag)
```

Typer equivalent using type hints:

```python
import typer
from enum import Enum

class Environment(str, Enum):
    staging = "staging"
    production = "production"

app = typer.Typer()

@app.command()
def deploy(
    env: Environment,
    tag: str = "latest",
    force: bool = typer.Option(False, help="Skip confirmation"),
):
    if not force:
        typer.confirm(f"Deploy {tag} to {env.value}?", abort=True)
    run_deploy(env.value, tag)
```

## Structured output (Checklist 2)

Add an `--output` flag that switches between human and JSON output:

```python
import json
import click

@click.command()
@click.option("--output", "output_format", type=click.Choice(["text", "json"]), default="text")
def deploy(output_format):
    result = {"status": "success", "data": {"deploy_id": "dep_abc123"}, "error": None}

    if output_format == "json":
        click.echo(json.dumps(result))
    else:
        click.echo(f"Deployed successfully: {result['data']['deploy_id']}")
```

## Help with examples (Checklist 3)

Click supports epilog for examples. Use `\b` to prevent Click from rewrapping text:

```python
@click.command(
    epilog="""
\b
Examples:
  mycli deploy --env staging
  mycli deploy --env production --tag v1.2.3
  mycli deploy --env staging --force
"""
)
def deploy():
    pass
```

## Actionable errors (Checklist 4)

Use `click.UsageError` for argument problems and `click.ClickException` for runtime failures:

```python
@click.command()
@click.option("--tag")
def deploy(tag):
    if not tag:
        raise click.UsageError(
            "No image tag specified.\n\n"
            "  mycli deploy --env staging --tag <image-tag>\n"
            "  Available tags: mycli build list --output tags"
        )
```

## Exit codes (Checklist 8)

```python
import sys

@click.command()
def deploy():
    try:
        result = run_deploy()
    except ConfigError:
        click.echo("Configuration error: ...", err=True)
        sys.exit(3)
    except PermissionError:
        click.echo("Permission denied: ...", err=True)
        sys.exit(5)
```

## Stdin support (Checklist 9)

Click has built-in stdin support with `click.File`:

```python
@click.command()
@click.option("--input", "input_file", type=click.File("r"), default="-")
def process(input_file):
    data = input_file.read()
```

The `default="-"` makes the command read from stdin when no file is specified.

## Env var support (Checklist 10)

Click supports env vars natively per option:

```python
@click.command()
@click.option("--token", envvar="MYCLI_TOKEN", help="API token [$MYCLI_TOKEN]")
@click.option("--env", envvar="MYCLI_ENV", help="Target environment [$MYCLI_ENV]")
def deploy(token, env):
    pass
```

## Command groups (Checklist 11)

```python
@click.group()
def cli():
    pass

@cli.group()
def service():
    pass

@service.command()
def create():
    pass

@service.command()
def list():
    pass
```

This produces `mycli service create`, `mycli service list`, etc.
