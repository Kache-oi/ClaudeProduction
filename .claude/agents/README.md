# Project subagents — cost-aware routing

Specialists live here as one Markdown file per agent (YAML frontmatter +
system prompt). Claude Code loads these at startup and uses `description`
to decide when a specialist may be useful; you can also invoke one by name.

## Roster

| Agent | Model | Tools | Role |
|---|---|---|---|
| `docs-auditor` | haiku | Read, Grep, Glob | Documentation accuracy, read-only |
| `explorer` | haiku | Read, Grep, Glob | Codebase search / file discovery, read-only |
| `feature-implementer` | sonnet | Read, Edit, Write, Bash, Grep, Glob | Scoped, approved feature work with tests |
| `verification-reviewer` | sonnet | Read, Grep, Glob, Bash | Do tests pass / does it meet acceptance criteria — read-only |
| `product-risk-reviewer` | sonnet | Read, Grep, Glob | Ship risk: user, privacy, security, migration, rollback — read-only |

Escalate to **Opus** case by case (in the parent task, not by editing these
files) for difficult architecture, ambiguous failure analysis, or high-risk
changes — only when your own evaluation shows it materially improves the
outcome.

## Why this split

Haiku is ~80% cheaper per token than Opus, Sonnet ~60% cheaper — not "90%
off an entire workflow." Read-only research and doc checks are bounded
tasks with cheap failure, so they start on Haiku. Editing code and running
commands gets a step up to Sonnet with explicit tool scope. Nothing here
gets broad write access by default, and reviewers stay read-only so they
check evidence instead of re-implementing the feature.

## Task template

```
Goal: [one measurable outcome]
Context: [relevant files, constraints, expected behaviour]
Use: docs-auditor and explorer first; feature-implementer only after a plan.
Must ask before: [deploying, sending, spending, deleting, changing access].
Finish with: tests run, diff summary, review findings, risks and rollback.
```

Don't run multiple agents to build the same feature in parallel — that
multiplies spend and creates merge work. Parallelize independent research
or review instead (e.g. verification-reviewer + product-risk-reviewer
after implementation).

## Measuring savings

Track cost per *accepted* task, not raw token cost — a cheap answer that
creates rework isn't a saving. See the setup guide for a 4-step evaluation
loop (select representative tasks → baseline → route deliberately →
compare accepted-task cost).

## Sources

- Claude Code — Subagents: https://code.claude.com/docs/en/sub-agents
- Claude Platform — Pricing: https://platform.claude.com/docs/en/about-claude/pricing
  (pricing changes — verify before publishing savings claims)
