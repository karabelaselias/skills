# Deepening

How to deepen a cluster of shallow modules safely, given its dependencies. Assumes the vocabulary in [LANGUAGE.md](LANGUAGE.md) — **module**, **interface**, **seam**, **adapter**.

The goal is always the same: **leverage at the interface, locality for maintainers, the simplest mechanism that gets you both.** Cleverness is not depth.

## Deepening gates

Before recommending a deepening, prove it earned its cost. A candidate should satisfy at least one gate:

- **Interface contract improves** - invariants, ownership, extents, layout, error modes, or threading assumptions move from caller folklore into one interface.
- **Caller obligations shrink** - N call sites stop repeating the same setup, checks, conversions, or cleanup.
- **Verification improves** - a deterministic public-interface test can now cover behavior that was previously spread across internals.
- **Variation is real** - two or more backends, data sources, kernels, or test stand-ins exist now, not in a speculative future.
- **Build/ABI risk is justified** - any public-header, object-layout, link-order, or generated-code change has a local reason and a rollback path.

If none of these are true, the module is probably not shallow enough to fix. Leave the code alone or recommend a note-level contract/comment/test ratchet instead.

## Dependency categories

When assessing a candidate for deepening, classify its dependencies. The category determines how the deepened module is tested across its seam.

### 1. In-process

Pure computation, in-memory state, no I/O. Always deepenable — merge the modules and test through the new interface directly. No adapter needed.

### 2. Local-substitutable

Dependencies that have local deterministic stand-ins: fixture meshes, small matrices, seeded random inputs, golden solver traces, in-memory files, fake clocks, ramdisks, or local-only tool invocations. Deepenable if the stand-in preserves the scientific contract being tested. The seam is internal; no port at the module's external interface.

### 3. Remote but owned (Ports & Adapters)

Owned dependencies across a process, library, accelerator, or generated-code boundary: internal meshing tools, solver subprocesses, IPC workers, shared libraries, GPU kernels, MPI/OpenMP coordination layers, or generated kernels. Define a **port** (interface) only when the dependency really varies or must be substituted. The deep module owns the logic; the transport or execution backend is an **adapter**. Tests use the lightest deterministic adapter that exercises the same contract. Production may use a shared-library call, pipe, generated source, accelerator launch, or process boundary.

Recommendation shape: *"Keep the physical update rule in one deep module; put a narrow backend seam at the generated-kernel boundary, with a fixture-backed CPU adapter for tests and the generated accelerator path in production."*

### 4. True external (Mock)

Third-party libraries and resources you don't control: BLAS/LAPACK/SuiteSparse/PETSc/Kokkos, proprietary meshers, vendor SDKs, hardware peripherals, GPU runtimes, kernel APIs, or lab/device interfaces that are not easily fakeable. The deepened module takes the external dependency through the narrowest useful interface; tests use deterministic fixtures, link-time fakes, or mocks only when fixture-based regression cannot cover the behavior.

## Language conventions

Different languages put the interface in different places. Pick the seam mechanism that matches the language's grain and the simplest one that delivers locality and leverage. This skill defaults to scientific C/C++; use the other language notes only for mixed repos, tooling glue, or non-C/C++ targets. The seams below are tools; **don't reach for the cleverest one because it's available**.

### TypeScript / JavaScript

Module = file or package; interface = exported types and functions plus invariants you've documented. Common seams: structural-typed interfaces, dependency injection via parameters, factory functions returning closures. Test stand-ins: in-memory implementations, MSW for HTTP, faker libraries.

### Go

Module = package; interface = exported identifiers plus the `interface{}` types they consume. Go's small interfaces ("accept interfaces, return structs") are deep by design. Test stand-ins are usually hand-written fakes in a `fake` subpackage.

### Python

Module = package or class; interface = public methods and the duck-typed protocol they consume. Common seams: dependency injection via constructor, `Protocol` types, monkeypatching at module boundaries (acceptable in tests, not in production code).

### Rust

