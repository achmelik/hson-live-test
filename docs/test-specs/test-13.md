# Test ID and name
Test 13: Web components and custom elements interop

---

# What is being tested

Whether LiveTree's parse and serialize cycle preserves custom element integrity when:
- Parsing HTML markup that contains registered custom elements
- Serializing that markup back to HTML
- Rendering the result as a live DOM

Specifically, the test validates that:
1. Custom element tag names (kebab-case names) are preserved exactly
2. Custom element attributes are accessible and reactive to changes made through LiveTree's attribute mutation API
3. Custom element lifecycle callbacks are fired in expected sequence during DOM rendering
4. Shadow DOM and slot projection structures remain intact and functional
5. Attribute changes via LiveTree mutators trigger attributeChangedCallback on the custom element
6. The custom element can coordinate its internal state with external mutations via LiveTree

---

# How it will be tested

Three separate pages will be created, each embedding a different category of custom element. The test does not pre-register components; the test HTML will assume they are already loaded and registered globally.

## Page 1: Vanilla Web Component
Markup containing a single vanilla (native Web Components API) custom element with:
- Registered tag name (e.g., `my-counter`)
- Constructor that establishes initial `count` property
- connectedCallback that logs a trace event
- disconnectedCallback that logs a trace event
- Static `observedAttributes` array listing one or more attributes (e.g., `["value"]`)
- attributeChangedCallback that updates internal state and re-renders content when observed attributes change
- Shadow DOM with internal slot for projected content

## Page 2: Lit-based Component
Markup containing a Lit component (Lit 3.x or compatible) with:
- Reactive properties and attribute bindings
- Lifecycle hooks that fire at expected times
- Slot-based content projection
- Attribute binding and reactivity

## Page 3: Stencil-built Component
Markup containing a Stencil component with:
- Reactive properties and state
- Lifecycle hooks that fire at expected times
- Slot-based content projection
- Host element styling

## Test flow (all pages)

1. **Load page** - HTML is loaded; components auto-register or are already registered globally
2. **Add to LiveTree** - Parse the page into the framework's live DOM
3. **Record baseline** - Capture initial DOM state, component lifecycle events, and attribute values
4. **Mutate via LiveTree** - Use the framework's attribute mutation API to change an observed attribute
5. **Verify reactivity** - Check that:
   - The DOM updates (custom element re-renders)
   - attributeChangedCallback fires (if not Lit/Stencil, which may abstract it)
   - Internal state changes are reflected in the DOM
6. **Serialize to HTML** - Serialize the live tree back to HTML
7. **Compare markup** - Check that re-emitted HTML includes the custom element tag, attributes, and slot structure
8. **Verify slot projection** - Confirm that slotted content remains in place and slot elements are not mutated

## Fixtures required

- A JavaScript bundle (or inline `<script>`) defining one vanilla Web Component that satisfies the criteria in "Page 1" above.
- A JavaScript bundle defining a Lit component that satisfies the criteria in "Page 2" above.
- A JavaScript bundle defining a Stencil component that satisfies the criteria in "Page 3" above.
- Three separate HTML files, each importing the respective component(s) and embedding markup that uses them.

No literal example payloads are provided; implementations must define working components with the stated properties.

---

# Likely failure modes and how they will be detected

## Failure mode 1: Custom element tag name not preserved in serialization
**What it is:** After parse and serialize, the custom element tag becomes a standard HTML tag or is malformed.
**How detected:** After serializing to HTML and parsing the result, check that the tag name matches the original (e.g., `my-counter` remains `my-counter`).

## Failure mode 2: Attributes on custom elements are lost or corrupted
**What it is:** Attributes passed to the custom element in source HTML are dropped or mangled during parse/add/serialize.
**How detected:** Compare the original source markup attributes with the re-emitted HTML. Verify via LiveTree's attribute API that attributes remain accessible.

## Failure mode 3: attributeChangedCallback not fired on LiveTree attr mutations
**What it is:** Changes made via LiveTree's attribute mutation API do not propagate to the underlying DOM element's attributeChangedCallback.
**How detected:** Instrument the custom element to log calls to attributeChangedCallback. Mutate via LiveTree and check that the callback log has new entries.

