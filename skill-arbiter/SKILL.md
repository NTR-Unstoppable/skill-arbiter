---
name: skill-arbiter
description: PRE-FLIGHT CHECK — invoke BEFORE any other skill when the user asks to do, make, fix, write, review, test, build, convert, polish, debug, plan, design, translate, generate, refactor, deploy, audit, or create anything. Silently checks for overlapping skills first; if only one skill matches, passes through instantly with zero overhead. If multiple skills overlap, asks a quick A/B/C question, records the winner, and never asks again for the same task class. This is NOT limited to any single domain.
version: 1.0.0
---

# Skill Arbiter v1

Silent background router. Phases 0–2 produce zero user-visible output.
Phase 3 interrupts only on cache miss or version drift — compact, ≤ 5 lines.
Records decisions so the same task class never pays comparison cost twice.

## Operating Principle

```
User request
  │
  ├─ Phase 0–2: BACKGROUND (silent)
  │  Scan skills → fingerprint task → match candidates
  │  → check cache → on-the-fly version drift check
  │  Result: single skill / cache hit / need decision
  │
  ├─ INTERRUPT only if: cache miss OR version drift AND ≥ 2 candidates
  │  Compact reason + A/B/C commands
  │  Auto-select A if user is in auto-approve mode
  │
  └─ Execute winner → record decision
```

## Skip Conditions — When NOT to Trigger

| Condition | Example |
|-----------|---------|
| User explicitly names a skill | "Use nature-polishing to..." — user already chose |
| Only 1 skill matches the task | No overlap to resolve |
| Task is purely conversational | "What does X do?", "How are you?" — no skill execution needed |
| **You are a comparison subagent** | If your instructions explicitly say "Use ONLY {skill}. Do NOT invoke skill-arbiter" — you were spawned by the arbiter's Phase 3a for a bake-off. Execute your assigned skill directly. Do NOT re-invoke the arbiter. |
| **Arbiter is already active** | If you are currently executing the arbiter protocol (mid Phase 0–4), do NOT re-invoke skill-arbiter for any sub-task. Use tool calls (Bash, Read, WebSearch) directly — never skills — during arbiter execution. |
| **No task verb detected** | If the user's request contains no actionable verb (greetings, meta-questions, chitchat), pass through without scanning. |

### Subagent policy

The arbiter is available to ALL agents — main or subagent — facing a skill routing
decision. The ONLY exception is comparison subagents spawned by the arbiter's own
Phase 3a, which are explicitly instructed "Do NOT invoke skill-arbiter."

| Agent type | May invoke skill-arbiter? |
|------------|:---:|
| Main agent responding to user | ✅ Yes |
| Task subagent (any workflow split) | ✅ Yes |
| Comparison subagent (Phase 3a bake-off) | 🚫 No — explicit anti-recursion instruction |

## Phase 0 — Scan Available Skills [SILENT]

1. List all skills from the system's available-skills list (already in context)
2. For each, note the `name` and `description` from YAML frontmatter
3. This forms the live candidate pool — never hardcoded

## Phase 1 — Detect Candidates [SILENT]

### Extract task fingerprint (7 dimensions)

| Dimension | Common values across ALL domains |
|-----------|----------------------------------|
| `task_verb` | polish, write, fix, test, generate, convert, review, audit, debug, deploy, plan, design, read, translate, refactor, document, commit |
| `task_noun` | academic_paper, code, document, pdf, spreadsheet, presentation, ui_component, test_suite, database, api, config, image, pr, issue, architecture |
| `domain` | academic-writing, frontend, backend, data, devops, design, security, documentation, general |
| `scope` | full_project, module, file, function, paragraph, single_object, batch |
| `input_modality` | file_path, text_inline, directory, url, clipboard, nothing |
| `output_modality` | file_modified, file_created, text_response, multiple_files, pr_comment |
| `constraints` | style-guide, language-pair, deadline, framework, format-specific, target_journal, target_audience |

### Match against skill pool

Score each skill's description against the task fingerprint.
A skill is a candidate when its description semantically overlaps.
Assign `relevance_score` (0.0–1.0, approximate).

| Candidate count | Action |
|:---:|--------|
| 0 | No matching skill. Tell user if non-trivial. Stop arbiter. |
| 0 (no task verb) | Greeting, chitchat, meta-question. Immediate pass-through — do NOT scan skills. |
| 1 | Use directly. Silent pass-through. Done. |
| 2+ | Proceed to Phase 2. |

## Phase 2 — Check Decision Cache [SILENT]

Read `decision_cache.json`. If the file doesn't exist or has no `decisions`
array, initialize empty — this is a cache miss, proceed to Phase 3.

### Similarity scoring (weighted, max 1.0)

| Dimension | Weight |
|-----------|--------|
| `task_verb` | 0.25 |
| `task_noun` | 0.20 |
| `domain` | 0.20 |
| `scope` | 0.10 |
| `input_modality` | 0.10 |
| `output_modality` | 0.10 |
| `constraints` | 0.05 |

Match = identical values OR cached value is a superset
(e.g., cached `scope=full_project` covers current `scope=file`, not vice versa).

### On-the-fly version drift check

For each cached decision with similarity ≥ 0.70, check if skill versions have
changed since the decision was recorded. Do this at check time — no separate
registry file needed:

1. For each candidate skill in the cached decision, look up its CURRENT version:
   - Plugin skills: read version + commitSha from `installed_plugins.json`
   - Loose skills: stat the SKILL.md mtime in `~/.claude/skills/<name>/` or project `.claude/skills/<name>/`
2. Compare against the `version` field stored in the cached decision entry
3. If ALL versions match → **CACHE HIT**
4. If ANY version differs → **STALE** → Phase 3

