---
name: brainstorm
description: Multi-model brainstorming with rich personas, Socratic questioning, and human steering
arguments:
  - name: topic
    description: The topic or question to brainstorm about
    required: true
  - name: --quick
    description: Skip steering checkpoint, auto-select branches
    required: false
  - name: --bias
    description: "Branch selection bias: novelty, balanced, practical"
    required: false
  - name: --providers
    description: "Comma-separated provider list (default: claude,codex,gemini)"
    required: false
---

Launch the brainstorm skill with the provided topic and flags.
