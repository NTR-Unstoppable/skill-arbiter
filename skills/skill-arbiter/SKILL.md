---
name: skill-arbiter
description: PRE-FLIGHT CHECK — invoke BEFORE any other skill when the user asks to do, make, fix, write, review, test, build, convert, polish, debug, plan, design, translate, generate, refactor, deploy, audit, or create anything. Silently checks for overlapping skills first; if only one skill matches, passes through instantly with zero overhead. If multiple skills overlap, asks a quick multiple-choice question via AskUserQuestion, records the winner, and never asks again for the same task class. This is NOT limited to any single domain.
version: 1.0.0
---

# Skill Arbiter v1

Silent background router. Phases 0–2, 4–5 produce zero user-visible output.
Phase 3 interrupts only on cache miss or version drift, using `AskUserQuestion`
for a native permission prompt.
Records decisions so the same task class never pays comparison cost twice.

## Operating Principle

```
User request
  │
  ├─ Phase 0–2: BACKGROUND (silent)
  │  Scan skills → fingerprint task → match candidates
  │  (semantic + overlap matrix) → check cache → version drift check
  │  Result: single skill / cache hit / need decision
  │
  ├─ INTERRUPT only if: cache miss OR version drift AND ≥ 2 candidates
  │  AskUserQuestion: Compare / Stars / Skip [+ Merge if aggregation-friendly]
  │  Auto-approve auto-selects 1st option (recommended default)
  │
  ├─ Phase 3a/3b/3d: Compare candidates → determine winner
  │
  ├─ Phase 4: Record decision to cache
  │
  └─ Phase 5: DISPATCH WINNER (silent)
     Invoke winner via Skill tool → winner executes
     Arbiter protocol ends. Do NOT re-invoke arbiter for the winner's main task.
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
| **Arbiter itself is the only match** | If the task is a meta-routing question ("which skill should I use?"), answer directly — do NOT invoke skill-arbiter recursively. |
| **Winner was chosen by arbiter** | After Phase 5 dispatches the winner, the winner's main task is explicitly chosen — do NOT re-invoke arbiter for it. Subtasks spawned BY the winner MAY use arbiter per subagent policy. |

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

**Primary path — semantic matching**: Score each skill's description against the task fingerprint.
A skill is a candidate when its description semantically overlaps.
Assign `relevance_score` (0.0–1.0, approximate).

**Secondary path — overlap matrix hard lookup**: Read `references/skill_overlap_matrix.md`.
For each `(task_verb, domain)` pair in the fingerprint, check if the matrix lists known
conflict groups. Any skill named in a matching row is a **confirmed candidate**, regardless
of its semantic score. This catches synonyms: when two skills use different words in their
descriptions for the same capability, the matrix ensures neither is missed.

**Merge**: Union of semantic matches + matrix matches. Deduplicate by skill name.
Matrix-listed candidates that were NOT found by semantic matching get `relevance_score`
from the nearest semantic match in their conflict group (or 0.50 if none).

### Multi-Verb Request Detection

If the user's request contains **2+ clearly distinct task_verbs** (e.g., "写论文并做图表"
→ `write` + `figure`), identify the **primary verb** (first/main action) and run
arbitration for it. Flag secondary verbs as **companion tasks** — after Phase 5 dispatch,
inform the user which companion tasks may benefit from separate skills. Example:

> "Dispatched nature-writing for '写论文'. Companion task '做图表' may need nature-figure.
> Reply 'figure' to invoke, or continue."

Do NOT attempt to auto-chain — the winner skill owns its workflow and may spawn subagents
that invoke arbiter for sub-tasks (R3).

### Aggregation-Friendly Task Detection

Some task_verbs produce **additive, non-exclusive results**. Running multiple skills and
merging their output is strictly better than picking one winner — each skill covers a
different source, angle, or dimension, and their results accumulate without conflict.

**Aggregation-friendly verbs** (results are additive, not mutually exclusive):

| Category | task_verbs | Why merge > pick |
|----------|-----------|------------------|
| Search | `search`, `research`, `find`, `检索`, `查`, `找` | Different databases cover different sources. Union > any single database. |
| Review/Audit | `review`, `audit`, `检查`, `审查`, `排查` | Different reviewers catch different issues. Each adds findings the others miss. |
| Verify/Validate | `verify`, `validate`, `test`, `校验`, `验证` | Multiple checks increase confidence. Complementary coverage. |
| Debug | `debug`, `diagnose`, `排查` | Different diagnostic angles find different root causes. |
| Cite | `cite`, `引用`, `加引用` | More sources = better coverage for different claims. |
| Explore | `explore`, `scan`, `inspect`, `探索` | Different tools find different aspects of the codebase. |

**Exclusive verbs** (must pick one, results conflict):

| Category | task_verbs | Why single winner |
|----------|-----------|-------------------|
| Write/Draft | `write`, `draft`, `generate`, `build`, `create`, `写`, `生成` | One final document. Multiple versions conflict. |
| Polish | `polish`, `refine`, `润色`, `修改` | One polished version. Different polishers produce incompatible edits. |
| Translate | `translate`, `翻译` | One translation. Different translators produce incompatible text. |
| Convert | `convert`, `转换` | One converted file. Different converters produce different outputs. |
| Fix/Patch | `fix`, `repair`, `patch`, `修复` | One fix. Multiple fixes may conflict. |

**Detection**: During Phase 1 fingerprint extraction, if `task_verb` is in the
aggregation-friendly set, set `aggregation_friendly = true`. This flag propagates
to Phase 3 where it enables Option D.

**Gray area**: When a task has both exclusive and aggregation-friendly aspects
(e.g., "查文献并写综述" → `search` + `write`), apply multi-verb detection first:
arbitrate `write` as primary (exclusive), and `search` is a companion that may
benefit from aggregation separately.

| Candidate count | Action |
|:---:|--------|
| 0 | No matching skill. Tell user if non-trivial. Stop arbiter. |
| 0 (no task verb) | Greeting, chitchat, meta-question. Immediate pass-through — do NOT scan skills. |
| 1 | Dispatch via Phase 5. Silent pass-through. Done. |
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
   - **Plugin skills** (installed via marketplace): read `version` + `commitSha` from `installed_plugins.json`
   - **Loose skills** (manual install in `~/.claude/skills/<name>/` or project `.claude/skills/<name>/`): stat the SKILL.md mtime (Unix timestamp)
   - **Built-in skills** (shipped with Claude Code): use the version string from the skill's YAML frontmatter
   - **Priority**: Plugin > Loose > Built-in. A skill may exist in multiple forms — use the highest-priority form's version.
2. Compare against the `version` field stored in the cached decision entry
3. If ALL versions match → **CACHE HIT**
4. If ANY version differs → **STALE** → Phase 3

**Cache hit**: Silently report "Using cached: **{winner}** (hit #{n}, ~{tokens} saved)."
Execute winner via Phase 5. Skip Phase 3–4.

**Cache miss or stale**: Proceed to Phase 3.

## Phase 3 — Interrupt: User Decision [USER-VISIBLE]

**Only reached when**: cache miss OR version drift AND ≥ 2 candidates.

Use the `AskUserQuestion` tool to present a native permission-style prompt. This
integrates with Claude Code's permission system: auto-approve users get the default
automatically, and all users get a structured UI instead of typing A/B/C.

### Option template (language follows user's locale)

All four options share the same description structure. Labels and descriptions are
written in the same language the user is using — no hardcoded locale.

```
Option structure:
  label:       "<short action name>"
  description: "<what it does>. ~<tokens> tokens. <when to use>"
