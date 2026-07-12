Critic found issues in one or more sub-audits:

## BASELINE_REVIEW

### BASELINE_REVIEW — sample 1/2

Now I have a thorough understanding of the entire codebase. Here is my complete code review:

---

## Code Review: Emmy

**Scope reviewed**: All 16 source files under `src/`, plus config files and workflow.

---

### 1. Environment / Platform Correctness

**Result: PASS** — The project correctly targets a browser-only runtime (`ssr: false`, `adapter-static`). 

- No Node.js globals (`Buffer`, `process`, `require`, `__dirname`, `fs`) appear in any client-shipped code. The only `process.env` reference is in `svelte.config.js`, which runs at build time in Node.
- `atob`/`btoa` in `github.js` are available in all modern browsers. ✅
- `createImageBitmap` in `upload.js` has a fallback for Safari/older browsers. ✅
- `localStorage` access is properly guarded with `typeof localStorage !== 'undefined'`. ✅

---

### 2. Hardcoded Values

**Result: PASS** — All branch names are fetched dynamically via `getDefaultBranch()` (with caching). Raw URLs are constructed from dynamic `owner`/`repo`/`branch` values. No hardcoded branch names, IDs, or URLs.

---

### 3. Security Basics

**Result: PASS** — The SVG XSS vector via `{@html}` is properly mitigated:
- `escapeAttr()` in `annotations.js` escapes `&`, `"`, `'`, `<`, `>` for all attribute values interpolated into SVG markup.
- `escapeXML()` handles text content.
- The token is stored in `localStorage` by design, with the tradeoff documented both in the UI modal and README. Acceptable for this single-user context.

No other secrets in client code. `crossOrigin = 'anonymous'` is set for canvas image loads. No overly permissive CORS patterns.

---

### 4. Remediation Verification

All 10 remediation items are correctly applied:

| # | Item | Status |
|---|------|--------|
| 1 | Decouple repo from PAT | ✅ Connect modal accepts repo without token |
| 2 | Allow gallery loading without auth | ✅ `loadGallery()` called with just `$githubRepo` |
| 3 | Gate write UI behind PAT | ✅ Upload/annotate/delete gated behind `$isConnected` |
| 4 | Deploy workflow branch trigger | ✅ `[main, master]` |
| 5 | GIF tags user-configurable | ✅ `gifTags` input in export section |
| 6 | Escape SVG attribute values | ✅ `escapeAttr()` in all interpolated positions |
| 7 | Static import of `getFile` | ✅ Moved to top-level import |
| 8 | Guard `handleKeydown` against inputs | ✅ Early return on INPUT/TEXTAREA |
| 9 | Dismiss text popup on tool switch | ✅ All tool buttons set `showTextInput = false` |
| 10 | Fix delete button logic inconsistency | ✅ Modal button no longer pre-closes before `handleDelete` |

---

### 5. New Issues Found

**ISSUE 1 — Touch + mouse event duplication in AnnotationEditor**  
**File:** `src/lib/components/AnnotationEditor.svelte`, lines 252-258

The SVG overlay registers **both** `onmousedown`/`onmousemove`/`onmouseup` **and** `ontouchstart`/`ontouchmove`/`ontouchend` event handlers. On touch devices, browsers synthesize mouse events from touch events, causing `handlePointerDown` to fire twice per gesture.

**How it breaks:** The touchstart handler correctly sets `startX`, `startY` to touch coordinates. The synthesized mousedown then fires and **overwrites** `startX`, `startY` with coordinates from the mouse event coordinate space (slightly different position due to how the browser maps them). For the `freehand` tool, `currentShape` is completely overwritten, losing the initial path data. The resulting shape's origin is shifted, and its extent is corrupted by interleaved touch and mouse coordinate updates during the drag.

**Fix:** Use Pointer Events API (`onpointerdown`/`onpointermove`/`onpointerup`) which unifies mouse and touch, or prevent default on touch events to suppress the synthesized mouse events, or add a guard that ignores synthetic mouse events after a touch event has been received.

