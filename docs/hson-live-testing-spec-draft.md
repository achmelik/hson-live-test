# hson-live real-world testing spec

## Package context

hson-live provides:
- Transformers between JSON, HTML, XML, SVG, and HSON via a shared node IR (HsonNode), claimed lossless and reversible across n round-trips
- A LiveTree extension that projects a live DOM from the node graph and updates the DOM in real time on graph changes
- A diagnostics suite

Details:
- Version: 2.1.0 (latest at time of spec)
- Author: neutralica @ terminal_gothic
- License: Parity-7.0.0
- Runtime deps: dompurify ^3.3.0
- ESM only
- Subpath exports: root, `/hson`, `/diagnostics`, `/types`
- Source: npm (`npm install hson-live`)
- Compiled `dist/` ships. `src/` does not. README references `/src/docs` which is not in the tarball.
- Demo site: terminalgothic.com

Author-stated status: experimental. Transformation core called stable. Surrounding APIs evolving. Not recommended for untrusted HTML or security-critical production use.

## Testing intent

The author's demo is a tech demo (animations, SVG cursor follower) and does not exercise real-world application patterns. This suite tests hson-live against patterns common in production applications to surface gaps between claims and actual behavior.

Each test is independent. There is no shared metric format, runtime, or report shape across tests.

## Architecture

### Backend
PHP. No build step required for backend. Backend produces responses (HSON, JSON, or other structured data) that the frontend consumes.

### Frontend
hson-live, plus libraries as needed per test.

### Websockets and other non-default behavior
Real websockets are NOT available because PHP does not support them by default. Any test referencing websockets, server-sent events, or any behavior PHP does not natively support must be mocked. Mocking approaches are open: the agent may have the frontend call a PHP endpoint that returns a payload simulating what the unsupported transport would have sent, may have frontend-only simulation, or any other approach. The agent does not need to determine HOW to use PHP. It must be clear in wording when behavior is mocked rather than real.

The point of any "streaming" or "websocket" test is to exercise the data-streaming-into-the-application pattern, not the wire protocol.

### Code layout
Each test has its own folder for code. Layout within the folder is the agent's decision.

## Universal guidelines

### Scope rules
- Describe WHAT is being tested and measured. Do not describe HOW to implement it.
- Do not write test code, fixtures, or datasets.
- Name a specific technology only when that technology IS what is being tested.
- If a fixture is required (e.g. "10k record dataset"), specify what it must contain. Do not generate it.
- Flag ambiguity. Do not guess.

### Reporting
- Reports must be accessible from a web-based frontend.
- Mechanisms are open. Examples: localStorage, JSON files consumed by a browser, an embedded HTML report page, hybrid approaches.
- Priority is in-browser testing. Secondary approaches (e.g. headless runs across many iterations) may be proposed where they add measurable value beyond the in-browser test. Not required.
- No shared report contract across tests. Each agent defines its own report shape based on what the test produces. The shape is part of the agent's spec output.

### Voice
- Concise. No filler, no adjectives, no dramatization.
- Specific. "Test thoroughly" or "ensure quality" is not acceptable.
- Direct.

## Required output structure per test

Each test spec must contain these sections, in this order:

1. **Test ID and name**
2. **What is being tested** (capability, performance, interaction, integration, maintainability, scalability, etc.)
3. **How it will be tested** (high-level approach: pages, builds, instrumentation, fixtures required)
4. **Likely failure modes and how they will be detected** (specific failure scenarios with specific detection methods)
5. **Measurement** (metrics, thresholds where applicable)
6. **Report shape** (what data is produced, where it is stored, how it reaches a browser)
7. **Out of scope** (explicit list of related concerns this test does not address)
8. **Tech stack required** (only when the tech is the test; otherwise "none specified")
9. **Mocking notes** (only if the test references non-default PHP behavior such as websockets)
10. **Assumptions and open questions** (anything decided without input, anything needing user review)

---

## Tests

### Test 01: Server returns HSON

**Context:** An interface requires a call to a backend service. The backend (PHP) returns HSON. The frontend renders the HSON to the page using hson-live.

**Constraint:** Response is HSON only. Not JSON. Not hybrid.

**Tests:** hson-live's ability to consume HSON delivered over the wire and render it correctly. The end-to-end path from a backend HSON producer to a rendered page.

