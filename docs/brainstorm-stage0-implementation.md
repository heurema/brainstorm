# Brainstorm v0.2.0 — Stage 0 CONTEXTUALIZE Implementation

**Version:** 0.2.0
**Date:** 2026-03-18
**Status:** implemented

## Overview

Stage 0 CONTEXTUALIZE runs before Diverge to scan local corpus, extract topic facets,
classify observation cards, and assemble persona-specific context packs for providers.

All Stage 0 failures degrade gracefully — the brainstorm run never aborts.

## File Layout

```
lib/stage0/
  facet_extractor.py       — Step 1: topic → {goal, audience, mechanism, ...} JSON
  corpus_scanner.py        — Step 2: Tier 1 local glob scan + Tier 1b brainstorm outputs
  dedup_classifier.py      — Step 3: field-based annotation (advisory-only)
  context_pack_builder.py  — Step 4: shared_core + persona packs + token budget
  config_parser.py         — brainstorm.local.md YAML parser
  tier3_web_scanner.py     — Tier 3: WebSearch worker (--news flag only)
  stage0_orchestrator.py   — main entry point, coordinates all steps
```

## Entry Point

```python
from lib.stage0 import run_stage0

result = run_stage0(
    topic="my brainstorm topic",
    project_root="/path/to/project",
    context_mode="auto",   # "auto" | "local" | "off" | "deep"
    news_enabled=False,    # True if --news flag set
    config_path=None,      # optional path to brainstorm.local.md
)
```

Returns:
```python
{
    "context_packs": {
        "shared_core": "## Shared Context\n...",
        "Explorer": "## Persona Context: Explorer\n...",
        "Operator": "## Persona Context: Operator\n...",
        "Contrarian": "## Persona Context: Contrarian\n...",
        "total_tokens": 8500,
        "summary": {...},
    },
    "classified_cards": [...],
    "facets": {...},
    "summary": {...},
    "context_summary_table": "### Context Summary\n...",
    "status": "ok",       # "ok" | "degraded" | "empty" | "no-context"
    "warnings": [],
}
```

## Failure Matrix

| Condition | Status | Behavior |
|-----------|--------|----------|
| `--context off` / `--no-context` | no-context | Skip Stage 0 entirely |
| Non-git directory | degraded | Skip git log; proceed with files |
| Empty corpus | empty | 0 cards; no Already Covered block |
| Facet extraction timeout | degraded | Keyword fallback |
| Bad brainstorm.local.md YAML | degraded | Warn; use defaults |
| WebSearch unavailable | degraded | Skip Tier 3; continue |
| `brainstorm.local.md` is symlink | degraded | Refuse; warn; use defaults |
| Zero observation cards | empty | Skip context injection; v0.1.0 |

## Security Policy

- All file access: `realpath` + verify within `PROJECT_ROOT`
- Symlinks: refused with warning
- Denylist patterns: `.env`, `secrets/`, `private/`, `credentials`, `*.key`, `*.pem`,
  `memory/bank/`, `.ssh/`, `.config/`, `node_modules/`, `vendor/`
- Snippet sanitization: strip base64 (40+ chars), email, IP, markdown images
- Web content: treated as DATA — security boundary injected in web worker prompts
- `yaml.safe_load()` for config parsing (no code execution)

## Token Budget

| Component | Target | Hard Cap |
|-----------|--------|----------|
| Protocol + persona + output | 2k | — |
| Shared core (facets + dedup) | 2-3k | — |
| Persona observation cards | 4-6k each | — |
| News block (Tier 3 only) | 2-4k | — |
| Reserve | 1-2k | — |
| **Diverge prompt total** | **12-18k** | **24k** |
| **Interrogate total** | 6-9k | — |

## Dedup Classification

Annotations are advisory-only — ideas are NEVER suppressed or blocked automatically.

| Condition | Annotation | Persona |
|-----------|-----------|---------|
| `facet_overlap_count >= 3 AND mechanism_match` | Already Covered | Operator |
| `facet_overlap_count 1-2 AND NOT mechanism_match` | Adjacent | Explorer |
| `recency_class = stale` | Stale | Explorer + Contrarian |
| `risk_flags non-empty` | Risk Signal | Contrarian |
| `facet_overlap_count = 0` | Dropped | None |

## CLI Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--context auto` | default | Full Tier 1 local scan |
| `--context local` | — | Tier 1 only (no web) |
| `--context off` | — | Skip Stage 0 (v0.1.0) |
| `--context deep` | — | v0.3.0+ (warns, falls back to auto) |
| `--no-context` | — | Alias for --context off; always wins |
| `--news` | — | Enable Tier 3 WebSearch worker |

## Interrogate (Round 2) Context

Pass 3-5 highest-ranking observation cards per selected branch ONLY.
NOT the full Stage 0 output.
Total Interrogate context budget: 6-9k tokens.
Compact persona definitions used (<=80 tokens per persona).
