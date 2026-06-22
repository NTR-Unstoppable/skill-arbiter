# Skill Arbiter v1.0.0

**Silent skill router for Claude Code.** When multiple skills could handle your request, the arbiter compares them and picks the best — so you don't have to remember 60+ skill names.

## The Problem

Claude Code environments accumulate skills. You might have 50+ installed across academic writing, code review, literature search, and more. When you say "polish this paper," three different skills could apply. Claude picks one arbitrarily — or you manually specify which to use, breaking your flow.

## What Skill Arbiter Does

```
You: "润色这篇论文"
     │
     ├─ Arbiter scans 60+ skills (silent)
     ├─ Detects 3 matching candidates (silent)
     ├─ Checks if it already knows the best one (silent)
     └─ Only interrupts you when it needs a decision:

[SKILL ARBITER] "润色论文" → 3 skills overlap
  nature-polishing vs paper-spine-humanize vs paper-spine-rewrite
  Why: both handle academic-writing polish | Cache: miss
Reply: A (compare) | B (stars) | C (skip)

You: "A"
     → Arbiter runs a quick sample comparison
     → paper-spine-rewrite wins (0.83 vs 0.74)
     → Dispatches winner, records decision
     → Next time "润色论文" skips comparison entirely
```

## Features

- **Zero-overhead pass-through**: If only one skill matches, it's dispatched instantly — no interruption, no cache check
- **Native permission prompt**: Phase 3 uses `AskUserQuestion` — auto-approve compatible, no typing A/B/C, structured click-to-choose UI
- **Sample-based comparison**: Phase 3a tests candidates on a representative sample (~500 words), not the full task, then Phase 5 runs the winner on the complete input
- **Aggregation mode**: For search/review/audit tasks, runs ALL candidates in parallel and merges results — more sources > single winner
- **Decision cache**: Once a task class is resolved, subsequent requests skip comparison entirely
- **Version-aware**: If a skill updates, the arbiter re-compares automatically
- **Anti-recursion**: 6 rules prevent the arbiter from invoking itself
- **Audit log**: Every decision is traceable via `audit_log.jsonl`
- **Overlap matrix**: Start with the template; populate as conflicts are discovered in your environment

## Quick Install

### Option 1: Plugin install (recommended)

```bash
# Via Claude Code marketplace (once registered)
/claude-plugin install skill-arbiter@skill-arbiter

# Or via npx
npx claude-plugin install skill-arbiter
```

### Option 2: Loose skill install

```bash
# Copy the skill directory to your Claude Code skills folder
cp -r skills/skill-arbiter/ ~/.claude/skills/skill-arbiter/
```

### Required: Add CLAUDE.md rule (loose install only)

**Plugin install**: The interception rule is **automatically injected** via the plugin's `CLAUDE.md` — no manual setup needed.

**Loose install**: You must manually add this to your `~/.claude/CLAUDE.md` as the **first section**:

```markdown
## 0. Skill 调用拦截规则（最高优先级）

**禁止直接调用除 `skill-arbiter` 之外的任何 skill。**
唯一豁免: `skill-arbiter` 自身。
```

Without this rule, the arbiter relies on Claude noticing its description — interception drops to ~50%. With the rule, it reaches ~80%. The remaining 20% gap is a Claude Code architecture limitation (no PreSkillUse hook).

## File Structure

```
skill-arbiter/
├── .claude-plugin/
│   ├── plugin.json                 # Plugin manifest
│   └── marketplace.json            # Marketplace registration
├── CLAUDE.md                       # Auto-injected interception rule (plugin install)
├── skills/
│   └── skill-arbiter/
│       ├── SKILL.md                # Main skill definition (553 lines)
│       ├── decision_cache.json     # Past comparison results (auto-populated)
│       └── references/
│           ├── skill_overlap_matrix.md   # Conflict group template (user-populated)
│           ├── CLAUDE_CN_RULE.md         # Chinese CLAUDE.md rule
│           ├── CLAUDE_EN_RULE.md         # English CLAUDE.md rule
│           └── audit_log.jsonl           # Diagnostic trace (append-only)
├── README.md
├── LICENSE
└── .gitignore
```

## How It Works (6 Phases)

| Phase | Visibility | What happens |
|-------|-----------|-------------|
| 0 — Scan | Silent | Enumerate all available skills from context |
| 1 — Match | Silent | Extract 7-dimension task fingerprint; semantic-match + matrix-lookup candidates; detect multi-verb and aggregation-friendly tasks |
| 2 — Cache | Silent | Check decision_cache.json; version drift check; cache hit = skip to Phase 5 |
| 3 — Decide | **Visible** | Only when ≥2 candidates + cache miss. Native `AskUserQuestion` prompt: Compare / Stars / Skip [+ Merge all] |
| 4 — Record | Silent | Write decision to cache so this task class never pays comparison cost twice |
| 5 — Dispatch | Silent | Invoke winner skill with FULL input (not the Phase 3a sample); arbiter protocol ends |

## Comparison Methods

| Option | When offered | Method | Token cost |
|--------|-------------|--------|-----------|
| **Compare** | Always (1st for exclusive, 2nd for aggregation) | Run each candidate on a sample, score 8 quality dimensions | ~2.5k/candidate |
| **Stars** | Always (2nd or 3rd) | Compare GitHub star counts of each skill's repo | ~200/candidate |
| **Skip** | Always (3rd) | You name the skill; records as `user_override` | 0 |
| **Merge all** | Search/review/audit tasks (1st) | Run ALL candidates in parallel, merge results | ~5k/candidate |

## Task Types

### Exclusive (pick one winner)
`write`, `polish`, `translate`, `convert`, `fix`, `generate`, `build`, `create`

### Aggregation-friendly (merge all results)
`search`, `review`, `audit`, `verify`, `debug`, `explore`, `cite`, `find`

## Limitations

1. **~80% interception ceiling**: Without a PreSkillUse hook in Claude Code, the arbiter cannot *force* interception. The CLAUDE.md rule is the strongest available mechanism.
2. **Sample-based comparison is approximate**: Phase 3a judges on ~1/3 of the input. The quality profile is reliable relative to other candidates but not absolute.
3. **First-run cost**: The initial comparison for a new task class costs tokens. Subsequent cache hits cost zero.
4. **Skill discovery is dynamic**: Newly installed skills are picked up automatically, but the overlap matrix must be manually updated for optimal coverage.

## Contributing

This is a community skill. Improvements welcome:

- **Overlap matrix**: Add new conflict groups as skills evolve. Start from the template — the full example matrix is in the repo as reference.
- **Aggregation verbs**: Propose additions to the aggregation-friendly verb list
- **Scoring metrics**: Refine the 8 quality dimensions
- **Edge cases**: Report missed overlaps or incorrect winner selections
- **Interception**: Experiment with stronger CLAUDE.md formulations

## License

MIT — use freely, modify, redistribute.
