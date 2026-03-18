---
name: brainstorm
description: Multi-model brainstorming with rich personas, Socratic questioning, and human steering — 5-step deliberation flow (Stage 0 CONTEXTUALIZE / Diverge / Steer / Interrogate / Synthesize)
arguments:
  - name: topic
    description: The topic or question to brainstorm about
    required: true
  - name: --quick
    description: Skip steering checkpoint, auto-select 3 branches deterministically (max-support + max-novelty + max-disagreement). Implicitly uses --context local unless --context is set explicitly.
    required: false
  - name: --context
    description: "Context gathering depth: auto (default, full Tier 1), local (Tier 1 only, no web), off (disable Stage 0 entirely), deep (v0.3.0+, falls back to auto with warning)"
    required: false
  - name: --no-context
    description: "Alias for --context off. Skips Stage 0 entirely, runs v0.1.0 behavior. Always wins over other context flags."
    required: false
  - name: --news
    description: "Force Tier 3 web/news scan (WebSearch) in addition to Tier 1 local scan. Ignored if --no-context or --context off is set."
    required: false
  - name: --bias
    description: "Branch selection bias preset: novelty (favor novel branches), balanced (default), practical (favor executable branches)"
    required: false
  - name: --providers
    description: "Comma-separated provider list (default: claude,codex,gemini). With 1 provider: runs 3 sequential calls with all 3 personas."
    required: false
---

# /brainstorm

Runs the brainstorm deliberation protocol on the given topic.

## Flow

0. **CONTEXTUALIZE** (Stage 0, v0.2.0) — scan local corpus, extract facets, classify observation
   cards, assemble persona-specific context packs. Degrades gracefully if no corpus found.
1. **Diverge** — 3 providers generate 4-6 idea cards each in parallel, each with a distinct
   persona (Explorer / Operator / Contrarian) assigned by `hash(topic) % 3` rotation.
   Providers receive shared_core + persona-specific context packs from Stage 0.
2. **Steer** — Claude clusters ideas into 5-8 branches; user selects 2-3 to deepen
   (or auto-selects with `--quick`).
3. **Interrogate** — Round 2: providers receive sparse branch summaries + 3-5 relevant
   observation cards (not full Stage 0 output) and produce questions, fatal risks, and
   strengthening moves.
4. **Synthesize** — Convergence detection (support_count, fatal_objections) produces
   Best Bets / Wild Cards / Open Questions / Next Experiments.

## Usage

```
/brainstorm "topic or question"
/brainstorm --quick "topic"
/brainstorm --context auto "topic"
/brainstorm --context local "topic"
/brainstorm --context off "topic"
/brainstorm --no-context "topic"
/brainstorm --news "topic"
/brainstorm --quick --news "topic"
/brainstorm --no-context --quick "topic"
/brainstorm --bias novelty "topic"
/brainstorm --bias balanced "topic"
/brainstorm --bias practical "topic"
/brainstorm --providers claude,codex "topic"
/brainstorm --providers gemini "topic"
/brainstorm --quick --bias novelty "topic"
```

## Flags

| Flag | Values | Default | Description |
|------|--------|---------|-------------|
| `--context` | `auto`, `local`, `off`, `deep` | `auto` | Context gathering depth |
| `--no-context` | (none) | — | Alias for `--context off`; always wins |
| `--news` | (none) | — | Force Tier 3 web/news scan (WebSearch); ignored with `--context off` |
| `--quick` | (none) | — | Skip steering checkpoint; implicit `--context local` downgrade |
| `--bias` | `novelty`, `balanced`, `practical` | `balanced` | Branch selection preset |
| `--providers` | comma-separated list | `claude,codex,gemini` | Restrict to subset of providers |

## Flag Precedence

| Combination | Effective behavior |
|-------------|-------------------|
| (no flags) | `--context auto` |
| `--quick` | `--context local` (implicit downgrade) |
| `--quick --context deep` | `--context deep` (explicit overrides implicit) |
| `--quick --no-context` | `--context off` (`--no-context` always wins) |
| `--quick --news` | `--context local` + Tier 3 news |
| `--no-context --news` | `--context off` (no-context wins, news ignored with warning) |
| `--context off --news` | `--context off` (off wins, news ignored with warning) |
| `--no-context` | `--context off` (alias) |

**Rule:** `--no-context`/`--context off` takes highest precedence. Explicit `--context <value>`
overrides implicit `--quick` downgrade. `--news` is additive (enables Tier 3) but cannot
override `off`.

## Context Modes

| Mode | Tier 1 | Tier 2 | Tier 3 | Notes |
|------|--------|--------|--------|-------|
| `auto` (default) | always | v0.3.0 | only with `--news` | Full local context |
| `local` | always | no | no | Faster, no web |
| `off` | no | no | no | v0.1.0 behavior |
| `deep` | always | always | always | v0.3.0+ (warns, falls back to auto) |

## Providers

Default providers: `claude`, `codex`, `gemini`

- **3 providers**: each gets a distinct persona via hash rotation
- **2 providers**: 2 personas assigned via rotation, 3rd skipped
- **1 provider**: 3 sequential calls with all 3 personas (Explorer, Operator, Contrarian)

## Output Sections

- **Run Summary** — Provider / Persona / Status / Time table
- **Context Summary** — Source / Items / Tokens table + dedup counts (from Stage 0)
- **Branches** — Numbered list with support count and persona attribution
- **Selected** — Which branches proceed to Round 2 and why
- **Interrogation** — Per-branch: questions, fatal risks, strengthening moves
- **Synthesis** — Best Bets, Wild Cards, Open Questions, Next Experiments

## Stage 0 Configuration

Optional `brainstorm.local.md` file (YAML frontmatter) in project root:

```yaml
---
corpus_globs:
  - "content/**/*.md"
  - "blog/**/*.md"
  - "posts/**/*.md"
---
```

Omit to use defaults: `content/**/*.md`, `blog/**/*.md`, `posts/**/*.md`, `docs/**/*.md`,
`notes/**/*.md`, `ideas/**/*.md`.

## Design

Full protocol specification: `docs/plans/2026-03-17-brainstorm-design.md`
Stage 0 design: `docs/plans/2026-03-18-brainstorm-v020-design.md`
