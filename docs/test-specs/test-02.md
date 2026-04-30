# Test 02: Server returns JSON

## 1. Test ID and name

**Test ID:** 02  
**Test name:** Server returns JSON

---

## 2. What is being tested

hson-live's ability to convert JSON directly to renderable form without HSON as an intermediate.

Specifically, this test validates that:

- A JSON response from a backend service can be loaded directly without requiring HSON syntax or server-side HSON generation.
- The JSON is parsed into a structure suitable for rendering according to the documented JSON mapping rules.
- The resulting node graph can be rendered to a live DOM without data loss.

This test does not validate JSON schema validation, application-level constraints, HTML rendering semantics, or security policies—only the data transformation path and structural fidelity.

---

## 3. How it will be tested

### Pages and setup
- A single frontend page that fetches JSON from a backend endpoint, parses it with hson-live, and renders it to a DOM container.
- Backend serves a JSON response (not HSON, not hybrid).

### Instrumentation
- Frontend captures JSON response before hson-live transformation.
- Captures the rendered DOM structure.
- Collects measurement data during conversion (timing, node count).

### Fixtures required
Two datasets, each specifying structural patterns (not literal values):

**Fixture A: Simple scalar and collection structures**
- Flat JSON object with string, number, boolean, and null properties.
- Shallow JSON array of mixed primitives.
- Validates basic primitive mapping and array indexing.

**Fixture B: Nested and mixed structures**
- JSON object containing nested objects and arrays at multiple depths.
- Arrays containing objects and nested arrays.
- Empty collections and null values at various positions.
- Validates recursive descent and structural wrapping.

Both fixtures must exclude:
- Undefined or symbol values.

---

## 4. Likely failure modes and how they will be detected

| Failure mode | Detection method |
|---|---|
| JSON parse error (malformed response) | Parsing fails before hson-live ingestion. Failure is detected immediately. |
| Type conversion mistakes (e.g., `1` becomes `"1"`) | Compare output structure against original JSON via structural equality. Assertion fails if values differ. |
| Missing or extra properties | Count properties in converted output against original object keys. Mismatch → test fails. |
| DOM not rendered or empty | Query rendered DOM; verify element count matches expected structure. |
| Structural invariant violation | Validation checks catch structural inconsistencies during conversion. |

---

## 5. Measurement

### Primary metrics
- **Transformation success rate:** Percentage of test cases (fixtures) where JSON conversion completes without error.
- **Data fidelity:** Boolean pass/fail for each fixture: Does the output structure contain all original values with correct types?
- **DOM projection success:** Boolean pass/fail: Is the rendered DOM non-empty and does it contain expected element counts?

### Secondary metrics
- **Execution time:** Time from JSON response to DOM render. Conversion should complete within typical interactive response time (no user-perceptible delay).
- **Node graph size:** Count of nodes in the output structure (should grow with JSON complexity).

---

## 6. Report shape

The test report will be stored and retrieved via browser localStorage. Report structure:

```json
{
  "testId": "02",
  "testName": "Server returns JSON",
  "timestamp": "ISO 8601 timestamp",
  "fixtures": [
    {
      "fixtureId": "simple-scalars",
      "transformSuccess": true,
      "dataFidelity": {
        "structurePass": true,
        "originalValueCount": 8,
        "recoveredValueCount": 8,
        "structureMismatch": null
      },
      "domProjection": {
        "rendered": true,
        "elementCount": 12,
        "expectedMinElements": 10
      },
      "executionTimeMs": 42,
      "nodeGraphSize": 15,
      "errors": []
    },
    {
      "fixtureId": "nested-structures",
      "transformSuccess": true,
      "dataFidelity": { ... },
      "domProjection": { ... },
      "executionTimeMs": 128,
      "nodeGraphSize": 67,
      "errors": []
    }
  ],
  "summary": {
    "totalFixtures": 2,
    "successCount": 2,
    "failureCount": 0,
    "overallPass": true
  }
}
```

Report will be accessible via a browser interface (e.g., read from localStorage on a report page; no external URLs or file paths assumed).

---

## 7. Out of scope

- HTML rendering appearance (CSS, layout, visual correctness).
- Websocket or real-time synchronization behavior (not applicable; JSON is static).
- Server-side HSON generation or HSON syntax parsing.
- JSON schema validation or semantic correctness beyond structure.
- Security vetting of JSON payloads (sanitization is separate concern).
- Performance optimization or benchmarking beyond basic timing.
- Conversion to/from HTML or HSON formats (JSON only).
- LiveTree DOM update reactivity (static render only; no mutations tested).
- Round-trip reversibility or JSON regeneration (covered separately).

---

## 8. Tech stack required

- **Frontend framework:** None required (vanilla DOM API sufficient).
- **hson-live:** Standard library exports as documented.
- **Backend:** Endpoint serving JSON (route/contract unspecified).
- **Fetch mechanism:** Browser HTTP API or equivalent.
- **DOM utilities:** Standard DOM API for element creation and insertion.
- **Storage:** Browser localStorage for report persistence.
- **Test harness:** JavaScript test runner with basic assertion support.

---

## 9. Mocking notes

- **JSON fetch:** No mocking required; backend serves real JSON.
- **localStorage:** Use native browser localStorage; no mocking needed.
- **DOM mutations:** Real DOM; no virtual DOM or shadow DOM mocking.
- **JSON conversion:** Not mocked. Test validates real hson-live parsing.
- **Async handling:** Fetch is async; test must await response before parsing.
- **Error handling:** Validation of errors during JSON conversion (malformed input, structural violations).

---

## 10. Assumptions and open questions

### Assumptions made

1. **Backend availability:** A backend endpoint is available and reachable (URL contract not specified; implementation handles routing).
2. **JSON validity:** Test fixtures will be valid JSON per ECMA-404. Malformed JSON is out of scope.
3. **Browser environment:** Test runs in a modern browser with ES2020+ support, localStorage, and HTTP fetch.
4. **Deterministic parsing:** JSON parsing is deterministic; running the same input twice produces identical output structures.

### Scope expansions

None. This test covers exactly what is stated in the assignment: JSON input, hson-live conversion, and rendering without HSON intermediate.

### Open questions

None identified. The assignment is sufficiently scoped, and hson-live's JSON parsing API is documented.
