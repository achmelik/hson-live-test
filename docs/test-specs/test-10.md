# Test 10: Data streaming in and out of the application

## 1. Test ID and name

**Test ID:** 10  
**Test name:** Data streaming in and out of the application

---

## 2. What is being tested

This test measures the ability of hson-live to support continuous data updates via payloads of different shapes sent to a LiveTree-managed DOM, with focus on:

- **Sustainable update rate:** DOM remains responsive and updates are not dropped or delayed under continuous payload delivery.
- **Correctness during ongoing mutations:** The live DOM accurately reflects all mutations applied through the node graph, regardless of payload shape or update frequency.
- **Behavior on burst traffic:** System remains stable and consistent when multiple payloads are delivered in rapid succession.
- **Payload shape comparison:** Cost (latency, memory, re-render impact) and developer ergonomics vary between HSON, structured JSON requiring no template parsing, and JSON requiring frontend template rendering. This test compares these.

The test exercises a primary chat/messaging pattern (required) plus at least one additional pattern (e.g., ticker, presence indicator, live notifications) to ensure the streaming mechanism is generalizable.

---

## 3. How it will be tested

### Frontend setup
1. A page is served with a DOM target element for streaming output.
2. The page initializes a LiveTree grafted to or created at that target using hson-live.
3. A mocked stream is established via one of:
   - Frontend-only simulation: JavaScript generates and injects payloads directly.
   - Backend endpoint polling: Frontend calls a backend endpoint that returns simulated stream payloads; frontend ingests payloads at a configurable interval.
   - Hybrid: Backend generates payloads; frontend delays/batches delivery to simulate arrival patterns.
   The mocking approach is the test implementer's choice. The mocking mechanism must be transparent in test output.

### Stream payload patterns
The mocked stream must deliver payloads in all three shapes:

1. **HSON-ready shape:** Payloads directly usable as pre-parsed HSON or HsonNode objects, passed directly to the framework's mutation methods.

2. **Structured JSON (no template required):** JSON objects with known, fixed structure that the frontend can ingest directly without additional parsing (e.g., a structured object with message metadata such as sender, text, and timestamp).

3. **Template-requiring JSON:** Raw JSON data (e.g., an object with name-value pairs) requiring the frontend to evaluate or parse against a template string to produce node structure before framework ingestion.

### Payload delivery scenarios
- **Continuous steady stream:** Single payload at regular intervals over a test duration (e.g., sustained delivery spanning 30 seconds).
- **Burst traffic:** Multiple payloads delivered within a short time window (e.g., a rapid batch of 50+ payloads within 500ms).
- **Mixed pattern:** Alternating steady and bursty delivery to simulate realistic chat + system notifications.

### Patterns exercised

#### Chat (required)
- Payloads represent individual messages or message batches (sender, text, timestamp).
- Each update appends or inserts messages into a container.
- Fidelity check: final DOM structure matches expected node graph; message count, order, text content, and timestamps are all correct.

#### Additional patterns (at least one, implementer's choice)
Examples include:
- **Ticker:** Payloads update a numeric value or price display; each update mutates a specific node's text or attribute. Measures: correctness of final value; no intermediate values are dropped; updates apply in order.
- **Presence indicator:** Payloads toggle or update status for multiple users (online/offline, "typing", etc.). Measures: all status changes are reflected; no race conditions in simultaneous updates to multiple nodes.
- **Live notifications:** Payloads append alert/notification elements that may auto-dismiss or require user dismissal. Measures: correct DOM insertion; correct removal (if auto-dismiss is implemented); order and timing.

### Instrumentation & fixtures required

**Fixtures:**
- A mock stream fixture that generates payloads in all three shapes. The fixture must:
  - Support configurable delivery patterns (steady, burst, mixed).
  - Support configurable payload structure to cover the three shapes.
  - Parameterize test variables (payload count, interval, shape mix).
- An HTML page template with:
  - A DOM target element.
  - hson-live library loaded (ESM).
  - Page-level JavaScript to initialize LiveTree and consume the mock stream.
  - Instrumentation hooks to capture: wall-clock time of each payload delivery, wall-clock time of each DOM mutation, final DOM state snapshot.

**Measurement instrumentation:**
- Timestamps recorded at: payload generation, payload delivery to frontend, framework mutation call, DOM mutation observable.
- Message/event counters: payloads sent, payloads processed, DOM mutations applied.
- Final state capture: serialization of the final node graph and final DOM tree for comparison.

---

## 4. Likely failure modes and how they will be detected

