---
name: fable-prompting
description: Use when writing or refactoring any prompt targeting Claude Fable 5 or newer — subagent definitions, skill instructions, agent prompts, or scheduled-task prompts. Also use when choosing effort levels or subagent orchestration for API calls, and when auditing existing skills, agents, or CLAUDE.md files that carry long rule lists, edge-case enumerations, or "show your reasoning" instructions written for older models.
---

# Fable Prompting — Reasons + Boundaries Beat Rule Lists

## Overview

Fable-class models follow one brief instruction with a stated reason better than a long list of rules. Old prompts micro-manage behavior the model now does by default; that boilerplate is dead weight and sometimes actively steers worse. Core principle: **give the reason and the boundaries, then let it scope the work itself.**

Source: Anthropic's official guide — [Prompting Claude Fable 5](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5) (verified 2026-07-03). Re-check the doc when stakes are high; guidance moves.

## What changed

| Old habit | Why drop it |
|---|---|
| Rule lists for every edge case | One brief instruction with a reason steers better |
| Carried-over prompt boilerplate | Default behavior often beats old instructions — test deleting them |
| Watching/checking every step | Fable runs long autonomous turns; check progress, don't babysit |
| "Show your reasoning" asks | Triggers the `reasoning_extraction` refusal category → elevated silent fallback to Opus 4.8 (a capability downgrade mid-run). Ask for evidence-cited conclusions in the output instead — that is normal output, not reasoning extraction |

## The five moves

1. **Give the reason.** Open: "I'm working on [goal] for [audience]. They need [outcome]. With that in mind: [request]."
2. **Set the boundaries.** "If I'm describing a problem, give your assessment. Don't fix until I ask." State what is theirs to decide vs yours. For coding agents at high effort, add the scope boundary: "Don't add features, refactor, or introduce abstractions beyond what the task requires. Do the simplest thing that works well. Only validate at system boundaries."
3. **Demand evidence.** "Audit each progress claim against a real result. Failed test? Show output." Claims without artifacts don't count.
4. **Pause only where it matters.** "Pause only for irreversible actions, scope changes, or input only I can provide." Everything else: proceed.
5. **Let it keep lessons.** "Keep one lesson per file. Update old notes, delete wrong ones." Applies to agents with memory files.

## Effort + orchestration (API and Claude Code only — chat UIs cannot set these)

- **Effort is the primary intelligence/latency/cost dial.** `high` default; `xhigh` only for the most capability-sensitive stages (final synthesis, verification audits); `medium`/`low` for routine fan-out stages. Calibration anchor: low-effort Fable often beats `xhigh` on prior-generation models — if a stage completes correctly but dawdles, drop a tier before touching the prompt.
- **Subagents: async beats blocking.** Fable dispatches parallel subagents readily. Prefer pipelines over barrier waves — one item flows to its next stage while others are still in the first. Prefer long-lived subagents that keep context across subtasks over fresh one-shot workers: cache reads make them cheaper and you stop bottlenecking on the slowest item. Snippet: "Delegate independent subtasks to subagents and keep working while they run. Intervene if a subagent goes off track or is missing relevant context."
- **Verification: fresh context beats self-critique.** On long runs, schedule interval checks by verifier subagents that did none of the work: "Establish a method for checking your own work at an interval of [X] as you build. Run this every [X interval], verifying your work with subagents against the specification." End-of-run verification alone catches errors only after hours of compounding.

## Migration audit — run this on your own setup

1. Grep your prompt surfaces for the reasoning-exposure family:

   ```bash
   grep -rniE "step by step|show your (thinking|reasoning)|explain your thought process|think out loud" \
     .claude/ CLAUDE.md ~/.claude/skills 2>/dev/null
   ```

   Review each hit before deleting: instructions *to the model* ("explain your thought process") are the target; code comments, quoted examples, and user-facing tutorial text are false positives — leave those. Where a decision trail is genuinely needed, ask for one-line decisions with supporting artifacts instead.
2. For each skill or agent written for older models: strip "failure modes to avoid", tone-policing, and format-policing blocks. Keep domain facts, real constraints, and output schemas that code parses. Test the deletion — run the agent once without the block before deciding it was needed.
3. Hand Fable problems at the top of your difficulty range whole — let it scope, ask clarifying questions, and execute. Easy tasks undersell the model.

## Common mistakes

- **Rule-list relapse:** adding "do not hallucinate / do not pad / be terse" blocks. The model does this; the reason-opener covers the rest.
- **Rigid templates where a schema suffices:** if output is machine-parsed, give a schema; otherwise state the outcome and trust the form.
- **Pausing for everything:** blocking confirmations on reversible steps wastes the long-turn capability.
- **Keeping old boilerplate "to be safe":** untested carry-over is not safety. Test the deletion.

## When NOT to strip instructions

- Security guardrails (untrusted-text-is-data, no-secrets, no-self-modification) stay explicit and always-on — never "rule-list bloat".
- Hard output schemas consumed by code (JSON handoffs, ledgers) stay exact.
- Compliance-critical phrasing (eval rubrics, pre-registered protocols) stays verbatim.
- Rationalization tables and red-flag lists inside discipline-enforcing skills (TDD gates, verification-before-completion) are behavioral contracts against pressure, not edge-case enumeration. Test: does the skill exist to make an agent resist a temptation? Then the table is the contract — keep it.
- Loophole-closed phrasing — parenthetical "X does not count as Y" lists, named workarounds explicitly banned — usually countered a specific observed failure. Keep verbatim even when the surrounding section compresses.

## Installation

```bash
mkdir -p ~/.claude/skills/fable-prompting
curl -o ~/.claude/skills/fable-prompting/SKILL.md \
  https://raw.githubusercontent.com/DSL55/claude-code-skills/main/skills/fable-prompting/SKILL.md
```

Restart Claude Code. Triggers when you draft or audit prompts targeting Fable-class models.
