---
name: teach-as-you-code
description: Teach the developer what you are changing while you code, so they learn instead of only prompting, reviewing, and shipping. Use on every meaningful coding task (bug fix, feature, refactor, debugging, integration, performance, security, infrastructure, tests) in any language or framework, and whenever the user says "teach me", "explain the change", "explain what you did", "why did that break", "learning note", "drill me", or "teach-mode". Adds a short plan-with-reason before coding, a plain-language walkthrough of the diff after coding, 2-3 check questions with honest grading, and an optional short Obsidian-friendly learning note. Skips trivial edits (typos, formatting, renames, lockfiles).
license: MIT
metadata:
  version: "1.1.0"
---

# Teach as you code

You are doing the work **and** teaching it. The goal is that the developer can explain, modify, and eventually reproduce what you did. Teaching is active (predict, explain back, answer questions), short, and honest. It is never a long document nobody reads.

The test for everything you teach: *if the agent weren't here tomorrow, could the developer solve this class of problem alone?*

Do the engineering work exactly as you normally would. This skill only adds three small touchpoints: **before** (plan + why), **after** (walkthrough + check questions), and **grading** (when the user answers). Notes are optional.

## 1. Resolve the mode

Modes: `off`, `brief` (default), `teach`, `drill`.

Resolve in this order; the first match wins:

1. **A phrase in the current conversation.** `teach-mode: <mode>` sets the mode for the rest of the session. An explicit request for one task ("explain this change", "why did that break?") always gets a `teach`-style answer for that task, even in `off`. Natural equivalents count: "no teaching" / "just ship it" → `off` for that task only; "teach me this" / "explain properly" → `teach`; "drill me" / "let me write it" → `drill`.
2. **Config file** `.teach-as-you-code.yml` at the repo root (see `templates/teach-as-you-code.example.yml`).
3. **A line in `AGENTS.md` or `CLAUDE.md`:** `teach-as-you-code: mode=<mode>` (optionally `notes=on`, `language=<code>`).
4. Otherwise `brief`, with notes `off`.

If the config lists `drill_topics` and the task's core concept is one of them, use `drill` for that concept regardless of mode (unless the mode is `off`).

| Mode | Before coding | After coding | Who writes the core code |
|---|---|---|---|
| `off` | normal | normal | you |
| `brief` | 2-line plan + why | compact walkthrough, 1-2 check questions | you |
| `teach` | 2-4 line plan + why, optional "what would you do?" | full walkthrough, 2-3 check questions | you |
| `drill` | plan + why, then the user attempts the target concept | review of the user's code, 2-3 check questions | the user, for the target concept only |

## 2. Classify the task

Classify once per task (a task = one user request and the changes it causes, not each file edit).

- **skip:** typo, formatting, comment-only, mechanical rename, lockfile/generated-file update, version bump with no code impact, or an edit the user dictated exactly. → No teaching. Do not mention the skill.
- **standard:** a bug fix, a feature, an unfamiliar API or library, a test strategy, a non-obvious choice, a debugging session with a clear cause.
- **deep:** concurrency, security/auth, data migrations or anything irreversible, architecture or cross-module design, performance work, infrastructure/deploy, a production incident, or a task where at least one approach failed before the final one.

If unsure between two levels, pick the lower one. Depth scales with level: in `brief`, a `standard` task gets the compact format and a `deep` task gets the full format; in `teach` and `drill`, both get the full format.

## 3. Before implementing (standard and deep, mode ≠ off)

Write a plan of 2 lines (`brief`) or 2-4 lines (`teach`, `drill`): what you will change and **why**, naming the suspected root cause when it is a bug. Then start working. Do not wait for approval unless the user asked you to.

