---
navigation_title: Implementing CLI Schema
---

# Implementing CLI Schema

This guide walks through adding a CLI Schema document to your tool. The goal is to go from zero to a valid, useful schema in four steps.

## Step 1: Choose a discovery mechanism

Consumers find your schema through two mechanisms. You should support at least one; both is better.

**Meta-command** — your binary responds to `__schema` by printing a JSON document and exiting 0:

```sh
mytool __schema   # prints schema to stdout
```

Keep this argument out of your help output. Register it in `reservedMetaCommands` so consumers know it's available.

**Sidecar file** — ship a file named `<binary>.cli-schema.json` next to your binary:

```
/usr/local/bin/mytool
/usr/local/bin/mytool.cli-schema.json
```

The sidecar lets consumers read your schema without spawning your process. This matters for sandboxed environments, offline use, and slow-starting runtimes.

## Step 2: Create a minimal schema

Only three fields are required. Start here:

```json
{
  "schemaVersion": 1,
  "name": "mytool",
  "version": "1.0.0"
}
```

Add description, tags, global options, and commands as you go. A partial schema is better than no schema.

## Step 3: Describe your commands

For each command, add a name, summary, and parameters. Here is a complete example for a destructive command:

```json
{
  "schemaVersion": 1,
  "name": "mytool",
  "version": "1.0.0",
  "commands": [
    {
      "name": "deploy",
      "summary": "Deploy the application to production",
      "intent": {
        "destructive": false,
        "idempotent": true,
        "scope": "global",
        "requiresConfirmation": false,
        "requiresAuth": true
      },
      "parameters": [
        {
          "role": "flag",
          "name": "env",
          "type": "enum",
          "enumValues": ["staging", "production"],
          "required": true,
          "summary": "Target environment"
        },
        {
          "role": "dryRun",
          "name": "dry-run",
          "type": "boolean",
          "required": false,
          "summary": "Preview what would be deployed without deploying"
        }
      ]
    },
    {
      "name": "destroy",
      "summary": "Tear down all deployed resources",
      "tags": ["dangerous"],
      "intent": {
        "destructive": true,
        "idempotent": false,
        "scope": "global",
        "requiresConfirmation": true,
        "requiresAuth": true
      },
      "parameters": [
        {
          "role": "confirmationSkip",
          "name": "yes",
          "type": "boolean",
          "required": false,
          "summary": "Skip confirmation prompt"
        }
      ]
    }
  ]
}
```

### Intent annotations

The intent object is the most valuable thing you can add. It tells consumers:

- `destructive: true` — this command deletes or irreversibly changes something
- `idempotent: true` — safe to run multiple times with the same result
- `scope` — how wide the blast radius is: `"file"`, `"directory"`, or `"global"`
- `requiresConfirmation: true` — without a `confirmationSkip` flag, the command blocks on stdin
- `requiresAuth: true` — this command needs an authenticated session

### Special parameter roles

Mark confirmation-skipping flags with `role: "confirmationSkip"`:

```json
{ "role": "confirmationSkip", "name": "yes", "type": "boolean", "required": false }
```

Mark dry-run flags with `role: "dryRun"`:

```json
{ "role": "dryRun", "name": "dry-run", "type": "boolean", "required": false }
```

Consumers use these roles to act safely without hardcoding flag names.

## Step 4: Validate your schema

Use the JSON meta-schema to validate your document:

```sh
npx ajv-cli validate \
  -s https://raw.githubusercontent.com/cli-schema/cli-schema/main/schema/cli-schema.meta-schema.json \
  -d mytool.cli-schema.json
```

Or clone the repository and validate locally:

```sh
npx ajv-cli validate \
  -s schema/cli-schema.meta-schema.json \
  -d mytool.cli-schema.json
```

## Grouping commands into namespaces

If your tool has subcommand namespaces (like `gh repo`, `gh auth`), use the `namespaces` field instead of flat `commands`:

```json
{
  "schemaVersion": 1,
  "name": "mytool",
  "version": "1.0.0",
  "namespaces": [
    {
      "segment": "config",
      "summary": "Manage configuration",
      "commands": [
        { "name": "get", "summary": "Get a configuration value" },
        { "name": "set", "summary": "Set a configuration value" }
      ]
    }
  ]
}
```

Namespaces can be nested arbitrarily to match your command tree.

## Documenting environment dependencies

Use the `environment` field to declare what env vars and config files your tool reads:

```json
{
  "environment": {
    "variables": [
      {
        "name": "MYTOOL_TOKEN",
        "required": false,
        "description": "API token; overrides interactive login"
      }
    ],
    "configFiles": [
      {
        "path": "~/.config/mytool/config.yml",
        "description": "User-level configuration"
      }
    ]
  }
}
```

## Declaring output formats

If your command supports machine-readable output, declare it:

```json
{
  "output": {
    "formats": ["json", "table"],
    "formatFlag": "--output"
  }
}
```

Consumers that need structured output can use `formatFlag` with value `"json"` without hardcoding flag names.
