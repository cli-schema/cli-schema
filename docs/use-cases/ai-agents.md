---
navigation_title: AI Agents
---

# AI Agents

cli-schema maps directly to the function-calling schemas used by LLMs. Once you have a cli-schema document, you can convert any command into a tool definition that an agent can call — without fine-tuning or training data.

## Why structured schemas matter for agents

LLMs infer CLI behavior from help text and documentation. This breaks on obscure flags, undocumented behavior, and commands that require confirmation or authentication. cli-schema eliminates the guesswork:

- **Command descriptions** are concise and machine-readable, not prose for humans.
- **Intent annotations** (`destructive`, `requiresConfirmation`, `idempotent`) let an agent decide what it can run autonomously.
- **Parameter types and constraints** map directly to JSON Schema types used in tool definitions.
- **Auth requirements** are declared explicitly so the agent knows when to prompt for login.

## Converting a command to a tool definition

```python
def command_to_tool(binary: str, path: list[str], command: dict) -> dict:
    properties = {}
    required = []

    for param in command.get("parameters", []):
        if param.get("hidden"):
            continue
        prop = {"description": param.get("description", param["name"])}
        if param.get("type") == "boolean":
            prop["type"] = "boolean"
        elif param.get("enumValues"):
            prop["type"] = "string"
            prop["enum"] = param["enumValues"]
        else:
            prop["type"] = "string"
        properties[param["name"]] = prop
        if param.get("required"):
            required.append(param["name"])

    return {
        "name": "_".join([binary] + path),
        "description": command.get("description", ""),
        "input_schema": {
            "type": "object",
            "properties": properties,
            "required": required,
        },
    }
```

## Deciding what an agent can run autonomously

Before executing a command, check its intent annotations:

```python
def is_safe_to_run_autonomously(command: dict) -> bool:
    intent = command.get("intent", {})

    if intent.get("destructive"):
        return False  # requires explicit human approval

    if intent.get("requiresConfirmation"):
        has_skip = any(
            p.get("role") == "confirmationSkip"
            for p in command.get("parameters", [])
        )
        if not has_skip:
            return False  # no way to bypass prompt non-interactively

    return True
```

When a command is not safe to run autonomously, surface it to the user with a description of what it will do and ask for confirmation before proceeding.

## Passing confirmation-skip flags automatically

Many CLIs have a `--yes` or `--force` flag. cli-schema marks these with `role: confirmationSkip`. Find it and include it when running non-interactively with user authorization:

```python
def build_noninteractive_args(command: dict) -> list[str]:
    return [
        f"--{p['name']}"
        for p in command.get("parameters", [])
        if p.get("role") == "confirmationSkip"
    ]
```

**Only pass confirmation-skip flags after explicit user approval on destructive commands.** Never bypass prompts silently.

## Requesting JSON output for structured responses

Agents work better with structured data than with prose. Check if the command supports JSON output and request it:

```python
def get_json_flag(command: dict) -> str | None:
    output = command.get("output", {})
    if "json" in output.get("formats", []):
        return output.get("formatFlag")
    return None
```

## Checking authentication before running

```python
def needs_auth(schema: dict, command: dict) -> bool:
    if schema.get("requiresAuth"):
        return True
    return bool(command.get("intent", {}).get("requiresAuth"))

def get_auth_commands(schema: dict) -> list[str]:
    return schema.get("authCommands", [])
```

If authentication is required, check for a valid session before attempting the command. Use `authCommands` to direct the user to the correct login command.

## Further reading

- [Consuming CLI Schema](../guides/consuming.md) — full guide to reading schema documents
- [Key Concepts](../concepts/index.md) — intent, parameters, output, and auth fields explained
- [v1 Specification](../spec/v1.md) — normative reference
