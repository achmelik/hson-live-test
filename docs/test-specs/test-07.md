# Test 07: Performance A/B (large-data business application)

## 1. Test ID and name
**Test ID:** 07  
**Test name:** Performance A/B (large-data business application)

## 2. What is being tested
Direct performance comparison between two implementations of the same client contract file interface, differing only in the rendering approach:
- **Implementation A (hson-live):** Uses hson-live's LiveTree to project and manage the DOM from a node graph.
- **Implementation B (vanilla JS):** Uses plain JavaScript and DOM manipulation.

Both implementations must render and interact with the same dataset identically. The test measures whether hson-live introduces overhead or provides performance benefits for large-data business applications with complex per-item templates, client-side pagination, and client-side filtering.

Scope: rendering performance and interaction responsiveness. Does not test hson-live's transformation accuracy or losslessness.

## 3. How it will be tested

**Test setup:**
- Generate a fixture containing 10,000 contract records. Each record must include client name, status, previous contract value, current contract value, contract value delta, and metadata sufficient to derive a yearly basis calculation.
- Two separate applications (A and B) render the same fixture, with identical UI and identical interaction patterns.

**Pages:**
- Page A: hson-live implementation. Renders initial view by transforming fixture data into an HsonNode graph, projects live DOM via LiveTree, and responds to user interactions through graph mutations.
- Page B: Vanilla JS implementation. Renders initial view through DOM manipulation, stores state in plain JavaScript objects, and responds to interactions by updating DOM directly.

**Page behavior (both implementations must be identical):**
- Initial render: Display records from the full dataset on initial page load.
- Each record displays: client name, status, previous contract value, current contract value, contract value delta, yearly basis, and 6 navigation icons (review, close, contact client, contact internal legal rep, contact internal stakeholder rep, export audit trail). Icons may be implemented as placeholders (no-op handlers); they do not need to navigate or perform actions.
- Pagination controls: Forward and backward navigation between pages. Page position indicator (page numbers or scroll position) is optional.
- Filter control: A search/filter input that filters records by client properties. Filter is applied client-side; filtered view updates immediately in response to input changes.
- Both pages must handle the same interaction sequence (detailed in Measurement section) with no deviations in behavior.

**Instrumentation:**
- Each page exposes performance metrics via in-page JavaScript (e.g., a global object or localStorage).
- Metrics are captured at specific interaction points (detailed in Measurement section).
- Metrics must be retrievable by a test harness (e.g., via `window.performance_metrics`, `localStorage`, or a data attribute on the page).

**Fixture:**
- Single 10k-record dataset used by both pages.
- Fixture must be static and reusable. Storage mechanism is open (JSON file, in-memory JavaScript object, or server endpoint).

## 4. Likely failure modes and how they will be detected

| Failure mode | Detection method |
|---|---|
| Inconsistent DOM structure between A and B | Visual regression: render both pages to the same state and compare DOM structure (element count, tag names, class names, text content). |
| Pagination produces different records between A and B | Verify record counts and client names match on each page after pagination. |
| Filter produces different result sets between A and B | Apply the same filter query to both and compare result counts and record IDs. |
| Performance metrics are missing or unreadable | Attempt to retrieve metrics from the nominated storage (e.g., `window.performance_metrics`); fail test if undefined or malformed. |
| Metrics are captured at inconsistent times (wall-clock drift, async delays) | Verify timestamps are ordered (each subsequent metric is >= previous). Flag if gaps exceed expected bounds (e.g., >500ms between sequential metrics within a single interaction). |
| Page becomes unresponsive during interaction sequence | Set a timeout per interaction (starting threshold: 10 seconds); fail if interaction does not complete within timeout. Timeout is open to adjustment based on initial runs. |
| Memory consumption grows unboundedly during repeated interactions | Monitor heap size before first interaction, after each interaction, and at end. Flag if heap grows >20% from baseline. This threshold is a starting point; refinement based on first run is expected. |

## 5. Measurement

**Metrics to capture (per-page, at each point listed below):**
- `time_to_initial_render_ms`: Time from page load to first paint of the record list (initial page of records visible).
- `time_to_interactive_ms`: Time from page load to the moment pagination and filter controls are responsive (e.g., first click is registered and acted upon).
- `time_to_paginate_forward_ms`: Duration from "next page" button click to all new records on page fully rendered and visible.
- `time_to_paginate_backward_ms`: Duration from "previous page" button click to all new records on page fully rendered and visible.
- `time_to_filter_apply_ms`: Duration from user input into filter field to all filtered records rendered and visible, including removal of non-matching records from DOM.
- `time_to_filter_clear_ms`: Duration from clearing filter input (or clicking a clear button) to all 10k records restored and visible.
- `filter_result_count`: Number of records matching the filter query. Captured at each filter interaction to verify A and B produce identical counts.
- `heap_size_after_initial_render_bytes`: Memory consumption immediately after initial render completes.
- `heap_size_after_sequence_ms`: Memory consumption at end of test sequence.

