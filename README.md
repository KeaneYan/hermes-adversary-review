# Hermes Adversary Review

A focused Hermes skill repo for **adversarial review** of model-generated drafts before sending them to users or posting them externally.

## Skill

### `adversary-review`
Use a delegated subagent to review a draft reply and catch:
- factual mistakes
- overclaiming
- missing caveats
- weak phrasing
- risky suggestions
- unnecessary verbosity

This skill is especially useful for:
- technical conclusions
- bug reports and root-cause summaries
- GitHub issues / PR text
- external-facing English drafts
- important decision recommendations

## Install

### Hermes tap flow

```bash
hermes skills tap add KeaneYan/hermes-adversary-review
hermes skills install adversary-review
```

### skills.sh / skills CLI flow

```bash
npx skills add KeaneYan/hermes-adversary-review
```

## Search Keywords

This repo is intentionally aligned with these queries:
- adversary review
- adversarial review
- draft review
- response review
- quality gate
- second-pass review
- technical draft checker
- hallucination check
- overclaiming check

## Repo Layout

```text
hermes-adversary-review/
├── README.md
└── adversary-review/
    └── SKILL.md
```

This flat layout is skills.sh-friendly.
