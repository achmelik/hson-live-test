# Test 14: Build Tool Integration

## 1. Test ID and name
**Test ID:** 14  
**Test name:** Build tool integration

## 2. What is being tested

- **Tree-shaking of unused exports.** Verify that when hson-live's public exports are consumed but only a subset are actually used in application code, the unused exports are removed from the final bundle.
- **Code splitting across subpath exports.** Verify that code imported from the subpath exports can be split into separate chunks and that each chunk contains only the code required for its exports.
- **Source map quality.** Verify that generated source maps accurately map bundle code back to original source files, with mappings that resolve to actual source positions.
- **HMR behavior on LiveTree changes.** Verify that the dev server's hot module replacement system correctly updates modules when LiveTree-managed content or its dependencies are modified, and that the DOM reflects changes without a full page reload.
- **Dev server vs production build differences.** Verify that development builds include source maps and unminified code, while production builds are minified and have source maps available via separate files or inline encoding.
- **ESM-only consumption friction points.** Identify and document friction specific to ESM-only consumption, such as:
  - lack of CommonJS support
  - inability to use dynamic `require()` patterns
  - compatibility issues with bundlers that default to CJS detection
  - missing polyfills for ESM-specific features
  - file extension handling in ESM contexts

## 3. How it will be tested

Three independent build tool configurations will be tested. The agent may choose to test one, several, or all of the following: Vite, webpack, Rollup, esbuild.

**For each build tool:**

1. **Create a minimal application project** that imports a small subset of hson-live's exports and uses the framework's main rendering capability. The application must run without errors in both development and production modes.

2. **Generate development and production builds** using the chosen tool with standard configuration (no aggressive optimization flags beyond what the tool enables by default for each mode).

3. **Inspect the resulting bundles:**
   - Extract the bundle size (unminified and minified).
   - For each subpath export tested, measure whether code from unused exports is present in the bundle (via string matching of exported symbols, class names, or function identifiers from the source).
   - Verify that source maps exist and contain mappings for all mapped source files.

4. **Test HMR behavior** by modifying the application's LiveTree-dependent code while the dev server is running. Observe whether the dev tool reports the module as updated and whether the page reflects changes without a full reload.

5. **Verify dev vs production differences:**
   - Confirm that development builds have readable code and embedded or linked source maps.
   - Confirm that production builds are minified and have source maps available.
   - Measure and record the difference in bundle size and load time between dev and production.

6. **Document ESM-only friction points** encountered during setup, configuration, or testing. Record errors or warnings related to module resolution, import handling, and build compatibility issues.

## 4. Likely failure modes and how they will be detected

| Failure mode | Detection method |
|---|---|
| Unused exports not tree-shaken | Bundle inspection: unused export symbols are present in final bundle |
| Code splitting failed | Bundle structure check: all subpath code bundled into a single file instead of separate chunks; or imports from separate paths are not honored |
| Source maps missing or malformed | Verify source map files exist; parse map files and confirm they contain valid JSON with `sources`, `mappings`, and `sourcesContent` arrays |
| Source map lines misaligned | Compare source map mappings against actual source file positions; flag if mappings do not align with source positions to within typical debugger tolerance |
| HMR not triggered | Modify application code, observe dev server output; if no module update message appears or if page requires manual refresh to reflect change, HMR failed |
| HMR triggered but DOM unchanged | HMR reports success but inspected DOM does not reflect the code change after the update completes |
| Dev build not readable | Attempt to search for a known exported function name in dev bundle; if unminified code is not found, dev build is minified when it should not be |
| Production build not minified | Search for function names, variable names, or keywords from source in production bundle; if not minified, readable identifiers appear |
| ESM import failures | Build process fails with errors related to module resolution or import handling in ESM contexts |

## 5. Measurement

**Bundle metrics (per build tool and configuration tested):**
- Unminified bundle size (in bytes).
- Minified bundle size (in bytes).
- Number of output chunks (if code splitting is enabled).
- Size of each output chunk.

**Tree-shaking metrics:**
- Count of unused export symbols present in final bundle (if any).
- Percentage reduction in bundle size if tree-shaking is enabled vs. disabled (if tool supports this comparison).

**Source map metrics:**
- Presence of source map file (yes/no).
- Validity of source map JSON structure (yes/no; report any parse errors).
- Alignment of mappings with original source positions (acceptable or not).

**HMR metrics:**
- Time from file save to module update reported by dev server (in milliseconds).
- Time from module update to DOM change observable (in milliseconds, measured via polling the DOM for the expected change).
- Count of successful HMR updates (without full page reload) over a small set of modifications.

**Dev vs production comparison:**
- Code readability ratio: percentage of identifiable function/variable names in dev bundle vs. production bundle.
- Source map availability: present in dev (yes/no), present in production (yes/no).
- Load time difference: (production load time) - (dev load time) in milliseconds.

**ESM friction points:**
- List of errors or warnings encountered during build setup or bundling, categorized by cause.
- Summary of configuration changes required to make the build succeed.

## 6. Report shape

**Format:** JSON or embedded HTML report viewable in browser.

**Structure:**

