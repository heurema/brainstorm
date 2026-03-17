# brainstorm

Multi-model brainstorming via deliberation loops — universal ideation using Claude, Codex, and Gemini with rich personas, Socratic questioning, and human steering.

## How it works

1. **Diverge** — 3 models generate ideas in parallel with rich personas (Explorer, Operator, Contrarian)
2. **Steer** — ideas clustered into branches, user selects 2-3 to deepen
3. **Interrogate** — Socratic round: models question branches, find fatal risks, suggest strengthening moves
4. **Synthesize** — Best bets / Wild cards / Open questions / Next experiments

## Installation

### Claude Code
```bash
claude plugin add heurema/brainstorm
```

### Codex CLI
```bash
# Add to .codex/agents/ or install via nex
nex install brainstorm
```

### Gemini CLI
```bash
# Add to GEMINI.md or install via nex
nex install brainstorm
```

## Usage

```
/brainstorm "topic or question"
/brainstorm --quick "topic"          # skip steering, auto-select branches
/brainstorm --bias novelty "topic"   # preset branch selection bias
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

## License

MIT
