---
navigation_title: Git example
---

# Git example

This page walks through the [`git` example schema](https://github.com/cli-schema/cli-schema/blob/main/spec/v1/examples/git.cli-schema.json), explaining the patterns it demonstrates. Git differs from `gh` in several ways: it has no authentication requirement at the root, uses flat top-level commands instead of namespaces, and several commands have dry-run flags.

## Root object

```json
{
  "schemaVersion": 1,
  "name": "git",
  "version": "2.44.0",
  "description": "Git — the stupid content tracker",
  "tags": ["vcs", "scm"],
  "requiresAuth": false,
  "reservedMetaCommands": ["__schema"]
}
```

`requiresAuth: false` at the root — git itself does not require auth to use. Individual commands like `push` and `pull` do (they carry `requiresAuth: true` in their own intent).

## Global options with repeatable flags

```json
{
  "globalOptions": [
    {
      "role": "flag",
      "name": "C",
      "type": "string",
      "required": false,
      "summary": "Run as if git was started in the given path",
      "repeatable": true
    }
  ]
}
```

`repeatable: true` on `-C` means it can be specified multiple times (`git -C /path1 -C /path2`). Consumers building completions or argument parsers use this to allow multiple values.

## Dry-run flag: `git commit`

```json
{
  "name": "commit",
  "intent": {
    "destructive": false,
    "idempotent": false,
    "scope": "file",
    "requiresConfirmation": false,
    "requiresAuth": false
  },
  "parameters": [
    {
      "role": "flag",
      "name": "message",
      "shortName": "m",
      "type": "string",
      "required": false,
      "repeatable": true
    },
    {
      "role": "dryRun",
      "name": "dry-run",
      "type": "boolean",
      "required": false,
      "summary": "Show what would be committed without actually committing"
    }
  ]
}
```

`scope: "file"` — git commit affects the local repository only, not remote state.

The `--message` flag is `repeatable: true` — `git commit -m "line 1" -m "line 2"` produces a multi-paragraph commit message.

The `dryRun` role on `--dry-run` lets consumers preview what would be committed before committing. An agent can run `git commit --dry-run` to check staged changes safely.

## Auth on specific commands: `git push`

```json
{
  "name": "push",
  "intent": {
    "destructive": false,
    "idempotent": false,
    "scope": "global",
    "requiresConfirmation": false,
    "requiresAuth": true
  },
  "parameters": [
    {
      "role": "dryRun",
      "name": "dry-run",
      "shortName": "n",
      "type": "boolean",
      "required": false,
      "summary": "Show what would be pushed without actually pushing"
    }
  ]
}
```

`requiresAuth: true` on the command's intent — even though the root says `requiresAuth: false`, `push` specifically requires credentials to talk to a remote.

The `dryRun` short name `"n"` means `git push -n` is equivalent to `git push --dry-run`. Consumers can use either form; short names are provided for display purposes.

## Dangerous command: `git reset`

```json
{
  "name": "reset",
  "tags": ["dangerous"],
  "intent": {
    "destructive": true,
    "idempotent": false,
    "scope": "directory",
    "requiresConfirmation": false,
    "requiresAuth": false
  }
}
```

`destructive: true` — `git reset --hard` discards working tree changes permanently.

`requiresConfirmation: false` — git reset does not prompt; it just runs. There is no `confirmationSkip` flag because there is nothing to skip. An agent must treat this command as high-risk and require human approval regardless of the confirmation state.

`scope: "directory"` — affects the local working tree, not remote state.

## Structured output: `git log`

```json
{
  "name": "log",
  "output": {
    "formats": ["json", "text", "oneline", "format"],
    "formatFlag": "--format"
  },
  "parameters": [
    {
      "role": "flag",
      "name": "format",
      "type": "string",
      "required": false,
      "summary": "Output format (oneline, short, medium, full, fuller, email, raw, or a format string)"
    }
  ]
}
```

`git log` supports multiple output formats. A consumer that needs JSON can pass `--format json`. The `formats` array documents what values are available.

## Environment declarations

```json
{
  "environment": {
    "variables": [
      {
        "name": "GIT_AUTHOR_NAME",
        "required": false,
        "description": "Author name for commits"
      },
      {
        "name": "GIT_DIR",
        "required": false,
        "description": "Path to the .git directory; overrides auto-detection"
      }
    ],
    "configFiles": [
      {
        "path": "~/.gitconfig",
        "description": "User-level git configuration"
      },
      {
        "path": ".git/config",
        "description": "Repository-level git configuration"
      }
    ]
  }
}
```

The environment object documents everything git reads from the environment. Consumers that need to run git in a controlled environment (CI, containers, agent sandboxes) can use this to understand what to set up.
