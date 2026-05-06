---
navigation_title: Home
---

# CLI Schema

CLI Schema is a JSON format for describing command-line interface programs in a machine-readable way. One document captures a program's commands, parameters, intent, authentication requirements, environment dependencies, and output capabilities — enough for tooling to understand a CLI without running it.

## Who it's for

**CLI authors** — ship a single schema document alongside your binary. Every consumer that understands CLI Schema immediately understands your tool.

**AI agent builders** — read the schema before executing commands. Know upfront which commands are destructive, which require confirmation, and how to request machine-readable output.

**IDE and tooling authors** — generate accurate autocomplete, documentation, and shell completions from one structured source rather than parsing help text.

## A quick example

Here is the `gh repo delete` command as a CLI Schema document:

```json
{
  "schemaVersion": 1,
  "name": "gh",
  "version": "2.45.0",
  "namespaces": [
    {
      "segment": "repo",
      "commands": [
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
      ]
    }
  ]
}
```

From this, a consumer immediately knows: the command is destructive and globally scoped, it requires authentication, it will block on an interactive prompt unless `--yes` is passed, and the tool author tagged it as `dangerous`. No guessing. No help-text parsing.

## Next steps

- [Why CLI Schema?](why.md) — the problem this solves and why now
- [Key concepts](concepts/index.md) — commands, parameters, intents, and discovery
- [Implementing CLI Schema](guides/implementing.md) — add a schema to your CLI tool
- [Full specification](spec/v1.md) — the normative v1 reference