```

| Option | label | description |
|--------|-------|-------------|
| A — Compare | Short action name | Runs each skill on a sample, scores 8 quality dimensions, picks winner. ~{A}tokens. Best for exclusive tasks (write/polish/translate/fix) where only one output is needed. |
| B — Stars | Short action name | Compares GitHub star counts of each skill's source repo. ~{B} tokens. Best when skills have clear community preference differences. |
| C — Skip | Short action name | User names the skill directly. Records preference for next time. ~0 tokens. Best when the user already knows which skill they want. |
| D — Merge all | Short action name | Runs ALL skills in parallel, deduplicates and merges results. ~{D} tokens. Best for aggregation-friendly tasks (search/review/audit/explore) where more sources beat a single winner. |

The model writes the actual label and description text at runtime, matching the
conversation locale. The structure is fixed; the language is dynamic.

### Option visibility and ordering

| Position | Option | Visible |
|----------|--------|---------|
| 1st | Recommended default | Always |
| 2nd | Alternative comparison | Always |
| 3rd | Skip / manual select | Always |
| 4th | Merge all | Only when `aggregation_friendly = true` |

The 1st option is the recommended default, auto-selected for auto-approve:

- `aggregation_friendly = true`: **Merge all** is 1st, Compare is 2nd
- `aggregation_friendly = false`: **Compare** is 1st, Stars is 2nd

### Auto-approve compatibility

When the user has "auto-approve all operations" enabled, `AskUserQuestion` auto-selects
the 1st option without prompting. The 1st option is always the recommended default for
that task type — no special handling needed.

### Token estimation for option descriptions

```
A_tokens = candidate_count × 2500 + 2000
B_tokens = candidate_count × 200
D_tokens = candidate_count × 5000 + 3000
```

Round to nearest 1000. Cost: $5/M input, $15/M output.

### Response mapping

`AskUserQuestion` returns the selected option. Map by position (not label text —
labels are locale-dependent):

| Selected option | Action |
|-----------------|--------|
| 1st option | Phase 3a (objective comparison) — or Phase 3d if aggregation_friendly and Merge is first |
| 2nd option | Phase 3b (GitHub stars comparison) — or Phase 3a if aggregation_friendly and Compare is second |
| 3rd option | Phase 3c — ask which skill → record `user_override` |
| 4th option (if present) | Phase 3d (parallel aggregation) |
| (auto-approve / no response) | Default = 1st option |

### Phase 3c — Manual Override

Reached when the user selects Skip (3rd option). The arbiter does not compare —
the user takes control.

1. Ask which skill the user wants to use: "Which skill should handle this?"
2. User names a skill → record as `user_override` in Phase 4
3. If the named skill doesn't exist or isn't in the candidate list: warn but proceed
4. Dispatch the user-chosen skill via Phase 5
5. `user_override` entries bypass future comparison for that task class (Guardrails: "User overrides stick")

### Phase 3a — Objective comparison

**Key principle**: Phase 3a is for COMPARISON, not execution. Subagents process a
representative sample — just enough to judge output quality. Phase 5 does the full job.

1. **Determine sample scope**: From the user's input, extract a representative subset:
   - Text/document tasks: first ~500 words or first 3 paragraphs
   - Code tasks: one representative file or function (~50 lines)
   - Search tasks: run the query but return only first 5 results
   - Multi-file tasks: process the first 2 files only
   The sample must be large enough to reveal style/quality differences but small
   enough to avoid redundant full execution.

2. Spawn parallel subagents, one per candidate. Each subagent MUST receive:
   - The explicit instruction: "Use ONLY {assigned_skill}. Do NOT invoke skill-arbiter
     or any other skill router. Execute the task directly with {assigned_skill}."
   - The **sample input** (NOT the full input). The subagent is told this is a
     comparison sample for skill evaluation.
   - The original full task description, so the subagent understands the complete goal.

3. **If a subagent fails**: exclude that candidate. If all fail, fall through to Phase 3b.
   If the user cancels during subagent execution: abort remaining subagents, do NOT
   write to cache, report "Comparison cancelled. No decision recorded."

4. After completion, score each output against the sample:

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

5. **Scoring caveat**: Scores are based on samples — they are reliable indicators
   of relative quality but not definitive. The winner's full output in Phase 5
   may differ in detail but should match in quality profile.

### Phase 3b — GitHub stars comparison

1. Identify source repo for each candidate (plugin marketplace, git remote, or search)
2. Fetch star count (WebSearch or `gh api`)
3. `star_score = stars / max_stars_in_group`
4. Highest wins. Tied (within 5%): prefer higher `relevance_score`.
5. **If stars unavailable** for all: use `relevance_score` as sole criterion.

### Phase 3d — Parallel Aggregation [USER-VISIBLE]

Only available when `aggregation_friendly = true`. No competition — all candidates
contribute, and their outputs are merged into a unified result.

1. Run ALL candidates in parallel via Skill tool invocations, collecting outputs.
   Use Agent tool with parallel calls when possible; fall back to sequential Skill calls.
2. **If a candidate fails**: skip it silently. Aggregation tolerates partial failure.
3. After all complete, synthesize:
   - **Deduplicate**: merge overlapping results (same paper, same bug, same finding).
     Flag each item with its source(s) so the user knows which skill found what.
   - **Organize**: group by theme/source/relevance as appropriate for the task type.
   - **Highlight gaps**: note if one source found something the others missed entirely.
4. Present the unified result with source attribution.
5. Record in cache: `evaluation_method = "aggregation"`, `winner = "aggregated"`
   (sentinel value indicating all candidates contributed equally),
   `candidates` lists all contributing skills.

**Cache behavior for aggregation**: Next time the same fingerprint hits the cache
with `evaluation_method = "aggregation"`, auto-aggregate — no Phase 3 interruption.

**Token cost note**: Option D runs ALL candidates but skips the scoring/judge phase,
making it generally cheaper than Option A for the same candidate count while producing
richer output. This is the recommended default for aggregation-friendly tasks.

### Ultimate Fallback Chain

When all comparison methods are exhausted:

```
Phase 3a (objective) → all subagents fail
  └─→ Phase 3b (stars) → no stars data available
       └─→ relevance_score → tie (within 0.05)
            └─→ alphabetical by skill name (deterministic, no bias)
