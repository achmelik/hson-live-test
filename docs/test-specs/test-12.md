# Test 12: Real-world HTML round-trip

## 1. Test ID and name
Test ID: 12
Test name: Real-world HTML round-trip

## 2. What is being tested
Structural stability and reversibility of hson-live's HTML parsing and serialization when applied to actual website HTML. The test verifies that:
- Actual HTML can be parsed into HsonNode internal format without crashing or data loss
- The node tree can be serialized back to HTML
- The output HTML preserves the essential structure, attributes, text content, and ordering of the input

The test does NOT validate visual rendering, CSS resolution, JavaScript behavior, or form functionality. It validates the transformation layer only.

Input HTML sources are not custom-built (not purpose-built fixtures). Each source category is chosen to exercise different structural patterns found in production HTML.

## 3. How it will be tested

### Test pages (6 required categories, specific URLs selected by implementing agent)
1. **News article**: Long-form text with mixed inline/block elements, media embeds (img, figure), bylines, timestamps
2. **E-commerce product page**: Forms (input, select, textarea), data attributes, microdata (schema.org or similar), images with alt text
3. **Documentation page**: Code blocks (pre/code), tables, anchor links (internal navigation), lists
4. **Blog post with embedded media**: Iframes (video embeds), oEmbed markers or similar external media, text flow around embeds
5. **SVG-heavy page**: Significant SVG content (inline or img), geometry attributes, gradients, groups
6. **Web components page**: Custom elements (defined tags), slot markers, shadow DOM references (if exposed via HTML source)

