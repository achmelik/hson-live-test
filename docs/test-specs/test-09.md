# Test 09: Render flicker on large updates

## 1. Test ID and name
- **ID:** 09
- **Name:** Render flicker on large updates

---

## 2. What is being tested

Whether large-scale content replacement via `graft()` produces visible flicker, FOUC (Flash of Unstyled Content), or layout shift.

**Specific behaviors under test:**
- Visibility and detectability of intermediate states during the replacement
- Whether the browser repaints/reflows with stale or partially-rendered content before the final state is painted
- Whether layout shifts occur (changes in element positions or dimensions between start and end of update)
- Whether CSS that applies to the root or body becomes briefly unapplied or inconsistent

---

## 3. How it will be tested

### Setup
1. A test page with a known initial DOM state (fixed-size container with styled content).
2. Instrumentation to capture rendering artifacts:
   - Observation of paint events during the update
   - Observation of layout shift events during the update
   - Capture of pixel data at fixed intervals during the update
   - Tracking of DOM mutations during the update
   - Sampling of computed style on key elements during the update
   - Measurement of layout dimensions (bounding box) before, during, and after the update

### Trigger
The update is triggered via one of:
- Direct call to `graft()` on the container element with new content
- Client-side function that replaces the node graph and causes a large update
- A simulated server-response handler that replaces content via the node graph

The trigger must replace a substantial amount of content (e.g., minimum 100 elements or equivalent depth), representing a large update.

### Measurement phases
1. **Before update:** Baseline paint state, layout dimensions, and pixel snapshot of rendered output.
2. **During update:** Capture paint events, layout shift events, DOM mutations, and pixel snapshots at short intervals (every 10–20ms or on animation frame boundary, whichever occurs first).
3. **After update:** Final paint state, layout dimensions, pixel snapshot, and comparison to baseline.

### Fixture requirements
- A container element (e.g., `<div id="app"></div>`) anchored in the DOM.
- Initial content: a styled tree with CSS applied (inline or linked stylesheet).
- New content payload (HSON, JSON, or HTML): structurally different from the initial content, with non-trivial volume.
- Optional: CSS animations or transitions on the container or its children to test interaction with rendering pipeline.

---

## 4. Likely failure modes and how they will be detected

### Failure Mode: Visible flicker
**Symptom:** User observes a brief flash of unstyled or partial content before the final render.
**Detection:**
- Pixel comparison: capture the rendered output at the frame where the update begins; if subsequent frames show intermediate states (not initial, not final), flicker is present.
- Paint events: if multiple paint events fire within a narrow time window during the update, the browser repainted multiple times due to intermediate layout states.

### Failure Mode: FOUC
**Symptom:** Content briefly renders without CSS styling.
**Detection:**
- Pixel inspection: look for color or opacity changes in elements that should be consistently styled.
- Computed style inspection: sample computed style on key elements during the update; if style values differ from expected, FOUC is occurring.

### Failure Mode: Layout shift
**Symptom:** Elements move or resize unexpectedly during the update.
**Detection:**
- Layout shift events: observe layout shift entries during the update; any entry indicating a shift without recent user input indicates an unexpected shift.
- Layout dimension comparison: capture the bounding box of the container before and after the update; if the box differs, a shift occurred. Track intermediate dimensions during the update to pinpoint when the shift happens.
- Cumulative Layout Shift (CLS) score: if any shift is detected, the score increases; the test records whether CLS was observed.

### Failure Mode: Long task blocks rendering
**Symptom:** The browser's main thread is busy for long enough that the user perceives a stalled render.
**Detection:**
- Observation of long-task events (if available in the browser).
- Measure time between update trigger and first paint; if the gap is > 50ms, flag as a potential blocking issue.

### Failure Mode: Stale content briefly visible
**Symptom:** Old content is visible before it is replaced.
**Detection:**
- Capture a pixel snapshot of the rendered DOM state immediately after `graft()` is called but before the promise resolves or the next paint fires.
- Compare the snapshot to the initial and final expected states; if it matches the initial state, old content was not immediately replaced.

---

## 5. Measurement

### Metrics collected
1. **Paint count:** Number of paint events between update trigger and final render.
   - Starting threshold: 0–2 (ideally 1, representing the final composite). >2 suggests multiple repaints.

