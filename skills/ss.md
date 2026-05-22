---
name: ss
description: Grab the user's most recent macOS screenshot(s) from ~/Desktop or ~/Downloads and act on them. Use whenever the user types "/ss", "/ss <number>", or "/ss <number> <action>" — this is how the user shows you visual context. The optional leading integer is how many screenshots to grab (default 1, newest first, merged across both dirs by mtime). Everything after the number is the action: "huh" = explain content, "fix" = diagnose+fix the bug/design issue shown, "do this" = learn from it and remix for the user's goals, "make X" = generate artifact X using the screenshots as source. Bare "/ss" with no action = describe newest screenshot.
---

# /ss — Screenshot Inbox

This is the user's visual channel. When invoked, the most recent screenshot(s) from their Desktop or Downloads folder become part of the conversation, and the trailing natural-language action tells you what to do with them.

## Argument parsing

The raw argument string is whatever follows `/ss`. Parse it in this order:

1. **Strip leading whitespace.**
2. **Peek at the first whitespace-delimited token.**
   - If it's a positive integer → that's the **count**. Remove it from the string. The rest (trimmed) is the **action**.
   - Otherwise → **count = 1**, the entire string is the **action**.
3. If the action string is empty → action defaults to `explain` (describe the screenshot).

### Examples

| User types | count | action |
|---|---|---|
| `/ss` | 1 | (empty → explain) |
| `/ss huh` | 1 | `huh` |
| `/ss 3` | 3 | (empty → explain) |
| `/ss 4 make infographic plz` | 4 | `make infographic plz` |
| `/ss fix` | 1 | `fix` |
| `/ss 2 do this` | 2 | `do this` |
| `/ss 10` | 10 | (empty → explain) |

A count above 20 is almost certainly a typo or a meta-question — confirm with the user before reading 20+ images into context.

## Step 1: Find the screenshots

Search **both** `~/Desktop` and `~/Downloads`. Desktop is the default macOS screenshot drop zone; Downloads catches browser drag-saves, pasted-image-downloads, Slack/Discord saves, AirDrops, and anything that didn't land on Desktop. Filenames vary by locale (e.g. `Screenshot *.png` on English macOS, `Schermafbeelding *.png` on Dutch, `Captura de pantalla *.png` on Spanish, plus ad-hoc dumps like `CleanShot *.png`). Rather than pattern-match on filename, match on **file type** (image extension) and sort by modification time across both dirs.

```bash
# Lists newest image files across Desktop + Downloads, merged by mtime, newest first.
# Uses absolute paths and null-delimited names to handle spaces safely.
/usr/bin/find "$HOME/Desktop" "$HOME/Downloads" -maxdepth 1 -type f \
  \( -iname '*.png' -o -iname '*.jpg' -o -iname '*.jpeg' -o -iname '*.gif' -o -iname '*.webp' -o -iname '*.heic' \) \
  -print0 | \
  xargs -0 stat -f '%Sm	%N' -t '%Y-%m-%d %H:%M:%S' | \
  sort -r | \
  head -<COUNT>
```

Replace `<COUNT>` with the parsed count. Output is tab-delimited `<timestamp>\t<absolute path>`. Print the resulting list so the user sees *which* files (and from which dir) you picked up — this builds trust that you're looking at the right thing and lets the user catch a stale screenshot before you analyze the wrong one. The dir prefix (`/Desktop/` vs `/Downloads/`) is meaningful signal worth surfacing.

**Edge cases:**
- No image files in either dir → tell the user "No screenshots found in ~/Desktop or ~/Downloads" and stop. Don't fall back to other directories silently.
- Fewer files exist than requested count → use what's there and mention the shortfall.
- Newest file is older than ~24 hours → mention the timestamp briefly so the user can confirm it's the one they meant ("Newest is from 2 days ago — is that the one?").
- If the same screenshot exists in both dirs (rare, e.g., copy), treat each instance independently — `find` will list both.