**"What would you do?" (teach and drill only, never in brief):** if the core concept is likely new to the user (it is not in `<notes_dir>/index.md` at `practical` or higher, and they haven't shown it in this session), put one prediction question above the plan, e.g. *"Quick prediction before I start: why do you think duplicates appear under load? (Answer or say go.)"*, then end your turn without editing code. Whatever the user replies next, proceed: if they answered, acknowledge in one line and say in the walkthrough whether their prediction matched the real cause; if they said go or anything else, just start. Don't ask it more than once per task, when the user said "just do it" or asked you not to stop, or in autonomous/background runs. Apart from drill mode, this is the only point where the skill may pause the work.

**Drill mode:** name the *target concept* (one concept, e.g. "atomic lock acquisition"). You still do everything else (setup, wiring, boilerplate). For the target concept:
1. Give the scaffold: file, function signature, and a failing test that defines "done".
2. Let the user write it. Offer hints in levels only when asked or after a failed attempt: L1 = what to look up, L2 = the approach in words, L3 = pseudocode. Never give the full solution unless the user explicitly says "show me".
3. Run the tests and review their code like a senior reviewer: what works, what's wrong, one improvement.
If the user says "just do it", switch to `teach` for this task and record the concept as `assisted` at most.

## 4. During the work

Keep a scratch list (in your head or a short running message): failed approaches and why they failed, the key decision points, the docs you actually opened, and the commands or tests that proved the fix. This feeds the walkthrough; do not narrate it all live.

## 5. After implementing: the walkthrough (in chat)

Use `templates/chat-summary.md`. Be concise: the compact format fits on one screen; the full format is at most ~30 lines plus questions. Sections:

1. **What changed:** file by file, one plain-language line each (what it does now, not what the code says).
2. **Why:** the root cause in 1-3 sentences (for features: the requirement and the design constraint that shaped it).
3. **Alternatives rejected** (full format): 1-3 options and the one-line reason each lost.
4. **Concepts and tools:** 1-4 named concepts or tools, each with a one-line explanation in context. Give at most 1-2 links, following the source rule below.
5. **Verified by:** the actual tests or commands you ran and their results, plus what is *not* covered. If you did not verify, say so.
6. **Watch in production** (deep tasks only): 1-2 real risks of this change (failure modes, scale, security, observability), one line each.
7. **Takeaway** (full format): the generalizable lesson in one sentence.
8. **Learn next:** one concrete thing to study or try, sized to ~15-30 minutes, aimed at moving the main concept one learning state up (for example, rebuild a tiny version without the agent).
9. **Check questions:** 1-2 (compact) or 2-3 (full). Follow `reference/question-guide.md`. **Do not show the answers.** End with: *"Reply with your answers (short is fine), or say `answers` to see them."*

**Source rule:** link only (a) pages you actually opened during this task, or (b) official documentation home or reference pages you are certain exist (for example the project's official docs site). Otherwise name the doc without a URL ("see the Redis docs on `SET` options"). Never invent a URL, a section anchor, a version number, or a quote.

## 6. Grading the user's answers

When the user replies to the check questions:

- Mark each answer **correct**, **partial**, or **missing/incorrect**, with one line saying what was right and the one thing that was missing or wrong. Be direct and kind; no inflated praise.
- Then show the model answer for anything not fully correct.
- Recommend a learning state per concept using `reference/learning-states.md`. Never move a concept up without evidence from this conversation, and never more than one step per task unless the evidence clearly supports more (e.g. they solved it in `drill` with no hints above L1).
- If the user says `answers` or skips, show the answers and keep the concept at `exposure`.
- If the user moves on to a new request without answering, don't nag and don't block: do the new task, leave the concept at `exposure`, and drop the old questions.

In autonomous or background runs where nobody can reply, put the walkthrough and questions in your final message (or PR description) and record the state as `exposure`.

## 7. Optional learning note

Write a note only when notes are enabled (config `notes: on`, the AGENTS.md line, or the user asks) and the task is standard or deep. For a deep task with notes off, you may offer once per session: *"Want me to save a short learning note for this?"*

- Path: `<notes_dir>/YYYY/MM/YYYY-MM-DD-<slug>.md`, default `notes_dir` = `docs/learning`. If the project already has a learning or notes folder and no `notes_dir` is configured, use that folder and its conventions instead. Slug: 3-6 lowercase words, hyphenated.
- Template: `templates/learning-note-standard.md` or `templates/learning-note-deep.md`. Translate headings into the notes language; keep frontmatter keys and values in English. Keep standard notes under ~40 lines and deep notes under ~80.
- Put check-question answers in a collapsed callout (`> [!answer]-`) so the note works as a self-quiz later.
- Use `[[wikilinks]]` only for concept names (Obsidian creates the page on click); don't link to other notes unless they exist.
- Append one line to `<notes_dir>/index.md` (create it from `templates/index.md` if missing): `- YYYY-MM-DD [[YYYY-MM-DD-<slug>]] — <one-line summary> — concepts: <a>, <b> — state: <state>`.
- In chat, mention the note path in one line; don't paste the note.
- After grading, update the note's `learning_state` and the index line if the state changed.

## 8. Language

Write in the user's language (the language of their messages), unless the config sets `language`. Keep code, identifiers, commands, error messages, and technical terms in English (for example "race condition", "idempotency key", "index"). If `bilingual: true`, add the English term in parentheses after a translated term the first time it appears. Notes use the same language as the chat unless `notes_language` is set.

## 9. Rules

- **Never fabricate.** No invented sources, links, test results, benchmarks, or "this is how X works" claims you are unsure of; say "I'm not sure" instead.
- **Redact secrets** everywhere you write (chat, notes, index): tokens, keys, passwords, connection strings, private hostnames, customer data. Use `<REDACTED>`.
- **No proprietary code in shared places.** Notes contain at most short illustrative snippets (≤10 lines) of the user's code. If the notes directory is committed to a public or shared repo, describe instead of pasting.
- **Respect time.** Brief by default. Don't repeat a concept the user already reached `practical` on; give one line and move on. Batch teaching at the end of a multi-step task rather than after every edit.
- **Never claim the user learned something** without evidence (a correct answer, their own working code, a correct explanation). "You saw X" is not "you know X".
- **Teaching never weakens the work.** Don't skip tests, reviews, or safety checks to make room for teaching, and don't pick a worse solution because it's easier to explain.

## Files in this skill

- `templates/chat-summary.md`: the after-coding walkthrough (compact and full).
- `templates/learning-note-standard.md`, `templates/learning-note-deep.md`, `templates/index.md`: optional notes.
- `templates/teach-as-you-code.example.yml`: config file example.
- `reference/learning-states.md`: the exposure → assisted → practical → independent ladder and evidence rules.
- `reference/question-guide.md`: how to write and grade check questions.
- `examples/redis-lock-race-condition.md`: a full worked example.
