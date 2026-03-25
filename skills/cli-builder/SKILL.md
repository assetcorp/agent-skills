---
name: cli-builder
description: Build CLIs that work for humans, agents, and automation. Covers non-interactive design, structured output, actionable errors, idempotency, and piping support. Use when creating new CLI tools, retrofitting existing ones for non-interactive use, auditing CLI agent-friendliness, or when the user mentions CLI design, command-line tool creation, or making a CLI work with LLM agents, CI/CD pipelines, or scripts.
---

# CLI Builder

Build command-line interfaces that work for both humans and machines. Every CLI you help create or improve should score well against the checklist below.

## How to use this skill

Detect the user's intent from context:

- **Building a new CLI**: Help pick the right framework for their language (see `references/`), then structure each command to pass the checklist.
- **Retrofitting an existing CLI**: Audit against the checklist, then make changes starting with the highest-impact gaps.
- **Auditing/reviewing**: Score each checklist item as pass/fail/partial with a concrete explanation. Recommend fixes in priority order.

## The Machine-Friendly CLI Checklist

Score every CLI against these 12 properties. Each one maps to a concrete, verifiable behavior.

### 1. Non-interactive by default

Every input must be passable as a flag. Interactive prompts work as a fallback when flags are missing, not the primary path. Automated callers (LLM agents, CI runners, shell scripts) can't navigate arrow-key menus or answer "y/n" at the right moment.

```bash
# blocks any automated caller
$ warehouse inventory
? Select warehouse region: (use arrow keys)

# passable as a flag
$ warehouse inventory --region us-east-1
```

### 2. Structured output mode

Support a `--output json` (or `--json`) flag for machine-readable output. Wrap every response in a consistent envelope so callers can parse success and failure the same way:

```json
{
  "status": "success",
  "data": {
    "migration_id": "mig_7f3a9b",
    "tables_affected": 12,
    "rollback_ref": "mig_7f3a9b_undo"
  },
  "error": null,
  "metadata": { "duration_ms": 8200 }
}
```

On failure, keep the same envelope shape but set `"status": "error"` and populate `error` with `code`, `message`, and `suggestion` fields. Human-friendly text remains the default when `--output json` is absent.

### 3. Discoverable, layered help

Give every subcommand its own `--help` and include at least two usage examples in each. Automated callers pattern-match off concrete invocations faster than they parse prose descriptions.

```bash
$ pkgr publish --help
Publish a package to the registry.

Options:
  --name      Package name (required)
  --version   Semver tag (required)
  --registry  Target registry URL (default: https://registry.pkgr.io)
  --access    Visibility: public or restricted (default: restricted)

Examples:
  pkgr publish --name @acme/utils --version 2.1.0
  pkgr publish --name @acme/utils --version 2.1.0 --access public
  pkgr publish --name @acme/utils --version 2.1.0 --registry https://internal.acme.co
```

Keep top-level help concise so callers discover subcommands first, then drill into the one they need.

### 4. Actionable errors

Fail immediately on bad input and show the correct invocation. Three parts: what went wrong, how to fix it, and where to look for valid values.

```bash
Error: --version is required but was not provided.
  pkgr publish --name @acme/utils --version <semver>
  See available versions: pkgr versions --name @acme/utils
```

Never hang waiting for input that won't arrive. Always exit with a non-zero code.

### 5. Idempotent commands

Running the same command twice should produce the same end state without side effects. If the resource already exists with the requested configuration, return a "no-op" status instead of duplicating it. Automated callers retry frequently due to timeouts and transient failures, so duplicate-safe behavior prevents cascading problems.

### 6. Dry-run support

Add `--dry-run` for any command that creates, modifies, or deletes resources. Print a human-readable preview of what would change, then exit without touching anything.

```bash
$ taskq worker scale --pool ingest --count 8 --dry-run
Would scale pool 'ingest' from 3 to 8 workers:
  - Provision 5 new instances (type: c6g.large)
  - Update load balancer target group
  - Estimated cost delta: +$0.42/hr
No changes applied.
```

### 7. Confirmation bypass

Add `--yes` or `--force` to skip interactive "are you sure?" prompts. Keep the confirmation as the default for human safety, but give automated callers a clean path through.

### 8. Meaningful exit codes

Use distinct exit codes for different failure categories so callers can branch logic without parsing stderr text.

| Code | Meaning |
| ---- | ------- |
| 0 | Success |
| 1 | General/unexpected error |
| 2 | Invalid arguments or usage |
| 3 | Configuration problem |
| 4 | Resource not found |
| 5 | Permission denied |
| 10 | Network or timeout failure |

Document the code table in `--help` or expose it through a dedicated subcommand.

### 9. Piping and composition

Accept input from stdin via a `--stdin` flag or by detecting a pipe. Write structured data to stdout and human-oriented messages to stderr, so callers can chain commands cleanly:

```bash
taskq job describe --id job_92f --output json | jq '.data.config' | taskq job create --stdin
pkgr publish --name @acme/utils --version $(pkgr next-version --name @acme/utils)
```

### 10. Auth via environment

Support tokens, API keys, and credentials through environment variables (e.g., `PKGR_TOKEN`, `TASKQ_API_KEY`). An interactive browser-based login flow is fine as a convenience, but it can't be the only auth mechanism. Config files work as a secondary option. List supported env vars in `--help`.

### 11. Consistent command structure

Pick one pattern and apply it across every resource. Two dominant conventions exist:

- **resource + verb**: `taskq job create`, `taskq job delete`, `taskq worker list`
- **verb + resource**: `taskq create job`, `taskq delete job`, `taskq list worker`

Whichever you choose, keep it uniform. When a caller learns one resource's commands, it should be able to predict every other resource's commands without reading docs.

### 12. Data on success

Return actionable information after every successful operation: identifiers, URLs, timestamps, and durations. These values let callers feed results into the next step of a pipeline.

```bash
published @acme/utils@2.1.0
registry: https://registry.pkgr.io
package_id: pkg_d41f8c
sha256: a9f3e7...
published_at: 2026-03-25T14:32:01Z
```

When `--output json` is active, return this same data inside the structured envelope from checklist item 2.

## Framework Selection

When the user needs help choosing or setting up a CLI framework, recommend based on their language:

| Language | Framework | Why | Reference |
| -------- | --------- | --- | --------- |
| Python | Click or Typer | Click holds 38.7% of Python CLI projects. Typer builds on Click with type-hint ergonomics. | [references/click-typer.md](references/click-typer.md) |
| Node.js/TypeScript | Commander | 35M+ weekly npm downloads, zero dependencies, ~18ms overhead. | [references/commander.md](references/commander.md) |
| Go | Cobra | Powers kubectl, Docker CLI, Hugo, and GitHub CLI across 173K+ projects. | [references/cobra.md](references/cobra.md) |
| Rust | Clap | 302M+ crate downloads, the standard Rust CLI library. | [references/clap.md](references/clap.md) |

Read the relevant reference file for framework-specific implementation patterns that map to the 12-point checklist.

## Env var override pattern

For every flag, support an equivalent environment variable. Use the prefix `APPNAME_` followed by the uppercase flag name. Flags take precedence over env vars, which take precedence over config file values.

```bash
TASKQ_POOL=ingest taskq worker scale --count 8
# --pool defaults to $TASKQ_POOL when the flag is absent
```

Document each flag's corresponding env var in `--help` output.
