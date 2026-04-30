# Test 08: JS-free Pages

## 1. Test ID and name

**ID:** 08  
**Name:** JS-free Pages

---

## 2. What is being tested

Whether hson-live can sustain an application pattern where:
- All business logic and state live on the server (PHP backend).
- The client contains no author-written JavaScript—only hson-live library code and infrastructure.
- User interactions (clicks, form submissions, etc.) trigger HTTP requests to the server.
- The server responds with updated HSON representing the new page state.
- hson-live receives the HSON, projects it to DOM, and updates the live DOM in place.

The test maps the failure surface of this pattern: where it works, where it breaks, and what constraints exist when no client-authored JS is available to handle nuances beyond HSON structure.

---

## 3. How it will be tested

### Setup
- Deploy a backend with endpoints that accept interaction events and return HSON.
- Build one or more HTML pages with hson-live injected as the only client library.
- Pages contain no `<script>` tags beyond the hson-live entry point.
- A fixture set defines typical app patterns: form submission, button interactions, conditional rendering, list updates, navigation state.

### Interaction Harness
- Browser automation navigates to pages and records initial page state.
- Test driver intercepts HTTP requests to verify structure (method, path, body content).
- Mock server responses using fixture HSON (pre-recorded or generated).
- Verify DOM updates match expected HSON-to-DOM projection.

### Instrumentation
- Log all HTTP requests and responses.
- Capture DOM snapshots before and after each interaction.
- Track which interactions succeeded (HTTP completed, DOM updated) and which failed.
- Record browser console errors and warnings during test execution.

### Pages and Scenarios
- **Counter page:** Button increments a count on the server; page updates on each click.
- **Form submission page:** Text input and submit button; server validates and echoes back state or error message.
- **Conditional visibility:** Server returns different HSON based on boolean state; client projects without writing any conditional logic.
- **List with dynamic content:** Server returns updated list; client appends/removes DOM nodes per HSON structure.
- **Focus and input preservation:** User types in a text field, submits, server returns updated HSON; measure whether input value/focus is correctly preserved or lost.

---

## 4. Likely failure modes and how they will be detected

### 4.1 Input Value Loss on Update

**Symptom:** User types in a text field, submits a form. Server returns HSON with updated page state but the new HSON does not include the text field value the user had just entered.

**Detection:**
- Record the input value before submission.
- After server response and DOM update, query the input element.
- Compare the new value against the user's entry.
- Fail if value is empty or differs from expected state.

### 4.2 Focus Loss After DOM Replacement

**Symptom:** User focuses an input, triggers an interaction, server returns HSON. hson-live replaces the DOM, focus is lost (no element is focused afterward).

**Detection:**
- Set focus to a known input before interaction.
- After HSON update, detect which element has focus.
- Fail if the focused element is not the expected input or if the body is in focus state.

### 4.3 Scroll Position Lost

**Symptom:** User scrolls to a section, triggers an interaction, server returns HSON. hson-live updates DOM; scroll position resets to top.

**Detection:**
- Measure scroll position to a target before interaction.
- After HSON update, measure scroll position.
- Fail if scroll position resets to 0 (or near 0).

### 4.4 Partial Updates Not Supported

**Symptom:** Server returns a full page HSON for every interaction. But the test expects partial updates (only changed subtrees sent); hson-live has no mechanism to apply a "patch" and must replace entire subtrees instead.

**Detection:**
- Design a scenario where only a small region of the page changes (e.g., one list item or button label).
- If test requires partial update semantics and hson-live only supports full-tree replacement, this pattern will fail for large pages (performance/flickering concern).
- Measure DOM diff size: if the entire page is replaced when only one node should change, flag as inefficiency.

### 4.5 Event Listeners Not Bound to Replaced DOM

**Symptom:** hson-live attaches listeners to initial DOM nodes. Server returns HSON; hson-live projects new DOM with new nodes. But if the new HSON describes the same structure, listeners are bound to old nodes, not new ones.

**Detection:**
- Establish a listener on a button before any interaction.
- Trigger an interaction that causes the button to be replaced (new HSON with same button tag).
- Try to click the button in its new position.
- Fail if the listener does not fire.

**Note:** This is a fundamental issue: HSON nodes have no mechanism to encode "attach a listener" directives. Listeners must be attached via JavaScript. If the page replaces the button, the old listener is lost unless the client re-attaches it.

### 4.6 Custom JavaScript for Complex Interactions

