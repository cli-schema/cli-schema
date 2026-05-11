---
navigation_title: Shell Completions
---

# Shell Completions

cli-schema is a single source of truth for tab completions across every shell. One schema produces bash, zsh, fish, and PowerShell completions — without writing separate completion scripts per shell per tool.

## How it works

Walk the command tree and emit completions for each level: top-level subcommands, then flags for the active command, then enum values for the active flag.

## Walking the full command tree

```python
def all_commands(
    node: dict,
    prefix: list[str] = []
) -> list[tuple[list[str], dict]]:
    results = []
    for cmd in node.get("commands", []):
        results.append((prefix + [cmd["name"]], cmd))
    for ns in node.get("namespaces", []):
        ns_prefix = prefix + [ns["segment"]]
        for cmd in ns.get("commands", []):
            results.append((ns_prefix + [cmd["name"]], cmd))
        results.extend(all_commands(ns, ns_prefix))
    return results
```

Skip parameters with `hidden: true` — they should not appear in completions.

## Generating fish completions

```python
def fish_completions(schema: dict) -> str:
    binary = schema["name"]
    lines = []

    for path, cmd in all_commands(schema):
        # Condition: previous args match the subcommand path
        cond_parts = " ".join(
            f"__fish_seen_subcommand_from {seg}" for seg in path[:-1]
        )
        cond = f"__fish_use_subcommand" if len(path) == 1 else cond_parts
        name = path[-1]
        desc = cmd.get("description", "").replace("'", "\\'")
        lines.append(
            f"complete -c {binary} -n '{cond}' -a '{name}' -d '{desc}'"
        )

        for param in cmd.get("parameters", []):
            if param.get("hidden"):
                continue
            flag = param["name"]
            pdesc = param.get("description", "").replace("'", "\\'")
            short = f" -s {param['short']}" if param.get("short") else ""
            enums = param.get("enumValues", [])
            if enums:
                for val in enums:
                    lines.append(
                        f"complete -c {binary} -l {flag}{short} -a '{val}'"
                    )
            else:
                lines.append(
                    f"complete -c {binary} -l {flag}{short} -d '{pdesc}'"
                )

    return "\n".join(lines)
```

## Generating zsh completions (_binary format)

```python
def zsh_completions(schema: dict) -> str:
    binary = schema["name"]
    lines = [f"#compdef {binary}", "", f"_{binary}() {{", "  local state", "  _arguments \\"]

    for cmd in schema.get("commands", []):
        desc = cmd.get("description", "").replace("'", "\\'")
        lines.append(f"    '{cmd['name']}[{desc}]' \\")

    lines += ["  ;", "}", f"_{binary}"]
    return "\n".join(lines)
```

## Enum value completions

When a parameter has an explicit set of allowed values, offer them:

```python
def param_completions(param: dict) -> list[str]:
    if vals := param.get("enumValues"):
        return vals
    for v in param.get("validations", []):
        if v.get("kind") == "allowed":
            return v.get("values", [])
    return []
```

## Omitting hidden and deprecated flags

```python
def visible_params(command: dict) -> list[dict]:
    return [
        p for p in command.get("parameters", [])
        if not p.get("hidden") and not p.get("deprecated")
    ]
```

Deprecated flags can optionally be included with a deprecation notice rather than omitted entirely, depending on your shell's completion API.

## Further reading

- [Consuming CLI Schema](../guides/consuming.md) — how to load and traverse schema documents
- [Key Concepts](../concepts/index.md) — parameters, enum values, and validations
- [v1 Specification](../spec/v1.md) — normative reference
