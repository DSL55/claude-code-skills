# claude-code-skills

A small, curated library of Claude Code skills — practical, battle-tested, drop-in.

Each file is a self-contained skill: copy it into `~/.claude/skills/<name>/SKILL.md` (or your project's `.claude/skills/`), restart Claude Code, and invoke.

## Skills

| Skill | Trigger | What it does |
|---|---|---|
| [ss](skills/ss.md) | `/ss [N] [action]` | Grab the user's newest macOS screenshot(s) and act on them — describe, fix the bug shown, remix the design, generate artifacts. Visual channel into Claude. |
| [fable-prompting](skills/fable-prompting/SKILL.md) | drafting/auditing prompts for Claude Fable 5 | Prompting patterns for Fable-class models from Anthropic's official guide — reasons + boundaries over rule lists, effort-level dial, async subagent orchestration, and a migration audit that finds `reasoning_extraction` triggers (silent Opus 4.8 fallback) in your existing skills and CLAUDE.md. |

## How to install a skill

```bash
mkdir -p ~/.claude/skills/<name>
curl -o ~/.claude/skills/<name>/SKILL.md \
  https://raw.githubusercontent.com/DSL55/claude-code-skills/main/skills/<name>.md
```

Restart Claude Code. The skill name from the file's frontmatter becomes the invocation trigger.

## Not a Claude Code user?

These skills are written as structured prompts. Paste the body of any `.md` into Claude.ai, ChatGPT, or any other capable model — the workflow still applies, you just lose the auto-trigger.

## License

MIT — see [LICENSE](LICENSE).

## Contributing

Issues and PRs welcome. Each new skill should:
- Stay in a single `.md` file
- Have valid YAML frontmatter (`name` + `description`)
- Be sanitised of personal context (no usernames, no private paths, no employer references)
- Include an Installation section
