# Test ID: 11

## Test name
Long-running memory stability

## What is being tested

Whether automatic cleanup claims hold under sustained churn. Specifically:
- Event listeners are removed from DOM elements on node detachment.
- CSS rules in the framework's managed stylesheet are removed on node detachment.

The test verifies that repeated mount/unmount cycles do not leak memory in heap size, listener counts, or CSS rule counts. These three signals together indicate whether cleanup is working.

## How it will be tested

1. **Instrumentation setup**: 
   - Capture baseline heap size, DOM listener count, and CSS rule count before any mounts.
   - Instrument via browser performance APIs (heap snapshots, listener tracking, stylesheet inspection).

2. **Churn loop**:
   - In a loop: mount a LiveTree-managed subtree → perform read/write operations → unmount.
   - Single iteration: create a node graph with mixed content (text, attributes, CSS rules, event listeners), render it to DOM via LiveTree, perform a few mutations, then remove the tree.
   - Repeat for a sufficient number of cycles to surface a leak (specifics open to tuning based on performance budget).

3. **Sampling**:
   - Capture heap, listener, and rule counts at intervals (e.g., every M cycles, plus start and end).
   - Between samples, trigger garbage collection if available (to stabilize heap measurements).

4. **Report and interpretation**:
   - Plot heap, listener count, and rule count over time.
   - Compute linear regression or trend analysis for each signal to detect drift.
   - Flag as failure if any signal shows persistent upward trend (slope significantly > 0) or exceeds threshold (e.g., +10% from baseline at end).
   - Pass if all three signals remain stable (flat trend, within noise tolerance).

## Likely failure modes and how they will be detected

**Failure: Event listener leak**
- Symptom: listener count grows monotonically with iterations.
- Root cause: The framework's listener-cleanup mechanism may not deregister all attached handlers, or handlers attached outside the managed listener system persist.
- Detection: listener count trend is significantly positive; compare DOM listener registry before/after each cycle.

**Failure: CSS rule leak**
- Symptom: rule count in the framework's managed stylesheet grows with iterations.
- Root cause: The framework's CSS-rule cleanup may not fully delete rules for detached nodes, or pseudo-rules may not be fully removed.
- Detection: CSS rule count trend is positive; inspect the managed stylesheet to confirm rules persist.

**Failure: Heap leak**
- Symptom: heap size grows monotonically despite GC.
- Root cause: node objects, event handler closures, or CSS rule maps remain reachable due to circular references or incomplete cleanup.
- Detection: heap snapshot analysis shows retained objects for old nodes; memory profiler shows allocation trend > 0.

**Failure: CSS singleton issues**
- Symptom: CSS rules accumulate even after multiple mount/unmount cycles.
- Root cause: The framework's CSS manager is a singleton; if document identity changes (e.g., in test harnesses that swap the global document), cached DOM references may become stale and new rules pile up without replacing old ones.
- Detection: The framework's managed stylesheet grows in size; or rules from old cycles remain visible in stylesheet.

## Measurement

**Primary signals**:
1. **Heap size**: Captured via heap snapshots or browser performance APIs. Unit: bytes. Baseline at iteration 0; measure at intervals.
2. **DOM listener count**: Count of registered listeners attached to elements in the tree. Method: count via standard DOM listener inspection or by instrumenting the framework's listener registration. Unit: count.
3. **CSS rule count**: Count of CSS rules in the framework's managed stylesheet. Unit: count. Can be derived from standard stylesheet inspection APIs.

**Secondary signals** (optional for deeper diagnosis):
- Count of rule scopes registered with the framework's CSS manager (may indicate CSS state accumulation).
- Pseudo-rule count from the framework's CSS manager.
- Total DOM element count (should remain bounded if mounting/unmounting is symmetric).

**Sampling interval**:
- Interval `S`: measure after every N cycles (e.g., N=10, so S=10).
- Minimum 10 samples across the full run (so if 100 cycles, measure at 0, 10, 20, ..., 100).

**Trend analysis**:
- Fit a line to each signal (linear regression) over all samples.
- Slope `m`: if the slope is close to zero relative to the baseline value per cycle, consider stable; otherwise flag trend.
- Alternatively, compute max–min over samples; if range < 5% of baseline, consider stable.

## Report shape

Format: JSON object stored in browser localStorage (or embedded in a rendered HTML report).

```json
{
  "testId": 11,
  "testName": "Long-running memory stability",
  "timestamp": "ISO8601",
  "config": {
    "totalCycles": <number>,
    "sampleIntervalCycles": <number>,
    "nodeStructure": "<description of the fixture tree>",
    "contentSize": "<e.g., 'small' | 'medium' | 'large'>"
  },
  "samples": [
    {
      "cycle": <number>,
      "heapBytes": <number>,
      "listenerCount": <number>,
      "cssRuleCount": <number>,
      "gcTriggered": <boolean>,
      "timestamp": "ISO8601"
    }
  ],
  "analysis": {
    "heapTrendSlope": <number>,
    "heapTrendStable": <boolean>,
    "listenerTrendSlope": <number>,
    "listenerTrendStable": <boolean>,
    "cssRuleTrendSlope": <number>,
    "cssRuleTrendStable": <boolean>,
    "overallResult": "PASS" | "FAIL",
    "failureReasons": [<string>]
  },
  "notes": "<optional context notes>"
}
```

