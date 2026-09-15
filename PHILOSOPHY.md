# Design Philosophy

These skills exist to make an agent's reasoning more reliable at the points where ordinary coding assistance tends to become vague: boundaries, failures, dependencies, concurrency, tests, and the handoff from a design decision to code.

## One source of engineering judgment

`coding-standards` owns the substantive guidance. It keeps concepts such as parsing, typed expected failures, module seams, observability, and test integrity in one place. It also uses focused references, so a task involving persistence does not need to load guidance intended only for Effect or concurrency.

The other TypeScript workflows are intentionally thin. A review, a technical specification, and an architecture scan should apply the same standards rather than inventing their own versions. This keeps the collection coherent as it evolves.

## Contracts before cleverness

The collection favors small, explicit contracts and deep modules over wide abstractions or architecture for its own sake. Boundary data is parsed before core logic sees it; expected failures are visible to callers; dependencies and side effects have real seams; tests verify what users or callers can observe.

These are defaults, not a demand to rewrite a whole codebase. The skills ask an agent to learn the local conventions first, retain compatibility at genuine boundaries, and make the smallest coherent improvement in the path being changed.

## Workflows are deliberate modes

Reviewing, writing a technical spec, searching for refactor opportunities, and stress-testing a design are different jobs. The workflow skills make those modes explicit and constrain their side effects:

- code review reports evidence; it does not silently edit;
- architecture exploration proposes candidates; it does not start a refactor;
- technical specs turn a chosen direction into interfaces, call stacks, and tests;
- grilling asks one question at a time so uncertainty is resolved with the person making the decision.

This separation makes agent work easier to audit and easier for a human collaborator to steer.

## Portable by default

No skill assumes Hermes, Codex, Claude Code, a specific IDE, a cloud vendor, or a project framework. The format is a directory with `SKILL.md` YAML frontmatter and optional relative resources. Any agent that supports the open skill format can consume the repository directly; the Skills CLI is simply a convenient installer and updater.

Effect is treated as an optional architectural context, not as the identity of the collection. The Effect reference applies only where a codebase already uses its services, layers, schemas, typed failures, or testing facilities. The general standards remain useful in TypeScript projects that use no Effect at all.

## Evolving the collection

Changes should preserve this split: stable, reusable judgment belongs in the standards; concrete operating procedures belong in a workflow skill; specialized detail belongs in a linked reference only when it changes decisions. A new rule should solve a demonstrated recurring problem, not memorialize every edge case from one project.