```

This chain ensures a winner is ALWAYS produced. Every step is deterministic and requires
no user input — the arbiter never deadlocks.

## Phase 4 — Record Decision [SILENT]

Append to `decision_cache.json`. Each task class naturally maps to one entry —
when a new decision for the same fingerprint is recorded, it replaces the old one.
No arbitrary entry limit.

Announce: "Recorded: **{winner}** wins for `{task_verb}` tasks. Next similar request skips comparison."

## Phase 5 — Dispatch Winner [SILENT]

The arbiter protocol ends here. The winner was **explicitly chosen through arbitration**,
which satisfies the "user explicitly names a skill" skip-condition equivalent.

1. Invoke the winner skill via the Skill tool.
2. The winner's SKILL.md loads. Execute the **full task** with the **complete input**
   (not the Phase 3a sample). The comparison was done on a representative sample;
   this is the real execution.
3. **Do NOT re-invoke skill-arbiter for the winner's main task.** The winner was deliberately
   selected — re-arbitrating would be infinite recursion.
4. Subtasks spawned BY the winner (e.g., "now search for citations") MAY invoke
   skill-arbiter per the subagent policy — this is legitimate sub-task routing.
5. The arbiter session is now complete. Resume normal operation.

**Edge cases handled by this phase:**

| Scenario | Behavior |
|----------|----------|
| Winner skill fails/errors | Report error to user. Do NOT re-arbitrate unless user asks for an alternative. |
| Winner skill has sub-steps that overlap with other skills | Winner's subagent may invoke arbiter for sub-task routing (allowed per subagent policy) |
| Single-candidate pass-through | Same dispatch — the 1 candidate is the winner, no comparison needed |
| Cache hit | Same dispatch — cached winner is the winner, skip comparison |
| Phase 3a used a sample | Phase 5 runs the FULL task with COMPLETE input. The sample assured quality; the full run delivers the result. Do NOT repeat the sample — restart from scratch with full input. |

## Edge Cases

### All candidates fail
See [Ultimate Fallback Chain](#ultimate-fallback-chain). Guaranteed to produce a winner.

### User cancels during Phase 3
- **During AskUserQuestion prompt**: No state written. Safe to retry next turn.
- **During Phase 3a subagent execution**: Abort remaining subagents. Do NOT write to
  `decision_cache.json`. Report: "Comparison cancelled. No decision recorded."
- **During Phase 5 (winner execution)**: Cache was already written in Phase 4. The
  decision stands — next similar request will cache-hit and re-dispatch.

### Plugin skill shadows loose skill
When the same skill name exists as both a plugin and a loose install, use the plugin's
version for drift checks. Plugin installs are managed and versioned; loose installs may
be manual edits.

### Skill renamed or removed
If a cached winner no longer exists in the candidate pool, mark the cache entry as
`stale:removed` and re-run Phase 1–3. Do NOT silently skip — the user's task still
needs a skill.

### Empty skill pool
If Phase 0 finds zero available skills (theoretical edge case), skip arbitration entirely.
Report: "No skills available in this environment."

### User request is ambiguous
If the task fingerprint is too vague to score (e.g., "帮我改一下" — no noun, no domain),
ask one clarifying question before proceeding. Do NOT guess.

## Audit Log

For diagnostic traceability, append one JSON line to `references/audit_log.jsonl` per
arbitration session. The log is append-only, human-readable, and never affects decisions.

### Schema (one JSON object per line)

```json
{
  "timestamp": "2026-06-22T20:45:00+08:00",
  "session_id": "uuid-short",
  "task_fingerprint": {
    "task_verb": "polish",
    "task_noun": "academic_paper",
    "domain": "academic-writing",
    "scope": "paragraph",
    "input_modality": "text_inline",
    "output_modality": "text_response",
    "constraints": "target_journal=Nature"
  },
  "phase1": {
    "semantic_matches": [{"skill": "nature-polishing", "score": 0.92}, {"skill": "paper-spine-humanize", "score": 0.85}],
    "matrix_matches": ["nature-polishing", "paper-spine-humanize"],
    "merged_candidates": ["nature-polishing", "paper-spine-humanize"],
    "multi_verb": false,
    "companion_tasks": [],
    "aggregation_friendly": false
  },
  "phase2": {
    "cache_check": "miss",
    "reason": "no prior entry for this fingerprint"
  },
  "phase3": {
    "method": "objective_metrics",
    "user_choice": "A",
    "scores": {
      "nature-polishing": {"objective": 0.94, "stars": null},
      "paper-spine-humanize": {"objective": 0.87, "stars": null}
    }
  },
  "winner": "nature-polishing",
  "winner_source": "objective_metrics",
  "duration_ms": 3200
}
```

### Rules

- Append-only, never rewrite. One line per arbitration.
- Max 1000 lines — rotate by renaming to `audit_log.{date}.jsonl` and starting fresh.
- If the file can't be written (disk full, permissions): skip silently. Audit failure
  must never block arbitration.
- The `session_id` ties together multi-turn arbitration chains for debugging.

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
    "evaluation_method": "objective_metrics | github_stars | user_override | aggregation",
    "aggregation_friendly": false,
    "created_at": "...",
    "hit_count": 0,
    "notes": "When evaluation_method is 'aggregation', winner is the sentinel value 'aggregated' — all candidates contributed equally."
  }]
}
```

