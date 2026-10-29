# teach-as-you-code

An [Agent Skill](https://agentskills.io/specification) that makes AI coding agents **teach you what they are changing while they change it**, so you actually learn instead of only prompting, reviewing, and shipping.

Works with Cursor, Claude Code, Codex, and other Agent Skills-compatible agents. One folder, plain Markdown, no installer, no server, no dependencies.

## The problem

You prompt a coding agent. It edits ten files. You review, run the tests, it works, you ship.

Then a week later the same class of bug comes back and you can't say why the first one happened, how it was fixed, which concepts were involved, or what else could have been done. You got faster, but your skills didn't move. Repeat for months and you're stuck in a loop where the agent knows your codebase better than you do.

Writing long post-task documentation doesn't fix this: nobody reads a 16-section report after every change. What works is **active** learning: predicting before you see the answer, explaining back, and answering a couple of sharp questions while the change is still fresh.

## What it does

| When | What the agent does |
|---|---|
| Trivial edit | Nothing. Typos, formatting, renames, and lockfiles are skipped. |
| Before coding | A 2-4 line plan **with the reason** (the suspected root cause for bugs). In `teach`/`drill` mode, an optional "what would you predict?" line that never blocks. |
| After coding | A short plain-language walkthrough in chat: what changed file by file, why, alternatives rejected, the concepts and tools involved (with real docs links only), how it was verified, and what to learn next. |
| Check questions | 2-3 short questions (explain-why, predict/what-if, transfer). Answers are hidden until you reply. |
| Grading | Your answers are marked correct / partial / missing, with model answers and an honest learning-state recommendation. |
| Notes (optional) | A short Obsidian-friendly Markdown note in `docs/learning/YYYY/MM/` plus a one-line entry in `docs/learning/index.md`. |

**Modes:** `off`, `brief` (default), `teach`, `drill`. In `drill`, the agent scaffolds the change and writes a failing test, and **you** write the core concept, with hints in levels if you ask.

**Depth scales with the task:** skip → standard → deep (concurrency, security, migrations, architecture, performance, infra, incidents, or failed attempts).

**Honest learning states:** `exposure → assisted → practical → independent`. A concept only moves up when you show evidence (a correct answer, your own passing code). Reading a summary is exposure, not knowledge.

## Before / after

**Before** (a typical agent reply):

> Fixed the duplicate emails. I added a Redis lock to `send_due_invoices` and updated `mark_sent()`. All tests pass.

**After** (with `teach-as-you-code`, `brief` mode):

> **Plan:** All 3 workers run the invoice job at once and read the same due list (check-then-act race). Adding a Redis lock plus an idempotent "mark sent".
>
> *(work happens)*
>
> **What changed**
> - `jobs/send_due_invoices.py`: takes a Redis lock first; other workers skip.
> - `lib/locks.py`: lock helpers with TTL and owner token.
> - `models/invoice.py`: email is sent only if "mark sent" actually updated the row.
>
> **Why:** the check ("is it due?") and the act ("send + mark") weren't atomic, so 3 workers all passed the check.
>
> **Concept:** **Atomic operation**: `SET NX` checks and sets in one step, so only one worker wins.
>
> **Verified by:** `pytest tests/test_send_due_invoices.py` → 1 email per invoice across 3 concurrent runs.
>
> **Check yourself:** 1. Why wouldn't `GET` then `SET` have fixed this?
>
> _Reply with your answer, or say `answers`._

See the [full worked example](skills/teach-as-you-code/examples/redis-lock-race-condition.md), including `teach` mode, grading, a learning note, and `drill` mode.

## Install

The skill is the folder `skills/teach-as-you-code/`. Put it in your agent's skills directory, either in a project or globally.

### Option A: `npx skills` (any supported agent)

The open [`skills` CLI](https://github.com/vercel-labs/skills) (Node.js 22.20+) detects your installed agents and copies the skill to the right place:

```bash
# into the current project
npx skills add mohamadreza-rezaei/teach-as-you-code

# globally, for all your projects
npx skills add mohamadreza-rezaei/teach-as-you-code -g

# for one agent only, e.g. Claude Code
npx skills add mohamadreza-rezaei/teach-as-you-code -a claude-code
```

### Option B: copy the folder

```bash
git clone --depth 1 https://github.com/mohamadreza-rezaei/teach-as-you-code /tmp/teach-as-you-code
SKILL=/tmp/teach-as-you-code/skills/teach-as-you-code
```

Then copy `$SKILL` into the directory for your agent:

| Agent | Project (commit it, the team gets it) | Global (all your projects) |
|---|---|---|
| **Cursor** | `.cursor/skills/` or `.agents/skills/` | `~/.cursor/skills/` or `~/.agents/skills/` |
| **Claude Code** | `.claude/skills/` | `~/.claude/skills/` |
| **Codex** (CLI, IDE, app) | `.agents/skills/` (scanned from the current directory up to the repo root) | `~/.agents/skills/` |

```bash
# examples
mkdir -p .agents/skills && cp -r "$SKILL" .agents/skills/          # Cursor + Codex, project
mkdir -p ~/.claude/skills && cp -r "$SKILL" ~/.claude/skills/      # Claude Code, global
```

Notes:
- One project copy in `.agents/skills/` is picked up by both Cursor and Codex. Claude Code reads `.claude/skills/`, so symlink it if you use both: `ln -s ../.agents/skills/teach-as-you-code .claude/skills/teach-as-you-code`.
- Cursor also loads `.claude/skills/`, `.codex/skills/`, `~/.claude/skills/`, and `~/.codex/skills/` for compatibility.
- Older Codex guides mention `~/.codex/skills`; current Codex docs list `~/.agents/skills` for user skills.
- Restart the agent if the skill doesn't show up. You can also invoke it explicitly: `/teach-as-you-code` in Cursor and Claude Code, `$teach-as-you-code` in Codex.

Sources: [Cursor: Agent Skills](https://cursor.com/docs/context/skills) · [Claude Code: Skills](https://code.claude.com/docs/en/skills) · [Codex: Skills](https://developers.openai.com/codex/skills) · [skills CLI: supported agents](https://github.com/vercel-labs/skills/blob/main/docs/agents.md)

### Option C: agents without skill support

Paste [`fallback/AGENTS-snippet.md`](fallback/AGENTS-snippet.md) into your `AGENTS.md`, `CLAUDE.md`, or your host's rules file. It holds a compact version of the rules that works on its own, and points to the full `SKILL.md` if you vendored it.

## Configure

Everything is optional. Pick one:

- **In chat:** `teach-mode: drill` (rest of the session). Natural phrases work too: "just ship it" turns teaching off for that task, "teach me this properly" switches to `teach`.
- **Config file:** copy [`templates/teach-as-you-code.example.yml`](skills/teach-as-you-code/templates/teach-as-you-code.example.yml) to your repo root as `.teach-as-you-code.yml`:

  ```yaml
  mode: teach          # off | brief | teach | drill
  notes: on            # write learning notes for standard/deep tasks
  notes_dir: docs/learning
  language: auto       # replies in the language you write in; technical terms stay English
  bilingual: false
  drill_topics: [sql-joins]   # concepts you always want to write yourself
  ```
- **One line in `AGENTS.md` / `CLAUDE.md`:** `teach-as-you-code: mode=teach notes=on`

## Using it with Obsidian

Notes are plain Markdown with small YAML frontmatter, `[[wikilinks]]` for concepts, and self-check answers in a collapsed `> [!answer]-` callout.

1. Turn notes on (`notes: on`).
2. In Obsidian, **Open folder as vault** and pick `docs/learning/` (or your whole `docs/`). Concept links like `[[Race Condition]]` turn into a graph of what you've touched across tasks.
3. Re-quiz yourself by opening old notes: the answers stay folded until you click.
4. Optional: install [kepano/obsidian-skills](https://github.com/kepano/obsidian-skills) as a companion so your agent writes richer Obsidian Markdown, Bases, or Canvas files. It isn't required.

Don't want notes in your repo? Point `notes_dir` at a folder inside your personal vault, or add `docs/learning/` to `.gitignore`. For proprietary code, the skill only describes changes in notes and caps snippets at 10 lines, but check before you commit notes to a shared repo.

## Feeding notes to a coach or mentor agent

The notes are built to be machine-readable as well:

- `docs/learning/index.md` is a one-line-per-task log with concepts and learning states.
- Each note's frontmatter has `concepts`, `tools`, `depth`, and `learning_state`.

Hand these to a personal coach agent, a mentor, or a study routine: "Here's my `index.md` for the last two weeks. Which concepts are stuck at `exposure`? Quiz me on three and build a plan." Because states only move with evidence, the log is an honest picture of what you can do, not just what your agent did.

## What's in the repo

```
skills/teach-as-you-code/
├── SKILL.md                         # the skill (the only required file)
├── templates/
│   ├── chat-summary.md              # after-coding walkthrough: compact + full + grading
│   ├── learning-note-standard.md
│   ├── learning-note-deep.md
│   ├── index.md
│   └── teach-as-you-code.example.yml
├── reference/
│   ├── learning-states.md           # the evidence ladder
│   └── question-guide.md            # writing and grading check questions
└── examples/
    └── redis-lock-race-condition.md # full worked example (teach, brief, drill)
fallback/AGENTS-snippet.md           # for agents without skill support
```

## Design principles

- **Learning over documentation:** questions and predictions, not reports.
- **Teach at three moments:** before coding (plan + why), after coding (walkthrough), and on your answers (grading).
- **Brief by default:** one screen for normal tasks; depth only when the task earns it.
- **Never fabricate:** no invented links, results, or claims that you learned something.
- **Lightweight:** no installer, taxonomy, dashboard, database, or required Obsidian.

## License

[MIT](LICENSE)
