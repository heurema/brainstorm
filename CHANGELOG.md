# Changelog

## [0.2.0] - 2026-03-18

### Added
- Stage 0 CONTEXTUALIZE — pre-brainstorm context gathering pipeline
- Facet extraction (inline LLM, 7 structured fields, keyword fallback)
- Tier 1 local corpus scan with denylist, realpath confinement, extension allowlist
- Field-based dedup classification (annotate-only, advisory)
- Persona-filtered context packs (Explorer/Operator/Contrarian routing)
- CLI flags: --context auto|off|local, --no-context, --news
- Compact persona definitions for Interrogate (Round 2)
- Data exposure policy with security boundary
- Failure matrix with graceful degradation (10 edge cases)
- Flag precedence table (8 combinations)
- Context Summary output block
- lib/stage0.py — 1441 lines, 37 functions (flat single-file)

## [0.1.0] - 2026-03-17

### Added
- Initial scaffold
- Design document from arbiter panel (Codex + Gemini consensus)
- Research synthesis: 48 claims, 20 T1 sources
