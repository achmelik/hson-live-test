# Test 01: Server returns HSON

## 1. Test ID and name

**Test ID:** 01  
**Test name:** Server returns HSON

---

## 2. What is being tested

The path from a PHP backend that produces HSON to a frontend that receives it, parses it, and renders it to the DOM via hson-live.

Specific scope:
- Backend HTTP endpoint responds with HSON syntax (valid according to HSON Spec 0 and 1).
- Frontend receives the response body as a string.
- Frontend parses the HSON string into a parsed structure.
- Frontend renders the parsed structure as a live DOM.
- The rendered DOM is visually and structurally correct and matches the intent of the HSON payload.

This tests the core parsing and rendering pipeline; it does NOT test DOM mutation, event handling, or state synchronization beyond initial mount.

---

## 3. How it will be tested

### Pages and builds
- A single HTML page that acts as the frontend entry point.
- The page includes hson-live as an ESM import.
- The page includes the DOMPurify dependency.
- Build setup is minimal: the page is served with ES modules enabled (e.g., via native ESM or a bundler configured for output).

### Fixtures required
- A valid HSON fixture covering multiple structural patterns:
  - HTML-model HSON (self-closing element syntax), including:
    - Simple text-only elements.
    - Nested elements with child content.
    - Elements with attributes (class, id, data attributes, style strings).
    - Mixed content (text nodes interleaved with element children).
    - Self-closing elements (e.g., `<br/>`, `<img/>`).
    - SVG elements and nested SVG structures if applicable.
  - JSON-model HSON (object/array syntax), including:
    - Nested objects.
    - Arrays.
    - Primitive values (strings, numbers, null, booleans).
  - The fixture should represent realistic document structures (e.g., a form, a list, a card layout) rather than trivial stubs.

- A separate PHP endpoint (deployed locally or remotely) that:
  - Accepts a request (method and path unspecified; describe the contract).
  - Returns the HSON fixture as plain text.
  - The response body is the complete HSON string (no wrapping, no embedded HTML).

### Instrumentation
- A test harness (HTML page) that:
  1. Fetches the HSON response from the backend endpoint.
  2. Parses the response body into a parsed structure.
  3. Renders the parsed structure into a container element in the DOM.
  4. Waits for the DOM to finish rendering.
  5. Captures metrics and visually inspects the rendered output.

---

## 4. Likely failure modes and how they will be detected

| Failure mode | Detection method |
|---|---|
| HSON syntax is invalid or malformed | Parser throws an error; test catches and reports the error message and line number if available. |
| Response body is not HSON (e.g., JSON or HTML) | Parser throws an error attempting to parse non-HSON syntax; error is reported. |
| Parsed structure is empty or null | Test checks that the parsed structure is defined and has a valid element tag; assertion fails if null or undefined. |
| Rendered DOM does not appear in document | Test queries for the container element and its children; element query returns null or child count is 0. |
| Rendered element tags do not match HSON source | DOM inspection compares element tag names to the parsed structure; mismatch reported. |
| Rendered attributes are missing or incorrect | Test reads presence and value of declared attributes on rendered elements and compares to parsed structure; mismatch reported. |
| Rendered text content does not match HSON source | Test compares element text content to the expected content derived from the parsed structure; mismatch reported. |
| Nested structure is flattened or re-ordered | Test traverses the rendered DOM tree and compares depth and sibling ordering to the parsed structure. |
| SVG namespace is incorrect (if SVG is in fixture) | Test checks namespace URI for SVG elements; should be the SVG namespace. |
| Style attributes are not applied | Test checks that style attributes are present on rendered elements and that declared CSS properties are applied. |
| Network error or timeout | Test catches fetch errors; reports endpoint unreachable or request timeout. |

---

## 5. Measurement

Metrics collected per test run:

- **Parse success**: Boolean. True if parsing completes without error.
- **Parse latency**: Time in milliseconds from parse start to completion.
- **Tree mount latency**: Time in milliseconds from structure creation to DOM element connected and visible in the document.
- **DOM structure match**: Boolean. True if rendered DOM tree depth, tag sequence, and sibling count match the parsed structure.
- **Attribute match rate**: Percentage of declared attributes present and correct in rendered elements (attributes found / attributes declared).
- **Text content match**: Boolean. True if all text nodes rendered match the HSON source.
- **SVG namespace correctness** (if applicable): Boolean. True if all SVG elements have correct namespace URI.
- **Style application** (if applicable): Boolean. True if declared CSS properties are correctly applied to rendered elements.
- **Error details**: Any parser errors, DOM query failures, or assertion failures with full stack trace and context.