---

**ISSUE 2 — Slideshow page requires auth for read-only access**  
**File:** `src/routes/slideshow/+page.svelte`, line ~35

```js
$effect(() => {
    if ($githubToken && $githubRepo) {
        loadEntries();
    }
});
```

The slideshow builder page gates the entry-loading `$effect` behind both `$githubToken` **and** `$githubRepo`. A user who has configured a repo name without a token (read-only mode) sees a perpetually loading spinner and cannot access the slideshow builder at all, even though building and previewing a slideshow is a read operation.

The export step (the write operation) is correctly gated behind `$isConnected` in the template — so the UX is inconsistent: the loading should work for read-only users, and only the export button should require auth.

Compare with the gallery page (`+page.svelte`, line ~63) which correctly only checks `$githubRepo`:
```js
$effect(() => {
    if ($githubRepo) {
        loadGallery();
    }
});
```

**Fix:** Change the `$effect` guard to `if ($githubRepo)` (removing the token check), since `fetchGallery` already works with an unauthenticated Octokit client for public repos.

---

**ISSUE 3 — Unused `strokeWidth` parameter in `arrowHead()`**  
**File:** `src/lib/components/AnnotationEditor.svelte`, module context, ~line 385

```js
function arrowHead(x1, y1, x2, y2, strokeWidth) {
    const angle = Math.atan2(y2 - y1, x2 - x1);
    const headLen = Math.min(12, ...);  // strokeWidth is never used
    ...
}
```

The `strokeWidth` parameter is accepted and passed by callers but never referenced in the function body. The arrow head size is always determined by a fixed formula regardless of line width. Callers pass `shape.strokeWidth || 2` as the argument, believing it affects the head — but it has no effect.

The same function in `annotations.js` (`renderShapeSVG` for `'arrow'` case) has the same logic inline and also doesn't factor `strokeWidth` into the head size, but at least doesn't have a misleading parameter.

**Fix:** Either use `strokeWidth` to scale the arrow head size (e.g., multiply `headLen` by a factor based on `strokeWidth`), or remove the parameter entirely for clarity.

---

**Summary:** The codebase is generally well-structured with proper error handling, loading states, and async safety. The three issues above are:

1. **Touch/mouse event conflict** — A real platform-compatibility bug on touch devices (medium severity)
2. **Auth gating inconsistency** — Contradicts the read-only browsing principle (low-medium severity)
3. **Dead parameter** — Cosmetic clarity issue (low severity)

RESULT: ISSUES


### BASELINE_REVIEW — sample 2/2

I now have a thorough understanding of all the code. Let me compile my complete findings.

---

## Code Review: Emmy

I've read every source file in the workspace. The previous audits found issues that have been remediated (PAT/repo decoupling, SVG escaping, deploy workflow branch, GIF tags input, static import, keyboard guard, delete logic). I'm evaluating **independently** for anything that remains.

### 1. Environment/Platform Correctness → CLEAN

The project correctly targets browser-only runtime (`ssr: false`, `adapter-static`, no API routes). No Node.js globals (`Buffer`, `process`, `require`, `__dirname`, `fs`) appear in any browser-shipped code. `process.env` in `svelte.config.js` runs at build time only. `localStorage` access is guarded with `typeof` checks. The `orchestrator.cjs` is a standalone Node script not imported by any app module. ✅

### 2. Hardcoded Values → CLEAN

All branch names are fetched dynamically via `getDefaultBranch()`. Raw URLs are constructed from `owner`, `repo`, and the dynamic branch. The one hardcoded URL (`github.com/settings/tokens/new?scopes=repo&description=Emmy`) is a legitimate deep-link per the spec. Deploy workflow now triggers on `[main, master]`. GIF tags are user-editable. ✅

### 3. Security Basics → CLEAN

