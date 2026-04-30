# Test 03: Client-side pagination

## 1. Test ID and name

**Test ID:** 03  
**Test name:** Client-side pagination

---

## 2. What is being tested

Capability of hson-live to manage a paginated data interface where:
- Backend provides raw data (JSON array of records) without pagination logic or pre-rendered markup
- Frontend loads the full dataset once and maintains pagination state entirely in the client
- LiveTree updates the rendered view as users navigate pages
- Updates are driven by client-side changes to the node graph

This tests whether hson-live is suitable for applications that require client-side pagination patterns: interactive page navigation, state management across page changes, DOM updates following node graph changes, and performance under repeated re-renders of paginated content.

---

## 3. How it will be tested

**Setup:** A single HTML page using hson-live to implement pagination.

**Instrumentation:** The page reports:
- Time to load and parse the dataset
- Time to render the first page
- Time to render each subsequent page (on user navigation)
- DOM node count at each page
- Listener count at each page (if readable)

**Fixture:** A JSON dataset of records (moderate size, representative of a realistic paginated dataset—such as a user directory or file listing). Each record should contain realistic content (text strings, dates, enumerations, numeric scores) sufficient to demonstrate pagination across multiple pages. The fixture must be complete and loaded once at page startup.

**Pagination interaction:**
- Users must be able to navigate forward and backward through pages
- The current page position must be visible to the user
- Navigation must span at least 10 page transitions per test run

**Flow:**
1. Page loads and fetches the fixture
2. Dataset is parsed and held in memory
3. First page of records is rendered
4. User navigates to another page
5. View updates to show the new page of records
6. Timing is captured at each step
7. Interaction continues for at least 10 page transitions per test run

---

## 4. Likely failure modes and how they will be detected

| Failure | Detection method |
|---------|------------------|
| Memory leak on repeated page updates | Heap size tracked during page transitions; size should remain stable ±10% across transitions |
| Listener accumulation on re-renders | Listener count (if accessible via DevTools or instrumentation) should not increase after initial page render |
| Sluggish re-render on page change | Render time for subsequent pages should remain ≤150% of first-page render time |
| Incorrect data rendered after navigation | Manual inspection of 5 randomly selected page transitions; verify record ranges match expected offsets |
| DOM not synchronized with node graph | After navigation, query rendered data in DOM and compare against node graph state; mismatch indicates desync |
| Excessive DOM churn | Capture DOM mutation count per page transition; should be proportional to the number of records on the page |

---

## 5. Measurement

**Primary metrics:**
- Initial load + parse time (ms)
- First-page render time (ms)
- Subsequent page render time (ms, aggregate: mean, p95, p99 across transitions 2–10+)
- Heap size before first render and after each major checkpoint (bytes)
- DOM mutation count per page transition

**Success thresholds:**
- Render time must not degrade by >25% across the observation period
- Heap size must remain within ±15% of initial post-render size
- DOM mutations per page transition should scale linearly with the number of records on the page

**Instrumentation method:** Timing captured via browser timing APIs; memory samples collected at checkpoints during the test run.

---

## 6. Report shape

**Storage:** A JSON file (written to localStorage or fetched from a backend endpoint), structured as:

```
{
  "testId": "03",
  "timestamp": "ISO 8601",
  "environment": {
    "browserUserAgent": "string",
    "pageUrl": "string (pathname only, no hostname)"
  },
  "fixture": {
    "recordCount": number,
    "pageTransitionsObserved": number
  },
  "results": {
    "loadParseTimeMs": number,
    "firstPageRenderMs": number,
    "subsequentPageRenderMs": {
      "mean": number,
      "p95": number,
      "p99": number,
      "samples": [number, ...]
    },
    "heapSizeBytes": {
      "initial": number,
      "checkpoint1": number,
      "checkpoint2": number,
      "checkpointN": number
    },
    "domMutationCount": {
      "mean": number,
      "samples": [number, ...]
    }
  },
  "notes": "string (qualitative observations)"
}
```

**Access:** A single-page HTML report (embedded or fetched from backend) displays the JSON data in tabular and chart form.

---

## 7. Out of scope

- Server-side pagination logic (backend returns only full dataset)
- Filtering, sorting, or search within the dataset
- Infinite scroll or virtual scrolling techniques
- Accessibility features (ARIA attributes)
- Styling beyond functional readability
- Mobile or responsive behavior
- Network latency simulation
- Dataset modifications (add/delete/edit records during pagination)
- Integration with advanced styling features

---

## 8. Tech stack required

None specified. hson-live (npm package) and standard browser APIs are sufficient.

---

## 9. Mocking notes

No mocking required. HTTP request to fetch the JSON fixture is real. No websockets, SSE, or non-standard backend behavior is involved.

---

## 10. Assumptions and open questions

### Assumptions made

1. **Dataset size and scope:** A moderate-sized dataset (in the range that produces multiple pages) is representative of a realistic paginated dataset. The implementing agent will define the specific record count and page size to meet this criterion.

2. **Page transitions:** At least 10 page transitions is sufficient to detect memory leaks and render time degradation. Longer observation (e.g., 20 or 40 transitions) may reveal additional patterns but is not required.

3. **Heap measurement:** Accurate heap snapshots may require manual intervention or browser API access. The test allows fallback to periodic sampling at test checkpoints.

4. **Single hson-live implementation:** The test measures hson-live's pagination behavior in isolation; no control implementation or comparative baseline is required.

### Scope expansions

None. The spec covers only pagination state and rendering performance as described in the assignment.

### Open questions

1. **Heap measurement frequency:** Should heap be sampled at every page transition or at intervals (e.g., every 5th page transition)? Every Nth page reduces overhead but may miss transient leaks. Recommend sampling at regular intervals for the first run; follow-up runs can increase frequency if instability is suspected.

2. **Listener detection method:** Browser DevTools exposes listener counts inconsistently across vendors. If automatic detection is unavailable, the test may report "listener count: not available" and rely on visual inspection of DevTools heap snapshots post-test.

3. **Report consumption:** Should the JSON report be embedded in the HTML page (inlined), fetched via fetch() after the page completes, or written to localStorage and read by a separate viewer page? Recommend localStorage for simplicity.

4. **Statistical rigor:** Are mean/p95/p99 sufficient, or are outliers and min/max also expected? Recommend mean/p95/p99 as baseline; raw samples array allows later analysis.
