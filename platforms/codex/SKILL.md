---
name: brainstorm
description: Multi-model brainstorming with rich personas and Socratic questioning. 4-step flow (Diverge / Steer / Interrogate / Synthesize). Use when exploring ideas on any topic.
---

# Brainstorm (Codex adapter)

Codex-specific adapter for the brainstorm deliberation protocol.

## Protocol

4-step deliberation flow:
1. **Diverge** — parallel idea generation with 3 personas (Explorer / Operator / Contrarian)
2. **Steer** — semantic clustering into 5-8 branches; user selects 2-3
3. **Interrogate** — Socratic Round 2: questions, fatal risks, strengthening moves
4. **Synthesize** — Best Bets / Wild Cards / Open Questions / Next Experiments

## Personas

| Persona | Role |
|---------|------|
| **Explorer** | Maximizes novelty; at least 1 non-obvious idea; no safe proposals |
| **Operator** | Execution realism; names 2 concrete blockers per idea |
| **Contrarian** | Attacks assumptions; challenges the premise; proposes inversions |

Persona rotation: `hash(topic) % 3` — avoids model↔persona bias confound.

## Flags

- `--quick` — skip steering checkpoint, auto-select 3 branches
- `--bias novelty|balanced|practical` — branch selection preset
- `--providers claude,codex,gemini` — restrict provider subset

## Dispatch in this platform

Codex is invoked via `timeout 120 codex exec --ephemeral` with persona system prompt.
With `--providers gemini` only (no codex), Codex is skipped.

**Safe prompt passing:** Never interpolate topic or persona content directly into the shell
command string. Use the Write tool to save the prompt to a temp file, then pass via stdin
redirect: `cat /tmp/brainstorm_prompt_$$.txt | timeout 120 codex exec --ephemeral`.
See the security section in `../../skills/brainstorm/SKILL.md` for full details.

## Full specification

See `../../skills/brainstorm/SKILL.md` and `../../docs/plans/2026-03-17-brainstorm-design.md`