Module = crate or module; interface = public types and the traits they implement / consume. Trait objects (`dyn Trait`) are runtime ports; generic bounds are compile-time ports. Test stand-ins via `cfg(test)` modules or feature-gated implementations.

### C / C++

The interface is **literally the header file** — declarations, types, and the contracts written in or alongside them. This makes C/C++ unusual: the seam location is forced by the language, but the seam *mechanism* is your choice.

The principle from the C++ Core Guidelines applies throughout: **express ideas directly, express intent, prefer compile-time checking, don't pretend.** A deep interface in C/C++:

- Has precise types (I.4) — no `void*`, no untyped buffers without a paired length, no `int` flags that should be enums.
- States preconditions and postconditions where they matter (I.5–I.8) — `assert`s, `Expects`/`Ensures`, or comments where the language doesn't help.
- Doesn't transfer ownership through raw pointers (I.11) — owning APIs use `unique_ptr` (C++) or paired alloc/free functions with documented ownership (C).
- Doesn't pass arrays as a single pointer (I.13) — pair pointer + length, or use `span<T>` / a struct that bundles both.
- Hides implementation details: PIMPL (I.27) when ABI stability or recompile cascades matter; opaque-pointer-and-`create`/`destroy` in C; private-section structs accessed only via the header's functions.

For scientific C/C++, the contract often needs more than type signatures. Check whether the interface states:

- units, coordinate frames, valid ranges, tolerances, and sentinel values;
- ownership, borrow duration, nullability, allocator/free pairing, and restore/release pairing;
- array extent, rank, stride, layout, alignment, padding, and AoS/SoA expectation;
- memory space, execution space, host/device accessibility, and whether copies are explicit or hidden;
- initialization state, reproducibility assumptions, and whether reductions must be deterministic;
- collectivity, thread-safety, reentrancy, callback lifetime, and `void*` context ownership;
- failure mode: return code, assertion, exception-free status path, undefined input, or documented no-op.

#### Seam mechanisms

| Mechanism | Reach for it when | Adapter pattern | Notes |
|---|---|---|---|
| **Buffer/view descriptor** (`span`, pointer+extent+stride, unmanaged view) | The seam is data access, not behavior. | Usually no adapter; one descriptor type states shape and borrow contract. | Carry extent/layout/stride/alignment/memory-space facts. Do not smuggle ownership or hidden deep copies through a "view". |
| **Callback + context** | Caller supplies domain equations, residuals, forcing functions, boundary conditions, or event hooks. | Function pointer plus typed or documented context; optional ops table if callbacks cluster. | State context lifetime, mutability, thread/collective behavior, and copy/share semantics across solver levels. |
| **Function-pointer struct** ("ops table" / "vtbl in C") | Default for runtime polymorphism in C, or C-style C++. Two+ implementations needed. | One struct literal per implementation; assign at construction. | Simplest, most readable port. Use this before reaching for virtuals if C is enough. |
| **Virtual interface** (abstract base class) | C++ with managed object lifetimes, multiple impls, and clear ownership. | One subclass per adapter. | Keep the base **empty** (I.25) — no state, no defaulted methods that subclasses depend on. State in a base = inheritance trap. |
| **PIMPL** | Stable ABI required, OR you want to break a recompile cascade across many translation units. | Single concrete impl, hidden behind the pointer. | This is a depth mechanism, not a real seam (one adapter = hypothetical seam). Don't reach for it without an ABI or build-time reason. Costs: one indirection per call, one heap alloc per object, more files. |
| **Templated policy / CRTP** | Compile-time polymorphism, zero runtime cost, 2+ policies you actually use. | One policy type per adapter. | Watch header bloat and error-message quality. Don't use to "make it generic for the future." |
| **Link-time substitution** | Test fakes for I/O. Production `.c` and fake `.c` satisfy the same headers. | Build system links one or the other. | Cleanest seam for OS-call wrappers (file, socket, time). No runtime indirection. |
| **Weak symbols** | Default implementation with optional override (drivers, plug-in points, RTOS HAL). | Default symbol always linked; override symbol wins when provided. | Toolchain-specific (`__attribute__((weak))`). Useful, easy to misuse. |
| **`#ifdef` / preprocessor** | Hardware variants where there are *no* two adapters at runtime — different code is compiled in entirely. | Different sources / blocks compiled per target. | Last resort. Hostile to readers and to tests. If anything else works, use that. |