**Symptom:** The app requires client-side logic beyond structure projection (e.g., client-side validation, custom animation, drag-and-drop, real-time filtering without server roundtrip).

**Detection:**
- Design a test that requires such behavior (e.g., "disable submit button until two fields are filled").
- Attempt to implement using only HSON and server state.
- If impossible without client JavaScript, fail and document as out-of-scope.

---

## 5. Measurement

### Success Metrics
- **Interaction completion rate:** Percentage of interactions (clicks, form submissions, navigations) that result in a successful HTTP request and DOM update without error.
- **DOM match rate:** Percentage of HSON-to-DOM projections that match expected structure (tags, attributes, content) with no stale DOM nodes or missing children.
- **Input preservation rate:** For form-based scenarios, percentage of interactions where user-entered data is preserved after server update (when the server echoes it back in HSON).
- **Listener re-attachment:** Number of interactions that require re-binding listeners due to DOM replacement (flagged as a workaround, not a success).
- **Time to interaction response:** Milliseconds from click to DOM update (includes network latency and rendering).

### Failure Metrics
- **Unhandled errors:** Count of JavaScript errors, uncaught exceptions, or console warnings during test execution.
- **DOM mismatches:** Count of interactions where the resulting DOM does not match expected HSON structure (wrong tags, missing attributes, incorrect content).
- **Listener failures:** Count of interactions where a bound listener does not fire on the updated DOM node.

---

## 6. Report shape

Reports are embedded in an HTML page served from the test folder. The page includes:

**Summary Section**
- Test date, duration, test environment (URL).
- Pass/fail verdict for each scenario.
- Overall interaction completion rate and DOM match rate.

**Per-Scenario Breakdown**
- Scenario name, description, expected behavior.
- List of interactions performed (button click, form submit, etc.).
- For each interaction: HTTP request details (method, path, payload), response HSON summary, DOM changes (before/after snapshots).
- Any errors or warnings captured in the console during that interaction.
- Listener binding status (if re-attachment was required).

**Failure Details**
- For each failed interaction or unmatched DOM: side-by-side comparison of expected vs. actual state.
- Stack traces or console logs if available.
- Recommendation: which constraint or failure mode was violated.

**Performance Metrics**
- Timeline of interaction latencies.
- Counts of DOM mutations per interaction.
- Network request size and count.

**Constraints and Limitations**
- Explicit listing of app patterns that this test does NOT cover (e.g., real-time bidirectional communication, client-side only validations).
- Recommendation for where client JavaScript would be required to augment the pattern.

---

## 7. Out of scope

- **Real-time / push notifications:** This test does not cover WebSocket or Server-Sent Events. All communication is request-response HTTP.
- **Client-only state management:** The test assumes all state lives on the server. Client-side transient state (e.g., animation frames, temporary UI toggling) is not tested.
- **Custom event handling:** The test does not author JavaScript event handlers beyond those needed to send HTTP requests. Complex interactions (drag-and-drop, multi-touch gestures, keyboard shortcuts) require client code and are not tested.
- **Accessibility testing:** The test does not measure WCAG compliance or assistive technology compatibility.
- **Performance optimization:** No load testing or stress testing. Test runs with small data sets and single-user interactions.
- **CSS or animation side effects:** This test does not verify that CSS animations, transitions, or media queries work correctly after HSON updates (only structural DOM correctness).
- **Browser compatibility:** Tests run against a single browser / version unless explicitly noted otherwise.

---

## 8. Tech stack required

### Frontend
- hson-live (specified version from root package).
- Browser with DOM API and standard event model (Chrome, Firefox, Safari, Edge).
- Browser automation tooling.

### Backend
- HTTP server capable of returning HSON.
- Endpoint contract: POST requests to a designated path (e.g., `/interact`), request body is form data or JSON with interaction details, response is HSON or JSON representation of HSON.

### Test Infrastructure
- Node.js runtime (or equivalent) to run test harness.
- HTTP client library for mocking or proxying requests (native fetch, axios, or equivalent).
- DOM snapshot / diff library (optional; can use string comparison of serialized HSON or HTML).

---

## 9. Mocking notes

### Server Responses
- Responses are pre-recorded HSON fixtures or generated from a simple state machine.
- Mocking does not require a real backend during development; test can run with a mock endpoint that returns fixture HSON based on interaction event keys.
- If a real backend is used, responses should be captured during a "recording" run and replayed during "replay" test runs to ensure deterministic behavior.