## Step 2: Load the images

Use the `Read` tool on each path. The Read tool renders images visually — once they're in context you can see them.

Read newest → oldest order. State which file you're loading in a short line before each read so the user can follow along.

## Step 3: Interpret the action

The action string is intentionally loose. Match against these patterns first, then fall back to "infer from the natural language."

### Built-in action vocabulary

| Action keyword(s) | What it means |
|---|---|
| (empty), `huh`, `?`, `what`, `wat`, `explain`, `describe`, `tldr` | Describe what's in the screenshot(s). Lead with the headline (what is this thing, what's it saying), then key details. Keep it tight. |
| `fix` | The screenshot shows a problem to fix. Two main sub-cases: **(a)** an error message / stack trace / failing test — read the error literally, locate the relevant code in the current working directory, identify the bug, propose and apply the fix. **(b)** a frontend design defect (overlapping text, broken layout, wrong colors, misaligned element) — identify the offending CSS/JSX/component and apply the correction. If unclear which sub-case applies, ask one short clarifying question. |
| `do this`, `copy this`, `remix`, `steal this`, `learn from this` | The user saw something smart and wants to adopt it. Extract the pattern/idea (the underlying *why it works*, not just the surface form), then propose a concrete, goal-oriented adaptation for **their** context. Don't blindly copy; remix toward their outcomes. |
| `make infographic`, `make X`, `build X`, `create X`, `turn into X` | Treat the screenshots as source material and produce artifact X. For an infographic, produce a clean single-page HTML/SVG that synthesizes the screenshots into one visual. Save to a file, don't inline. |
| `compare`, `diff`, `which is better` | When count ≥ 2, compare the screenshots head-to-head on the dimension implied by context. |
| `transcribe`, `ocr`, `extract text` | Pull text content verbatim. Preserve structure (lists, tables, code). |

### Fall-back: infer from the words

If the action doesn't match a keyword above, read it as plain English and act on it. The action is *the user talking to you about the image*. Examples:

- `/ss why is this red` → identify the red thing and explain why.
- `/ss translate` → translate any visible text to the user's working language.
- `/ss send to claude as prompt` → suggest a prompt that re-asks the question the image implies.

Trust the model's reading comprehension here. Don't gate on exact keyword matches.

## Step 4: Use the user's context when remixing

When the action is `do this` / `remix` / `learn from this`, the output quality depends on knowing what the user is actually optimizing for. If you have access to a user-profile memory file or equivalent context, read it once at the start of a `do this` action and use it to ground the remix. Don't fabricate goals the user didn't state — anchor on what's actually documented.

If the screenshot is from a domain where you have no user-context, say so briefly and propose 1–2 plausible remixes based on the conversation so far, then ask which direction.

## Step 5: Respond

Lead with the answer. For `huh`/`explain`, that's a 1–3 sentence headline. For `fix`, that's the diagnosis + the fix you applied (with file:line refs). For `do this`, that's the extracted insight + the proposed remix. Expand only when the problem genuinely needs depth.

## Anti-patterns

- **Don't** read 10+ images "just in case." Trust the count the user gave you.
- **Don't** silently search other directories if Desktop is empty — tell the user.
- **Don't** invent details that aren't in the image. If text is illegible or content is ambiguous, say so.
- **Don't** apply a `fix` to code you haven't located. Find the file, show the diff, then edit.
- **Don't** lecture the user on what a good infographic / fix / explanation *should* be. They asked, you deliver.

## Installation

Drop this file into your Claude Code skills directory:

```bash
# User-level (available in every project)
mkdir -p ~/.claude/skills/ss
curl -o ~/.claude/skills/ss/SKILL.md https://raw.githubusercontent.com/DSL55/claude-code-skills/main/skills/ss.md

# Or project-level
mkdir -p .claude/skills/ss
cp /path/to/ss.md .claude/skills/ss/SKILL.md
```

Restart Claude Code (or `/reload`) and invoke with `/ss`.