```
{
  "testId": 14,
  "testName": "Build tool integration",
  "timestamp": "ISO 8601 datetime",
  "toolsTested": ["Vite", "webpack", ...],
  
  "builds": [
    {
      "tool": "Vite",
      "configuration": "dev" | "production",
      "metrics": {
        "bundleSizeUnminified": <bytes>,
        "bundleSizeMinified": <bytes>,
        "numChunks": <number>,
        "chunks": [
          {
            "name": "main.js",
            "sizeMinified": <bytes>,
            "exports": [<symbol names from this chunk>]
          }
        ],
        "unusedExportsInBundle": [<symbol names that are unused but present>],
        "sourceMapValid": true | false,
        "sourceMapErrors": [<errors if any>],
        "sourceMapAlignment": "acceptable" | "misaligned",
        "codeReadability": "<percent>%"
      }
    }
  ],
  
  "hmrTesting": {
    "tool": "Vite",
    "successfulUpdates": <number>,
    "failedUpdates": <number>,
    "avgTimeToModuleUpdate": <ms>,
    "avgTimeToDomChange": <ms>,
    "hmrWorking": true | false,
    "notes": "..."
  },
  
  "devVsProduction": {
    "tool": "Vite",
    "devMetrics": {
      "codeReadable": true | false,
      "hasSourceMaps": true | false,
      "loadTimeMs": <ms>
    },
    "productionMetrics": {
      "codeMinified": true | false,
      "hasSourceMaps": true | false,
      "loadTimeMs": <ms>
    },
    "loadTimeDifference": "<percent>% faster"
  },
  
  "esmFrictionPoints": [
    {
      "category": "missing file extension",
      "tool": "webpack",
      "description": "Module resolution failed due to missing file extension in import statement",
      "resolution": "Added file extension to import statement"
    },
    ...
  ],
  
  "summary": {
    "overallPass": true | false,
    "toolsWithIssues": [<tool names>],
    "keyFindings": ["..."]
  }
}
```

Report will be stored in a location accessible to the browser (e.g., in the test directory as `report.html` or `report.json`), and a URL or local path will be provided to access it.

## 7. Out of scope

- Performance benchmarking beyond bundle size and load time measurement.
- Testing dynamic imports or code splitting patterns beyond the implicit splitting provided by subpath exports.
- Testing against specific framework integration scenarios (e.g., React, Vue, Svelte). Focus is on the raw bundler behavior, not framework-specific tooling.
- Testing with non-standard bundler configurations or plugins (e.g., custom tree-shaking directives, manual chunk splitting).
- Compatibility testing with specific versions of bundlers. Use current major version of each tool selected.
- Testing security or integrity of minified code.
- Verification of dompurify dependency bundling or tree-shaking (external dependency).

## 8. Tech stack required

**Build tools (agent selects one or more):**
- Vite (with default esbuild backend) OR
- webpack 5+ OR
- Rollup OR
- esbuild

**Dev tooling:**
- A JavaScript runtime supporting modern ESM and a package management mechanism.
- A local dev server for HMR testing (provided by Vite, webpack-dev-server, or equivalent).

**Browser for HMR and report viewing:**
- Any modern browser with DevTools (Chrome, Firefox, Safari, Edge).

**Instrumentation:**
- Bash or Node.js scripts to extract bundle metrics and inspect structure.
- Optional: a source map parsing utility, agent's choice.

## 9. Mocking notes

- **Real websockets not available.** Dev servers (Vite, webpack-dev-server) provide HMR via HTTP polling or long-polling by default. No additional mocking is required; use the bundler's built-in HMR transport mechanism.
- **File watching:** Local file system watchers (provided by bundlers) will be used to trigger recompilation and HMR. No mocking needed.
- **Source map inspection:** Source map files can be read and parsed as JSON from disk; no runtime mocking required.

## 10. Assumptions and open questions

### Assumptions made

1. **Single-page application assumption:** Tests assume a simple HTML page that imports and uses hson-live. Frameworks, routing, or advanced SPAs are out of scope.
2. **Standard bundler defaults:** Configuration will use each tool's default settings for its chosen mode (dev vs. production) without aggressive custom optimization flags (e.g., no aggressive minification plugins beyond standard production settings).
3. **Subpath exports are tested sequentially, not all together:** If all subpath exports are imported into a single app, tree-shaking results will show which exports are removed. But separate test runs with different imports are not required; one bundle can be inspected to measure tree-shaking for all subpaths.
4. **HMR test assumes module isolation:** The modification to LiveTree-dependent code will affect one or a few modules; HMR is expected to update only the changed modules, not trigger a full page reload.
5. **Source map testing does not require debugger integration:** Inspection is via file reading and mapping validation, not via breakpoint stepping.
6. **Bundle inspection via string matching:** No specialized bundle analyzer plugins are required; string matching for export symbols is sufficient to detect unused code.
7. **Dev server runs locally:** The dev server for HMR testing runs on localhost; no remote deployment is required.

### Scope expansions

None. This test covers only the build integration surface described in the assignment.

### Open questions

1. **Multiple bundlers or just one?** The assignment says "agent may test one, several, or all." Should the agent test all four (Vite, webpack, Rollup, esbuild) or is testing one or two sufficient?

2. **Source map validation depth:** Should source maps be tested only for existence and JSON validity, or should additional validation checks be performed?

3. **Code splitting: implicit vs. explicit?** Subpath exports naturally create separate entry points, but should the test also verify code splitting with dynamic imports (`import()`), or only static imports?

4. **HMR test duration:** How many modifications should trigger HMR before declaring success/failure?

5. **ESM friction definition:** Should the test also document build-time vs. runtime ESM friction separately, or is a combined list sufficient?

6. **Report hosting:** Should the report be saved locally in the test directory, or uploaded to a central location?
