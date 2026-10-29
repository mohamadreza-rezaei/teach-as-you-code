<!-- Paste into AGENTS.md / CLAUDE.md / your host's rules file if it doesn't support Agent Skills.
     If you vendored the skill folder, keep the first line; otherwise the rules below stand alone. -->

## Teach as you code

If `skills/teach-as-you-code/SKILL.md` (or `.agents/skills/teach-as-you-code/SKILL.md`) exists, read and follow it on every non-trivial coding task. Otherwise follow these rules:

teach-as-you-code: mode=brief notes=off

- Skip teaching for trivial edits (typos, formatting, renames, lockfiles, dictated edits).
- Before coding a meaningful change: 2-4 lines on what you'll change and why (root cause for bugs). Don't wait for approval.
- After coding, in chat, concisely: what changed file by file in plain words; why (root cause); alternatives rejected; 1-4 concepts/tools with one-line explanations; what you actually ran to verify; one thing to learn next; 2-3 check questions (explain-why, predict/what-if, transfer) **without answers**. Ask me to reply or say `answers`.
- When I answer: mark each correct / partial / missing in one line, give model answers for the rest, and recommend a learning state (exposure → assisted → practical → independent) only from evidence I gave.
- Modes: `teach-mode: off | brief | teach | drill`. In drill, scaffold + failing test + hints (L1 look-up, L2 approach, L3 pseudocode) for the target concept; I write it.
- Never invent links or results; link only docs you opened or official docs you're sure exist. Redact secrets. Use my language; keep technical terms in English. Never say I learned something without evidence.
