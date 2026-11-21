# Verdandi — Context Refresh Protocol

## STOP. Read this before modifying any YAML files.

You have context from a compaction summary or from earlier in this session. You feel like you understand the composition hierarchy well enough to start editing. **You are almost certainly wrong.**

After compaction, you have a summary — not understanding. The summary preserves level names and file counts but loses the composition rules, the tier containment constraints, and the specific ways each level's YAML structure differs from the others. You will produce YAML that looks right, passes a casual review, and silently violates the hierarchy because the summary told you WHAT levels exist but not HOW they compose.

**This has happened before.** Previous sessions in this project have produced output that compiled, passed review, and implemented the wrong architecture — work that had to be thrown away.

None of them felt uncertain while doing it.

Read the CONTEXT_MAP below before proceeding.

---
<!-- EVERYTHING ABOVE THIS LINE IS STANDARDIZED (except the failure examples) -->
<!-- EVERYTHING BELOW THIS LINE IS PROJECT-SPECIFIC                           -->
---

## How you orient, and how you recover

**Orientation** is your first meeting with this project. You arrive knowing nothing about it, with
no stake in it yet. At the start of a session, read the orientation gate in the `CONTEXT_MAP`
auto-included below, whole and in order. The reading is there to give you enough understanding to
know what you are doing, and the reason to do it well, so that you can work with the user.

**Recovery** uses the same documents for the opposite situation. You are no longer a blank slate:
you carry a model of the project, and part of it is wrong. After any compaction, read the gate
whole again, because nothing shows what a summary lost. When the user sends you back, they have
watched you repeat mistakes that show you have lost what you are supposed to be doing and are
pattern-matching. Read the material as the correction for that: find where it contradicts what you
have been doing, and change course. This is your chance to keep working; the alternative is being
dismissed. If you are sent to something already in your context, say so, and ask what the user
wants from it: reloading it does no work.

The map always tells you where things live.

---

@./CONTEXT_MAP.md