### Network Failures
- Optional: Include mock scenarios for HTTP errors (404, 500, timeout) to verify graceful degradation (e.g., error message shown in place of content).
- For initial test runs, assume 100% success rate; fault injection can be added later.

### DOM Event Triggering
- Browser automation simulates user interactions (click, text input, form submit) via native DOM APIs or equivalent automation commands.
- No custom event mocking required; standard browser events suffice.

---

## 10. Assumptions and open questions

### 10.1 Assumptions made

1. **Server has a single endpoint or a small set of endpoints** that accept interaction events and return updated HSON. The exact URL structure is deployment-agnostic; the test expects an endpoint contract, not a fixed path.

2. **hson-live is configured and injected before any user interactions occur.** The test assumes a single entry point that mounts the initial page.

3. **HSON does not encode event handler callbacks or listener directives.** The test acknowledges that HSON is a structural representation only. Client-side listener attachment must be done via JavaScript (the hson-live infrastructure provides mechanisms for this). The "no client-authored JS" constraint means the test harness (not the app itself) handles listener attachment for interaction triggering.

4. **Form values and input focus are managed by the DOM directly.** The server does not encode "focus on this input" or "preserve this input's value" in HSON; the client must manage these via DOM APIs or hson-live state helpers.

5. **All test interactions are synchronous (request → response → DOM update).** No queued interactions or concurrent updates.

6. **The backend is stateless or session-aware but does not require persistent TCP connections.** The test does not cover WebSocket or streaming scenarios.

7. **QUID (stable node identity) is available to match old DOM nodes with new HSON nodes** when determining which listeners need re-attachment or which form values to preserve.

### 10.2 Scope expansions

1. **Listener binding and re-attachment:** The test explicitly checks whether event listeners survive DOM projection (a known constraint in hson-live). This is technically a separate concern from HSON structure projection but is critical to understanding where the "JS-free" pattern breaks down.

2. **Input and focus preservation:** User-centric concerns (input value, focus state) are not part of HSON spec itself but are critical to app usability. The test treats these as measurable failure modes.

### 10.3 Open questions

1. **How should the test harness attach listeners for initial page interactivity if no client JS is allowed?**
   - Current understanding: The test harness itself is JavaScript (browser automation); it attaches listeners programmatically to trigger HTTP requests. The "no client-authored JS" constraint applies to the app code itself, not the test infrastructure.
   - Clarification needed: Does "no client-authored JavaScript" mean the entire page must have zero JavaScript runtime, or only that the app developer does not write handlers?

2. **Should the test cover partial HSON updates, or assume full-page HSON on every interaction?**
   - Current understanding: Each server response is a full HSON tree. Partial updates are not tested.
   - Clarification needed: Does hson-live support merging partial HSON patches into a live tree, or must it replace the entire tree?

3. **What is the expected behavior when QUID changes or nodes are not matched across updates?**
   - Current understanding: If QUID is not present, hson-live may not correctly track which old DOM node corresponds to which new HSON node, leading to lost listeners and focus.
   - Clarification needed: Does the backend HSON response include QUIDs, or are they assigned client-side? If client-side, do QUIDs persist across server responses?

4. **Should the test include scenarios where the server response is an error (e.g., HTTP 400 validation error) and the client must display it without crashing?**
   - Current understanding: Out of scope for the initial test (assume 100% success). Fault scenarios can be added in a later phase.
   - Clarification needed: If included, what HSON structure represents an error state, and how should hson-live render it?

5. **Is the backend a requirement, or can the test backend be any HTTP server that produces HSON?**
   - Current understanding: Any HTTP server is acceptable; the test is deployment-agnostic.
   - Clarification needed: Should the spec include a minimal example, or is the endpoint contract sufficient?

6. **How should the test handle form serialization (urlencoded vs. JSON) and multipart data (e.g., file uploads)?**
   - Current understanding: Initial test focuses on simple text inputs and button clicks. File uploads are out of scope.
   - Clarification needed: Should form submission scenarios include validation, and what format should validation errors be returned in (HSON, JSON, or HTML)?

7. **Should listener re-attachment be measured as a failure, a workaround, or a known limitation to document?**
   - Current understanding: Treated as a failure mode and documented as a constraint of the pattern.
   - Clarification needed: If hson-live has a built-in mechanism to re-attach listeners after DOM updates, should the test expect it to be used, or is manual re-attachment acceptable?