## Guardrails

| Rule | Detail |
|------|--------|
| Phases 0–2, 4–5 silent | Zero user output from scan, fingerprint, cache query, record, dispatch |
| Phase 3 uses AskUserQuestion | Native permission prompt — auto-approve compatible, structured response, no text parsing |
| Auto-approve → first option | First option is always the recommended default (Merge for aggregation, Compare otherwise) |
| Never fabricate | Record `null` for undetermined scores |
| Dynamic discovery | Always scan at runtime, never hardcode |
| Version check on-the-fly | No registry file — check versions when checking cache. Plugin > Loose > Built-in priority. |
| Err on inclusion | More candidates > missing one. Matrix is the safety net for semantic gaps. |
| User overrides stick | `user_override` bypasses future comparison for that task class |
| Single candidate = pass-through | No interruption, no cache check, dispatch via Phase 5 |
| Subagent failure → fallback | If all fail in Phase 3a, use Phase 3b |
| Ultimate fallback is deterministic | Phase 3a → 3b → relevance_score → alphabetical. Never deadlocks. |
| Version → cache key | Version is part of the cached decision record, checked on-the-fly |
| Skip when user names a skill | Explicit skill invocation bypasses arbiter |
| Comparison subagents never re-invoke arbiter | Only Phase 3a bake-off subagents are restricted. All other subagents may freely use the arbiter. |
| During arbiter execution, use tools only | Phase 0–5 uses Bash/Read/WebSearch/Skill directly — but never re-invoke skill-arbiter itself. Skill tool is used ONLY in Phase 5 (dispatch) and Phase 3a (spawning comparison subagents). |
| No task verb → no scan | Greetings, chitchat, meta-questions bypass Phase 0 entirely |
| Cancel = no cache write | User cancellation during Phase 3 comparison does not write to decision_cache.json |
| Stale cache entries → re-arbitrate | Skill renamed/removed since cache was written triggers fresh Phase 1–3 |
| Audit failure is non-blocking | If audit_log.jsonl can't be written, skip silently and continue |
| Multi-verb = primary + companion | Detect 2+ verbs, arbitrate primary, flag companions after dispatch |
| Aggregation-friendly → Merge 1st | Search/review/verify/debug/explore tasks default to parallel merge as the recommended option |
| Aggregation is non-competitive | Phase 3d runs all candidates without scoring. Partial failures are tolerated. |
| Aggregation cache → auto-merge | Cached aggregation decisions skip Phase 3 interruption and auto-run all candidates |

