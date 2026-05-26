---
name: to-prd
description: Synthesize current conversation and repo context into a Product Requirements Document, design brief, or engineering requirements brief for downstream planning, issue decomposition, or Haft/commission framing. Use when the user wants to turn messy intent into a durable requirements artifact, especially before creating implementation issues or handoff work.
---

# To PRD

Turn current conversation context and codebase understanding into a durable requirements artifact. Do not treat this skill as a decision engine or execution planner. Its job is to preserve intent, requirements, constraints, assumptions, and open questions so the next rotary can work from clean input.

Default to producing a draft in chat or a file. Publish to an issue tracker only when the user explicitly asks for publication or the local workflow instructions require it.

## Process

### 1. Gather Authority

Work from existing conversation context first. If the user passes an issue, plan, PRD, URL, or file path, read it before drafting.

When repo context matters, inspect the local authority chain:

- repo-local instructions such as `AGENTS.md`
- `.haft/` decisions, problems, plans, notes, or commission artifacts when present
- `CONTEXT.md`, ADRs, architecture docs, or domain glossaries
- build/test conventions for the affected area

Use the project's domain vocabulary. If existing decisions contradict the requested direction, surface the conflict instead of smoothing it over.

### 2. Identify Existing Comparables

Look for the current system, comparable products, previous features, prior issues, tests, examples, or workflows that show how the problem is handled today.

For each useful comparable, capture:

- what exists
- how it is used
- where it fails
- what requirement that failure implies

Do not over-research. Stop when you have enough evidence to state requirements and uncertainty honestly.

### 3. Extract Requirements

Requirements are not implementation decisions. A requirement states a characteristic the solution must meet to count as successful.

For each requirement, include:

- **Requirement** - the characteristic the design must meet
- **Role** - `constraint`, `target`, or `observation`
- **Needed because** - why it belongs in scope
- **Feasibility** - known feasibility evidence or risk
- **Verification** - how a later agent or human can check it
- **Source** - conversation, repo artifact, issue, test, comparable, or explicit assumption
- **Tradeoffs** - conflicts with other requirements, if known

Apply three filters:

- If a requirement is not needed to solve the problem, leave it out.
- If feasibility is unknown, mark it as uncertain instead of pretending it is settled.
- If requirements conflict, name the tradeoff and leave the decision open unless the user already resolved it.

### 4. Choose The Artifact Shape

Use the shape that fits the work:

- **Product PRD** - product-facing user behavior, UX, or feature workflows.
- **Engineering Requirements Brief** - infrastructure, libraries, APIs, tools, scientific code, C/C++, runtime workflows, or internal platform work.
- **Haft Framing Brief** - non-trivial decision work that needs parity comparison, evidence, invariants, or WorkCommission scoping.

Do not force fake user stories onto engineering work. Use user stories only when they clarify real product actors and outcomes.

### 5. Draft The Brief

Use this template. Omit sections that are genuinely irrelevant, but preserve uncertainty and open decisions.

<brief-template>

## Problem / Intent

State the problem in one or two paragraphs. Prefer: `[who] needs [what] because [why]`.

## Target User / Operator

Who uses, maintains, operates, or depends on the result.

## Current System / Comparables

- What exists:
- How it is used:
- Where it fails:
- Requirements implied:

## Requirements

List requirements using the fields from step 3. Keep them specific and verifiable.

## Proposed Shape

- Areas or modules likely affected:
- Interfaces likely involved:
- Candidate paths or ownership boundaries:
- Out of scope paths or behaviors:

For scientific or performance-sensitive C/C++ work, preserve local style, data layout, numerical semantics, ABI, build reality, and hot-path constraints. Do not assume that extracting a "deep module" is better than explicit loops, flat data, or simple functions.

## Decisions

### Known Decisions

Decisions already made by the user, repo artifacts, ADRs, or Haft records. Include source/provenance.

### Open Decisions

Choices that still require the user, a decision record, or a comparison process. Do not launder these into requirements.

### Assumptions

Assumptions that are acceptable for drafting but must be checked before execution.

### Non-Decisions

Important things this brief intentionally does not settle.

## Testing / Evidence

- Behavioral checks:
- Regression tests:
- Build/lint/typecheck commands:
- Manual or measurement evidence:
- Evidence freshness risks:

## Handoff

- Suggested next rotary: `to-issues`, Haft frame/decision, WorkCommission, manual review, or other
- Ready for issue decomposition: yes/no, with reason
- Ready for Haft framing: yes/no, with reason
- Ready for decision: yes/no, with reason
- Ready for commission/execution: yes/no, with reason

## Further Notes

Any remaining context that helps the next agent avoid rediscovery.

</brief-template>

## Publishing

Publish only after the user approves the draft or explicitly asks for publication. When publishing, follow the local issue tracker and triage label vocabulary. If no tracker conventions are available, stop with the draft and say what information is missing.
