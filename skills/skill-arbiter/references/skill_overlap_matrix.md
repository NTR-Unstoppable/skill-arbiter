# Skill Overlap Reference

Dynamic discovery (Phase 0–1) is the source of truth. This file is a quick
cross-reference of common overlap patterns. **It starts nearly empty — you
populate it as you discover conflicts in your own skill environment.**

## How to use

When the arbiter's Phase 3 surprises you (wrong winner, missed candidate), add
the conflict to the relevant section below. Format:

```
| task_verb | Candidate skills | Overlap note |
|-----------|-----------------|-------------|
| polish | skill-a vs skill-b | Brief reason they overlap |
```

## How it works

Phase 1 reads this file as a hard-match fallback. If semantic matching misses a
skill, but that skill appears in a matrix row matching the current `(task_verb, domain)`,
it's added as a confirmed candidate. This catches synonyms — when two skills describe
the same capability using different words.

Skills listed here that are NOT installed are silently ignored (filtered by Phase 0).

## Template — copy and fill for your domains

### Academic Writing

| task_verb | Candidate skills | Overlap note |
|-----------|-----------------|-------------|
| polish | *add your skills here* | e.g. nature-polishing vs paper-spine-humanize — both polish academic prose |

### Code Review & Quality

| task_verb | Candidate skills | Overlap note |
|-----------|-----------------|-------------|
| review | *add your skills here* | e.g. code-review vs security-review — both review code, different focus |

### Literature Search

| task_verb | Candidate skills | Overlap note |
|-----------|-----------------|-------------|
| search | *add your skills here* | e.g. cnki-search vs wfdata-search — both search Chinese databases |

### Planning & Workflow

| task_verb | Candidate skills | Overlap note |
|-----------|-----------------|-------------|
| plan | *add your skills here* | e.g. brainstorming vs writing-plans — both are pre-coding planning steps |

### PR & Git Management

| task_verb | Candidate skills | Overlap note |
|-----------|-----------------|-------------|
| review-pr | *add your skills here* | e.g. review vs code-review — PR-level vs diff-level review |

### Skill Management

| task_verb | Candidate skills | Overlap note |
|-----------|-----------------|-------------|
| create-skill | *add your skills here* | e.g. skill-creator vs writing-skills — both help create new skills |

### Your custom domains

| task_verb | Candidate skills | Overlap note |
|-----------|-----------------|-------------|
| *verb* | *add your skills here* | *describe the conflict* |
