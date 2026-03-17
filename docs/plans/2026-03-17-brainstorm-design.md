# Design: Brainstorm v0.1.0

DATE: 2026-03-17
STATUS: draft
SOURCES: arbiter panel (Codex GPT-5.4 + Gemini 0.33.1), deep research (6 agents, 48 claims, 20 T1 sources)
RESEARCH: ~/vicc/docs/research/2026-03-17-multi-model-brainstorming-synthesis.md

## Problem

Single-model brainstorming produces ideas shaped by one model's training biases.
Existing arbiter modes don't cover ideation:
- `panel` — same prompt, compare answers (not structured ideation)
- `diverge` — 3 independent implementations (code artifacts, not idea exploration)
- `quorum` — go/no-go gate (not creative)

## Solution

`arbiter brainstorm` = **branch generator + human steering + Socratic narrowing**.

## Architecture

### Round Structure (hybrid: assertions → questions)

```
┌─────────────────────────────────────────┐
│ Round 1: DIVERGE (parallel)             │
│                                         │
│  Explorer ──┐                           │
│  Operator ──┼── 4-6 idea cards each     │
│  Contrarian ┘   + assumptions           │
│                 + questions for user     │
│                 + anti-goals             │
├─────────────────────────────────────────┤
│ STEERING CHECKPOINT                     │
│                                         │
│  Claude clusters → 5-8 branches         │
│  User selects 2-3 OR bias preset        │
│  (novelty / balanced / practical)       │
├─────────────────────────────────────────┤
│ Round 2: SOCRATIC INTERROGATION         │
│                                         │
│  Models see branch summaries only       │
│  (sparse communication)                 │
│  Output per model:                      │
│  - questions that would change ranking  │
│  - fatal risks                          │
│  - strengthening moves                  │
├─────────────────────────────────────────┤
│ SYNTHESIS (Claude orchestrator)         │
│                                         │
│  Best bets (support ≥2, 0 fatal)        │
│  Wild cards (support=1, high novelty)   │
│  Open questions                         │
│  Next experiments                       │
└─────────────────────────────────────────┘
```

### Personas (hardcoded, rotation by topic hash)

| Persona | Worldview | Style | Constraints |
|---------|-----------|-------|-------------|
| **Explorer** | Maximizes novelty. Ignores feasibility early. Outsider perspective. | Provocative, uses metaphors, questions status quo | Must produce at least 1 idea nobody asked for. No "safe" proposals. |
| **Operator** | Maximizes execution realism. Surfaces dependencies, adoption path. | Direct, structured, references prior art | Must name 2 concrete blockers for each idea. No hand-waving. |
| **Contrarian** | Attacks assumptions. Proposes inversions and anti-goals. | Skeptical, asks "what if the opposite is true?" | Must challenge the premise itself. Cannot agree without evidence. |

**Rotation:** persona assignment by `hash(topic) % 3` to avoid model↔persona bias confound.
Not: "Gemini=Explorer always" → model bias + persona bias conflated.

### Provider Dispatch

Round 1 + Round 2: all 3 providers via `run_in_background: true` (same as arbiter panel).

| Round | Claude | Codex | Gemini |
|-------|--------|-------|--------|
| 1 Diverge | Persona X (subagent) | `codex exec --ephemeral` | `gemini -p` |
| Steering | Orchestrator (inline) | — | — |
| 2 Socratic | Persona Y (subagent) | `codex exec --ephemeral` | `gemini -p` |
| Synthesis | Orchestrator (inline) | — | — |

### Convergence Detection (deterministic, no ML)

```
support_count = models that independently proposed similar branch
fatal_objections = models that found blocking issue in Round 2

Best bet:     support_count >= 2 AND fatal_objections == 0
Wild card:    support_count == 1 AND novelty flag set
Needs work:   fatal_objections > 0 (listed with specific objections)
```

Fixed stop after 2 model rounds + synthesis. No adaptive stopping in v1.
Simpson index: log as telemetry only if easy.

### Non-interactive Mode

If no TTY or `--quick` flag:
- Skip steering checkpoint
- Auto-select: max support branch + max novelty branch + max disagreement branch
- Proceed to Round 2 with those 3

### CLI Surface

```
/brainstorm "topic"                        → full flow (diverge → steer → interrogate → synthesize)
/brainstorm --quick "topic"                → skip steering, auto-select branches
/brainstorm --bias novelty "topic"         → preset: favor novel branches
/brainstorm --bias practical "topic"       → preset: favor executable branches
/brainstorm --bias balanced "topic"        → preset: balanced selection (default for --quick)
/brainstorm --providers claude,codex       → specific providers only
```

### Output Format

```markdown
## Brainstorm: "<topic>"

### Run Summary
| Provider | Persona | Status | Time |
|----------|---------|--------|------|
| Claude   | Explorer | ok    | 5.2s |
| Codex    | Operator | ok    | 8.1s |
| Gemini   | Contrarian | ok  | 6.3s |

### Branches (from Round 1)
1. [Branch name] — 2/3 support, described by Explorer + Operator
2. [Branch name] — 1/3 support, Explorer only (wild card)
...

> Selected: branches 1, 3 (user chose "practical" bias)

### Interrogation (Round 2)
**Branch 1:**
- Q: "What happens when X scales beyond Y?" — Contrarian
- Fatal risk: none
- Strengthening: "Add Z constraint" — Operator

**Branch 3:**
- Q: "Has anyone tried this outside domain W?" — Explorer
- Fatal risk: "Depends on assumption A which is unvalidated" — Contrarian
- Strengthening: "Validate A first with experiment E" — Operator

### Synthesis

**Best Bets**
1. [Branch 1]: [summary + why it's strong]

**Wild Cards**
2. [Branch 5]: [summary + what makes it interesting]

**Open Questions**
- [Question that no model could answer]
- [Question that would change the ranking]

**Next Experiments**
- [Concrete action to validate top branch]
- [Concrete action to de-risk wild card]
```

## Research Findings Applied

| # | Finding | Status | How |
|---|---------|--------|-----|
| 1 | Heterogeneous models | v1 | 3 real providers |
| 2 | Rich personas | v1 | Explorer/Operator/Contrarian with worldview+style+constraints |
| 3 | Socratic mode | v1 | Round 2 = questions, not assertions |
| 4 | 2-3 rounds max | v1 | Fixed 2 rounds |
| 5 | Divergent→Convergent | v1 | Round 1 → Steering → Round 2 → Synthesis |
| 6 | Simpson index | v1 telemetry only | Log, don't use for stopping |
| 7 | Human steering | v1 | 1 checkpoint between rounds |
| 8 | Sparse communication | v1 | Round 2 sees branch summaries, not full transcripts |
| 9 | 3 parallel critics | defer v2 | Overlaps with arbiter verify |
| 10 | Six Thinking Hats | defer v2 | Optional persona preset |

## Effort

| Component | Effort |
|-----------|--------|
| Skill SKILL.md (full protocol) | M |
| Command brainstorm.md | S |
| Persona templates | S |
| Platform adapters (codex, gemini) | S |
| Integration tests | M |
| **Total** | **M (2-3 days)** |

## Not in v1

- Adaptive stopping (Simpson-based)
- Separate critic swarm
- Six Thinking Hats preset
- >2 external rounds
- Dynamic persona authoring
- Session persistence / resume
