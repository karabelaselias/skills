# Interface Design

When the user wants to compare alternative interfaces for a chosen deepening candidate, route through this. Based on "Design It Twice" (Ousterhout) — your first idea is unlikely to be the best.

Uses the vocabulary in [LANGUAGE.md](LANGUAGE.md) — **module**, **interface**, **seam**, **adapter**, **leverage**.

There are two paths: the **haft-native path** (preferred when the project uses haft) and the **fallback path** (any other project). Pick based on what's available.

## Path A — Haft-native (preferred when haft is present)

Haft already has a comparison protocol designed exactly for this kind of decision: `/h-frame → /h-char → /h-explore → /h-compare → /h-decide`. Use it. Don't reinvent the comparison machinery.

The skill's job is to prepare the brief; haft owns the structured comparison.

### 1. Frame the problem space

Write a problem-frame brief for the chosen candidate covering:

- **Signal** — what observation triggered this (the friction from the candidate, in the codebase's actual terms).
- **Constraints** — what any new interface MUST satisfy (ABI, dependencies, callers' existing expectations, performance budgets).
- **Dependency category** — in-process / local-substitutable / remote-owned / true-external (see [DEEPENING.md](DEEPENING.md)).
- **Acceptance** — what "solved" looks like measurably (callers reduced from N to M, this test now expressible at the interface, drift on these files resolves).
- **Reversibility** — tactical / standard / deep.
- **A rough illustrative sketch** — not a proposal, just a way to make the constraints concrete.

Show the brief to the user.

### 2. Hand off to haft

Suggest `/h-frame` to record the problem (the brief above maps directly: signal, constraints, acceptance, reversibility). Then `/h-char` to declare the comparison dimensions before generating variants — typical dimensions for an interface decision:

- **Depth** (leverage at the interface)
- **Locality** (where change concentrates)
- **Seam placement** (where the interface lives — internal seam vs external port)
- **Test surface** (what a test must mock or substitute)
- **Build/ABI cost** (recompile cascade, header pollution, ABI stability)
- **Caller migration cost** (how invasive is the switch for existing call sites)

Mark each as `constraint` (hard limit), `target` (optimize), or `observation` (watch but don't optimize for — Anti-Goodhart). For C/C++ specifically, ABI stability is often a constraint; build-graph cost is typically a target.

Then `/h-explore` to generate variants. Suggest seed directions so the variants differ *in kind*, not degree:

- **Minimize the interface** — 1–3 entry points. Maximize leverage per entry point. PIMPL or a single facade function.
- **Maximize flexibility** — many use cases, extension points, multiple seams.
- **Optimize the common caller** — make the default case trivial; advanced uses pay extra.
- **Cross-seam (Ports & Adapters)** — if the dependency category is remote-owned or true-external. One port, prod and test adapters.
- **Stepping stone** — a simpler interim shape that doesn't solve everything but unblocks the test surface and leaves room to grow.

Each variant should carry its **weakest link** (what bounds its quality — e.g., "leverage is high but compile times grow ~25% from header changes"; "tests get clean but vtable costs ~3ns per call on the hot path").

`/h-compare` produces the Pareto front against the dimensions from `/h-char`. The user — not the agent — picks at the Choose → Execute boundary. `/h-decide` records the contract.

### 3. Cross-project precedents

Before or during exploration, call `haft_query(action="similar", ...)` for the problem area. Past interface decisions in this language (CL2) or others (CL1) often surface a variant the team would otherwise miss. Include the strongest precedents in the brief, tagged with their congruence level so the user can weigh transferability.

### 4. The skill's role at each phase

- **Frame** — prepare the brief; surface drift / linked decisions from the pre-check.
- **Characterize** — propose dimensions; the user adjusts.
- **Explore** — propose variant directions; let `/h-explore` generate them. If your harness has parallel sub-agents, use them for variant generation. Otherwise, generate sequentially. Either way, the variants get persisted by `/h-explore`.
- **Compare** — present each variant in prose with its weakest link, then let `/h-compare` produce the Pareto computation and parity check.
- **Decide** — out of scope. The user decides.

Be opinionated in the prose: name your favourite and why. The user wants a strong read, not a menu. The Transformer Mandate still applies — the user chooses, but they choose better when they know what you'd pick.

## Path B — Fallback (no haft)

Same shape, less machinery. The discipline is the same; you just don't have the artifact graph.

### 1. Frame

Same brief as Path A — signal, constraints, dependency category, acceptance, reversibility, sketch. Show to the user.

### 2. Generate variants — design twice

Generate 3+ genuinely distinct variants for the deepened module's interface. They must differ in kind, not degree. Use the same seed directions as Path A (minimize / maximize flexibility / optimize common caller / cross-seam / stepping stone).

If your harness has a sub-agent / parallel-execution helper, fan out variant generation. Otherwise, do them sequentially. Each variant produces:

1. **Interface** — types, methods, params; plus invariants, ordering, error modes.
2. **Usage example** — how callers use it.
3. **Implementation behind the seam** — what the module hides.
4. **Dependency strategy and adapters** — see [DEEPENING.md](DEEPENING.md).
5. **Trade-offs** — where leverage is high, where it's thin. Name the **weakest link**.

### 3. Compare and recommend

Present designs sequentially so the user can absorb each, then compare them in prose. Contrast by **depth** (leverage at the interface), **locality** (where change concentrates), **seam placement**, **test surface**, and **build/ABI cost** if relevant.

After comparing, give your own recommendation: which design you think is strongest and why. If elements from different designs would combine well, propose a hybrid. Be opinionated.

The user chooses. The skill stops there — implementation is the user's call.