2. **Time to first paint:** Milliseconds from update trigger to first paint event.
   - Starting threshold: < 50ms (browser's typical frame budget).

3. **Layout shift events:** Count and cumulative shift value during the update window.
   - Starting threshold: 0 (no shifts).

4. **Flicker presence:** Binary (yes/no) based on pixel comparison.
   - Method: Capture pixel data at regular intervals; if an intermediate frame does not match the initial state and does not match the final state, flag as flicker.

5. **FOUC presence:** Binary (yes/no) based on computed style sampling.
   - Method: Sample computed style on a representative element during the update; if a property value temporarily differs from its final value, flag as FOUC.

6. **Update duration:** Milliseconds from trigger to final paint event.
   - Starting threshold: < 100ms for large but manageable updates.

7. **Layout shift total:** Cumulative shift value from all layout shift events.
   - Starting threshold: 0 or near-zero (< 0.1).

### Reported metrics
All metrics are reported per test run. The report should include:
- Summary: pass/fail based on presence of flicker, FOUC, or layout shift.
- Breakdown: per-metric status (paint count, time to first paint, CLS, etc.).
- Timeline: list of events (paint, shift) in order with timestamps.

---

## 6. Report shape

**Storage:** Report is stored as a JSON object in the browser's local storage or as a JSON file served via a server endpoint.

**Structure:**
```json
{
  "testId": "09",
  "testName": "Render flicker on large updates",
  "timestamp": "2026-04-27T14:32:00Z",
  "runs": [
    {
      "runId": 1,
      "updateTrigger": "graft-direct",
      "initialStateChecksum": "abc123...",
      "finalStateChecksum": "def456...",
      "metrics": {
        "paintCount": 1,
        "timeToFirstPaintMs": 12,
        "layoutShiftCount": 0,
        "cumulativeLCS": 0.0,
        "updateDurationMs": 45,
        "flickerDetected": false,
        "foucDetected": false,
        "layoutShiftDetected": false
      },
      "events": [
        { "timestamp": 0, "type": "update-trigger", "detail": "graft() called" },
        { "timestamp": 12, "type": "paint", "detail": "first paint" },
        { "timestamp": 45, "type": "paint", "detail": "final paint" }
      ],
      "verdict": "PASS"
    }
  ],
  "summary": {
    "totalRuns": 1,
    "passCount": 1,
    "failCount": 0,
    "failureReasons": []
  }
}
```

**Retrieval:** Report is accessible via:
- Local storage key: `hson-live-test-09-report`
- Server endpoint (if backend is present): `/api/test-report/09` or similar contract, returning the JSON object.

---

## 7. Out of scope

- Testing of CSS animation or transition performance (separate from render-blocking behavior).
- Testing of LiveTree mutation methods other than those involved in the update trigger (e.g., `append`, `removeChildren`).
- Testing of different rendering engines or browsers (test is written to be browser-agnostic, but results may vary).
- Testing of accessibility (a11y) during the update.
- Benchmarking of LiveTree vs. other frameworks or manual DOM manipulation.
- Testing of updates triggered by non-graft methods (e.g., direct `innerHTML` assignment), unless the test explicitly compares them.

---

## 8. Tech stack required

### Frontend
- **hson-live** (runtime): Root export for graft, node-graph manipulation, and LiveTree construction.
- **Browser environment:** Must support rendering-event observation (paint and layout-shift events) and pixel-capture capability (to record visual state during the update).

### Backend (optional)
- An endpoint that:
  - Serves the test page HTML
  - (Optional) Provides a JSON/HSON payload to be grafted
  - (Optional) stores or retrieves test reports

### Build & Runtime
- ESM-compatible bundler or ES modules directly in the browser (hson-live is ESM-only).
- No specific HTTP server required; test can run locally or on any static file server.

---

## 9. Mocking notes

**Real websockets:** Not used. Any live updates that would normally come from a server are simulated via JavaScript function calls or pre-computed payloads.

**Paint and layout-shift events:** Supported by modern browsers natively. If unavailable, the test gracefully degrades: event-based metrics are unavailable, but pixel-based flicker detection still works.

**Pixel capture:** Pixel-sniffing requires rendering to a canvas. If the test environment does not support pixel capture (e.g., Node.js headless), flicker detection is skipped. Paint and layout shift metrics are still available.

**Layout shift observation:** Layout shift events are available in most modern browsers. Older browsers do not populate this API; if unavailable, the test records `layoutShiftDetected: null` (unknown) rather than false.

---

## 10. Assumptions and open questions

### Assumptions made
1. The container element into which content is grafted is a direct child of `document.body` and is styled such that its layout is not dependent on parent dimensions (i.e., it has a declared width/height or is flex-able).
2. The initial and final content states are visually distinct enough that flicker is detectable via pixel comparison (e.g., different background colors or text content).
3. The test environment supports ES modules and modern browser APIs for rendering observation and style inspection.
4. CSS animations and transitions on the page are either disabled or have zero duration during the test (to avoid confounding repaints).
5. The "large update" is defined as one that involves replacing >= 100 elements or a tree depth of >= 10 levels. The exact threshold is implementation-dependent.
6. Paint events have sufficient granularity to distinguish between initial, intermediate, and final states (browser-dependent; some browsers may batch them).

### Scope expansions
None. The test scope is limited strictly to detecting flicker, FOUC, and layout shift during large updates via the node graph, as described in the context.

### Open questions
1. **Pixel capture granularity:** What interval should pixel snapshots be taken at—fixed time (e.g., every 10ms), animation frame boundary, or event-driven (on mutation/paint)?

2. **Flicker definition:** Is a single intermediate frame showing partial content sufficient to flag flicker, or should a duration threshold apply (e.g., flicker must be visible for >= 16ms / one full frame)?

3. **FOUC detection accuracy:** Sampling computed style at discrete intervals may miss brief FOUC events. Should the test also monitor for changes in style rules to detect style mutations?

4. **Initial content structure:** Should the initial content be a realistic page (e.g., a blog post with header, sidebar, article) or a synthetic stress test (e.g., a deeply nested tree)?

5. **Update trigger variability:** The context states that the trigger mechanism is the agent's choice. Should the test run multiple trigger types and compare results?

6. **Report accessibility:** Should the report be human-readable or machine-readable, or both?
