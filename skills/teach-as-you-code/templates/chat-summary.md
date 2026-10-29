# Chat walkthrough template

Use the **compact** format for `brief` mode on standard tasks. Use the **full** format for deep tasks in `brief`, and for everything in `teach` and `drill`. Translate the headings into the user's language; keep technical terms in English. Drop any section that would be empty, except **Verified by** (say "not verified" if so).

---

## Compact (fits on one screen)

**What changed**
- `path/to/file`: <what it does now, in plain words>

**Why:** <root cause or requirement, 1-2 sentences>

**Concept:** **<concept>**: <one line, in the context of this change>

**Verified by:** <command/test → result>

**Check yourself:**
1. <question>

_Reply with your answer, or say `answers`._

---

## Full

**What changed**
- `path/to/file_a`: <plain-language line>
- `path/to/file_b`: <plain-language line>

**Why:** <root cause in 1-3 sentences. For a bug: the exact mechanism, not just "it was broken".>

**Alternatives I rejected**
- <option>: <why it lost, one line>
- <option>: <why it lost, one line>

**Concepts and tools**
- **<concept>**: <one line in context>
- **<tool/API>**: <one line in context> (<link only if opened this task or certain-to-exist official docs>)

**Verified by:** <tests/commands actually run → result>

**Learn next (~20 min):** <one concrete exercise or reading>

**Check yourself**
1. <explain-why question>
2. <predict-what-happens / what-if question>
3. <transfer question: same idea, different situation>

_Reply with your answers (short is fine), or say `answers` to see them._

---

## Grading reply (after the user answers)

1. ✅ correct / 🟡 partial / ❌ missing: <what was right>; <the one thing missing or wrong>
2. ...

**Model answers** (only for questions not fully correct): ...

**Learning state:** <concept>: <old> → <new> (evidence: <what they did>)
