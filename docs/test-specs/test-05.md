# Test 05: Canvas (driven by hson and LiveTree)

## 1. Test ID and name

**Test ID:** 05
**Test name:** Canvas (driven by hson and LiveTree)

---

## 2. What is being tested

Whether hson-live's node graph and LiveTree can drive the state and behavior of an HTML canvas element, including:

- State representation: expressing canvas drawing state (width, height, context settings) as hson nodes or node properties.
- State synchronization: ensuring that changes to the node graph or LiveTree properties trigger canvas redraws.
- Interaction handling: whether user interactions on the canvas (mouse events, keyboard) can be captured, processed, and fed back into the node graph to update subsequent state.

Constraint: Canvas state must be actively driven by hson and LiveTree, not merely coexist with them. Having a canvas on the same page as hson-rendered content is insufficient; this tests integration where canvas updates flow from node changes.

---

## 3. How it will be tested

### Pages and fixtures

- A single page hosting a canvas element created or integrated within a LiveTree.
- A backend endpoint that supplies drawing instructions as an HSON or JSON payload.
- The payload describes a sequence of canvas operations (e.g., state of width, height, drawing context properties, and a list of draw commands).

### Instrumentation and setup

1. **Canvas creation:** Canvas is created as a LiveTree node and integrated into the node graph.
2. **State representation:** Drawing state (context properties and command queue) is encoded in the node graph as attributes or child nodes. The schema must represent:
   - Canvas dimensions and metadata.
   - Current drawing context state (fill style, stroke style, line width, etc.).
   - A sequence of drawing commands to be executed.
3. **Redraw trigger:** A change handler observes node mutations and redraws the canvas when attributes or children change.
4. **Interaction capture:** Mouse and keyboard listeners attached to the canvas capture user input, translate events into state updates (e.g., new draw command nodes or property changes), and append them back to the node graph.
5. **Backend integration:** Backend serves drawing instructions as HSON/JSON; frontend parses and integrates instructions into the LiveTree.
6. **Report data:** Test collects metrics on redraw frequency, command execution latency, and interaction handling reliability.

### Fixture requirements

- Drawing instruction payload must cover:
  - Basic context properties (fill style, stroke style, line width, font, etc.).
  - Geometric primitives (filled rectangles, stroked rectangles, arcs, bezier curves, etc.).
  - Text rendering (filled text, stroked text).
  - Image data (if applicable).
  - Transformation matrices (translate, rotate, scale, etc.) if testing advanced features.
- Interaction test fixtures:
  - Click and drag sequences to draw on canvas.
  - Keyboard input to modify drawing state (e.g., change color via key press).

---

## 4. Likely failure modes and how they will be detected

### Failure mode 1: Canvas pixel state is not part of the DOM tree

**Why it fails:** Canvas renders imperative drawing commands to a pixel buffer. The buffer's contents do not appear in the DOM tree and cannot be inspected or mutated by querying nodes. LiveTree and hson operate on node graphs; they have no native way to represent or synchronize arbitrary pixel data.

**Detection:** 
- Attempt to integrate an existing canvas with drawn content into the node graph. Verify whether pixel data is captured. Expect: **no pixel data is captured**; only element metadata (width, height, attributes).
- Attempt to redraw the canvas by mutating the node graph alone (without imperative canvas API calls). Expect: **canvas remains blank or stale**; changes to the node graph do not automatically redraw pixels.

### Failure mode 2: Canvas internal context is not exposed as node properties

**Why it fails:** Canvas 2D context (fill style, stroke style, line width, etc.) is mutable state held in the context object, not in the DOM or node structure. Changing the context does not trigger DOM mutations, and LiveTree's change observers and DOM synchronization do not capture context changes.

**Detection:**
- Create a LiveTree canvas and set context properties via the canvas API (e.g., setting fill style to red).
- Query the node graph for attributes or children representing fill style. Expect: **no representation exists**; the node graph is unaware of context state.
- Attempt to revert a fill style change by modifying the node graph. Expect: **the change is lost**; the context state is not synchronized with the node graph.