---

### Test 02: Server returns JSON

**Context:** An interface requires a call to a backend service. The backend (PHP) returns JSON. hson-live converts and renders the JSON. Per the README, hson-live should translate from JSON natively without requiring HSON.

**Constraint:** Response is JSON only. Not HSON. Not hybrid.

**Tests:** hson-live's claim that JSON can be converted to renderable structure without an HSON intermediate produced by the server.

---

### Test 03: Client-side pagination

**Context:** Server returns raw data with no rendering structure. The client handles pagination and rendering updates on the client side using hson-live.

**Constraint:** All pagination state and update logic is client side.

---

### Test 04: Component templates (marketing site)

**Context:** Determines whether hson-live patterns support breaking files into reusable components. The README does not describe component decomposition patterns; this test discovers what is achievable.

**Required components:** header, footer, hero banner, three-card display.

**Site context:** Basic marketing site composed from the four components above.

**Constraint:** Each component has its own HTML and its own CSS. CSS must use hson-live's scoped CSS (QUID-based scoping per the README).

---

### Test 05: Canvas (driven by hson and LiveTree)

**Context:** An HTML canvas whose interactivity is driven by hson and LiveTree.

**Constraint:** Canvas state must be driven by hson and LiveTree. A canvas merely coexisting on the page alongside hson-live-rendered content is NOT what this tests. Coexistence is likely to work; the question is integration and capability.

**Tests:** Whether hson and LiveTree can drive a canvas (state changes, redraws, interaction handling).

**User note:** Likely failure point.

---

### Test 06: Chart library integration

**Context:** A page using a popular chart library alongside hson-live, rendering a variety of chart types. hson-live needs to play well with other libraries.

**Constraint on library choice:** Popular, currently maintained. Specific library is up to the agent.

**Constraint on variety:** Multiple chart types must be exercised. Picking one chart type does not satisfy the test.

**Tests:** How well hson-live coexists with a third-party library that owns its own DOM region (e.g. a chart that renders into a canvas or contained div). The library is not ingested by LiveTree; it operates alongside.

---

### Test 07: Performance A/B (large-data business application)

**Context:** Business applications can be heavy-handed with data. Two pages render the same large dataset, one with hson-live and one with vanilla JS, so performance can be directly compared.

**Domain framing:** A file interface for client contracts. 10k record dataset stubbed in.

**Per-page requirements:**
- Renders the 10k dataset
- Client-side pagination
- Client-side filtering
- Per-item rendering complex enough to stress rendering performance
- Performance instrumentation built into the page (the page reports its own metrics)

**Per-item rendering (ILLUSTRATIVE, NOT AUTHORITATIVE):** Agent may expand or contract these as needed to achieve sufficient rendering complexity:
- Client name
- Status
- Previous contract value
- Current contract value
- Contract value delta enumerated to a yearly basis
- Navigation icons for: review, close, contact client, contact internal legal rep, contact internal stakeholder rep, export audit trail

The above set is illustrative of the kind of per-item complexity intended to stress rendering. The agent can adjust the specific fields and icons as long as the per-item complexity remains comparable.

**Required metrics (illustrative, not exhaustive):** time to initial render, time to update on pagination, time to update on filter. Agent identifies any other relevant metrics.

**Constraint:** Both pages must be implemented and measured equivalently so the comparison is meaningful. Same dataset, same fixture, same interaction pattern, same metrics.

---

### Test 08: JS-free pages

**Context:** Pages where interactivity is driven entirely by HSON structures, with no client-authored JavaScript. User interactions trigger calls to the server. The server (PHP) returns updated HSON. The HSON itself encodes handler callbacks and rendering, and hson-live applies them on receipt.

**Tests:** Whether a "no-JS-on-the-client" application pattern is achievable with hson-live, and where it breaks down.

**User note:** Expected failure case. The intent is to document where the approach breaks down and the boundaries of what is achievable without client-authored JS.

---

### Test 09: Render flicker on large updates

**Context:** Triggers a large-scale content replacement via the LiveTree node graph and observes whether the page flickers, FOUCs, or shifts during the update.

**Background:** Per the README, LiveTree's `graft()` "replaces contents of document.body". Large updates via the node graph may produce a visible re-render or layout shift. This test characterizes that behavior.

