---
name: brainstorm
description: Use when the user wants to brainstorm, ideate, explore ideas, generate alternatives, think through options, or needs creative input on any topic — code, product, business, strategy, life decisions. Triggers on "brainstorm", "let's think about", "what are the options", "explore ideas", "ideate", "what if we", "help me think through". Always use this skill for open-ended creative exploration, even if the user doesn't say "brainstorm" explicitly.
---

# Brainstorm — Multi-Model Deliberation Protocol

Full 4-step flow: **Diverge → Steer → Interrogate → Synthesize**

Design spec: `docs/plans/2026-03-17-brainstorm-design.md`

---

## Overview

`/brainstorm` runs structured multi-model ideation using 3 providers (Claude, Codex, Gemini) with
3 hardcoded personas (Explorer, Operator, Contrarian), a human steering checkpoint between rounds,
sparse communication in Round 2, and deterministic convergence detection (support_count,
fatal_objections) to produce structured output.

---

## CLI Surface

```
/brainstorm "topic"                          full flow (diverge → steer → interrogate → synthesize)
/brainstorm --quick "topic"                  skip steering, auto-select 3 branches
/brainstorm --bias novelty "topic"           preset: favor novel branches
/brainstorm --bias balanced "topic"          preset: balanced selection (default for --quick)
/brainstorm --bias practical "topic"         preset: favor executable branches
/brainstorm --providers claude,codex "topic" restrict to subset of providers
```

Parse flags before topic text. `--quick` and `--bias` may be combined. `--bias` without
`--quick` suggests priorities during interactive selection.

---

## Personas

Three personas are hardcoded. Each provider is assigned a persona via **persona rotation**:
`persona_index = hash(topic) % 3` where hash is a simple sum of character code points mod a large
prime. Rotation starts at persona_index; the three providers get personas 0, 1, 2 cyclically from
that offset. This avoids model↔persona bias confound (not "Gemini=Explorer always").

### Explorer
- **Worldview:** Maximizes novelty. Ignores feasibility early. Outsider perspective.
- **Style:** Provocative, uses metaphors, questions status quo.
- **Constraint:** Must produce at least 1 novel idea nobody asked for. No "safe" proposals. Must challenge conventional framing.

### Operator
- **Worldview:** Maximizes execution realism. Surfaces dependencies, adoption path.
- **Style:** Direct, structured, references prior art.
- **Constraint:** Must name 2 concrete blockers per idea. No hand-waving on implementation.

### Contrarian
- **Worldview:** Attacks assumptions. Proposes inversions and anti-goals.
- **Style:** Skeptical, asks "what if the opposite is true?".
- **Constraint:** Must challenge the premise itself. Cannot agree without evidence. Must propose at least one inversion.

---

## Provider Dispatch

### Provider Binding

**Provider binding** rules based on `--providers` flag:

- **3 providers** (default: claude, codex, gemini): each gets a distinct persona via rotation.
- **2 providers**: assign 2 personas via hash rotation (skip 3rd persona).
- **1 provider**: invoke that provider 3 times sequentially with different system prompts — one per persona — so all 3 personas always run. Sequential calls preserve independence (no shared context between calls). In single-provider mode, `support_count` counts per-persona (not per-provider): it equals the number of personas whose idea cards independently cluster into the same branch. Max support_count = 3 (all 3 persona passes). Best Bet threshold (>= 2) applies to persona-count in this mode.

### Dispatch Commands

| Provider | Round 1 | Round 2 |
|----------|---------|---------|
| Claude   | Subagent (run_in_background: true) | Subagent (run_in_background: true) |
| Codex    | `timeout 120 codex exec --ephemeral` | `timeout 120 codex exec --ephemeral` |
| Gemini   | `timeout 120 gemini -p` | `timeout 120 gemini -p` |

Each dispatch runs with a 120-second timeout (120s per provider). On timeout, mark provider
status as "timeout" in Run Summary and continue with available results.

**Safe prompt passing (shell injection prevention):** Never interpolate `<TOPIC>` or persona
content directly into shell command strings. Use the Write tool to write the prompt to a
temp file, then pass via `--url-file` or stdin redirection (e.g.,
`timeout 120 codex exec --ephemeral < /tmp/brainstorm_prompt_$$.txt`). This matches the
safe dispatch pattern used in arbiter to prevent shell injection on user-controlled input.

---

## Step 1: DIVERGE (Round 1)

Dispatch all active providers in parallel (or sequentially for 1-provider mode).

### System prompt per provider (inject persona)