### Failure mode 3: Integration does not preserve canvas pixel state

**Why it fails:** When a canvas is integrated into the node graph, the parsing extracts element metadata (tag, attributes, children) but cannot access the canvas's pixel buffer or context state.

**Detection:**
- Draw on a canvas in the DOM.
- Integrate the canvas element into a LiveTree.
- Verify that the LiveTree node includes context state. Expect: **context state is not captured**; only attributes are preserved.

### Failure mode 4: Command representation mismatch

**Why it fails:** Canvas drawing commands are imperative (drawImage, fillRect, etc.) and stateful (context properties persist across calls). HSON nodes are declarative. Representing a sequence of imperative commands as nodes requires encoding semantics (order, context state at each step) that hson does not naturally express.

**Detection:**
- Define a node structure representing a draw command (a node with attributes for command type, x, y, width, height, etc.).
- Implement a redraw handler that iterates the command nodes and executes corresponding canvas operations.
- Change the command node's x or y attribute. Expect: **redraw occurs correctly**, but only if explicit redraw logic is implemented. Without it, **the canvas is stale**.
- Reorder or insert command nodes. Expect: **the redraw behavior depends entirely on custom logic**; hson itself provides no automation.

### Failure mode 5: Event propagation and state coherence under rapid updates

**Why it fails:** Interaction-driven canvas updates (e.g., drawing on mousemove) occur at high frequency. Each event may trigger a node mutation, which may trigger a redraw. The redraw is asynchronous. Multiple events can queue up faster than redraws complete, causing visual lag, duplicate draws, or missed commands.

**Detection:**
- Implement a drag-to-draw interaction on the canvas.
- Drag quickly across the canvas, generating many mousemove events.
- Collect all draw commands executed. Compare against expected sequence. Expect: **commands may be lost, duplicated, or reordered** if event queueing and redraw pacing are not carefully synchronized.
- Measure the time between event and visible redraw (visual latency). Expect: **latency is high or inconsistent** because the redraw pipeline is not optimized for canvas.

### Failure mode 6: Canvas does not support child node semantics

**Why it fails:** Canvas elements cannot have DOM children (they behave like empty elements). Attempting to add child nodes under a canvas node may fail, silently no-op, or produce unexpected results.

**Detection:**
- Create a canvas LiveTree node and attempt to append child nodes. Expect: **error or silent failure**; canvas does not render children.
- Attempt to integrate a canvas with nested HTML children into the node graph. Expect: **children are lost or not integrated** (or parsing fails).

---

## 5. Measurement

### Metrics to collect

1. **Redraw completeness:** Count of draw commands successfully executed on the canvas vs. commands in the node graph.
2. **Redraw latency:** Time between node mutation and visible pixel change on canvas.
3. **Interaction throughput:** Number of user events processed per second during drag-to-draw.
4. **Interaction precision:** Verify that coordinates in draw commands match the screen coordinates of user interaction.
5. **Context state leakage:** Verify that context state (fill style, etc.) does not leak between separate commands or persist incorrectly.

### Success criteria

- All draw commands in the node graph are reflected on canvas within a reasonable latency (within one display frame).
- Canvas state (width, height, context properties) can be mutated via the node graph and the changes are visible on canvas.
- Interaction events are captured, translated to node mutations, and subsequent redraws reflect the user's input without loss or lag.

---

## 6. Report shape

Reports are generated in-browser and stored as JSON in localStorage or embedded in an HTML artifact.

### Report structure