**Interaction sequence:**
1. Load page, wait for initial render metric to complete.
2. Verify pagination and filter controls are interactive (measure `time_to_interactive_ms`).
3. Click "next page" button, measure `time_to_paginate_forward_ms`, record page number and displayed record IDs.
4. Click "next page" again, measure `time_to_paginate_forward_ms` (second iteration), record page number and displayed record IDs.
5. Click "previous page" button, measure `time_to_paginate_backward_ms`, record page number and displayed record IDs.
6. Enter a filter query, measure `time_to_filter_apply_ms`, record result count.
7. Modify filter query to a more restrictive match, measure `time_to_filter_apply_ms` (second iteration), record result count.
8. Clear filter, measure `time_to_filter_clear_ms`, record result count (should be 10k).
9. Click "next page" (while filter is cleared), measure `time_to_paginate_forward_ms` (third iteration).
10. Repeat steps 6–8 two more times with different filter queries to capture variance.

**Agent's role in metric selection:**
The above metrics are illustrative. Agent may add or refine:
- Frame rate or layout thrashing metrics if rendering complexity warrants instrumentation.
- Interaction latency (e.g., time from click event fire to state update) if user-perceivable delay is a concern.
- DOM diff or reconciliation metrics if frameworks are involved.
- Any other metrics that directly measure responsiveness or resource consumption.

**Comparison criteria:**
- A and B must produce identical `filter_result_count` at each filter step. Mismatch indicates a logic error in filtering.
- A and B must display identical record sets (by client name and ID) at each pagination step. Mismatch indicates a logic error in pagination.
- Latency metrics (time_to_*) are compared by ratio: `latency_B / latency_A`. A ratio of 1.0 means identical performance. Ratios >1.2 or <0.8 are initial thresholds indicating a meaningful performance difference and should be flagged for human review. These thresholds are open to refinement after first run.
- Memory metrics are compared by absolute difference: `heap_size_B - heap_size_A` in bytes. Differences >10 MB are initial thresholds and should be flagged. This threshold is open to refinement after first run.

## 6. Report shape

**Storage:** Metrics and comparison results are stored in-browser via localStorage or as a JSON object exposed on `window`. Backend storage (e.g., a database or file) is optional.

**Structure:**
```
{
  "test_id": "07",
  "test_name": "Performance A/B (large-data business application)",
  "timestamp": "ISO 8601 string",
  "implementation_a": {
    "name": "hson-live",
    "metrics": [
      {
        "interaction": "initial_render",
        "time_to_initial_render_ms": <number>,
        "time_to_interactive_ms": <number>,
        "heap_size_after_initial_render_bytes": <number>
      },
      {
        "interaction": "paginate_forward",
        "iteration": 1,
        "time_to_paginate_forward_ms": <number>,
        "page_number": <number>,
        "record_ids": [<array of client IDs or names>]
      },
      {
        "interaction": "filter_apply",
        "iteration": 1,
        "filter_query": "<string>",
        "time_to_filter_apply_ms": <number>,
        "filter_result_count": <number>
      },
      {
        "interaction": "filter_clear",
        "iteration": 1,
        "time_to_filter_clear_ms": <number>,
        "filter_result_count": <number>
      }
    ]
  },
  "implementation_b": {
    "name": "vanilla-js",
    "metrics": [<same structure as implementation_a>]
  },
  "comparison": {
    "filter_result_count_matches": true,
    "record_set_matches": true,
    "latency_ratios": [
      {
        "metric": "time_to_initial_render_ms",
        "ratio": 1.05,
        "ratio_interpretation": "hson-live is ~5% slower"
      },
      {
        "metric": "time_to_paginate_forward_ms",
        "ratio": 0.95,
        "ratio_interpretation": "hson-live is ~5% faster"
      }
    ],
    "memory_diff_bytes": 2048000,
    "memory_diff_interpretation": "vanilla JS consumed ~2 MB more"
  }
}
```

**Retrieval:** Test harness reads metrics from localStorage or window object, parses JSON, and displays results in an HTML report. Report includes tables comparing A and B metrics, latency ratio charts, and memory charts.

## 7. Out of scope
- Transformation accuracy or losslessness of hson-live (covered by other tests).
- CSS rendering performance or style computation (both implementations use the same stylesheet).
- Network latency or asset loading time (both implementations load the same assets).
- Accessibility, usability, or semantic correctness.
- Comparison with frameworks other than vanilla JS (e.g., React, Vue).
- Security or XSS mitigation (both implementations use the same sanitization approach, if any).
- Behavior with datasets <10k records or >10k records (exactly 10k is scoped).

## 8. Tech stack required

**Backend:**
- Capability to serve static HTML files and JSON fixtures.
- Ability to serve endpoints that return HSON, JSON, or fixture data as needed.

**Frontend (Implementation A - hson-live):**
- hson-live 2.1.0 (ESM).
- LiveTree API for DOM projection.
- JavaScript for performance instrumentation and event handling.
- DOM for rendering.