## Failure mode 4: Shadow DOM is flattened or lost
**What it is:** The shadow root structure is not preserved; shadow children appear in light DOM or are dropped.
**How detected:** Query the custom element's `shadowRoot` after adding to LiveTree and rendering. Verify structure and children match expectations.

## Failure mode 5: Slot projection breaks
**What it is:** Slotted content is not projected into slots correctly; content appears in the wrong DOM location or is duplicated/missing.
**How detected:** Query slot elements and their assigned nodes. Verify that `slot.assignedNodes()` returns the expected light DOM children.

## Failure mode 6: Lifecycle callbacks fire at wrong times or not at all
**What it is:** connectedCallback, disconnectedCallback, or similar do not fire during add/rendering.
**How detected:** Instrument the custom element to log lifecycle events. Add to LiveTree and verify event order and counts match expectations.

## Failure mode 7: Attribute reactivity breaks on Lit/Stencil components
**What it is:** Lit reactive properties or Stencil props do not update when the HTML attribute changes.
**How detected:** For Lit, check that re-renders occur and rendered output updates. For Stencil, verify that lifecycle hooks fire and re-renders occur.

## Failure mode 8: Serialization includes unwanted Shadow DOM internals
**What it is:** Shadow DOM children or host styles leak into the serialized HTML.
**How detected:** Serialize to HTML and verify that shadowRoot elements do not appear as siblings or children in the light DOM output.

---

# Measurement

## Success criteria (all must pass)

1. **Tag preservation:** Original and re-emitted custom element tag name are identical.
2. **Attribute round-trip:** All attributes present in source are present after parse-add-serialize; attribute values match.
3. **Lifecycle firing:** connectedCallback fires exactly once during add; if elements are removed, disconnectedCallback fires.
4. **Attribute reactivity:** Setting an observed attribute via LiveTree's attribute mutation API triggers the custom element's attributeChangedCallback and updates its rendered output.
5. **Shadow DOM integrity:** Shadow DOM structure is preserved after rendering; `shadowRoot` remains accessible and unmodified.
6. **Slot projection:** Slot assignment works after rendering; `slot.assignedNodes()` returns expected light DOM children.
7. **Serialization safety:** Serialized HTML does not include shadow DOM children, host styles, or internal component state as light DOM nodes.

## Counts to track

- **Lifecycle events fired:** Number of connectedCallback, disconnectedCallback, attributeChangedCallback invocations for each custom element instance.
- **Attribute mutations:** Number of times attribute mutations are applied; corresponding count of attributeChangedCallback fires.
- **Shadow DOM queries:** Number of successful `shadowRoot` access queries after rendering.
- **Slot queries:** Number of successful `slot.assignedNodes()` calls returning correct node counts.

---

# Report shape

A JSON report containing:

```json
{
  "test_id": 13,
  "test_name": "Web components and custom elements interop",
  "timestamp": "ISO 8601",
  "pages": [
    {
      "page_id": "vanilla",
      "component_type": "vanilla web component",
      "tag_name": "my-counter",
      "results": {
        "tag_preserved": true/false,
        "attributes_round_trip": {
          "source_attrs": {"count": "5", "label": "Counter"},
          "regrafted_attrs": {"count": "5", "label": "Counter"},
          "match": true/false
        },
        "lifecycle_events": {
          "connected_callback_count": <number>,
          "disconnected_callback_count": <number>,
          "attribute_changed_callback_count": <number>,
          "expected_count": <number>,
          "match": true/false
        },
        "attribute_reactivity": {
          "mutations_applied": <number>,
          "callbacks_fired": <number>,
          "dom_updates": <number>,
          "all_match": true/false
        },
        "shadow_dom": {
          "accessible": true/false,
          "structure_intact": true/false,
          "children_count": <number>
        },
        "slot_projection": {
          "slots_found": <number>,
          "assigned_nodes_correct": true/false,
          "details": ["slot name", "assigned node count"]
        },
        "serialization": {
          "shadow_children_leaked": true/false,
          "tag_preserved": true/false,
          "attributes_preserved": true/false
        }
      }
    },
    {
      "page_id": "lit",
      "component_type": "lit component",
      "tag_name": "<tag name>",
      "results": { /* same shape as above */ }
    },
    {
      "page_id": "stencil",
      "component_type": "stencil component",
      "tag_name": "<tag name>",
      "results": { /* same shape as above */ }
    }
  ],
  "summary": {
    "total_checks": <number>,
    "passed": <number>,
    "failed": <number>,
    "failure_categories": ["tag_preservation", "attribute_reactivity", ...]
  }
}
```