```json
{
  "testId": "05",
  "testName": "Canvas (driven by hson and LiveTree)",
  "timestamp": "ISO8601 timestamp",
  "results": {
    "failureModes": [
      {
        "mode": "failure mode name",
        "tested": true | false,
        "passed": true | false,
        "details": "description of what was tested and outcome"
      }
    ],
    "metrics": {
      "redrawCompleteness": {
        "totalCommands": number,
        "executedCommands": number,
        "completenessPercent": number
      },
      "redrawLatency": {
        "min": number,
        "max": number,
        "mean": number,
        "unit": "ms"
      },
      "interactionThroughput": {
        "eventsPerSecond": number,
        "result": "pass" | "fail"
      }
    },
    "summary": "concise pass/fail summary"
  }
}
```

---

## 7. Out of scope

- Comparing performance of canvas-via-hson against imperative canvas code (no performance benchmarks).
- Advanced canvas features (WebGL, workers, video capture, effects filters).
- Accessibility of canvas content (ARIA, alt text).
- Canvas-as-media-element (video, audio playback inside canvas).
- Testing non-2D canvas contexts (3D, OffscreenCanvas, transferable buffers).
- Security or XSS vectors related to canvas.
- Persistence and serialization of canvas state (reserved for round-trip testing in other tests).

---

## 8. Tech stack required

- **Backend:** PHP (no specific version constraint).
- **Frontend:**
  - hson-live or compatible.
  - Browser with native canvas 2D support (all modern browsers).
  - No external canvas libraries; canvas must be driven directly by hson and LiveTree.
- **Testing instruments:**
  - Browser DevTools or custom instrumentation to measure latency.
  - Event queueing instrumentation to track interaction throughput.
- **Report delivery:**
  - localStorage or sessionStorage for JSON storage.
  - HTML artifact for visual report display.

---

## 9. Mocking notes

- **Real websockets:** Not available. Canvas state updates from the backend are polled or fetched via HTTP GET/POST, not WebSocket push.
- **Backend state sync:** If the test requires synchronizing canvas state with backend, use polling or explicit HTTP requests triggered by user interaction. State changes on canvas are reflected in HTTP POST payloads sent to the backend; backend responds with updated drawing instructions.
- **Animation and timing:** Canvas redraws are scheduled to stay in sync with browser rendering cycles. Redraw scheduling coordinates with the browser's rendering pipeline.

---

## 10. Assumptions and open questions

### Assumptions made

1. A "canvas driven by hson" will require custom application-level redraw logic. hson-live does not automatically execute canvas operations; the test implementation must bridge the gap by listening to node mutations and invoking the canvas API.
2. Canvas state (context properties, command queue) will be encoded as node attributes or child nodes, using a schema designed for this test. The schema is not defined by hson-live and is left to the test implementation.
3. The test will focus on 2D context only; 3D/WebGL is out of scope.
4. Interaction latency is acceptable if within one display frame at typical refresh rates. Faster is better but not required.

### Scope expansions

(None at this time. Interaction handling is explicitly part of the assignment.)

### Open questions

1. **Canvas state representation schema:** How should context state and drawing commands be represented as HSON nodes? As attributes (representing context/metadata), child nodes (representing commands), or a hybrid?
   - Proposed resolution: Node attributes represent canvas metadata and context state (width, height, fill style, stroke style, line width). Child nodes represent commands (each command is a node with a type indicator and attributes for parameters). This is a test-specific schema, not part of hson-live's core.

2. **Redraw pacing:** Should redraws be on-demand (triggered by mutation) or periodic (on a rendering cycle)? On-demand risks race conditions; periodic risks lag. How aggressively should redraws be paced?
   - Proposed resolution: Redraw on the next rendering cycle after any node mutation. Batch mutations within a single rendering cycle into one redraw.

3. **Drawing commands in scope:** Should the test cover advanced drawing operations (transformation matrices, image manipulation, text metrics) or focus on basic geometric primitives and fill/stroke?
   - Proposed resolution: Cover basic operations (rectangles, circles, lines, text, fill, stroke). Advanced transformations are optional; failure to support them is not a test failure.

---

**Document version:** 2.0
**Last updated:** 2026-04-27