- **PAT storage**: localStorage by design, documented in both the UI modal and README — correct for this single-user context.
- **XSS in SVG overlays**: The remediation correctly added `escapeAttr()` to all interpolated attribute values in `renderShapeSVG`. Text content is escaped via `escapeXML()`. Svelte's auto-escaping protects the editor's template-rendered SVG. I verified all three injection points (gallery `{@html}`, slideshow `{@html}`, editor template) are safe.
- **CORS**: `crossOrigin = 'anonymous'` is correctly set on canvas image loads for GIF export.
- **No secrets in source**: No hardcoded tokens, keys, or credentials anywhere in the codebase.

✅ No outstanding security issues.

### 4. Framework Footguns → CLEAN (with observations)

- **No `<button>` inside `<form>` without type**: There are no `<form>` elements anywhere in the app. All buttons use `onclick` handlers. ✅
- **Async error handling**: Every async operation (`handleUpload`, `handleEditSave`, `handleDelete`, `handleExport`, `saveAnnotations`, `loadGallery`, `loadEntries`) has proper try/catch, loading states, and error display. ✅
- **Missing error/loading states**: All async paths have both loading and error states wired. ✅
- **The `handleKeydown` guard**: Properly returns early for INPUT/TEXTAREA targets. ✅
- **Text input popup dismissal on tool switch**: All tool buttons set `showTextInput = false`. ✅
- **Delete button logic**: `handleDelete` guard (`if (detailEntry?.id === id) closeDetail()`) is now reachable — modal button no longer pre-closes. ✅

### 5. New Issues Found

These are issues that were not flagged in the remediation or prior audits:

---

**ISSUE A — GIF export text annotation rendering does not match SVG preview**

**Files:** `src/lib/gif-export.js:163-171` (`drawAnnotationsOnCanvas`, text case)  
vs `src/lib/annotations.js:97-99` (`renderShapeSVG`, text case)  
vs `src/lib/components/AnnotationEditor.svelte:293-300` (editor template, text case)

There are **three different rendering paths** for text annotations, and they produce different visual results:

| Path | Stroke applied? | Fill behavior for transparent fill |
|---|---|---|
| AnnotationEditor SVG template | **No** — no `stroke` or `stroke-width` attributes | Text uses `strokeColor` as fill |
| Gallery/detail SVG overlay (`renderAnnotationsSVG` via `{@html}`) | **Yes** — `stroke` and `stroke-width` from shared `style` variable | Text uses `strokeColor` as fill |
| GIF export canvas (`drawAnnotationsOnCanvas`) | **Yes** — explicit `ctx.strokeText()` call after `ctx.fillText()` | `fillStyle = strokeColor`, then both `fillText` AND `strokeText` called |

In the canvas path, for **transparent-fill** text:
```js
if (isTransparent(shape.fillColor)) {
    ctx.fillStyle = shape.strokeColor;   // fill = stroke color
}
ctx.fillText(shape.text || '', x, y);
ctx.strokeText(shape.text || '', x, y); // also drawn with stroke color + default lineWidth=1
```
This renders the text twice with the same color, making it **visibly bolder** than the SVG version which renders once.

For **filled** text, `strokeText` adds a 1px outline that doesn't exist in the editor SVG template (which has no `stroke` attribute on `<text>`).

**Impact:** GIF exports will show text annotations with a different appearance (bolder / outlined) than what the user sees in the annotation editor or gallery preview. The spec (§4 Phase 3) requires annotations to be "burned into the frame" with annotations preserved, but does not specify they must visually match the overlay — however, the mismatch is a clear user-facing quality issue.

---

**ISSUE B — Dynamic imports of local modules create unnecessary async boundaries**

