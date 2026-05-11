---
navigation_title: Use Cases
---

# Use Cases

cli-schema is infrastructure. The specification describes a CLI once in a structured, machine-readable format — and anything can consume it.

## Who builds on cli-schema

| Use case | What it enables |
|----------|----------------|
| [AI agents](ai-agents.md) | Give LLMs a structured view of any CLI without fine-tuning or training data |
| [IDE integrations](ide-integrations.md) | Autocompletion, inline docs, and flag validation inside editors and devtools |
| [Reference generation](reference-generation.md) | Always-accurate CLI reference pages, man pages, and `--help` output |
| [Shell completions](shell-completions.md) | Tab completions for bash, zsh, fish, and PowerShell from one schema |

All four use cases read a cli-schema document the same way — via the `__schema` meta-command or a sidecar file. See the [consuming guide](../guides/consuming.md) for the shared foundation.