#### Anti-patterns to flag, not propose

- **One-subclass abstract base class.** "I'll make `IFoo` so we can swap it later." With one adapter the seam is hypothetical; you've added indirection for nothing. Plus, if the base has state, it violates I.25 and creates an inheritance trap.
- **Reflexive `shared_ptr<everything>`.** I.11 is about *ownership transfer at interfaces*, not "every object should live in a shared_ptr." Aliased ownership has real costs (atomic refcount, heap, lifetime confusion). Reach for stack values, `unique_ptr`, or borrowed references first.
- **PIMPL on every class.** PIMPL pays for itself when ABI stability or build-time decoupling are at stake. Otherwise you've added a heap allocation and an indirection per call to hide implementation details from no one.
- **Templates / virtuals where a function-pointer struct in plain C reads better.** C inside C++ is fine when it's clearer. P.1 / P.3 are about expressing intent; the most "modern C++" mechanism is not automatically the most legible.
- **"Modernising" raw arrays into `vector<T>`/`array<T,N>` at the interface for no reason.** If callers want `vector`, take `vector` (or `span<T>`); if they want a fixed buffer, take pointer + length. The interface should match how callers actually own data.
- **A "view" that hides a copy, layout conversion, allocator, or device transfer.** If data movement happens, make it explicit in the interface or keep it inside one deep module with tests.
- **Callback seams with undocumented context lifetime.** A `void* ctx` is tolerable in C-style scientific code only if ownership, mutability, and call ordering are documented.

#### Build-graph friction

A deepening that looks clean in source can be expensive at the build level:

- Adding a member to a header struct cascades a recompile across every translation unit that includes it.
- Promoting a `.c`-private function to the header pulls its dependencies into every includer.
- Introducing a virtual interface forces clients of the concrete type to include the abstract base header.
- Changing a public type's layout breaks ABI for any consumer that links against you.
- Moving logic between translation units can disturb Makefile link order, generated-file rules, symbol visibility, or static-library grouping.

Surface this as a trade-off in candidates that touch headers, not as a blocker. PIMPL and link-time seams exist precisely to manage this; flag when they'd help.

## Seam discipline

- **One adapter = hypothetical seam. Two adapters = real seam.** Don't introduce a port, virtual interface, or function-pointer struct unless at least two adapters are justified (typically production + test). A single-adapter seam is just indirection.
- **Internal seams vs external seams.** A deep module can have internal seams (private to its implementation, used by its own tests) as well as the external seam at its interface. Don't expose internal seams through the interface just because tests use them.
- In scientific C/C++, default to header contracts, file-local helpers, link-time substitution, compile-time policies, or fixture-backed tests before inventing runtime ports. Runtime indirection needs a concrete reason: multiple backends, test substitution that cannot be done at link time, ABI stability, or an actual process/device seam.

## Testing strategy: replace, don't layer

- Old unit tests on shallow modules become waste only after tests at the deepened module's interface cover the same behavior. Until then, keep them as scaffolding. Delete only with evidence, not because the architecture diagram looks tidier.
- Write new tests at the deepened module's interface. The **interface is the test surface**.
- Tests assert on observable outcomes through the interface, not internal state.
- Tests should survive internal refactors — they describe behaviour, not implementation. If a test has to change when the implementation changes, it's testing past the interface.
- C/C++ test stand-ins commonly used: fff (fake function framework, C), GoogleMock and trompeloeil (C++), link-time fakes for I/O wrappers. Use the simplest one that lets the test exercise the interface.
- Prefer deterministic scientific regression tests over mocks when fixtures can capture the invariant: known mesh topology, conservation check, tolerance-bounded residual, reproducible solver trace, or public-header smoke test.
- Add assertions at the interface for cheap invariants and tests for the expensive ones. Assertions are not a replacement for regression tests; they make invalid states fail closer to the caller.
