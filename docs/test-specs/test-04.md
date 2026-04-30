# Test 04: Component templates (marketing site)

## 1. Test ID and name

**Test ID:** 04  
**Test name:** Component templates (marketing site)

---

## 2. What is being tested

Whether hson-live patterns support breaking a multi-section marketing site into reusable component files, where each component has isolated HTML structure and CSS styling. Specifically:

- **Reusability:** Can four distinct components (header, footer, hero banner, three-card display) be defined once and instantiated multiple times on the same page without duplication or style collision?
- **CSS isolation:** Does each component's CSS remain scoped to its own elements, preventing CSS from leaking across components?
- **QUID scoping:** Does hson-live's QUID mechanism enable CSS scoping to isolate styles per component instance?
- **LiveTree composition:** Can components be created as separate component trees, then appended into a parent structure to form a complete page?

The test does NOT evaluate:
- Component template syntax or special declarative language (hson-live uses standard HTML/HSON).
- Slot mechanisms, prop drilling, or state management.
- Performance characteristics or rendering speed.
- Server-side template processing (this test executes client-side only).

---

## 3. How it will be tested

### Setup phase

1. **Component definitions:** Create four HTML component templates (header, footer, hero banner, card component):
   - Each component is standalone HTML with semantic structure (e.g., `<header>`, `<section>`, `<article>`).
   - Each has its own scoped CSS rules targeting its elements.
   - CSS includes at least one property unique to each component to detect CSS leaking to other components.
   - CSS may include pseudo-classes (`:hover`, `:active`) or pseudo-elements (`::before`, `::after`) to test scoping on pseudo rules.

2. **Backend endpoint:** PHP backend exposes an endpoint that returns component templates as HSON/JSON. Endpoint returns each component independently and/or as a list.

3. **Frontend page:** Single-page frontend loads hson-live:
   - Fetches component templates from the backend.
   - Parses each template into a separate component tree.
   - Appends each component into a container under a root tree.
   - Each appended component receives a unique identifier assigned (automatic per-instance assignment).
   - Frontend renders the tree to the page.

4. **Styling:** Component CSS is applied via the framework's CSS management system, with rules scoped to individual component instances. CSS rules must be scoped per component to prevent cross-component interference.

### Test execution

1. **Initial render:** Verify that all four components are visible and styled correctly in the browser DOM.

2. **CSS scoping validation:**
   - Inspect the rendered stylesheet and confirm that each CSS rule is associated with a unique component instance identifier.
   - Verify that a property set on one component instance does not appear on another component instance.
   - Confirm pseudo-rules (if present) are also scoped to the component instance that owns them.

3. **Multiple instantiation:**
   - Append the same component template twice to the page (hero banner, for example).
   - Confirm that each instance receives a distinct identifier.
   - Modify CSS on one instance and verify the other instance is not affected.
   - Inspect the generated CSS rules and confirm separate scoped rules for each instance.

### Instrumentation

- Console logging: Log identifier assignments as components are appended.
- DOM inspection: Verify instance identifiers are present and correct.
- Stylesheet inspection: Extract and log the generated CSS text before and after mutations.
- JSON snapshots: Log component node graphs to verify structural integrity.

---

## 4. Likely failure modes and how they will be detected

| Failure mode | Detection method |
|---|---|
| CSS leaks across components | One component's CSS property appears on another component in computed style or visual inspection. Inspect stylesheet scoping rules; if a property lacks proper instance scoping or targets a broad selector, report as leak. |
| Identifier not assigned to component | Instance identifier missing from DOM element or identifier registries empty. Log identifier values during append; if undefined or empty, flag failure. |
| CSS rules target wrong instance | A scoped rule targets one instance's identifier but that property does not apply to the correct element. Inspect stylesheet source and compare against runtime CSS state. |
| Pseudo-rules not scoped | A rule like `:hover { ... }` appears in stylesheet without instance scoping, or pseudo-selector suffix is missing. Inspect all `:hover`, `:active`, `::before`, `::after` rules; each must have full instance scoping. |
| Multiple instances collide | Two instances of the same component template receive the same identifier, or CSS rules from one affect the other. Log identifier for each instance; confirm they differ. Modify one instance's CSS and snapshot the stylesheet; confirm the other instance's rules unchanged. |
| Component CSS is inline instead of scoped | CSS appears in `style="..."` attribute on elements instead of in a stylesheet. Inspect generated HTML; if components have inline styles, flag as failure. Confirm CSS is managed by the stylesheet scoping system. |

---

## 5. Measurement

### Pass criteria

1. **All four component templates load and render** without JavaScript errors.
2. **Each component instance receives a distinct, non-empty identifier** on first append or projection.
3. **All CSS rules in the stylesheet are scoped to component instances** via the framework's scoping mechanism.
4. **No CSS property from one component instance appears on another component instance** (verified via stylesheet inspection and computed styles).
5. **Pseudo-rules (`:hover`, `::before`, etc.) are fully scoped** to their component instance, not global.
6. **Multiple instances of the same component template are assigned distinct identifiers** and their CSS does not collide.

### Measured values

- **Component count:** 4 (header, footer, hero, three-card display).
- **CSS rule count:** Count of total CSS rules generated; expect one rule per distinct instance + property combination, plus pseudo-variants.
- **Instance identifier count:** Count of unique identifiers assigned to all component instances on the page.
- **Mutation success rate:** Percentage of CSS property changes that successfully apply (target 100%).
- **Multi-instance collisions:** Count of instances where two components received the same identifier (target 0).

