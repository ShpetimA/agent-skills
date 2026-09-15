# Agent Skills for TypeScript Design

Portable skills for designing, reviewing, planning, and changing TypeScript systems. They are deliberately agent-neutral: the skills describe engineering decisions and workflows, not a particular assistant, editor, runtime, or deployment platform.

## Install

The [Skills CLI](https://github.com/vercel-labs/skills) installs this repository into the native skill directory for the agents you select.

```sh
# See the available skills first
npx skills add ShpetimA/agent-skills --list

# Let the CLI choose agents interactively
npx skills add ShpetimA/agent-skills

# Install selected skills for specific agents
npx skills add ShpetimA/agent-skills \
  --skill coding-standards \
  --skill code-review \
  --agent codex \
  --agent claude-code \
  --agent opencode
```

Use `-g` to install globally. Omit `--agent` to let the CLI detect and prompt for installed agents. The same repository also works with Hermes Agent through `--agent hermes-agent`; no Hermes-specific fork is needed.

## Skills

| Skill | Purpose |
| --- | --- |
| `coding-standards` | Shared TypeScript design standards, with focused references for domains, modules, parsing, errors, async work, testing, TypeScript, and Effect. |
| `code-review` | A review-only workflow that requires concrete evidence and applies the relevant standards. |
| `tech-spec` | A typed, code-shaped architecture handoff with contracts, seams, call stacks, and a test-first plan. |
| `improve-codebase-architecture` | A planning-only scan for high-leverage, standards-backed refactor opportunities. |
| `postgresql` | PostgreSQL-specific guidance for schemas, queries, transactions, indexes, MVCC, vacuum, and production diagnosis. |
| `grilling` | A one-question-at-a-time design interview. |
| `grill-me` | An explicit entrypoint for a grilling session. |
| `grill-with-docs` | A grilling session that also develops ADR and glossary material. |
| `tdd` | Red-green-refactor guidance focused on observable behavior and real seams. |

## Why these exist

Read [PHILOSOPHY.md](PHILOSOPHY.md) for the reasoning behind the collection: why the standards are centralized, why workflows stay thin, and why the skills prefer explicit contracts and evidence over broad prescriptions.

## Repository layout

```text
skills/
  coding-standards/
  code-review/
  tech-spec/
  improve-codebase-architecture/
  postgresql/
  grilling/
  grill-me/
  grill-with-docs/
  tdd/
```

Each directory is self-contained and starts with a `SKILL.md` file. Supporting Markdown files are relative resources loaded only when the task needs them.

## Attribution

This collection is derived from the local `dmmulroy/skills` checkout. The grilling and TDD skills are derived from [Matt Pocock's skills](https://github.com/mattpocock/skills), with local adaptations. See [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).