---

## 6. Report shape

Reports are stored and accessed via the browser (primary mechanism).

Storage and retrieval:
- Test results are JSON objects stored in browser storage at a fixed key per test run (e.g., `hson-test-01-<timestamp>`).
- A simple HTML report page queries storage, retrieves all stored results, and renders them in a table or card view.
- Results include:
  - Test name and ID.
  - Timestamp of run.
  - Pass/fail status (boolean).
  - All metrics from section 5.
  - Error message (if applicable).
  - Human-readable summary (e.g., "Parse success: true, DOM match: true, 95% attributes correct").

No specific report URL or file path is prescribed. The report retrieval logic is part of the test framework and may be embedded in a single HTML page or served separately.

---

## 7. Out of scope

The following concerns are NOT tested here:

- DOM mutation and real-time synchronization (e.g., graph changes updating the DOM) — covered by separate mutation tests.
- Event handling (click, change, input events) — covered by separate event tests.
- CSS animation and transition behavior — covered by separate animation tests.
- Styling beyond static style attribute application — covered by styling tests.
- Data binding and reactive updates — covered by binding tests.
- Security (HTML injection, XSS prevention) — covered by separate security tests.
- Performance under very large documents (>10k nodes) — covered by separate scalability tests.
- Transformation round-trips (HSON ↔ JSON ↔ HTML) — covered by separate transformation tests.
- Diagnostics suite functionality — covered by separate diagnostics tests.

---

## 8. Tech stack required

**Frontend:**
- Browser environment with native ESM support.
- hson-live (ESM export from root or designated subpath).
- dompurify.
- Fetch API or equivalent for HTTP requests.

**Backend:**
- PHP capable of returning plain-text HTTP responses.
- No specific framework required (framework-agnostic endpoint contract).

**Test framework:**
- No external test framework required for basic pass/fail.
- Assertion logic may be custom (vanilla JS) or use a lightweight library.
- Browser storage API for result persistence.

---

## 9. Mocking notes

Real websockets, Server-Sent Events (SSE), or other real-time transports are not available in standard PHP. This test uses HTTP (stateless request/response) only. No mocking is required; the endpoint is a standard HTTP GET or POST that returns HSON in the response body.

If future tests require subscription or real-time updates, a mock or wrapper will be needed; this is documented separately.

---

## 10. Assumptions and open questions

### Assumptions made

1. **HSON fixture is static**: The backend endpoint returns the same HSON payload on every request (no dynamic or randomized generation per request).

2. **Response encoding is UTF-8**: The HTTP response uses UTF-8 character encoding; test assumes standard text decoding is safe without explicit recoding.

3. **No Content-Security-Policy restrictions**: The test page is served without CSP restrictions that would block inline script or dynamic imports of hson-live.

4. **DOM container is available**: The frontend page provides a fixed container element where the structure will be rendered.

5. **Single document per test run**: Each test run receives and renders one complete HSON document from the backend; no multi-document or streaming scenarios are tested.

6. **HSON consistency within fixture**: The fixture uses either HTML-model HSON (self-closing element syntax) or JSON-model HSON (object/array syntax) consistently; mixing models in a single payload is not tested.

7. **No external CSS dependencies**: Styles in the fixture are inline or part of the HSON; no linked stylesheets are required for the test to pass.

8. **Fetch succeeds with valid HTTP**: The backend endpoint responds with HTTP 200 and a valid HSON body; HTTP error codes (4xx, 5xx) and incomplete responses are not explicitly tested.

### Scope expansions

None. This test covers exactly the path described: backend HSON → frontend parse → live DOM render.

### Open questions

1. **Fixture complexity**: Should the HSON fixture include nested arrays or objects (JSON-model HSON)? Or should it focus on HTML-model structures?

2. **SVG support**: Should the HSON fixture include SVG elements and nested SVG trees? The spec mentions SVG support; including it here would verify that claim.

3. **Style validation scope**: Should the test verify computed styles or only check that the style attribute is present? Computed styles require the browser to have fully laid out and rendered, which may add latency.

4. **Error recovery**: If the backend returns invalid HSON (malformed syntax), should the test attempt to recover or should it fail fast?

5. **Timing and assertion**: Should the test use browser rendering events to wait for render completion, or is a fixed delay sufficient? How much time should the test wait before declaring a timeout?

6. **Endpoint contract specifics**: Should the endpoint accept a query parameter to select different HSON payloads? Or should it return a single fixed payload?