---

## 6. Report shape

A JSON report object stored in browser `localStorage` with key `hson_test_04_report`:

```json
{
  "testId": "04",
  "testName": "Component templates (marketing site)",
  "timestamp": "2026-04-27T15:30:00Z",
  "results": {
    "componentsLoaded": {
      "header": true,
      "footer": true,
      "heroBanner": true,
      "threeCardDisplay": true
    },
    "instanceIdentifierAssignment": {
      "identifiersCount": 4,
      "identifiersUnique": 4,
      "identifiersPerComponent": {
        "header": "abc123def456",
        "footer": "xyz789uvw012",
        "heroBanner": "pqr345stu678",
        "threeCardDisplay": "lmn901opq234"
      }
    },
    "cssScoping": {
      "scopedRulesCount": 18,
      "unscopedRulesCount": 0,
      "pseudoRulesScopedCount": 6,
      "pseudoRulesUnscopedCount": 0,
      "cssLeaksDetected": false,
      "cssLeakDetails": []
    },
    "multiInstanceTest": {
      "instanceCount": 2,
      "distinctIdentifierCount": 2,
      "collisionsDetected": false,
      "cssCollisionDetails": []
    }
  },
  "summary": {
    "passed": true,
    "failureReasons": [],
    "warnings": []
  }
}
```

Report is generated client-side and written to localStorage. If a UI/dashboard consumes it, the mechanism is a simple fetch to `window.localStorage.getItem('hson_test_04_report')` and JSON.parse().

---

## 7. Out of scope

- Server-side rendering or templating languages (SSR, Handlebars, EJS, etc.).
- Reactive data binding or state synchronization between instances.
- Performance benchmarks (render time, memory usage, CSS bundle size).
- Accessibility testing or semantic validation.
- Browser compatibility (test assumes modern ES2020+ browser with WeakMap, crypto, and standard DOM APIs).
- Nested component composition (components containing other components; only flat composition on a single page).
- Dynamic component loading from external URLs or modules (components are fetched once from the backend endpoint).
- CSS animations or transitions (structural and state are tested; animation execution is not).
- Component documentation generation or meta-information (e.g., prop tables).
- HTML round-trip serialization and re-mutation (covered by Test 12).

---

## 8. Tech stack required

| Layer | Technology |
|---|---|
| Frontend | hson-live JavaScript library with component tree support |
| CSS manager | Framework's CSS management system with instance-scoped styling |
| DOM binding | Framework's LiveTree DOM projection with automatic instance assignment |
| Testing approach | Browser test harness with DOM inspection and stylesheet analysis capabilities |
| Backend | PHP endpoint serving component templates as HTML, JSON, or HSON |
| Report storage | Browser `localStorage` (required); optional: JSON file export via browser download |

---

## 9. Mocking notes

1. **WebSockets:** Not used. No mocking required.
2. **SSE (Server-Sent Events):** Not used. No mocking required.
3. **Backend fetch:** Standard HTTP GET/POST to the PHP endpoint returning component templates. If testing without a real backend, mock the fetch response with static template strings.
4. **DOM:** Real browser DOM. No mocking of DOM APIs.
5. **Crypto API:** hson-live uses `crypto.getRandomValues()` to mint identifiers; fallback to timestamp-based IDs if crypto is unavailable. No mocking required; hson-live handles both paths.
6. **CSS injection:** CSS management system automatically creates and manages a stylesheet in the document. No external stylesheet loading; all CSS is generated in-memory.

---

## 10. Assumptions and open questions

### Assumptions made

1. **Component HTML is trusted:** The test assumes component templates received from the backend are trusted HTML (not user-generated or adversarial). dompurify is a runtime dependency but is not explicitly tested.

2. **Single-page deployment:** The test assumes a single HTML page that loads hson-live once. No multi-document or iframe scenarios.

3. **Deterministic identifier generation:** Instance identifiers are generated via cryptographic random values (or timestamp-based fallback). Test assumes identifiers are stable during a single page load and unique across all instances.

4. **No external CSS files:** The test does not load external stylesheets. All CSS is generated by the CSS management system and injected into a managed stylesheet.

5. **Components are acyclic:** No component contains another component (flat composition only). Nested composition is out of scope.

6. **Identifier stability:** Once assigned, instance identifiers are preserved and associated with their component instances throughout the page lifetime.

7. **CSS property coverage:** The test does not require exhaustive CSS property testing; a representative sample (background color, font size, padding, border, pseudo-class opacity) is sufficient.

8. **Browser environment:** Test runs in a modern browser with ES2020+ support (WeakMap, Optional chaining, Nullish coalescing, etc.).

### Scope expansions

1. **CSS pseudo-class and pseudo-element testing:** The assignment specifies "each component has its own CSS" but does not detail pseudo-rule coverage. This spec tests `:hover`, `:active`, `::before`, and `::after` to validate full instance-scoped CSS.

2. **Multi-instance collision detection:** The assignment does not explicitly require testing multiple instances of the same component; this spec includes that to validate instance identifier uniqueness and CSS isolation across instances.

### Open questions

None. The spec covers component decomposition, instance-scoped CSS isolation, composition via LiveTree append, and complete transformation through multiple instantiation. All mechanisms (instance identifiers, CSS management, scoped CSS rules) are documented in hson-live's source and API docs.
