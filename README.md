# brainstorm

Multi-model brainstorming via deliberation loops — universal ideation using Claude, Codex, and Gemini with rich personas, Socratic questioning, and human steering.

## How it works

4-step deliberation flow:

1. **Diverge** — 3 models generate 4-6 idea cards each in parallel, each with a distinct persona (Explorer, Operator, Contrarian) assigned by `hash(topic) % 3` rotation
2. **Steer** — Claude clusters ideas into 5-8 branches by theme/approach; user selects 2-3 to deepen (or auto-selects with `--quick`)
3. **Interrogate** — Socratic Round 2: models receive sparse branch summaries only (not full Round 1 output) and produce questions, fatal risks, and strengthening moves
4. **Synthesize** — Convergence detection (support_count, fatal_objections) produces Best Bets / Wild Cards / Open Questions / Next Experiments

## Personas

| Persona | Role | Constraint |
|---------|------|-----------|
| **Explorer** | Maximizes novelty, outsider perspective | At least 1 non-obvious idea; no safe proposals |
| **Operator** | Execution realism, surfaces dependencies | Must name 2 concrete blockers per idea |
| **Contrarian** | Attacks assumptions, proposes inversions | Must challenge the premise; cannot agree without evidence |

Persona rotation by `hash(topic) % 3` prevents model↔persona bias confound.

## Installation

### Claude Code
```bash
claude plugin add heurema/brainstorm
```

### Codex CLI
```bash
nex install brainstorm
```

### Gemini CLI
```bash
nex install brainstorm
```

## Usage

```
/brainstorm "topic or question"
/brainstorm --quick "topic"            skip steering, auto-select branches
/brainstorm --bias novelty "topic"     preset: favor novel branches
/brainstorm --bias balanced "topic"    preset: balanced selection (default for --quick)
/brainstorm --bias practical "topic"   preset: favor executable branches
/brainstorm --providers claude,codex "topic"   restrict to 2 providers
/brainstorm --providers gemini "topic"         single provider (3 sequential persona calls)
```

## Flags

| Flag | Description |
|------|-------------|
| `--quick` | Skip steering checkpoint; deterministic auto-select (max-support + max-novelty + max-disagreement) |
| `--bias novelty\|balanced\|practical` | Branch selection preset |
| `--providers` | Comma-separated provider list (default: claude,codex,gemini) |

## Output Format

```markdown
## Brainstorm: "<topic>"

### Run Summary
| Provider | Persona | Status | Time |
...

### Branches (from Round 1)
1. [Branch] — 2/3 support, Explorer + Operator
...

> Selected: branches 1, 3

### Interrogation (Round 2)
**Branch 1:** Q / Fatal risk / Strengthening

### Synthesis
**Best Bets** / **Wild Cards** / **Open Questions** / **Next Experiments**
```

## Research

Design based on deep research: 48 claims, 20 T1 sources, 6 research agents.
See `docs/plans/2026-03-17-brainstorm-design.md` for full design document.

Key findings applied:
- Heterogeneous models beat homogeneous (ICLR 2025)
- Rich personas: +4.76 diversity (d=2.88) vs +0.62 without
- Socratic mode: -11.31% when removed (MARS, AAAI 2026)
- Human steering: novelty accuracy 13.79% → 89.66%
- 2-3 rounds max — beyond = insignificant gains
- Sparse communication in Round 2 preserves model independence

## License

MIT