### Test procedure
1. **Fetch phase**: Retrieve raw HTML source from each URL via HTTP GET. Request must respect robots.txt and user-agent headers; use a reasonable browser user-agent.
2. **Parse phase**: Parse the HTML string into the framework's internal structure (HsonNode). Capture any parsing errors, warnings, or exceptions.
3. **Serialize phase**: Serialize the internal node structure back to an HTML string. Capture any serialization errors or warnings.
4. **Diff phase**: Compare input and output HTML using a diff tool (selection is the implementing agent's decision). Diff must report:
   - Missing elements (tags present in input, absent in output)
   - Extra elements (tags in output, absent in input)
   - Attribute mismatches (attribute presence, name, or value divergence per element)
   - Text content mismatches (whitespace normalization rules per agent decision)
   - Child ordering mismatches (element order within parent)
5. **Aggregation**: Collect results across all 6 pages. Calculate per-page success rate (percentage of elements/text nodes without divergence) and overall aggregate success rate.

### Fixtures required
- No synthetic fixtures. Real URLs are required to meet the "real-world" criterion.

### Instrumentation
- Capture and log parse phase duration (ms) per page
- Capture and log serialize phase duration (ms) per page
- Count of mismatches per category (missing, extra, attribute, text, ordering)
- Total bytes input vs. output per page (as-is, not normalized)

## 4. Likely failure modes and how they will be detected

### Parse-phase failures
- **Malformed or non-standard HTML**: Pages with unclosed tags, misnested elements, or browser-specific quirks may fail XML-like parsing. Detection: exceptions during HTML ingestion.
- **Entity expansion or character encoding issues**: Non-ASCII text, rare entities, or mismatched charset declarations. Detection: garbled text or parse errors on specific strings.
- **Unsupported content**: Doctypes, processing instructions, or other non-element nodes. Detection: missing doctype or other metadata in output.

### Serialize-phase failures
- **Attribute or element name escaping**: Special characters in attribute names, values, or tag names. Detection: malformed HTML in output (e.g., unescaped quotes in attributes).
- **Whitespace handling**: Collapsing of significant whitespace in pre/code blocks or text nodes. Detection: text content mismatch in diff phase.
- **Void element serialization**: Self-closing tags (img, br, input) may not serialize correctly. Detection: extra closing tags or missing /> markers.

### Round-trip mismatches
- **Normalization effects**: hson-live may normalize spacing, reorder attributes, or expand shorthand. Detection: attributes in different order or whitespace differences.
- **Content wrapping**: Mixed content (text + elements) may be restructured. Detection: text nodes appear in different positions or within different parents.
- **Media and dynamic content**: Iframes, scripts, and external embeds may be sanitized or preserved inconsistently. Detection: iframe src attributes changed or scripts removed.
- **Boolean attributes**: HTML5 boolean attributes (checked, disabled, etc.) may be expanded to name="name" form or vice versa. Detection: attribute value changes.
- **SVG namespace handling**: SVG elements may lose or gain xmlns declarations or namespace prefixes. Detection: xmlns attributes appearing/disappearing.
- **Web component markers**: Custom element tags should survive; slot and shadow DOM markers may not round-trip losslessly. Detection: custom tags present but attributes or children altered.

## 5. Measurement

### Per-page metrics
- **Bytes in / bytes out**: Raw serialized sizes (not normalized for whitespace)
- **Parse time (ms)**: Duration of HsonNode creation
- **Serialize time (ms)**: Duration of HTML string generation
- **Mismatch count**: Number of structural/attribute/text mismatches detected by diff tool
- **Success rate**: Percentage of top-level elements and text nodes that matched (calculated as: (total nodes - mismatches) / total nodes * 100)

### Aggregate metrics
- **Overall success rate**: Mean success rate across all 6 pages
- **Pages fully lossless**: Count of pages with 100% success rate
- **Category performance**: Success rate grouped by page category (news, e-commerce, documentation, blog, SVG, web components)
- **Timing summary**: Min/max/mean parse and serialize times across all pages
- **Common mismatch patterns**: Frequency of mismatch types (missing elements, attribute mismatches, etc.)

## 6. Report shape

Report is generated as a JSON document and consumed by a web-based frontend (in-browser rendering via JavaScript).

### Report structure
```
{
  "testId": 12,
  "testName": "Real-world HTML round-trip",
  "timestamp": "ISO 8601 datetime",
  "overallSuccessRate": number (0-100),
  "pageCount": number,
  "fullyLosslessCount": number,
  "pages": [
    {
      "url": string,
      "category": string (one of: news, ecommerce, documentation, blog, svg, webcomponents),
      "byteInput": number,
      "byteOutput": number,
      "parseTimeMs": number,
      "serializeTimeMs": number,
      "successRate": number (0-100),
      "mismatchCount": number,
      "mismatchesByType": {
        "missingElements": number,
        "extraElements": number,
        "attributeMismatches": number,
        "textMismatches": number,
        "orderingMismatches": number
      },
      "sampleMismatches": [
        {
          "type": string,
          "selector": string (CSS path or XPath to element in input),
          "description": string
        }
      ]
    }
  ],
  "categoryAggregates": [
    {
      "category": string,
      "pageCount": number,
      "meanSuccessRate": number,
      "minSuccessRate": number,
      "maxSuccessRate": number
    }
  ],
  "timingAggregates": {
    "parseTimeMs": {
      "min": number,
      "max": number,
      "mean": number
    },
    "serializeTimeMs": {
      "min": number,
      "max": number,
      "mean": number
    }
  },
  "mismatchPatterns": {
    "type": string,
    "count": number,
    "percentage": number (of total mismatches)
  }
}
```

Report is stored in a location accessible to the test's web frontend (e.g., in localStorage, as a JSON file served by backend, or embedded in an HTML report page). The frontend will retrieve and display the report without requiring server-side aggregation.

## 7. Out of scope

- **Visual rendering validation**: No screenshot or CSS layout comparison
- **JavaScript behavior**: No execution of inline or external scripts; pages may be pre-rendered or scripts removed
- **Form submission or interaction**: Forms are validated structurally only, not functionally
- **Network behavior**: Lazy loading, resource fetching, or DNS resolution not tested
- **Browser-specific DOM quirks**: Test uses hson-live's parsing logic, not browser's HTML5 parser, so behavior may differ from browser rendering
- **Performance benchmarking**: Timing is logged for context but not subjected to formal performance thresholds
- **Accessibility validation**: WCAG, ARIA, or semantic correctness not measured
- **Third-party embed behavior**: External scripts or data-driven content not executed; embeds captured as-is

## 8. Tech stack required

### Frontend
- hson-live 2.1.0 (root export for transformation API)
- Diff tool (agent selects)
- Browser (ESM capable; modern Chrome, Firefox, Safari, or similar)
- JSON serialization (built-in)

### Backend
- PHP (stateless HTTP server; no build step required)
- HTTP client capable of fetching arbitrary URLs
- HTML parsing capable of returning raw source (no DOM mutations)

### Optional
- Report viewer library for category performance visualization (though not required)

## 9. Mocking notes

### Websocket / SSE
Not applicable. No real-time communication required. All data flows are request-response.

### External content
Pages are fetched as-is. External resources (CSS, JavaScript, images) are NOT fetched; only the HTML source is captured. If a page requires JavaScript to render, the test captures the initial HTML state before rendering (static source).

### Robots.txt and rate limiting
Implement reasonable crawling practices:
- Respect robots.txt if present
- Use a descriptive user-agent header
- Implement per-page delays if multiple pages from the same host (agent decision on duration)
- Do not retry failed pages unless explicitly part of the test design

## 10. Assumptions and open questions

### Assumptions made
1. **HTML parsing uses the framework's ingestion path**: The test assumes hson-live's HTML ingestion parses HTML into internal structure. No sanitization is applied to preserve full round-trip fidelity.
2. **Input HTML is UTF-8**: All real-world pages are assumed to be UTF-8 encoded or auto-detected as such. Charset mismatches are not a test focus.
3. **Whitespace normalization is acceptable in output**: Text nodes may have leading/trailing whitespace normalized or collapsed. A mismatch in whitespace alone (e.g., newline vs. space) is counted as a mismatch but not a failure.
4. **URLs are publicly accessible**: All selected page URLs are assumed reachable via HTTP(S) without authentication, CAPTCHAs, or geofencing.
5. **Single-pass transformation**: The test performs HTML → HsonNode → HTML in one pass. Round-trip stability is not measured beyond this single cycle.
6. **Diff tool is deterministic**: The chosen diff tool produces consistent results across multiple runs on the same input pair.

### Scope expansions
None. The test covers only the HTML round-trip transformation and does not extend into LiveTree mutation, DOM projection, CSS management, or animation handling.

### Open questions
1. **URL selection criteria**: The parent spec requires 6 page categories but does not specify exact URLs or quantity per category. Should the agent select 1 URL per category (6 total) or multiple per category (12+)?
2. **Diff granularity**: Should the diff tool measure success at the element level (each element is pass/fail) or text-node level (each text node is pass/fail), or both?
3. **Whitespace handling rules**: Should attribute whitespace (e.g., `  data-x  =  "value"  `) be normalized before diff, or should any spacing change count as a mismatch?
4. **Media embed sanitization**: Should iframes (YouTube, etc.) be expected to survive unchanged, or is removal/alteration acceptable?
5. **Error handling strategy**: If a page fetch fails (HTTP error, timeout), should the test skip that page, retry, or abort entirely?
