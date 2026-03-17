---
name: brainstorm
description: Multi-model brainstorming with rich personas, Socratic questioning, and human steering — 4-step deliberation flow (Diverge / Steer / Interrogate / Synthesize)
arguments:
  - name: topic
    description: The topic or question to brainstorm about
    required: true
  - name: --quick
    description: Skip steering checkpoint, auto-select 3 branches deterministically (max-support + max-novelty + max-disagreement)
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

1. **Diverge** — 3 providers generate 4-6 idea cards each in parallel, each with a distinct
   persona (Explorer / Operator / Contrarian) assigned by `hash(topic) % 3` rotation.
2. **Steer** — Claude clusters ideas into 5-8 branches; user selects 2-3 to deepen
   (or auto-selects with `--quick`).
3. **Interrogate** — Round 2: providers receive sparse branch summaries only (not full idea
   cards) and produce questions, fatal risks, and strengthening moves.
4. **Synthesize** — Convergence detection (support_count, fatal_objections) produces
   Best Bets / Wild Cards / Open Questions / Next Experiments.

## Usage

```
/brainstorm "topic or question"
/brainstorm --quick "topic"
/brainstorm --bias novelty "topic"
/brainstorm --bias balanced "topic"
/brainstorm --bias practical "topic"
/brainstorm --providers claude,codex "topic"
/brainstorm --providers gemini "topic"
/brainstorm --quick --bias novelty "topic"
```

## Flags

| Flag | Values | Description |
|------|--------|-------------|
| `--quick` | (none) | Skip steering checkpoint; deterministic auto-select |
| `--bias` | `novelty`, `balanced`, `practical` | Branch selection preset |
| `--providers` | comma-separated list | Restrict to subset of providers |

## Providers

Default providers: `claude`, `codex`, `gemini`

- **3 providers**: each gets a distinct persona via hash rotation
- **2 providers**: 2 personas assigned via rotation, 3rd skipped
- **1 provider**: 3 sequential calls with all 3 personas (Explorer, Operator, Contrarian)

## Output Sections

- **Run Summary** — Provider / Persona / Status / Time table
- **Branches** — Numbered list with support count and persona attribution
- **Selected** — Which branches proceed to Round 2 and why
- **Interrogation** — Per-branch: questions, fatal risks, strengthening moves
- **Synthesis** — Best Bets, Wild Cards, Open Questions, Next Experiments

## Design

Full protocol specification: `docs/plans/2026-03-17-brainstorm-design.md`
