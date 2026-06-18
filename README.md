# Skill Arbiter

A silent background arbiter that automatically routes tasks to the optimal skill
when multiple installed skills overlap. Works across ALL domains.

## What it does

When you say "polish this paper" and both `nature-polishing` and `ppw-polish` can
handle it, the arbiter:

1. **Silently** scans all installed skills, fingerprints your task, checks a
   versioned decision cache (Phases 0–2)
2. **Interrupts only if needed** — a compact 3-line prompt with A/B/C options
3. **Records the winner** so the same task class never pays comparison cost twice
4. **Auto-selects A** for users with auto-approve mode enabled

## Installation

Place in any skill directory Claude Code scans:

```
.claude/skills/skill-arbiter/
├── SKILL.md
├── README.md
├── decision_cache.json
└── references/
    └── skill_overlap_matrix.md
```

## Supported domains

| Domain | Example overlap scenarios |
|--------|--------------------------|
| Academic writing | polish, de-ai, translate, write, cite, review-response |
| Document processing | convert, create-pdf, create-docx, create-pptx |
| Code & design | design-ui, create-art, build-webapp |
| Testing & quality | verify-change, review-code, debug |
| Planning | plan, brainstorm, execute-plan |
| PR & Git | review-pr, finish-branch, manage-issues |
| Skill management | create-skill, build-mcp, learn-codebase |

Plus any future skill suites — discovery is dynamic at runtime.

## How it works

```
User request
  → Phase 0–2: SILENT (scan, fingerprint, cache check)
  → Phase 3: INTERRUPT only on cache miss (A/B/C, ≤ 5 lines)
  → Phase 4: SILENT (record winner)
```

## Decision cache

`decision_cache.json` stores past comparisons with:
- Task fingerprint (7 dimensions)
- Candidate versions at decision time
- Scores and evaluation method
- Hit counter for cache effectiveness tracking

Cache hits require: similarity ≥ 0.70 AND on-the-fly version drift check passes.

## Requirements

- Claude Code (uses Agent for parallel comparison, Bash for version checks)
- No external dependencies (pure Markdown + JSON)

## Trigger Priority

By default, skill-arbiter competes with other skills for matching. Two methods
ensure it always gets first look:

### Method 1 — Aggressive description (built-in)

The SKILL.md description starts with `PRE-FLIGHT CHECK — invoke BEFORE any other skill`
and lists 16 common action verbs. This covers most task-oriented requests out of
the box. No extra config needed.

### Method 2 — CLAUDE.md routing protocol (guaranteed priority)

For absolute certainty, add this to your project's `CLAUDE.md` or global
`~/.claude/CLAUDE.md`:

```markdown
# CRITICAL — DO NOT SKIP
Before invoking any skill, check if multiple skills could handle the user's
request. If 2+ skills overlap, read skill-arbiter/SKILL.md and follow its
protocol. If only one skill matches, use it directly.
```

Method 1 handles ~90% of cases. Method 2 guarantees 100%.

## Limitations (honest)

- The arbiter relies on Claude's skill-matching to be invoked. If another skill
  has a more specific description match, the arbiter may not fire.
- Use Method 2 above for mission-critical routing.
- The arbiter cannot intercept skill calls mid-execution — it must be invoked
  before the target skill runs.

## Version

1.0.0 — 2026-06-18

