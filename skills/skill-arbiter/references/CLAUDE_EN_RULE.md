# Skill Arbiter — Installation Rule (English)

Add this to your `~/.claude/CLAUDE.md` as the **first section** (before all other rules).

Without it, interception rate ≈ 50%. With it, ≈ 80%.

---

## 0. Skill Interception Rule (highest priority, before all other rules)

**NEVER directly invoke any skill except `skill-arbiter`.**

### Correct flow

```
User request → skill-arbiter (arbitrate) → arbiter chooses and invokes target skill
```

### Forbidden

```
User request → directly invoke nature-polishing / code-review / ...
               or any non-skill-arbiter skill  ← violation
```

### Only exception

**`skill-arbiter` itself.** Invoking skill-arbiter does not require prior arbitration.