### Anti-Recursion Rules

These prevent infinite loops where arbiter invokes itself:

| # | Rule | Enforced at |
|---|------|------------|
| R1 | **Arbiter never invokes arbiter** | Phase 0 entry: if the "task" is a meta-routing question, answer directly. skill-arbiter is excluded from the candidate pool during Phase 1 scan. |
| R2 | **Winner main task is final** | Phase 5: after dispatching winner, its main task is explicitly chosen — no re-arbitration. Equivalent to "user explicitly named this skill." |
| R3 | **Subtask routing is allowed** | Winner's subagents may invoke arbiter for sub-task routing. This is NOT recursion — it's hierarchical delegation (different task, different scope). |
| R4 | **Comparison subagents are isolated** | Phase 3a subagents receive explicit instruction: "Do NOT invoke skill-arbiter." They execute their assigned skill directly. |
| R5 | **Arbiter excluded from candidate pool** | During Phase 1 semantic matching, skip `skill-arbiter` itself. It is never a candidate for task execution — only for meta-routing decisions. |
| R6 | **CLAUDE.md bootstrap only** | If CLAUDE.md says "always invoke skill-arbiter before any skill," this is a **top-level entry gate** only. It ensures the first user request in a turn passes through arbiter. It does NOT force re-arbitration for: (a) skill-arbiter itself, (b) a winner dispatched by Phase 5 (R2), or (c) subagents executing within an already-arbitrated context. Subagents in that context MAY invoke arbiter per R3 but are never MANDATED to by CLAUDE.md — the bootstrap obligation was already discharged. |
