# Writing and grading check questions

Check questions turn a walkthrough into learning. They should take the user under two minutes to answer in a sentence or two.

## Types (use a mix; full format = one of each)

1. **Explain why:** "Why did two requests both pass the stock check?" Tests the root cause.
2. **Predict / what-if:** "What happens if the lock TTL is shorter than the slowest checkout?" Tests the mental model, not memory.
3. **Transfer:** "Where else in a typical web app could the same check-then-act bug appear?" Tests whether the idea generalizes.

Optional, for `teach`/`drill`: **spot the bug**: show a 3-8 line variant with one flaw and ask what breaks.

## Good questions

- Are about **this** change and its concepts, not trivia ("what's the flag name?").
- Have one clear best answer you can state in 1-3 sentences.
- Can't be answered by copying a line from the walkthrough.
- Use the user's language; keep terms in English.
- No multiple choice by default (it trains recognition, not recall). Use it only if the user asks for quicker checks.

## Bad questions

- "Do you understand the change?" (yes/no, no evidence)
- "What does line 42 do?" (reading, not thinking)
- Questions whose answer is in the previous sentence.
- Trick questions, or questions about things you didn't explain or the user can't see.

## Hiding answers

- In chat: don't print answers. End with the reply prompt. Show answers only after the user answers, or says `answers` / "skip".
- In notes: put answers inside a collapsed Obsidian callout: `> [!answer]-`.

## Grading

- **correct ✅**: core mechanism right; wording doesn't matter.
- **partial 🟡**: right direction, missing the key mechanism or a condition (e.g. "because of concurrency" without saying check and write aren't atomic).
- **missing ❌**: wrong, blank, or "I don't know" (which is fine to say; praise honesty, not guessing).

Give one line per answer: what was right + the one most important gap. Then the model answer for non-correct items. Then the state recommendation (see `learning-states.md`). Keep the whole grading reply under ~12 lines.
