---
name: improve-codebase-architecture
description: Find codebase-native improvement opportunities in existing scientific-computing C and C++ repositories. Use when the user wants safer scientific/HPC C/C++ code, architecture review, refactoring candidates, public-header cleanup, ownership/lifetime clarification, numerical-invariant documentation, performance-aware simplification, Makefile-oriented code inspection, or testability improvements that fit the local codebase. First read repo-local instructions, Makefiles, compile_commands.json or actual compile flags, decision records, local style, data layout, and numerical contracts. Uses haft natively when present for related-decision pre-checks, cross-project precedents, and framing handoff.
---

# Improve Codebase Architecture

Surface improvement opportunities that make an existing scientific C/C++ codebase safer, easier to verify, and more maintainable while preserving local style, numerical semantics, data layout, and build reality.

The original deepening vocabulary is still useful: **module**, **interface**, **implementation**, **depth**, **seam**, **adapter**, **leverage**, and **locality**. Use it consistently; see [LANGUAGE.md](LANGUAGE.md). But do not treat "deepening" as the automatic goal. In scientific code, explicit loops, flat data, and simple functions are often the right shape.

## Reference Map

- Read [SCIENTIFIC_CPP.md](SCIENTIFIC_CPP.md) for the scientific/HPC reconnaissance checklist, Makefile and `compile_commands.json` defaults, hot-path caveats, and external research anchors.
- Read [DEEPENING.md](DEEPENING.md) only when a candidate genuinely involves module depth, seam choice, or interface/test-surface design.
- Read [INTERFACE-DESIGN.md](INTERFACE-DESIGN.md) when the user chooses a candidate and wants alternative interface designs compared.

## Process

### 1. Start With Local Authority

Repo-local instructions override this skill. Read `AGENTS.md`, `.haft/`, `CONTRIBUTING`, architecture docs, local style notes, and the user's task framing before applying defaults.

In haft projects, check active decisions before proposing changes. If a candidate contradicts a live decision, drop it or surface the decision ID and frame it as a reopening question.

### 2. Produce A Codebase Fit Brief

Before proposing candidates, inspect the repo enough to fill this compact brief:

```text
Codebase Fit Brief
- Repo-local instructions:
- Build authority:
- Compile database / inspection flags:
- Numerical domain:
- Interface/header style:
- Data layout and ownership:
- Error/assertion style:
- Allocation/performance conventions:
- Tests/benchmarks:
- Local idioms to preserve:
- Safety/correctness friction:
- Uncertainty:
```

Default assumptions: Makefiles first; CMake secondary unless clearly authoritative. Use `compile_commands.json` or actual compiler invocations before making tooling or include/macro claims. If a downstream host repo is supplied, treat it as read-only adoption/style evidence unless the user explicitly scopes edits there.

### 3. Explore Scientific Friction

Look first for scientific C/C++ problems:

- implicit numerical invariants, units, tolerances, solver regimes, or reproducibility assumptions;
- unclear array extents, stride, aliasing, ownership, lifetime, or error handling;
- hidden allocation, copies, virtual dispatch, RTTI, exceptions, branchy control flow, or other hot-path costs;
- headers that expose implementation details, trigger rebuild cascades, or hide contracts;
- tests that miss public scientific behavior or rely on mocks where deterministic regression tests would be stronger.

Then apply the original depth/deletion-test lens. A module is worth deepening only if it improves locality, caller leverage, or verification without harming the scientific constraints.

### 4. Present Candidates

Use a numbered list. For each candidate include:

- **Files** - exact files/modules involved.
- **Problem** - concrete friction observed in this codebase.
- **Local fit** - conventions preserved, including Makefile/test/tooling impact.
- **Scientific constraint** - numerical semantics, data layout, hot path, ABI, parallelism, reproducibility, or dependency constraints.
- **Solution** - the smallest change that improves correctness, safety, locality, or verification.
- **Safety/correctness gain** - what risk becomes visible, typed, tested, or removed.
- **Performance/build risk** - layout, allocation, inlining, vectorization, compile time, link order, ABI, or rebuild impact.
- **Evidence** - repo-local facts, linked decisions, tests/benchmarks/profilers/sanitizers, and any external references used.
- **Ceremony level** - `note-level`, `tactical`, `standard`, or `deep`.

Do not propose final interfaces immediately. Ask which candidate to explore unless the user explicitly asks for implementation.

### 5. Design Or Implement

For note-level/tactical work, sketch the minimal patch and verification command. For standard/deep work, use haft or the inline decision framework before changing public interfaces, memory layout, ABI, or solver contracts.

The user owns domain-critical decisions: numerical model, acceptable tolerance, performance budget, and physical meaning. Do not silently decide those.