```
You are the <PERSONA> in a multi-model brainstorm on: "<TOPIC>"

<PERSONA_DEFINITION>

Produce exactly 4-6 idea cards. Each idea card must follow this format:

### Idea: <short title>
**Core concept:** One sentence.
**Assumptions:** List 2-3 assumptions this idea depends on.
**Questions for user:** 1-2 questions that would clarify feasibility or scope.
**Anti-goals:** What this idea explicitly does NOT try to solve.

Stay in persona. Do not break character. Do not explain your persona role.
```

Replace `<PERSONA>` with the assigned persona name. Replace `<PERSONA_DEFINITION>` with the
persona's worldview, style, and constraint block verbatim.

### Orchestrator aggregation

Collect raw idea cards from all providers. Do not reorder or filter. Do not show providers
which ideas other providers produced (preserve independence for Round 2).

---

## Step 2: STEER (Steering Checkpoint)

**Note on `--providers` and the Claude orchestrator:** `--providers` controls which models run
in Round 1 (Diverge) and Round 2 (Interrogate) dispatch. Claude ALWAYS performs the Steer
(clustering) and Synthesize phases as orchestrator regardless of the `--providers` value —
Claude is not a provider in those phases. If Claude is excluded from `--providers`, it still
runs Steer and Synthesize inline; it only skips the Round 1/Round 2 provider dispatch.

Claude orchestrator (inline, not subagent) performs **semantic clustering** of all idea cards
into **5-8 branches** grouped by theme or approach. Clustering is deterministic: group ideas that
share a common solution mechanism, target beneficiary, or technology approach. Label each branch
with a short descriptive name.

For each branch, compute:
- `support_count`: number of distinct providers whose idea cards cluster into this branch (max = number of active providers)
- `novelty_score`: flag as high-novelty if at least one contributing idea card came from Explorer persona and has no direct precedent in other branches
- `disagreement_score`: count of providers whose idea cards do NOT cluster into this branch

### Interactive Mode (default, TTY present, no --quick)

Present branches to user:

```
### Branches (from Round 1)
1. [Branch name] — X/3 support, by [Persona list]
2. [Branch name] — X/3 support, by [Persona list]
...

Select 2-3 branches to deepen (enter numbers separated by commas),
or enter a bias preset (novelty / balanced / practical):
```

User may type branch numbers (e.g. "1,3") or a preset name. `--bias` flag prefills the
suggestion but user can override.

### Auto-select (--quick or non-TTY / non-interactive)

If `--quick` flag present or TTY is not detected (non-interactive, non-TTY execution), skip
the steering checkpoint entirely. Deterministically auto-select exactly 3 distinct branches
(or all available if fewer than 3 exist) using the following ordered criteria with dedup:

1. **max-support branch** — pick the branch with the highest support_count. On tie, pick the
   one with the lowest branch index (earliest in list).
2. **max-novelty branch** — from the remaining branches NOT already selected, pick the branch
   with the highest novelty_score (high-novelty flag, Explorer-sourced). On tie, lowest branch
   index. If the max-novelty branch was already selected in step 1, skip it and pick the
   next-best novelty branch from the remaining pool.
3. **max-disagreement branch** — from the remaining branches NOT already selected in steps 1
   or 2, pick the branch with the highest disagreement_score. On tie, lowest branch index.

**Deduplication rule:** Each branch may appear in at most one selection slot. If a branch wins
multiple criteria, it occupies the slot for the first criterion it satisfies; subsequent
criteria draw from the remaining pool. Always produce exactly 3 distinct branches (or all
available if fewer than 3 exist). Log auto-selection rationale and which branches were skipped
due to dedup.

### Bias Presets

| Preset | Selection rule |
|--------|---------------|
| `novelty` | Sort by novelty_score desc, pick top 3 |
| `practical` | Sort by support_count desc, pick top 3 |
| `balanced` | Pick 1 max-support + 1 max-novelty + 1 max-disagreement (default for --quick) |

In interactive mode, `--bias` displays the preset selection as a suggestion. User can confirm
or override.

---

## Step 3: INTERROGATE (Round 2 — Socratic Interrogation)

Dispatch all active providers again in parallel (or sequentially for 1-provider mode).

### Sparse communication constraint

Providers receive **ONLY** sparse branch summaries — NOT the full idea cards or raw Round 1
output. The orchestrator writes a 2-3 sentence summary for each selected branch based on Round 1
output. Summaries include: branch_id, branch_title, and a 2-3 sentence description of the core
approach and key assumptions. This preserves model independence for Round 2.

### System prompt per provider (interrogation)