Reports are stored as JSON files in a test-local results directory and optionally parsed and displayed in an in-browser HTML dashboard (e.g., localStorage-backed or fetched from a JSON endpoint).

---

# Out of scope

- Custom elements that do not follow the Web Components standard (non-standard registration APIs).
- Server-side rendering or pre-rendered shadow DOM snapshots.
- Comparing LiveTree with other DOM libraries or frameworks.
- Testing event listener binding on custom elements (covered by general event handling tests).
- Testing CSS custom properties or inheritance through shadow boundaries.
- Testing form submission or form-associated custom elements.
- Testing performance or memory usage of custom element instantiation at scale.
- Testing custom elements inside iframes or cross-origin contexts.

---

# Tech stack required

- **Frontend:**
  - hson-live
  - Three working custom element implementations:
    - Vanilla Web Components (no framework)
    - Lit (3.x compatible)
    - Stencil (latest compatible)
  - HTML/CSS/JS test harness to load components and run test flow

- **Backend:**
  - PHP endpoint(s) serving:
    - Three separate HTML test pages (each with the respective custom element markup)
    - Component registration scripts (optional; may be inline in HTML)
    - JSON endpoint for storing/retrieving test results (optional; can use localStorage)

- **Test runtime:**
  - Browser with support for Shadow DOM, DOM querying, and mutation observation
  - ES2020+ JavaScript support
  - DOM parsing and serialization capabilities

---

# Mocking notes

- Custom element lifecycle callbacks are mocked by wrapping the component or instrumenting via a test helper that logs calls to a trace array.
- attributeChangedCallback is logged directly; no mocking needed if the custom element is instrumented in its definition.
- Shadow DOM is not mocked; it is native browser functionality.
- Slot projection is native; no mocking required.
- If the backend does not support websockets, result persistence is via localStorage (client-side) or JSON POST endpoints (server-side).

---

# Assumptions and open questions

## Assumptions made

1. **All three component types are already defined and registered globally via `customElements.define()`** The test assumes components are loaded and their registration completed by the time the test harness starts.
2. **Custom elements are vanilla Web Components API compliant** (not transpiled or polyfilled in a way that breaks standard lifecycle).
3. **Shadow DOM is supported by the test runtime** (modern browser with DOM Level 4+).
4. **The components do not require a build step or bundler.** They are either pre-built or provided as plain JS files.
5. **`observedAttributes` is correctly declared** in vanilla components; Lit and Stencil are assumed to expose attribute reactivity via their standard mechanisms.
6. **Slot assignment is deterministic.** Slotted content is stable and does not change during the test.
7. **The test harness can instrument custom elements** by wrapping their lifecycle methods or listening to their events; no access to internal component state is required.
8. **Serialization preserves custom element tag names verbatim** (no lowercasing or normalization that would break kebab-case tags).
9. **`customElements.get()` can be used to retrieve and verify registration state** of each custom element.

## Scope expansions

None. This spec covers exactly the intersection of "parse custom elements," "mutate via LiveTree," "render to live DOM," and "serialize back to HTML."

## Open questions

1. **Should the test verify that slot names are preserved in re-emitted HTML?** For example, if the source has `<slot name="header">`, should the re-emitted HTML also have `<slot name="header">`? This may depend on whether the custom element's shadow DOM is serialized at all.

2. **Is comparing old vs. new DOM state via serialization a valid round-trip test, or should the test re-parse the serialized HTML and render it again to a fresh DOM?** The spec assumes the former; clarify if the latter is required.

3. **For Lit and Stencil components, should the test verify framework-specific lifecycle or only the Web Components standard API (attribute changes)?** The spec assumes framework-specific hooks are nice-to-have, not required.

4. **Should the test verify that mutating the tree and then re-serializing produces HTML that reflects those mutations?** Or is serialization only tested on the initial render without intermediate mutations?

5. **If a custom element has slots with default content, should that default content appear in the serialized output?** Or is only the projection (assigned nodes) expected?
