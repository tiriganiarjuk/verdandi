# Verdandi — Context Refresh Protocol

## STOP. Read this before modifying any YAML files.

You have context from a compaction summary or from earlier in this session. You feel like you understand the composition hierarchy well enough to start editing. **You are almost certainly wrong.**

After compaction, you have a summary — not understanding. The summary preserves level names and file counts but loses the composition rules, the tier containment constraints, and the specific ways each level's YAML structure differs from the others. You will produce YAML that looks right, passes a casual review, and silently violates the hierarchy because the summary told you WHAT levels exist but not HOW they compose.

**This has happened before.** Previous sessions in this project have produced output that compiled, passed review, and implemented the wrong architecture — work that had to be thrown away.

None of them felt uncertain while doing it.

**The test:** Can you name all 15 levels in load order and state which tier each belongs to? Can you state what a `simplegroup` can contain that a `group` cannot, and vice versa? If not, you don't have enough context. Read the CONTEXT_MAP below before proceeding.

---
<!-- EVERYTHING ABOVE THIS LINE IS STANDARDIZED (except the failure examples) -->
<!-- EVERYTHING BELOW THIS LINE IS PROJECT-SPECIFIC                           -->
---

## How you orient — first time or recovery (same thing)

Orienting and recovering are one act: read the project's main documents, **routed by the
`CONTEXT_MAP` auto-included below** — start with its **orientation gate**, which *is* the
recovery-source list. If already verifiably oriented you needn't re-read everything; the map
always tells you where things live.

---

@./CONTEXT_MAP.md