**Files:**
- `src/lib/upload.js:197` — `const { putBinaryFile, putFile, getDefaultBranch } = await import('./github.js');`
- `src/lib/gif-export.js:303-305` — `const { putBinaryFile, putFile, getDefaultBranch } = await import('./github.js');` and `const { generateImageId, buildMetadata, blobToBase64 } = await import('./upload.js');`
- `src/lib/gif-export.js:232` — `const { GIFEncoder, quantize, applyPalette } = await import('gifenc');` (gifenc is an external module, this one is acceptable as it enables code splitting)

The dynamic imports of **local modules** (`./github.js`, `./upload.js`) serve no purpose — there are no circular dependencies, and these modules will be bundled by Vite anyway. The SvelteKit/Vite build produces a single SPA bundle or a few chunks; dynamic imports of local files just add an async failure mode with no benefit. These should be static imports at the top of the file for consistency with the rest of the codebase.

---

**ISSUE C — Slideshow play interval is not cleaned up on component destroy**

**File:** `src/routes/slideshow/+page.svelte:86-103`

The `setInterval` created in `startPlay()` (line 90) is only cleared by `stopPlay()`, which is called when the user clicks pause or the playback completes its loop. If the user navigates away from the page while playing (e.g., clicks "← Back" at line 186, or "Edit Annotations" at line 353), the interval continues running:

```js
playInterval = setInterval(() => {
    const nextIndex = (previewIndex + 1) % slides.length;
    if (nextIndex === 0) stopPlay();
    else previewIndex = nextIndex;
}, slides[previewIndex]?.delay || defaultDelay);
```

No `onDestroy` lifecycle hook is used. This causes:
- A memory leak (the interval + closures holding references to component state)
- Continued mutations to `isPlaying`, `previewIndex` after the component is unmounted

**Fix:** Import `onDestroy` from `'svelte'` and add a cleanup:

```js
import { onDestroy } from 'svelte';
// ...
onDestroy(() => {
    if (playInterval) {
        clearInterval(playInterval);
        playInterval = null;
    }
});
```

---

**ISSUE D — Dead code: `getLatestCommitSha` exported but never used**

**File:** `src/lib/github.js:214-228`

```js
export async function getLatestCommitSha(octokit, owner, repo) {
    try {
        const resp = await octokit.rest.repos.getCommit({ owner, repo, ref: 'HEAD' });
        return resp.data.sha;
    } catch {
        return null;
    }
}
```

This function is exported but never imported anywhere in `src/`. It's dead code that adds unnecessary maintenance surface and API usage to an otherwise tight module.

---

**ISSUE E — `arrowHead` logic is duplicated**

**Files:**
- `src/lib/annotations.js` (inline in `renderShapeSVG` arrow case, lines ~110-114)
- `src/lib/components/AnnotationEditor.svelte:380-390` (module-level `arrowHead` function)

The same arrowhead triangle calculation exists in two places. Any change to arrow rendering (e.g., head size, angle, styling) must be manually synchronized. The function in `annotations.js` is the canonical SVG renderer; the template in `AnnotationEditor.svelte` uses `arrowHead()` in two places (lines 283, 331) to compute the `<polygon points="...">` for arrowheads in the interactive editor. This should be exported from `annotations.js` and imported in the editor component.

---

### Summary

| Category | Finding | Severity |
|---|---|---|
| Rendering bug | **Issue A**: GIF export text annotations render differently (bolder/outlined) than the editor/gallery SVG overlay — three inconsistent rendering paths for the same data | Medium |
| Code quality | **Issue B**: Unnecessary dynamic imports of local modules in `upload.js` and `gif-export.js` | Low |
| Memory leak | **Issue C**: Slideshow play interval not cleaned up on component destroy | Low |
| Dead code | **Issue D**: `getLatestCommitSha` exported but never used | Low |
| Code quality | **Issue E**: `arrowHead` logic duplicated across two files | Low |

No environment/platform mismatches, no hardcoded values that should be dynamic, no security vulnerabilities, and no framework footguns related to form submission, promise handling, or missing error states.

RESULT: ISSUES