**How the update is triggered does not matter.** A JS function that replaces page content via the node graph is acceptable. A server-returned payload that triggers a graft is acceptable. A client-only simulation is acceptable. The trigger mechanism is the agent's choice; the focus is the visible result of a large node-graph-driven update.

**Tests:** Whether large-scale node-graph-driven updates produce visible flicker, FOUC, or layout shift, and under what conditions.

**Open:** Detection and quantification approach is up to the agent.

---

### Test 10: Data streaming in and out of the application

**Context:** A page receiving real-time-style updates as a stream of payloads, with updates flowing into LiveTree.

**Patterns to exercise (multiple required):**
- Chat (REQUIRED)
- Additional patterns chosen by the agent (e.g. ticker, presence indicator, collaborative cursor, live notifications)

**Payload shape variety:** The mocked stream must include multiple payload shapes:
- HSON ready to inject
- Structured JSON convertible to structural data the frontend can inject directly
- JSON data the frontend must parse into a template before rendering

**Tests:** Sustainable update rate, fidelity under continuous mutation, behavior on burst traffic, and how each payload shape compares in handling cost and developer ergonomics.

**Mocking notes:** Real websockets are not available. The streaming behavior must be mocked. The frontend may call a PHP endpoint that returns simulated stream payloads, simulate the stream entirely in the frontend, or use any other mocking approach. The mocking approach is the agent's choice. Test must be self-contained and independent.

---

### Test 11: Long-running memory stability

**Context:** Mount and unmount LiveTree-managed content in a loop over an extended period.

**Tests:** Whether the claimed automatic teardown holds under churn. Per the README, hson-live claims to automatically clean up:
- Event listeners on node removal
- Scoped CSS rules in the `<hson-_style>` stylesheet on node removal

If these claims hold, heap, listener count, and CSS rule count should remain stable over many mount/unmount cycles. If they leak, this test surfaces the leak.

**Open:** Test approach, monitoring approach, and reporting approach are up to the agent. Duration, cadence, and what specifically is sampled are agent decisions.

---

### Test 12: Real-world HTML round-trip

**Context:** Real-world HTML pages are ingested by hson-live, converted to HSON, converted back to HTML, and the output is diffed against the input. Tests the core stability and reversibility claim against inputs the author likely did not fixture.

**Open:** Specific page selection, diff framework, monitoring, and reporting are up to the agent.

**Page categories to cover (agent selects specific URLs within each):**
- News article (long form, mixed media)
- E-commerce product page (forms, images, microdata)
- Documentation page (code blocks, tables, anchors)
- Blog post with embedded media (iframes, video, oEmbed)
- A page with significant SVG content
- A page using web components

The categories above are required. The specific URLs are up to the agent.

**Diff scope:** Structural, attribute, text content, ordering. Aggregate success rate across the set.

---

### Test 13: Web components and custom elements interop

**Context:** A page containing third-party custom elements is processed by LiveTree. LiveTree must parse and re-emit markup that includes custom elements.

**Tests:** Whether LiveTree's parse-and-re-emit preserves:
- Custom element registration
- Lifecycle callbacks (connectedCallback, disconnectedCallback, attributeChangedCallback)
- Shadow DOM
- Slot projection
- Attribute reactivity inside the custom element

**Distinction from chart-library testing:** This test is about ingestion. LiveTree takes over markup that contains custom elements and must not break their lifecycle or internal behavior. The chart library test focuses on coexistence with libraries that own their own DOM region (e.g. canvas-rendered charts), which is a different failure surface.

**Element categories to include (agent selects specific implementations within each):**
- Native vanilla custom element (built directly on the Web Components API)
- Lit-based component
- Stencil-built component

---

### Test 14: Build tool integration

**Context:** hson-live consumed inside a build pipeline.

**Tests:**
- Tree-shaking of unused exports
- Code splitting across subpath exports (`/hson`, `/diagnostics`, `/types`)
- Source map quality
- HMR behavior on changes to LiveTree-managed content
- Dev server vs production build differences
- Friction points specific to ESM-only consumption

**Open:** Tool choice is up to the agent. Candidates: Vite, webpack, Rollup, esbuild. Agent may test one, several, or all.

**Open:** Test approach, monitoring, and reporting are up to the agent.
