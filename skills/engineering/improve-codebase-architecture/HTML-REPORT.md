# HTML Report Format

Use this only when the user explicitly asks for a visual report. The normal output of this skill is the numbered candidate list in `SKILL.md`, not an HTML artifact.

Render the architectural review as a static HTML file in the OS temp directory. Prefer inline CSS for portability. Use Mermaid from a CDN only when the environment allows network access and graph layout is worth the dependency; otherwise use simple HTML/SVG diagrams. Mermaid handles graph-shaped diagrams reliably; hand-built divs and inline SVG handle mass diagrams, data-layout sketches, and before/after call-flow cross-sections.

## Scaffold

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title>Architecture review — {{repo name}}</title>
    <script type="module">
      import mermaid from "https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs";
      mermaid.initialize({ startOnLoad: true, theme: "neutral", securityLevel: "loose" });
    </script>
    <style>
      /* small custom layer for dashed seam lines, arrow heads, and callouts */
      body { margin: 0; background: #fafaf9; color: #0f172a; font-family: system-ui, sans-serif; }
      main { max-width: 72rem; margin: 0 auto; padding: 3rem 1.5rem; }
      section { margin-top: 2.5rem; }
      .files { font-family: ui-monospace, SFMono-Regular, Menlo, monospace; font-size: 0.875rem; }
      .module-label { font-size: 0.75rem; text-transform: uppercase; letter-spacing: 0.06em; }
      .diagram { border: 1px solid #cbd5e1; background: #fff; padding: 1rem; }
      .seam { stroke-dasharray: 4 4; }
      .leak { stroke: #dc2626; }
      .deep { background: linear-gradient(135deg, #0f172a, #1e293b); }
    </style>
  </head>
  <body>
    <main>
      <header>...</header>
      <section id="candidates">...</section>
      <section id="top-recommendation">...</section>
    </main>
  </body>
</html>
```

## Header

Repo name, date, git head if available, and a compact legend: solid box = module, dashed line = seam, red arrow = leakage, thick dark box = deep module, amber label = unresolved evidence. No introduction paragraph — straight into the candidates.

## Candidate card

The diagrams carry the weight. Prose is sparse, plain, and uses the glossary terms ([LANGUAGE.md](LANGUAGE.md)) without ceremony.

Each candidate is one `<article>`:

- **Title** — short, names the deepening (e.g. "Name the contact-clearance invariant").
- **Badge row** — recommendation strength (`Strong` = emerald, `Worth exploring` = amber, `Speculative` = slate), plus a tag for the dependency category (`in-process`, `local-substitutable`, `ports & adapters`, `mock`).
- **Files** — monospaced list, `.files`.
- **Before / After diagram** — the centrepiece. Two columns, side by side. See patterns below.
- **Problem** — one sentence. What hurts.
- **Solution** — one sentence. What changes.
- **Wins** — bullets, <=6 words each. e.g. "tests hit one interface", "units become explicit", "delete 4 shallow wrappers".
- **Haft callout** (if applicable) — one line with linked decision/problem/commission refs or an explicit "no linked refs found".

No paragraphs of explanation. If the diagram needs a paragraph to be understood, redraw the diagram.

## Diagram patterns

Pick the pattern that fits the candidate. Mix them. Don't make every diagram look the same — variety is part of the point.

### Mermaid graph (the workhorse for dependencies / call flow)

Use a Mermaid `flowchart` or `graph` when the point is "X calls Y calls Z, and look at the leak." Wrap it in a plain bordered block. Style with classDef to colour leakage edges red and the deep module dark. Sequence diagrams work well for "before: 6 caller obligations; after: 1 interface contract."

```html
<div class="diagram">
  <pre class="mermaid">
    flowchart LR
      A[Contact Step] --> B[Projection]
      B --> C[Clearance Check]
      C -.leak.-> D[GUI Radius Assumption]
      classDef leak stroke:#dc2626,stroke-width:2px;
      class C,D leak
  </pre>
</div>
```

### Hand-built boxes-and-arrows (when Mermaid's layout fights you)

Modules as `<div>`s with borders and labels. Arrows as inline SVG `<line>` or `<path>` elements positioned absolutely over a relative container. Reach for this when you want the "after" diagram to feel like one thick-bordered deep module with greyed-out internals — Mermaid won't render that with the right weight.

### Cross-section (good for layered shallowness)

Stack horizontal bands (`h-12 border-l-4`) to show layers a call passes through. Before: 6 thin layers each doing nothing. After: 1 thick band labelled with the consolidated responsibility.

### Mass diagram (good for "interface as wide as implementation")

Two rectangles per module — one for interface surface area, one for implementation. Before: interface rectangle is nearly as tall as the implementation rectangle (shallow). After: interface rectangle is short, implementation rectangle is tall (deep).

### Call-graph collapse

Before: a tree of function calls rendered as nested boxes. After: the same tree collapsed into one box, with the now-internal calls shown faded inside it.

## Style guidance

- Lean technical, not corporate-dashboard. Generous whitespace only where it helps diagrams read.
- Colour sparingly: one accent plus red for leakage and amber for unresolved evidence.
- Keep diagrams ~320px tall so before/after sits comfortably side by side without scrolling.
- Use `.module-label` for module labels inside diagrams — they should read as schematic, not as UI.
- The only optional script is the Mermaid ESM import. The report is otherwise static — no app code, no interactivity beyond Mermaid's own rendering.

## Top recommendation section

One larger card. Candidate name, one sentence on why, anchor link to its card. That's it.

## Tone

Plain English, concise — but the architectural nouns and verbs come straight from [LANGUAGE.md](LANGUAGE.md). Concision is not an excuse to drift.

**Use exactly:** module, interface, implementation, depth, deep, shallow, seam, adapter, leverage, locality.

**Never substitute:** component, service, unit (for module) · API, signature (for interface) · boundary (for seam) · layer, wrapper (for module, when you mean module).

**Phrasings that fit the style:**

- "Contact projection module is shallow — interface nearly matches the implementation."
- "Radius semantics leak across the seam."
- "Deepen: one interface, one place to test."
- "Two adapters justify the seam: generated kernel in prod, fixture-backed CPU path in tests."

**Wins bullets** name the gain in glossary terms: *"locality: bugs concentrate in one module"*, *"leverage: one interface, N call sites"*, *"interface shrinks; implementation absorbs the wrappers"*. Don't write *"easier to maintain"* or *"cleaner code"* — those terms aren't in the glossary and don't earn their place.

No hedging, no throat-clearing, no "it's worth noting that…". If a sentence could be a bullet, make it a bullet. If a bullet could be cut, cut it. If a term isn't in [LANGUAGE.md](LANGUAGE.md), reach for one that is before inventing a new one.