```
You are the <PERSONA> reviewing proposed solution branches for: "<TOPIC>"

<PERSONA_DEFINITION>

The following branch summaries have been selected for deeper analysis:

<BRANCH_SUMMARIES>
(Format: Branch N: <title> — <2-3 sentence summary>)

For each branch, produce:
a) Questions that would change ranking: 1-2 questions whose answers would significantly
   raise or lower this branch's priority.
b) Fatal risks or blocking issues: specific, concrete blockers that make this branch
   unviable as stated (or "none" if no fatal issues).
c) Strengthening moves: 1-2 concrete actions that would make this branch stronger.

Stay in persona throughout.
```

### Orchestrator collection

Collect Round 2 outputs. For each branch, aggregate across providers:
- `fatal_objections`: count of distinct providers that identified a fatal risk/blocking issue
- Collect all questions and strengthening moves by branch

---

## Step 4: SYNTHESIZE

Claude orchestrator (inline) applies **convergence detection** and produces final output.

### Convergence rules

```
support_count = number of distinct providers (or personas in single-provider mode) that
               independently proposed idea cards clustering into the same branch
               (max = number of active providers; in single-provider mode max = 3 personas)
fatal_objections = number of distinct providers (or personas) that identified a blocking
               issue in Round 2

Best Bet:   support_count >= 2 AND fatal_objections == 0
Wild Card:  support_count == 1 AND novelty flag set (high-novelty, Explorer-sourced)
            AND fatal_objections == 0
Needs Work: fatal_objections > 0 (EXCLUSIVE — a branch with any fatal objection is always
            Needs Work, never Best Bet or Wild Card, regardless of support_count)
```

Apply rules to all selected branches. Categories are mutually exclusive: fatal_objections > 0
classifies a branch as Needs Work ONLY. A Best Bet branch must have fatal_objections == 0 with
no exceptions. List fatal objections in the Needs Work section for remediation.

### Output format

Produce this exact structure:

```markdown
## Brainstorm: "<topic>"

### Run Summary
| Provider | Persona | Status | Time |
|----------|---------|--------|------|
| Claude   | <Persona> | ok/timeout/error | <Xs> |
| Codex    | <Persona> | ok/timeout/error | <Xs> |
| Gemini   | <Persona> | ok/timeout/error | <Xs> |

### Branches (from Round 1)
1. [Branch name] — X/3 support, described by [Persona] + [Persona]
2. [Branch name] — 1/3 support, [Persona] only (wild card)
...

> Selected: branches N, N (user chose / auto-selected with [rationale])

### Interrogation (Round 2)
**Branch N: [Branch name]**
- Q: "[question that would change ranking]" — [Persona]
- Fatal risk: [specific blocker or "none"]
- Strengthening: "[concrete action]" — [Persona]

(repeat for each selected branch)

### Synthesis

**Best Bets**
N. [Branch name]: [summary + why it's strong]

**Wild Cards**
N. [Branch name]: [summary + what makes it interesting despite low support]

**Needs Work**
N. [Branch name]: [fatal objection details + which persona raised it]

**Open Questions**
- [Question no model could answer definitively]
- [Question whose answer would change the ranking]

**Next Experiments**
- [Concrete action to validate top Best Bet]
- [Concrete action to de-risk a Wild Card or Needs Work branch]
```

---

## Non-Interactive / --quick Execution Notes

- Detect TTY: if `test -t 0` is false or `--quick` flag is present, run in non-interactive mode.
- Auto-select branches deterministically (max-support, max-novelty, max-disagreement).
- Skip all interactive prompts. Produce valid complete output without pausing.
- Log in Run Summary: "auto-selected (--quick)" or "auto-selected (non-TTY)".

---

## Telemetry (optional)

If easy to compute, log Simpson diversity index of idea cards after Round 1:
`D = 1 - sum((n_i/N)^2)` where n_i = ideas per provider, N = total ideas.
Do not use D for stopping or selection decisions — log only.

---

## Error Handling

- Provider timeout (120s): mark status "timeout", continue with remaining providers.
- Provider error: mark status "error", continue. If all providers fail, abort with message.
- <2 providers responding: warn user, proceed with available results.
- Empty idea cards: skip that provider's output in clustering.

---

## Examples

**Round 1 dispatch (3 providers, topic="AI tutoring app"):**
- hash("AI tutoring app") % 3 = 0 → Claude=Explorer, Codex=Operator, Gemini=Contrarian
- All 3 dispatch in parallel with 120s timeout

**Round 1 dispatch (1 provider, --providers gemini):**
- Sequential 3 calls to gemini: first with Explorer prompt, then Operator, then Contrarian
- provider binding: 3 sequential calls, all 3 personas always run

**Auto-select (--quick):**
- Branch 2: support_count=3 → max-support
- Branch 5: novelty_score=high → max-novelty
- Branch 1: disagreement_score=2 → max-disagreement
- Selected: 2, 5, 1
