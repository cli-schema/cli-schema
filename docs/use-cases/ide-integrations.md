---
navigation_title: IDE Integrations
---

# IDE Integrations

cli-schema gives editors and devtools a structured model of any CLI — enabling autocompletion, inline documentation, and flag validation without per-tool plugins or custom parsers.

## What an IDE integration can do

With a cli-schema document, an editor can:

- **Autocomplete** subcommands, flags, and positional arguments as the user types in a terminal pane or run configuration.
- **Show inline docs** — command descriptions, flag types, and whether a flag is required — on hover or in a sidebar.
- **Validate arguments** before the user runs the command, catching unknown flags or missing required parameters.
- **Warn on destructive commands** by reading the `intent.destructive` field.

## Loading a schema for a binary

```typescript
import { execFile } from "node:child_process";
import { readFile } from "node:fs/promises";
import { which } from "which";

async function loadSchema(binary: string): Promise<CliSchema | null> {
  // 1. Try sidecar file
  try {
    const binPath = await which(binary);
    const sidecar = binPath + ".cli-schema.json";
    const raw = await readFile(sidecar, "utf8");
    return JSON.parse(raw);
  } catch {}

  // 2. Try meta-command
  return new Promise((resolve) => {
    execFile(binary, ["__schema"], { timeout: 5000 }, (err, stdout) => {
      if (err || !stdout) return resolve(null);
      try { resolve(JSON.parse(stdout)); }
      catch { resolve(null); }
    });
  });
}
```

## Building a completion item list

Walk the command tree to produce a flat list of completion candidates for the current cursor position:

```typescript
interface CompletionItem {
  label: string;
  detail: string;
  documentation?: string;
  deprecated?: boolean;
}

function commandCompletions(command: Command): CompletionItem[] {
  return (command.parameters ?? [])
    .filter(p => !p.hidden)
    .map(p => ({
      label: `--${p.name}`,
      detail: p.type ?? "string",
      documentation: p.description,
      deprecated: p.deprecated != null,
    }));
}

function subcommandCompletions(node: Schema | Namespace): CompletionItem[] {
  const items: CompletionItem[] = [];
  for (const ns of node.namespaces ?? []) {
    items.push({ label: ns.segment, detail: "namespace", documentation: ns.description });
  }
  for (const cmd of node.commands ?? []) {
    items.push({ label: cmd.name, detail: "command", documentation: cmd.description });
  }
  return items;
}
```

## Showing intent warnings

Surface destructive or confirmation-requiring commands before the user runs them:

```typescript
function getIntentWarning(command: Command): string | null {
  const intent = command.intent ?? {};
  if (intent.destructive) {
    return `⚠ This command is destructive and cannot be undone.`;
  }
  if (intent.requiresConfirmation) {
    return `This command will prompt for confirmation before proceeding.`;
  }
  return null;
}
```

## Providing enum value completions

When a flag has an explicit set of allowed values, offer them as completions:

```typescript
function valueCompletions(param: Parameter): string[] {
  if (param.enumValues) return param.enumValues;
  const allowed = (param.validations ?? [])
    .find(v => v.kind === "allowed");
  return allowed?.values ?? [];
}
```

## Further reading

- [Consuming CLI Schema](../guides/consuming.md) — full guide to reading schema documents
- [Key Concepts](../concepts/index.md) — command tree, parameters, intent, and deprecation
- [v1 Specification](../spec/v1.md) — normative reference
