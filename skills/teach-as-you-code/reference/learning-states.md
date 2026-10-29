# Learning-state ladder

Four states per concept. They describe what the developer has **shown**, not what the agent did.

| State | Meaning | Evidence required to reach it |
|---|---|---|
| `exposure` | Saw it used in a change. | The concept appeared in a walkthrough. This is the default after any task, and the ceiling when questions were skipped or no one could answer. |
| `assisted` | Understands this specific change. | Correctly answered the "explain why" question about this change (correct, not partial), or correctly explained the diff in their own words. |
| `practical` | Can work with it, with some help. | Correctly answered a predict/what-if **and** a transfer question, or modified or extended the solution themselves and it passed tests, or completed a `drill` task using hints up to L2. |
| `independent` | Can solve a comparable problem without the agent's core reasoning. | Completed a `drill` task with no hints above L1, or described a comparable problem they solved alone in enough detail to be checkable, across at least two separate tasks. |

## Rules

1. **Never upgrade without evidence from the conversation.** Reading a walkthrough is exposure. Saying "got it" is not evidence.
2. **One step per task by default.** Jump more than one step only when the evidence for the higher state is present in this task (e.g. a clean L1-or-less drill).
3. **Partial answers don't upgrade.** They hold the current state; note what was missing.
4. **States can go down.** If the user later gets a basic question about a concept wrong, recommend dropping one step, and say why.
5. **Recommend, don't decide.** Phrase it as "I'd put this at `assisted` because ...". The user can disagree; record their call if they do, marked `(self-assessed)`.
6. **Where state lives.** In the note's frontmatter `learning_state` and the index line. With notes off, mention the state in chat only; nothing is stored.
7. **Reading prior state.** Before treating a concept as new, check `docs/learning/index.md` (or the configured `notes_dir`). Use the latest line that mentions the concept.