**Frontend (Implementation B - vanilla JS):**
- Vanilla JavaScript (no frameworks).
- DOM for rendering.
- JavaScript for performance instrumentation and event handling.

**Testing/Instrumentation:**
- Modern browser supporting ES modules, high-resolution timing API, and DOM API.
- Browser DevTools or programmatic heap size measurement.
- localStorage or window object for metrics storage.

**Test harness:**
- Browser automation or manual runner (agent determines).
- Ability to load both pages sequentially and extract metrics.
- HTML report generation (static or client-side).

## 9. Mocking notes

**Websockets and SSE:** Not required. Both pages load a static fixture and operate entirely client-side. No real-time sync is tested.

**Backend endpoints:** If fixture data is served via an endpoint (e.g., `/api/contracts`), the backend must return the fixture with minimal overhead (single request). Response time is NOT measured; only client-side rendering is in scope. If latency between fetch and rendering is a concern, fixture may be inlined in HTML or served as a single JSON file.

**High-resolution timing:** A high-resolution browser timing API is expected to be available. If unavailable, fall back to lower-resolution timing with a note that precision is lost.

**Heap size measurement:** Heap size measurement APIs are non-standard and not available in all browsers. If unavailable, agent may omit heap size metrics or estimate memory consumption by other means (e.g., timing DOM traversals, counting object instances). Document fallback in report.

## 10. Assumptions and open questions

### Assumptions made

1. **Fixture size and distribution:** The 10k-record fixture is uniformly distributed (no clustering or hotspots that favor one implementation over another). Record names and values are generated deterministically (not random) so the same fixture is reproducible.

2. **Per-item rendering complexity:** The illustrative list (client name, status, previous value, current value, delta, yearly basis, 6 icons) provides sufficient DOM complexity to stress rendering. Agent may expand (e.g., add nested containers, conditional elements, animations) if initial implementation shows no measurable difference between A and B. Minimum complexity: each item must render >10 DOM nodes.

3. **Pagination page size:** Default is based on achieving meaningful latency measurement between operations. Agent may adjust page size if this results in pages that render too quickly or too slowly to measure latency (target: 100–500 ms per interaction).

4. **Filter matching:** Filter updates records based on client properties. Agent may extend to multi-field search if complexity is needed.

5. **Interaction timing:** Metrics are captured using high-resolution timing from the page itself (not externally by a test harness). Agent is responsible for ensuring that clock skew or timer resolution does not introduce >10% measurement error.

6. **Baseline performance:** Neither implementation is optimized beyond reasonable defaults. Code is written for clarity, not micro-optimized. Both are subjected to the same toolchain and minification.

7. **DOM visibility:** Elements must be in the DOM and visible (not display: none) for metrics to be meaningful. "Rendered and visible" means the browser has completed layout, paint, and compositing.

8. **Memory measurement:** If heap size measurement is unavailable, agent will document the fallback approach (e.g., measurement by indirect timing). Memory metrics are secondary; latency metrics take priority.

9. **Browser environment:** Test runs in a modern browser supporting ES modules, the DOM API, and localStorage. Test results are specific to that browser; cross-browser variance is not measured.

### Scope expansions

1. **Heap size monitoring:** Originally out of scope (performance test focuses on latency), but included here as a secondary metric to catch memory leaks or unbounded growth during repeated interactions.

2. **DOM structure comparison:** Originally out of scope (content and DOM semantics are not being tested), but included as a failure detection method to verify that A and B produce equivalent pages.

3. **Interaction latency beyond rendering:** Metrics include `time_to_interactive_ms`, which measures responsiveness of event handlers, not just rendering speed. This is a mild expansion to isolate input lag from rendering lag.

### Open questions

1. **Fixture storage:** Should the 10k-record fixture be served by the backend, inlined in the HTML page, or pre-loaded via a JavaScript module? Backend serving allows easier fixture updates; inlining avoids a network request and isolates page latency.

2. **Filter UI detail:** Should the filter be a text input with live update on keypress, or a button-click to apply? Live update (keypress) adds noise to latency measurement due to debounce/throttle and may favor implementations with aggressive caching. Button-click is cleaner but less realistic.

3. **Report delivery:** Should the HTML report be self-contained (all data embedded in one .html file) or split between a separate JSON file and an HTML viewer? Self-contained is simpler for sharing; split is easier to integrate with CI/CD.

4. **Baseline vs. regression threshold:** What latency ratio or memory delta constitutes a meaningful difference for flagging? Current thresholds (1.2x or 0.8x for latency, ±10 MB for memory) are starting points. Should these be configurable or data-driven (e.g., set to ±1 standard deviation from a baseline)?

5. **Repeated interaction sampling:** The interaction sequence includes 3 filter iterations and 3 pagination interactions to capture variance. Is 3 iterations sufficient, or should the test loop the sequence N times? More iterations increase confidence but may hit browser resource limits or show fatigue effects.