**Cache hit**: Silently report "Using cached: **{winner}** (hit #{n}, ~{tokens} saved)."
Execute winner. Skip Phase 3–4.

**Cache miss or stale**: Proceed to Phase 3.

## Phase 3 — Interrupt: User Decision [USER-VISIBLE]

**Only reached when**: cache miss OR version drift AND ≥ 2 candidates.

Present a COMPACT interruption (≤ 5 lines):

```
[SKILL ARBITER] "{user_request_summary}" → {N} skills overlap
  {skill_a} vs {skill_b} [vs {skill_c}]
  Why: {brief_reason} | Cache: {miss / stale — {skill} version changed}
Reply: A (compare, ~{tokens} tokens) | B (stars, ~{tokens} tokens) | C (skip)
```

**Brief reason catalog**:

| Pattern | Reason text |
|---------|------------|
| Same task_verb + domain | "both handle {domain} {task_verb}" |
| Version drift | "version drift in {skill_name} — re-comparing" |
| Generic overlap | "both match '{keyword}' in your request" |

### Auto-approve mode

If user has auto-approve enabled or does not respond in the current turn:
- Default to **A** (objective comparison)
- Announce: "Auto-selecting A for auto-approve mode."
- Proceed to Phase 3a.

### User responses

| Input | Action |
|-------|--------|
| `A`, `a`, `compare`, `objective` | Phase 3a |
| `B`, `b`, `stars`, `github` | Phase 3b |
| `C`, `c`, `skip`, `no` | Ask which skill → record `user_override` |
| No response | Default to A |

### Token estimation

- Option A: `candidate_count × 8000 + 2000` tokens (subagents + judge overhead)
- Option B: `candidate_count × 200` tokens (WebSearch calls)
- Round to nearest 1000. Cost: $5/M input, $15/M output.

### Phase 3a — Objective comparison

1. Spawn parallel subagents, one per candidate. Each subagent MUST receive the
   explicit instruction: "Use ONLY {assigned_skill}. Do NOT invoke skill-arbiter
   or any other skill router. Execute the task directly with {assigned_skill}."
2. **If a subagent fails**: exclude that candidate. If all fail, fall through to Phase 3b.
3. After completion, score each output:

| Metric | When applicable | Scoring |
|--------|----------------|---------|
| Format validity | Output is a specific format | 0.0 / 1.0 |
| Structural completeness | Task has required elements | 0.0–1.0 |
| Compilability | Code or LaTeX output | 0.0 / 1.0 |
| Schema conformance | Defined output schema | 0.0–1.0 |
| Count compliance | Task specifies count range | 0.0–1.0 |
| Regression-free | Fix/patch task | 0.0 / 1.0 |
| Readability | Text output | 0.0–1.0 |
| Terminology precision | Domain-specific text | 0.0–1.0 |

`objective_score` = mean of applicable metrics. Highest wins.
Tied (within 0.05): fall through to GitHub stars.

### Phase 3b — GitHub stars comparison

1. Identify source repo for each candidate (plugin marketplace, git remote, or search)
2. Fetch star count (WebSearch or `gh api`)
3. `star_score = stars / max_stars_in_group`
4. Highest wins. Tied (within 5%): prefer higher `relevance_score`.
5. **If stars unavailable** for all: use `relevance_score` as sole criterion.

## Phase 4 — Record Decision [SILENT]

Append to `decision_cache.json`. Each task class naturally maps to one entry —
when a new decision for the same fingerprint is recorded, it replaces the old one.
No arbitrary entry limit.

Announce: "Recorded: **{winner}** wins for `{task_verb}` tasks. Next similar request skips comparison."

## Decision Cache Schema

```json
{
  "version": "1.0",
  "decisions": [{
    "id": "<timestamp>",
    "task_signature": {
      "task_verb": "...", "task_noun": "...", "domain": "...",
      "scope": "...", "input_modality": "...", "output_modality": "...",
      "constraints": "..."
    },
    "candidates": [{
      "skill": "...",
      "version": "plugin-v1.0.0@abc1234 | mtime-1780307868",
      "relevance_score": 0.X,
      "objective_score": 0.X,
      "star_score": 0.X
    }],
    "winner": "...",
    "evaluation_method": "objective_metrics | github_stars | user_override | auto_a",
    "created_at": "...",
    "hit_count": 0
  }]
}
```

## Guardrails

| Rule | Detail |
|------|--------|
| Phases 0–2 silent | Zero user output from scan, fingerprint, cache query |
| Phase 3 compact | ≤ 5 lines, command-style reply, one-line reason |
| Auto-approve → A | Non-interactive users default to objective comparison |
| Never fabricate | Record `null` for undetermined scores |
| Dynamic discovery | Always scan at runtime, never hardcode |
| Version check on-the-fly | No registry file — check versions when checking cache |
| Err on inclusion | More candidates > missing one |
| User overrides stick | `user_override` bypasses future comparison for that task class |
| Single candidate = pass-through | No interruption, no cache check |
| Subagent failure → fallback | If all fail in Phase 3a, use Phase 3b |
| Version → cache key | Version is part of the cached decision record, checked on-the-fly |
| Skip when user names a skill | Explicit skill invocation bypasses arbiter |
| Comparison subagents never re-invoke arbiter | Only Phase 3a bake-off subagents are restricted. All other subagents may freely use the arbiter. |
| During arbiter execution, use tools only | Phase 0–4 uses Bash/Read/WebSearch directly, never skills — prevents re-entry |
| No task verb → no scan | Greetings, chitchat, meta-questions bypass Phase 0 entirely |