Report retrieval: JavaScript running in the test page reads from localStorage and renders a summary table or chart. Alternatively, report is written to a plain-text or HTML file in the session output directory for post-session inspection.

## Out of scope

- Testing transformations (HSON ↔ JSON ↔ HTML ↔ XML ↔ SVG) are out of scope; this test is LiveTree-only.
- CSS animation or keyframe cleanup is not explicitly tested (implicitly covered via CSS rule count, but no dedicated animation harness).
- Custom event handlers attached outside the managed listener system are not in scope. Only handlers registered via hson-live's listener API are tracked.
- Testing under extreme churn (millions of cycles) is not in scope; practical limits suffice to detect leaks.

## Tech stack required

**Backend**: None required. Test runs entirely in the browser.

**Frontend**:
- hson-live package (ESM).
- Browser APIs: performance measurement APIs (heap snapshots or performance monitoring), listener inspection (where available), stylesheet inspection (standard DOM).
- GC trigger: browser-specific (e.g., available triggering mechanisms in DevTools contexts with garbage collection exposure).
- DOM testing environment: virtual or real browser (specific environment choice is open; heap and listener measurements may differ by implementation).

**Test framework**: None mandated. Plain script or lightweight harness.

**Fixture**:
- Node graph fixture with: HTML elements (div, p, span, etc.), text content, attributes (id, class, data-*), CSS rules (color, opacity, transform), event listeners (click, input, mouseover). The fixture must demonstrate a tree with mixed content, attributes, listeners, and CSS rules.

## Mocking notes

- **No real WebSockets**: Test does not use WebSockets.
- **GC triggering**: If garbage collection triggering is not available, fall back to available alternatives or accept variance in heap measurements.
- **Listener tracking**: If standard listener inspection APIs are unavailable, instrument the framework's listener registration to count active listeners.
- **CSS rule inspection**: Standard stylesheet inspection APIs work across browsers; parsing textContent is a fallback.

## Assumptions and open questions

### Assumptions made

1. **CSS management is centralized**: Multiple mount/unmount cycles reuse the same CSS management instance. If document identity changes (test harness behavior), the CSS manager resets appropriately and cached DOM references are updated. This reset is assumed to work correctly; the test does not mock document swaps.

2. **Cleanup is synchronous**: Node detachment immediately removes listeners and CSS rules. The test assumes no async cleanup hooks; measurements are taken synchronously after unmount.

3. **Event listener API is available**: The test assumes listener inspection or an equivalent tracking mechanism exists. Browsers differ in availability; fallback to manual instrumentation of the framework's listener system.

4. **CSS rule removal is complete**: The framework's CSS-rule cleanup fully removes all scoped rules. Pseudo-rules are removed. No partial or orphaned rules remain. The test trusts these mechanisms without patching the CSS manager.

5. **Baseline stability**: The first sample (cycle 0, before any churn) represents a stable baseline. Subsequent samples are compared against this baseline.

6. **GC pauses are visible**: Heap measurements after forced GC are stable enough to detect trends. High GC variance may obscure slow leaks; test duration and sampling density are sufficient to detect meaningful drift.

7. **No external listener interference**: The test does not attach DOM listeners outside the framework's listener system. All tracked listeners are managed by the library.

8. **Node fixture is representative**: The fixture tree is realistic enough to surface leaks at scale without requiring millions of cycles.

### Scope expansions

None identified. Test is strictly scoped to memory stability under mount/unmount churn.

### Open questions

1. **Listener counting method**: How are event listeners counted if standard listener inspection APIs are unavailable? Should the test instrument the framework's listener management to track registration/deregistration, or inspect the listener state directly?

2. **CSS pseudo-rule cleanup**: Are pseudo-rules (`:hover`, `::before`, etc.) fully exercised in the fixture? If not tested, leaks in pseudo-rule cleanup may go undetected. Should the fixture include nodes with pseudo-state CSS?

3. **GC timing**: How long should tests wait after triggering GC before measuring heap? Different JavaScript engines have different GC latencies. Should the test wait for a callback or use a heuristic delay?

4. **Heap measurement API**: Should the test use performance monitoring APIs (with their inherent granularity limitations) or heap snapshots (more accurate, slower)? Heap snapshots are more reliable but may require DevTools integration.

5. **Threshold for failure**: Is a +10% drift in any signal a hard failure, or should the test tolerate small trends (e.g., <5% drift) as noise? Should thresholds be relative to baseline or absolute?

6. **Document swap handling**: The test assumes document identity is stable. If the test runner swaps the global document between cycles, should the test verify that the CSS manager resets appropriately, or should it avoid document swaps entirely?

7. **Listener system internals**: Should the test inspect the framework's listener state directly to detect leaks, or only observe DOM-level listener counts? Inspecting internals gives precise leak detection but exposes implementation details.
