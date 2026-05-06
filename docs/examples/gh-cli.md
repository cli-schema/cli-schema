---
navigation_title: GitHub CLI example
---

# GitHub CLI example

This page walks through the [`gh` example schema](https://github.com/cli-schema/cli-schema/blob/main/spec/v1/examples/gh.cli-schema.json), explaining the key patterns it demonstrates.

## Root object

```json
{
  "schemaVersion": 1,
  "name": "gh",
  "version": "2.45.0",
  "description": "GitHub CLI — bring GitHub to your terminal",
  "tags": ["github", "vcs", "devtools"],
  "requiresAuth": true,
  "authCommands": ["auth login", "auth logout", "auth status"],
  "reservedMetaCommands": ["__schema"]
}
```

**`requiresAuth: true`** signals that the program as a whole needs authentication. A consumer setting up an environment can check this and direct the user to authenticate before running any commands.

**`authCommands`** tells consumers exactly which commands handle login and logout — no need to guess or parse help text.

**`reservedMetaCommands: ["__schema"]`** confirms that `gh __schema` is a supported discovery mechanism.

## Global options

```json
{
  "globalOptions": [
    {
      "role": "flag",
      "name": "repo",
      "shortName": "R",
      "type": "string",
      "required": false,
      "summary": "Select another repository using the [HOST/]OWNER/REPO format"
    }
  ]
}
```

Global options appear on every command. Consumers building completion or documentation can apply these to all commands without repeating them.

## Auth namespace

```json
{
  "segment": "auth",
  "commands": [
    {
      "name": "login",
      "intent": {
        "destructive": false,
        "idempotent": true,
        "scope": "global",
        "requiresConfirmation": true,
        "requiresAuth": false
      },
      "parameters": [
        {
          "role": "confirmationSkip",
          "name": "git-credential-helper",
          "type": "string",
          "required": false,
          "summary": "Set credential helper (skips prompt)"
        }
      ]
    }
  ]
}
```

`auth login` has `requiresAuth: false` (it's the command that establishes auth) but `requiresConfirmation: true` — it blocks on stdin by default. The `confirmationSkip` parameter (`--git-credential-helper`) allows non-interactive use.

`idempotent: true` means running `gh auth login` again when already logged in is safe.

## Destructive command: `repo delete`

```json
{
  "name": "delete",
  "summary": "Delete a GitHub repository",
  "tags": ["admin", "dangerous"],
  "intent": {
    "destructive": true,
    "idempotent": false,
    "scope": "global",
    "requiresConfirmation": true,
    "requiresAuth": true
  },
  "parameters": [
    {
      "role": "positional",
      "name": "repository",
      "type": "string",
      "required": false,
      "summary": "Repository to delete in OWNER/REPO format"
    },
    {
      "role": "confirmationSkip",
      "name": "yes",
      "type": "boolean",
      "required": false,
      "summary": "Skip confirmation prompt"
    }
  ]
}
```

This command is the canonical example of a high-risk operation fully described by CLI Schema:

- `tags: ["admin", "dangerous"]` — conventional tags consumers may use to apply extra caution
- `destructive: true` — the repository is permanently deleted
- `idempotent: false` — once deleted, the resource is gone; calling it again will error
- `scope: "global"` — affects a cloud resource, not just local files
- `requiresConfirmation: true` — blocks on stdin without `--yes`
- The `confirmationSkip` role on `--yes` lets consumers pass it automatically — but **only with explicit user authorization** given the `destructive: true` annotation

## Structured output: `repo list`

```json
{
  "name": "list",
  "output": {
    "formats": ["json", "table"],
    "formatFlag": "--json"
  },
  "parameters": [
    {
      "role": "flag",
      "name": "limit",
      "shortName": "L",
      "type": "integer",
      "required": false,
      "defaultValue": "30",
      "validations": [
        { "kind": "range", "min": "1", "max": "1000" }
      ]
    }
  ]
}
```

`output.formats: ["json", "table"]` declares JSON support. `output.formatFlag: "--json"` tells consumers which flag to use. The `--limit` flag has a `range` validation constraint (1–1000) that consumers can surface in UI and completions.

## Long-running streaming command: `run watch`

```json
{
  "name": "watch",
  "streaming": true,
  "longRunning": true,
  "intent": {
    "destructive": false,
    "idempotent": true,
    "scope": "global",
    "requiresConfirmation": false,
    "requiresAuth": true
  }
}
```

`streaming: true` tells consumers the command produces continuous output — don't buffer it. `longRunning: true` tells consumers not to expect a quick exit — set timeouts accordingly or display progress indicators.

## Aliases: `repo list`

```json
{
  "name": "list",
  "aliases": ["ls"]
}
```

Aliases let consumers surface alternative command names in completions and documentation without duplicating the full command definition.