| Failure Mode | Detection Method |
|---|---|
| Dropped payloads | Payload counter at ingestion point < expected count; final node count < expected count. |
| Out-of-order updates | Final DOM message timestamps or ticker values are not monotonic; presence states do not match delivery order. |
| Stale DOM | Final node graph differs from final DOM; DOM element count does not match expected node count; text content mismatch. |
| Latency spikes under burst | Median or p99 latency (delivery → mutation applied) increases significantly during burst windows. |
| Memory leaks | Heap size grows monotonically with no plateau; object counts in DevTools increase unbounded. |
| Payload shape handling bugs | One or more shapes fail to parse or update correctly; latency/memory varies significantly by shape without clear explanation. |
| Race conditions in multi-pattern updates | Updates to independent nodes interfere with each other (e.g., a presence indicator mutation affects a chat message). |
| Incorrect template evaluation | Template-requiring JSON produces wrong node structure; text or attribute values mismatch expected template output. |

---

## 5. Measurement

### Latency metrics (per-payload)
- **Delivery latency:** Wall-clock time from payload generation to first DOM change triggered (milliseconds).
- **Mutation latency:** Time to apply a single payload to the framework and observe DOM change (milliseconds).
- **p50, p95, p99** latencies across all payloads; report separately by payload shape.

### Throughput metrics
- **Payloads processed per second:** Total payloads delivered / total test duration.
- **DOM mutations per second:** Total observable DOM changes / total test duration.

### Fidelity metrics
- **Final payload count accuracy:** Expected count vs. actual count in final node graph. Report as delta.
- **Message order correctness:** Presence of out-of-order messages. Report as % in correct order.
- **Value correctness (ticker/presence patterns):** Final state of each non-message node matches expected value. Report as % correct.
- **Template accuracy (template-requiring shape):** For JSON payloads requiring template rendering, % of rendered nodes that match template expectation.

### Burst impact
- **Burst window latency increase:** Median latency during burst window vs. baseline steady-state median. Report as percentage increase.
- **Burst completion time:** Wall-clock time from first to last payload in burst reaching stable DOM state.

### Memory & resource metrics
- **Peak heap size:** Maximum observed during test (MB).
- **Heap growth rate:** Linear regression slope of heap size over time. Should be near zero during steady stream.
- **DOM node count:** Expected vs. actual at end of test.

### Payload shape comparison
For each shape (HSON, structured JSON, template JSON), report:
- Median latency (ms).
- p95 latency (ms).
- Mean memory delta per payload (bytes, if measurable).
- Developer ergonomics score: subjective but document in report (e.g., practical assessment of parsing overhead, verbosity, and evaluation cost per shape).

---

## 6. Report shape

The test report is a self-contained JSON document (or browser-consumed HTML rendering a JSON payload) with the following structure:

```json
{
  "testId": "10",
  "testName": "Data streaming in and out of the application",
  "executedAt": "ISO8601 timestamp",
  "duration": { "seconds": number, "ms": number },
  "streamConfig": {
    "mockingApproach": "string (e.g., 'frontend-only', 'backend-polling', 'hybrid')",
    "totalPayloadsDelivered": number,
    "payloadShapeMix": {
      "hson": number,
      "structuredJson": number,
      "templateJson": number
    },
    "deliveryPatterns": {
      "steadyStream": { "duration": "seconds", "interval": "ms" },
      "burst": { "count": number, "window": "ms" },
      "mixed": "boolean"
    }
  },
  "results": {
    "latency": {
      "perShape": {
        "hson": { "p50": number, "p95": number, "p99": number },
        "structuredJson": { "p50": number, "p95": number, "p99": number },
        "templateJson": { "p50": number, "p95": number, "p99": number }
      },
      "overall": { "p50": number, "p95": number, "p99": number, "unit": "ms" }
    },
    "throughput": {
      "payloadsPerSecond": number,
      "domMutationsPerSecond": number
    },
    "fidelity": {
      "payloadCountAccuracy": {
        "expected": number,
        "actual": number,
        "delta": number
      },
      "messageOrderCorrect": { "percentage": number, "unit": "%" },
      "valueCorrectness": { "percentage": number, "unit": "%" },
      "templateAccuracy": { "percentage": number, "unit": "%" }
    },
    "burst": {
      "baselineLatency": number,
      "burstLatency": number,
      "latencyIncrease": { "value": number, "unit": "%" },
      "completionTime": number
    },
    "memory": {
      "peakHeapSizeMb": number,
      "heapGrowthRatePerSecond": number,
      "expectedNodeCount": number,
      "actualNodeCount": number
    },
    "patterns": {
      "chat": {
        "messagesDelivered": number,
        "messagesInCorrectOrder": number,
        "status": "pass | fail"
      },
      "additional": [
        {
          "name": "string (e.g., 'ticker', 'presence')",
          "valuesCorrect": number,
          "status": "pass | fail",
          "notes": "string"
        }
      ]
    }
  },
  "failures": [
    {
      "type": "string (e.g., 'dropped-payload', 'out-of-order', 'stale-dom')",
      "count": number,
      "details": "string"
    }
  ],
  "notes": "string (free-form observations, edge cases, constraints encountered)"
}
```

