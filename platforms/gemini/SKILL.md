---
name: brainstorm
description: Multi-model brainstorming with rich personas and Socratic questioning. 5-step flow (Stage 0 CONTEXTUALIZE / Diverge / Steer / Interrogate / Synthesize). Use when exploring ideas on any topic.
---

# Brainstorm (Gemini adapter)

Gemini-specific adapter for the brainstorm deliberation protocol.

## Protocol

5-step deliberation flow:
0. **Stage 0 CONTEXTUALIZE** — scan local corpus, extract facets, build persona-specific context packs
1. **Diverge** — parallel idea generation with 3 personas (Explorer / Operator / Contrarian)
2. **Steer** — semantic clustering into 5-8 branches; user selects 2-3
3. **Interrogate** — Socratic Round 2: questions, fatal risks, strengthening moves
4. **Synthesize** — Best Bets / Wild Cards / Open Questions / Next Experiments

See `../../skills/brainstorm/SKILL.md` for Stage 0 CONTEXTUALIZE implementation details.
Stage 0 implementation: `../../lib/stage0.py` (flat single-file; also available as `../../lib/stage0/stage0_orchestrator.py`)

## Personas

| Persona | Role |
|---------|------|
| **Explorer** | Maximizes novelty; at least 1 non-obvious idea; no safe proposals |
| **Operator** | Execution realism; names 2 concrete blockers per idea |
| **Contrarian** | Attacks assumptions; challenges the premise; proposes inversions |

Persona rotation: `hash(topic) % 3` — avoids model↔persona bias confound.

## Flags

- `--context auto|local|off|deep` — context gathering depth (auto is default)
- `--no-context` — skip Stage 0 entirely, v0.1.0 behavior
- `--news` — force Tier 3 web/news scan
- `--quick` — skip steering checkpoint, auto-select 3 branches; implicit --context local
- `--bias novelty|balanced|practical` — branch selection preset
- `--providers claude,codex,gemini` — restrict provider subset

## Dispatch in this platform

Gemini is invoked via `timeout 120 gemini -p` with persona system prompt.
When Stage 0 is active, the prompt includes shared_core FIRST, then Gemini's persona context pack.
With `--providers claude,codex` (no gemini), Gemini is skipped.

**Safe prompt passing:** Never interpolate topic or persona content directly into the shell
command string. Use the Write tool to save the prompt to a temp file, then pass via stdin
redirect: `cat /tmp/brainstorm_prompt_$$.txt | timeout 120 gemini -p`.
See the security section in `../../skills/brainstorm/SKILL.md` for full details.

## Full specification

See `../../skills/brainstorm/SKILL.md` and `../../docs/plans/2026-03-17-brainstorm-design.md`
Stage 0 design: `../../docs/plans/2026-03-18-brainstorm-v020-design.md`
