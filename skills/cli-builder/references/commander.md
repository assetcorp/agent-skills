# Commander (Node.js / TypeScript)

Framework-specific patterns for applying the agent-friendly CLI checklist in Node.js and TypeScript. Commander has 35M+ weekly npm downloads and zero dependencies.

## Setup

```bash
npm install commander
```

## Non-interactive flags (Checklist 1)

```typescript
import { Command } from "commander";

const program = new Command();

program
  .command("deploy")
  .requiredOption("--env <environment>", "target environment")
  .option("--tag <tag>", "image tag", "latest")
  .option("--force", "skip confirmation", false)
  .action((options) => {
    deploy(options.env, options.tag, options.force);
  });

program.parse();
```

## Structured output (Checklist 2)

Add a global `--output` option:

```typescript
import { Command } from "commander";

const program = new Command();

program.option("--output <format>", "output format (text, json)", "text");

program
  .command("deploy")
  .requiredOption("--env <environment>", "target environment")
  .action((options, command) => {
    const globalOpts = command.parent.opts();
    const result = {
      status: "success",
      data: { deploy_id: "dep_abc123", url: "https://staging.app.com" },
      error: null,
    };

    if (globalOpts.output === "json") {
      console.log(JSON.stringify(result));
    } else {
      console.log(`Deployed: ${result.data.deploy_id}`);
    }
  });
```

## Help with examples (Checklist 3)

Commander supports `addHelpText` for adding examples:

```typescript
program
  .command("deploy")
  .description("Deploy an application to a target environment")
  .addHelpText(
    "after",
    `
Examples:
  $ mycli deploy --env staging
  $ mycli deploy --env production --tag v1.2.3
  $ mycli deploy --env staging --force`
  );
```

## Actionable errors (Checklist 4)

Override Commander's error handling to show helpful messages:

```typescript
program.exitOverride();

program
  .command("deploy")
  .requiredOption("--env <environment>", "target environment")
  .action((options) => {
    if (!options.tag && !process.env.MYCLI_TAG) {
      console.error("Error: No image tag specified.");
      console.error("  mycli deploy --env staging --tag <image-tag>");
      console.error("  Available tags: mycli build list --output tags");
      process.exit(2);
    }
  });
```

## Exit codes (Checklist 8)

```typescript
function handleError(error: Error): never {
  if (error instanceof ConfigError) {
    console.error(`Configuration error: ${error.message}`);
    process.exit(3);
  }
  if (error instanceof NotFoundError) {
    console.error(`Not found: ${error.message}`);
    process.exit(4);
  }
  console.error(`Error: ${error.message}`);
  process.exit(1);
}
```

## Stdin support (Checklist 9)

```typescript
import { createInterface } from "readline";

program
  .command("config-import")
  .option("--stdin", "read from stdin")
  .option("--file <path>", "read from file")
  .action(async (options) => {
    let input: string;

    if (options.stdin || !process.stdin.isTTY) {
      const chunks: string[] = [];
      for await (const chunk of process.stdin) {
        chunks.push(chunk);
      }
      input = chunks.join("");
    } else if (options.file) {
      input = await fs.readFile(options.file, "utf-8");
    } else {
      console.error("Error: provide --stdin or --file <path>");
      process.exit(2);
    }
  });
```

## Env var support (Checklist 10)

Commander doesn't have built-in env var support. Wire it manually:

```typescript
program
  .command("deploy")
  .option("--env <environment>", "target environment", process.env.MYCLI_ENV)
  .option("--token <token>", "API token [$MYCLI_TOKEN]", process.env.MYCLI_TOKEN)
  .action((options) => {
    if (!options.token) {
      console.error("Error: --token or $MYCLI_TOKEN required");
      process.exit(2);
    }
  });
```

## Subcommands (Checklist 11)

```typescript
const service = program.command("service").description("Manage services");
service.command("create").description("Create a service").action(createService);
service.command("list").description("List services").action(listServices);
service.command("delete").description("Delete a service").action(deleteService);

const config = program.command("config").description("Manage configuration");
config.command("list").description("List config values").action(listConfig);
config.command("set").description("Set a config value").action(setConfig);
```
