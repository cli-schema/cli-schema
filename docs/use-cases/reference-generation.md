---
navigation_title: Reference Generation
---

# Reference Generation

cli-schema is a single source of truth for a CLI's command surface. From it you can generate accurate reference documentation — HTML pages, man pages, Markdown files, or `--help` output — without manual maintenance.

## Why generate from the schema

Hand-written docs drift. A flag gets renamed, a command gets deprecated, a new subcommand ships — and the documentation lags behind. Generating from cli-schema means docs are always in sync with the binary's declared behavior.

## Generating a Markdown reference page

```python
def render_command(path: list[str], command: dict) -> str:
    lines = []
    full_name = " ".join(path + [command["name"]])
    lines.append(f"## `{full_name}`\n")

    if desc := command.get("description"):
        lines.append(f"{desc}\n")

    if command.get("intent", {}).get("destructive"):
        lines.append("> **Warning:** This command is destructive and cannot be undone.\n")

    params = [p for p in command.get("parameters", []) if not p.get("hidden")]
    if params:
        lines.append("### Flags\n")
        lines.append("| Flag | Type | Required | Description |")
        lines.append("|------|------|----------|-------------|")
        for p in params:
            name = f"`--{p['name']}`"
            if short := p.get("shortName"):
                name += f" / `-{short}`"
            typ = p.get("type", "string")
            required = "Yes" if p.get("required") else "No"
            desc = p.get("description", "")
            if p.get("deprecated"):
                desc = f"*(deprecated)* {desc}"
            lines.append(f"| {name} | `{typ}` | {required} | {desc} |")
        lines.append("")

    return "\n".join(lines)


def render_schema(schema: dict) -> str:
    sections = []
    sections.append(f"# {schema['name']} CLI Reference\n")

    def walk(node: dict, path: list[str]):
        for cmd in node.get("commands", []):
            sections.append(render_command(path, cmd))
        for ns in node.get("namespaces", []):
            walk(ns, path + [ns["segment"]])

    walk(schema, [schema["name"]])
    return "\n".join(sections)
```

## Generating a man page

```python
import datetime

def render_man(schema: dict, command: dict, path: list[str]) -> str:
    name = " ".join([schema["name"]] + path + [command["name"]])
    date = datetime.date.today().strftime("%Y-%m-%d")
    lines = [
        f'.TH "{name.upper()}" "1" "{date}" "{schema["name"]} {schema.get("version","")}" ""',
        ".SH NAME",
        f"{name} \\- {command.get('description', '')}",
        ".SH SYNOPSIS",
        f".B {name}",
        "[OPTIONS]",
        ".SH OPTIONS",
    ]
    for p in command.get("parameters", []):
        if p.get("hidden"):
            continue
        flag = f"--{p['name']}"
        if short := p.get("shortName"):
            flag = f"-{short}, {flag}"
        lines.append(f".TP")
        lines.append(f".B {flag}")
        lines.append(p.get("description", ""))
    return "\n".join(lines)
```

## Respecting deprecation

Parameters and commands can declare `deprecated` with a `since` version, a `message`, and an optional `removedIn` version. Always surface this in generated docs:

```python
def deprecation_notice(item: dict) -> str | None:
    dep = item.get("deprecated")
    if not dep:
        return None
    msg = dep.get("message") or f"Deprecated since {dep.get('since', '?')}."
    if removed := dep.get("removedIn"):
        msg += f" Will be removed in {removed}."
    return msg
```

## Output format availability

Some commands can return structured JSON. Documenting this is useful for readers building scripts:

```python
def output_note(command: dict) -> str | None:
    output = command.get("output", {})
    formats = output.get("formats", [])
    if "json" in formats:
        flag = output.get("formatFlag", "--output")
        return f"Pass `{flag} json` to get machine-readable output."
    return None
```

## Further reading

- [Consuming CLI Schema](../guides/consuming.md) — how to load and traverse schema documents
- [GitHub CLI example](../examples/gh-cli.md) — a real-world schema with rich parameter metadata
- [v1 Specification](../spec/v1.md) — normative reference for all fields
