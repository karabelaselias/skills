# Scientific C/C++ Reference

Use this reference for scientific, numerical, solver, geometry, FEM, simulation, HPC, or C-style C++ codebases.

## Build And Inspection Defaults

Makefiles are the default assumption. Inspect in this order:

1. `Makefile`, included `*.mk`, build scripts, documented targets.
2. Actual compile/link commands via `make -n`, verbose build output, existing logs, or `compile_commands.json`.
3. If clang tooling matters and `compile_commands.json` is absent, suggest the least invasive Makefile-compatible path: repo target if present, `bear -- make`, `compiledb make`, or captured verbose build output.
4. CMake only if the repo clearly uses it as the authoritative build graph.

Record compiler family, language standard, optimization/debug/sanitizer targets, warning policy, include paths, generated files, macros, external numerical libraries, link order, static libraries, OpenMP/MPI/CUDA/SYCL/Kokkos/BLAS/LAPACK usage, tests, and benchmarks.

Do not make clangd/clang-tidy claims without a compilation database or equivalent flags. Headers are often parsed through including translation units; missing macros and include paths create fake diagnostics.

Prefer `arguments` entries over shell `command` entries when available; they avoid shell quoting ambiguity. If only `command` is present, use a shell-aware parser before interpreting flags.

`json-compact` can be useful for LLM inspection after parsing entries into a tabular summary, but it is not a compile database parser and must not become canonical tooling input. Raw `compile_commands.json` compaction may save little because most bytes live in nested argument arrays. Better pattern: parse each entry into fields such as `file`, `compiler`, `std`, `opt`, `includes`, `defines`, `warnings`, `openmp`, `pic`, and `output`, then compact that projection for context.

## Downstream Host Repo Mode

If the user provides a working repo plus a downstream host repo, treat them differently:

- Working repo: candidate source for changes.
- Host repo: read-only evidence for build style, interface style, naming, data carriers, compile flags, and adoption constraints.
- Do not propose edits to the host repo unless the user explicitly scopes them.
- Compare the staged integration surface against host precedents before proposing API changes.
- Report mismatches as adoption friction, not as defects in the host repo.

## What Good Often Means Here

Scientific C++ often differs from mainstream application C++ in one sharp direction: memory throughput and compute throughput may dominate developer-convenience abstractions.

Common local patterns may be legitimate:

- flat arrays, spans/views, or `std::vector` instead of pointer-linked structures;
- structs and namespaces over deep object hierarchies;
- compile-time dispatch/templates in hot paths instead of virtual dispatch;
- no `new`/`delete` inside time loops;
- preallocated work buffers;
- explicit alignment, stride, AoS/SoA layout, and member ordering;
- `-fno-exceptions`, `-fno-rtti`, LTO, restrict qualifiers, SIMD pragmas, or platform/compiler-specific paths in solver/kernel code.

Do not cargo-cult these rules. Verify whether the local codebase actually uses them and whether the path is performance-critical. Standard library use is fine when local code accepts it and it does not violate ownership, allocation, ABI, or hot-path constraints.

## Improvement Lens

Prefer candidates that improve one of these without damaging the others:

- **Numerical correctness:** units, coordinate frames, tolerances, conservation laws, solver regimes, reproducibility, deterministic reductions where needed.
- **Memory behavior:** hidden copies, aliasing, contiguous layout, preallocation, alignment, cache locality, vectorization.
- **Interface clarity:** public headers state caller obligations, ownership, array extents, valid ranges, lifetime, error modes, and threading assumptions.
- **Build clarity:** Makefile targets express real workflows; generated artifacts and compiler flags are visible; `compile_commands.json` exists when code tools need it.
- **Verification:** tests hit public scientific behavior; regression data is versioned appropriately; profilers/benchmarks guard hot paths before optimization claims.
- **Portability:** compiler, platform, and dependency assumptions are explicit.

Common candidate types:

- **Public contract cleanup:** ownership, units, lifetimes, extents, error modes.
- **Host adoption friction:** staged interface differs from host Makefile/header idioms.
- **Numerical invariant visibility:** assumptions are present but not named or tested.
- **Hot-path safety:** hidden allocation, copy, dispatch, aliasing, or branch risk.
- **Build evidence gap:** Makefile target or compilation database does not support inspection.
- **Test ratchet gap:** behavior exists but no public-interface regression protects it.

## Anti-Patterns To Reject

- Virtual interfaces, factories, adapters, or dependency-injection scaffolding around one real numerical implementation.
- Abstractions that hide data layout, units, or hot-loop cost.
- "Modernizing" arrays into containers without checking ABI, allocation, caller layout, and vectorization.
- Splitting formula code until the reader must chase one scalar update across five files.
- Treating mocks as stronger evidence than deterministic scientific regression tests.
- Running clang tools without actual compile flags and believing the diagnostics.
- Imported style-guide changes that fight repo-local style without a correctness, safety, or reproducibility reason.

## External Anchors

Use these as background, not law:

- PETSc style and usage guidance: scientific C APIs treat names, documentation, developer notes, Fortran/interface constraints, and Makefile conventions as maintainability surface.
- Kokkos coding standards: performance-portable C++ should avoid syntactic noise and respect host/device annotation realities.
- JPL Power of Ten / JPL C / JSF AV C++: safety-critical subsets are useful for hot-path discipline, especially bounded control flow, allocation restrictions, and avoidance of runtime features.
- C++ Core Guidelines and SEI CERT C/C++: ownership, lifetime, contracts, resource safety, undefined behavior, buffers, and integer safety.
- Wilson et al., "Best Practices for Scientific Computing": automate workflows, make small changes, assert/test expected behavior, optimize after correctness, and document design/purpose.
- Memory-layout research such as LLAMA/AoS-vs-SoA work: layout is architecture in scientific codes, not an implementation detail.

When an external anchor conflicts with local style, state the conflict and prefer the minimal local-compatible fix unless correctness, safety, or reproducibility requires a stronger change.