The report is stored in the browser's localStorage under key `test-10-report` or served as a static JSON file at a path analogous to `/reports/test-10.json` (exact path not prescribed; implementer decides). A basic HTML page in the same folder displays the report with tables and charts for latency and throughput.

---

## 7. Out of scope

- Browser-specific rendering optimizations or paint timing (test does not measure RequestAnimationFrame or paint events).
- Network latency simulation for backend-based payloads (if a backend is used, payloads are assumed to arrive instantly; network jitter is not modeled).
- Concurrency stress (e.g., competing LiveTree mutations on the same tree from multiple timer sources or Workers).
- Advanced scheduling (e.g., prioritizing urgent updates over batch updates).
- Security or XSS prevention for untrusted payloads (hson-live's sanitization is tested elsewhere; this test assumes trusted input).
- Animator or animation timing (payload mutations do not include CSS animations or transitions).
- Resizing or layout thrashing (test does not stress CSS that requires recalculation on every mutation).

---

## 8. Tech stack required

**Frontend:**
- ES2020+ JavaScript (ESM modules).
- hson-live (main, /hson, /types exports).
- (Optional: build tool if bundling is desired, but not required).
- Browser DevTools APIs for memory/heap measurement (if measurable; fallback to manual GC observation).
- DOM change detection capability.

**Backend (if used):**
- A backend environment capable of returning JSON or plain text (mocked payloads).
- No specialized libraries or extensions required beyond standard request/response handling.

**Test environment:**
- Modern browser with ES2020 support.
- Test runner (optional; can be manual or built into the page).

---

## 9. Mocking notes

**Real websockets are not available.** Streaming is mocked as follows (implementer chooses approach):

1. **Frontend-only:** Page generates payloads in JavaScript using scheduled timer functions; no backend call. Payloads are pre-programmed fixtures or generated deterministically.

2. **Backend polling:** Frontend calls a backend endpoint at a fixed interval. The endpoint returns a JSON payload simulating the next message or ticker update. State is maintained server-side (or simulated on each call with a counter/seed).

3. **Hybrid:** Backend generates and returns a batch of payloads on a single call; frontend delays delivery using scheduled timers to simulate real-time arrival.

The choice does not affect test validity as long as:
- The mocking mechanism is documented in the test report.
- Payloads are deterministic and reproducible across runs.
- Delivery timing is controlled and measurable.

---

## 10. Assumptions and open questions

### Assumptions made

1. **Trusted input:** All mocked payloads are trusted; no XSS or sanitization is tested. The test assumes hson-live's sanitization (tested in security/safety tests elsewhere) works correctly.

2. **Single LiveTree instance:** The test manages one LiveTree root and mutates it throughout. Multiple independent LiveTrees are not tested.

3. **Synchronous mutations:** All mutations are applied synchronously (no async operations between payload receipt and mutation application). Timing measurements reflect JavaScript execution time, not browser event loop scheduling.

4. **No CSS rendering:** DOM mutations are measured as observable node/attribute changes; CSS layout/paint timing is not instrumented (test does not depend on computed styles or layout recalculation for payload processing).

5. **Message/payload fidelity:** Final state comparison assumes serialization of nodes and DOM is stable and reproducible. No clock skew or floating-point comparison issues are expected in timestamp fields.

6. **Memory measurement:** Peak heap size and growth rate are measured via DevTools heap snapshots or performance API if available; fallback is manual observation (may be imprecise).

7. **Payload delivery order:** Payloads are delivered in the order mocked; no network reordering or out-of-order delivery is simulated.

8. **No concurrent updates:** Frontend does not apply mutations from multiple sources simultaneously (e.g., only one stream consumer).

### Scope expansions

None. This specification is narrowly scoped to streaming data updates via multiple payload shapes into a single LiveTree instance under controlled mocking.

### Open questions

1. **Heap measurement mechanism:** How is peak heap size and growth rate captured? Should the test use browser DevTools heap snapshots (manual), performance API (if available), or a custom memory tracker? Recommend documenting the approach in the test report so results can be interpreted correctly.

2. **Template evaluation approach:** For template-requiring JSON payloads, what is the template evaluation approach? How are payloads transformed into node structure? This affects latency baseline and should be specified before implementation to ensure fair comparison across shapes.

3. **Auto-dismiss timing for notifications (if presence/notifications pattern is chosen):** Should notifications auto-dismiss after a fixed delay, require user dismissal, or neither? This affects final state fidelity measurement and should be decided early.

4. **Burst definition:** What constitutes a "burst"? (e.g., 50 payloads in 500ms, or 10x baseline rate for 2 seconds). Recommend defining exact burst parameters (count, window, number of bursts) before implementation to ensure reproducibility.
