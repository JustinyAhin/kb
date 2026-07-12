# Codex Subagents and Model Routing

This runbook describes a practical Codex setup for routing work to specialized
subagents with different models and reasoning levels.

## Recommended layout

Keep the main Codex thread focused on requirements, decisions, and the final
result. Delegate bounded work to specialized agents:

| Agent | Model | Reasoning | Use for |
| --- | --- | --- | --- |
| `<project>_architect` | `gpt-5.6-sol` (Sol) | `high` | Architecture, migrations, ambiguous planning, difficult tradeoffs |
| `scout` | `gpt-5.6-luna` | `low` or `medium` | File and symbol lookup, simple searches, shallow summaries |
| `explorer` | `gpt-5.6-terra` | `low` or `medium` | Codebase mapping, large-file review, documentation research |
| `implementer` | `gpt-5.6-terra` | `medium` | Normal feature work and focused fixes |
| `<project>_reviewer` | `gpt-5.6-sol` (Sol) | `high` | Security, correctness, regression, and final review |

Use `gpt-5.6-luna` for cheap, repetitive, or high-volume scouting when it is
available in the Codex client. Escalate to Terra when exploration requires
tracing real execution paths or synthesizing behavior across files. Luna is
not the default for risky edits.

OpenAI's current Codex guidance says to start with `gpt-5.6-sol` for demanding
agents and use `gpt-5.6-terra` for faster, lighter subagent work. Higher
reasoning levels improve difficult work but increase latency and token usage.

Use the exact model identifiers exposed by the Codex model picker. In
ChatGPT-authenticated Codex sessions, `gpt-5.6-sol` is the supported Sol
identifier; do not assume the API alias `gpt-5.6` works for delegated agents.

## Example custom agents

Project-scoped agents belong in `.codex/agents/` and can be committed to Git.

`.codex/agents/<project>_architect.toml`:

```toml
name = "<project>_architect"
description = "Architecture and planning specialist for ambiguous, high-impact work."
model = "gpt-5.6-sol"
model_reasoning_effort = "high"
sandbox_mode = "read-only"
developer_instructions = """
Focus on system boundaries, data flow, interfaces, tradeoffs, risks, and a
concrete implementation plan. Do not modify application code.
"""
```

`.codex/agents/explorer.toml`:

```toml
name = "explorer"
description = "Fast, read-only codebase explorer."
model = "gpt-5.6-terra"
model_reasoning_effort = "medium"
sandbox_mode = "read-only"
developer_instructions = """
Trace the relevant execution paths, identify files and symbols, and return
concise evidence for the parent agent. Do not modify files.
"""
```

`.codex/agents/scout.toml`:

```toml
name = "scout"
description = "Cheap first-pass agent for locating files, symbols, and obvious usages."
model = "gpt-5.6-luna"
model_reasoning_effort = "low"
sandbox_mode = "read-only"
developer_instructions = """
Perform focused searches and return file paths, symbols, and concise findings.
Do not infer architecture or modify files. Escalate complex code-flow questions
to the explorer agent.
"""
```

`.codex/agents/implementer.toml`:

```toml
name = "implementer"
description = "Implementation agent for bounded changes after the work is understood."
model = "gpt-5.6-terra"
model_reasoning_effort = "medium"
sandbox_mode = "workspace-write"
developer_instructions = """
Make the smallest defensible change, follow repository conventions, run the
most relevant checks, and report the files changed and validation performed.
"""
```

## Delegation rules

Put routing guidance in the repository's `AGENTS.md` or in a project skill:

```md
For architecture, migrations, and ambiguous planning, delegate to `<project>_architect`.
For simple file/symbol lookup and shallow summaries, delegate to `scout`.
For execution-path tracing and deeper API research, delegate to `explorer`.
For bounded implementation after the plan is clear, delegate to `implementer`.
Use `<project>_reviewer` for security-sensitive, cross-cutting, or final validation work.
Keep the main thread responsible for requirements, decisions, and synthesis.
```

Codex can delegate when asked directly or when applicable `AGENTS.md` or skill
instructions request it. If a subagent's `model` and
`model_reasoning_effort` are omitted, Codex may choose a setup balancing
intelligence, speed, and price for that task.

This is context-based delegation, not a silent model change inside the current
main thread. The main thread keeps its selected model; the delegated agent uses
its own configuration.

## Sharing instructions with Claude Code

If `AGENTS.md` is a symlink to `CLAUDE.md`, keep that file limited to rules
that both tools should follow. Put Codex-specific model and delegation behavior
in Codex-native files instead:

```text
AGENTS.md -> CLAUDE.md       # shared project conventions
.codex/agents/*.toml         # project-specific agent models and roles
.codex/skills/               # Codex-specific workflows
~/.codex/AGENTS.md            # personal Codex preferences
```

Use separate real `AGENTS.md` and `CLAUDE.md` files when the top-level prose
itself must differ. A shared source file can be used to generate their common
sections. Do not rely on `AGENTS.override.md` or fallback filenames as a
per-tool overlay: Codex checks the override first and includes at most one
instruction file per directory.

Codex discovers global and project instructions in order, then merges them from
the repository root toward the current directory. Use nested `AGENTS.md` files
for directory-specific rules, and `AGENTS.override.md` when a subtree should
replace its normal instructions.

## Git and security policy

Commit project behavior that the team should share:

- `.codex/agents/*.toml`
- project-safe `.codex/config.toml`
- project-specific `.codex/skills/`
- `AGENTS.md`

Do not commit personal or sensitive state:

- `~/.codex/` credentials, sessions, caches, and personal configuration
- API keys, MCP tokens, or provider credentials
- machine-specific absolute paths
- local logs and temporary files

Review project Codex files like development tooling. They affect how an agent
reads the repository, runs commands, edits files, and delegates work.

## Practical defaults

- Use `gpt-5.6-sol` with `high` reasoning for architecture and final review.
- Use `gpt-5.6-luna` with `low` reasoning for cheap first-pass scouting.
- Use `gpt-5.6-terra` with `medium` reasoning for everyday implementation.
- Use Terra for exploration that requires code-flow understanding or synthesis.
- Lower Terra to `low` for straightforward work where speed matters.
- Prefer a plan before changes that touch several files.
- Keep the main thread clean by returning summaries rather than raw exploration
  output from subagents.

## References

- [Codex subagents and model selection](https://learn.chatgpt.com/docs/agent-configuration/subagents)
- [Codex configuration reference](https://learn.chatgpt.com/docs/config-file/config-reference)
- [OpenAI model guidance](https://developers.openai.com/api/docs/models)
