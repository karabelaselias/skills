# Deepening

How to deepen a cluster of shallow modules safely, given its dependencies. Assumes the vocabulary in [LANGUAGE.md](LANGUAGE.md) — **module**, **interface**, **seam**, **adapter**.

The goal is always the same: **leverage at the interface, locality for maintainers, the simplest mechanism that gets you both.** Cleverness is not depth.

## Dependency categories

When assessing a candidate for deepening, classify its dependencies. The category determines how the deepened module is tested across its seam.

### 1. In-process

Pure computation, in-memory state, no I/O. Always deepenable — merge the modules and test through the new interface directly. No adapter needed.

### 2. Local-substitutable

Dependencies that have local test stand-ins (PGLite for Postgres, in-memory filesystem, ramdisk, fake clock). Deepenable if the stand-in exists. The deepened module is tested with the stand-in running in the test suite. The seam is internal; no port at the module's external interface.

### 3. Remote but owned (Ports & Adapters)

Your own services across a network or process boundary (microservices, internal APIs, IPC). Define a **port** (interface) at the seam. The deep module owns the logic; the transport is injected as an **adapter**. Tests use an in-memory adapter. Production uses an HTTP / gRPC / queue / shared-memory / pipe adapter.

Recommendation shape: *"Define a port at the seam, implement an HTTP adapter for production and an in-memory adapter for testing, so the logic sits in one deep module even though it's deployed across a network."*

### 4. True external (Mock)

Third-party services and resources you don't control — Stripe, Twilio, vendor SDKs, hardware peripherals, kernel APIs that aren't easily fakeable. The deepened module takes the external dependency as an injected port; tests provide a mock adapter.

## Language conventions

Different languages put the interface in different places. Pick the seam mechanism that matches the language's grain — and the simplest one that delivers the locality and leverage. The seams below are tools; **don't reach for the cleverest one because it's available**.

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

#### Seam mechanisms

| Mechanism | Reach for it when | Adapter pattern | Notes |
|---|---|---|---|
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

#### Build-graph friction

A deepening that looks clean in source can be expensive at the build level:

- Adding a member to a header struct cascades a recompile across every translation unit that includes it.
- Promoting a `.c`-private function to the header pulls its dependencies into every includer.
- Introducing a virtual interface forces clients of the concrete type to include the abstract base header.
- Changing a public type's layout breaks ABI for any consumer that links against you.

Surface this as a trade-off in candidates that touch headers, not as a blocker. PIMPL and link-time seams exist precisely to manage this; flag when they'd help.

## Seam discipline

- **One adapter = hypothetical seam. Two adapters = real seam.** Don't introduce a port, virtual interface, or function-pointer struct unless at least two adapters are justified (typically production + test). A single-adapter seam is just indirection.
- **Internal seams vs external seams.** A deep module can have internal seams (private to its implementation, used by its own tests) as well as the external seam at its interface. Don't expose internal seams through the interface just because tests use them.

## Testing strategy: replace, don't layer

- Old unit tests on shallow modules become waste once tests at the deepened module's interface exist — delete them.
- Write new tests at the deepened module's interface. The **interface is the test surface**.
- Tests assert on observable outcomes through the interface, not internal state.
- Tests should survive internal refactors — they describe behaviour, not implementation. If a test has to change when the implementation changes, it's testing past the interface.
- C/C++ test stand-ins commonly used: fff (fake function framework, C), GoogleMock and trompeloeil (C++), link-time fakes for I/O wrappers. Use the simplest one that lets the test exercise the interface.
