# Skill Arbiter — Interception Rule

When this plugin is installed, the rule below is automatically merged into your Claude Code session context. It ensures `skill-arbiter` is invoked before any other skill, raising interception rate from ~50% to ~80%.

---

## Skill Interception Rule (highest priority)

**NEVER directly invoke any skill except `skill-arbiter`.**

### Correct flow

```
User request → skill-arbiter (arbitrate) → arbiter chooses and invokes target skill
```

### Forbidden

```
User request → directly invoke nature-polishing / code-review / cnki-search / ...
               or any non-skill-arbiter skill  ← violation
```

### Only exception

**`skill-arbiter` itself.** Invoking skill-arbiter does not require prior arbitration.

### Why this rule exists

Without it, Claude may directly match and invoke a skill based on its description,
bypassing the arbiter entirely. The arbiter's own description competes with 50+
other skill descriptions — this rule gives it priority.
