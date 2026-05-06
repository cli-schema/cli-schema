---
navigation_title: Consuming CLI Schema
---

# Consuming CLI Schema

This guide covers how to read and use CLI Schema documents as a consumer — whether you're building an AI agent, an IDE extension, a shell completion generator, or any other tool that works with CLIs.

## Discovering a schema

Check for sidecar files first (no process spawn needed), then fall back to the meta-command:

```python
import subprocess, json, os, shutil

def get_schema(binary: str) -> dict | None:
    # 1. Try sidecar file
    binary_path = shutil.which(binary)
    if binary_path:
        sidecar = binary_path + ".cli-schema.json"
        if os.path.exists(sidecar):
            with open(sidecar) as f:
                return json.load(f)

    # 2. Try meta-command
    try:
        result = subprocess.run(
            [binary, "__schema"],
            capture_output=True, text=True, timeout=5
        )
        if result.returncode == 0:
            return json.loads(result.stdout)
    except (subprocess.TimeoutExpired, FileNotFoundError, json.JSONDecodeError):
        pass

    return None
```

Check `reservedMetaCommands` on the schema to confirm `__schema` is supported before calling it.

## Reading the command tree

Commands live at the root in `commands` (flat list) or under `namespaces` (grouped). Namespaces can be nested. To find a specific command like `gh repo delete`:

```python
def find_command(schema: dict, path: list[str]) -> dict | None:
    # path = ["repo", "delete"] for "gh repo delete"
    node = schema
    for segment in path[:-1]:
        namespaces = node.get("namespaces", [])
        node = next((n for n in namespaces if n["segment"] == segment), None)
        if node is None:
            return None
    name = path[-1]
    commands = node.get("commands", [])
    return next((c for c in commands if c["name"] == name), None)
```

## Making risk decisions with intent

Before executing a command, check its intent:

```python
def is_safe_to_run_autonomously(command: dict) -> bool:
    intent = command.get("intent", {})

    if intent.get("destructive"):
        return False  # requires human approval

    if intent.get("requiresConfirmation"):
        # safe only if there's a confirmationSkip flag we can pass
        has_skip = any(
            p["role"] == "confirmationSkip"
            for p in command.get("parameters", [])
        )
        if not has_skip:
            return False

    return True
```

When a command is destructive or requires confirmation, surface it to a human or request explicit approval before running it.

## Passing confirmation-skip flags

When running a command non-interactively, find its `confirmationSkip` parameter and include it:

```python
def build_noninteractive_args(command: dict) -> list[str]:
    args = []
    for param in command.get("parameters", []):
        if param["role"] == "confirmationSkip":
            args.append(f"--{param['name']}")
    return args
```

**Only do this with explicit user authorization on destructive commands.** A command marked `destructive: true` with `requiresConfirmation: true` should never be run non-interactively without user approval.

## Using dry-run mode

Before running a write command, check if it supports dry-run:

```python
def get_dry_run_flag(command: dict) -> str | None:
    for param in command.get("parameters", []):
        if param["role"] == "dryRun":
            return f"--{param['name']}"
    return None

# Usage
dry_run_flag = get_dry_run_flag(command)
if dry_run_flag:
    result = subprocess.run([binary, *cmd_args, dry_run_flag], ...)
    # show result to user, then ask for confirmation
```

## Requesting structured output

Check the command's `output` field to know if JSON is available and how to request it:

```python
def get_json_flag(command: dict) -> str | None:
    output = command.get("output", {})
    if "json" in output.get("formats", []):
        return output.get("formatFlag")
    return None

# Usage
json_flag = get_json_flag(command)
if json_flag:
    args = [binary, *cmd_args, json_flag, "json"]
```

Note: some CLIs use `--json fields` (like `gh`), others use `--output json`. Check the `formatFlag` value — it's the flag name, not the value.

## Checking authentication requirements

```python
def needs_auth(schema: dict, command: dict) -> bool:
    if schema.get("requiresAuth"):
        return True
    intent = command.get("intent", {})
    return bool(intent.get("requiresAuth"))

def get_auth_commands(schema: dict) -> list[str]:
    return schema.get("authCommands", [])
```

If the program or command requires auth, check for a valid session before running. Use `authCommands` to direct users to the right login command.

## Shell completion generators

To generate completions, walk the command tree and emit completions for each command's parameters:

```python
def all_commands(schema: dict, prefix: list[str] = []) -> list[tuple[list[str], dict]]:
    results = []
    for cmd in schema.get("commands", []):
        results.append((prefix + [cmd["name"]], cmd))
    for ns in schema.get("namespaces", []):
        ns_prefix = prefix + [ns["segment"]]
        for cmd in ns.get("commands", []):
            results.append((ns_prefix + [cmd["name"]], cmd))
        results.extend(all_commands(ns, ns_prefix))
    return results
```

For each command, flags with `hidden: true` should be omitted from completions. Parameters with `enumValues` can offer value completions. Parameters with `validations` of kind `"allowed"` can similarly offer value completions.

## Validating schema documents

Before acting on a schema, validate it against the meta-schema:

```sh
npx ajv-cli validate \
  -s schema/cli-schema.meta-schema.json \
  -d mytool.cli-schema.json
```

Treat schema documents as untrusted input — don't assume `destructive: false` means a command is safe. Apply your own risk reasoning on top of the schema's declarations.
