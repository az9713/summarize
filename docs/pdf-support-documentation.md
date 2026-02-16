# PDF URL Support for Chrome Extension — Implementation & Troubleshooting Log

## 1. What Was Changed

### Problem Statement

When a user navigated to a PDF URL (e.g., `https://arxiv.org/pdf/2507.11538`) and clicked Summarize in the Chrome extension, the extension tried to extract text via its content script DOM extraction. Chrome's built-in PDF viewer renders PDFs in an embedded `<embed>` element with no accessible text in the DOM, so the extraction returned empty or garbage content (e.g., "S M O" — 3 words, 23 chars). The LLM then had nothing meaningful to summarize.

The CLI already handled PDFs correctly via its asset pipeline, but the daemon's URL flow (used by the extension) only supported HTML pages.

### Changes Made (5 files modified, across 3 layers)

#### Layer 1: Core Library — PDF URL Detection

**File: `packages/core/src/content/url.ts`**

Added `isPdfUrl()` function that detects PDF URLs by:

- Checking if the pathname ends with `.pdf` (case-insensitive)
- Special-casing `arxiv.org/pdf/<id>` URLs which serve PDFs without a `.pdf` extension

Updated `shouldPreferUrlMode()` to include `isPdfUrl(url)` in the OR chain, so PDF URLs are recognized as needing the URL-mode fast path (same treatment as YouTube, Twitter, podcasts).

**File: `packages/core/src/content/index.ts`**

Added `isPdfUrl` to the barrel export from `"./url.js"`.

#### Layer 2: Chrome Extension — Route PDFs Through URL Fast Path

**File: `apps/chrome-extension/src/entrypoints/background.ts`**

Three changes:

1. **Import:** Added `isPdfUrl` to the import from `@steipete/summarize-core/content/url`.

2. **Fast path detection (line ~1149):** Updated `wantsUrlFastPath` to trigger on PDF URLs:

   ```typescript
   const wantsUrlFastPath =
     Boolean(tab.url && (isYouTubeWatchUrl(tab.url) || isPdfUrl(tab.url))) &&
     opts?.inputMode !== "page" &&
     prefersUrlMode;
   ```

3. **Status messages (lines ~860, ~1156):** Show "Downloading PDF…" instead of "Fetching transcript…" or "Extracting video transcript…" when the URL is a PDF.

#### Layer 3: Daemon — PDF Content Extraction & Summarization

**File: `src/daemon/summarize.ts`**

This was the critical fix. The daemon has two functions that process URLs:

1. **`extractContentForUrl()`** — Used for extract-only mode. Added a PDF interception block before `runUrlFlow()` that:
   - Checks `isPdfUrl(url) && hasUvxCli(env)`
   - Downloads the PDF bytes via `fetch()`
   - Converts to markdown via `convertToMarkdownWithMarkitdown()` (which calls `uvx markitdown`)
   - Returns a fully-populated `ExtractedLinkContent` object

2. **`streamSummaryForUrl()`** — Used by the Chrome extension's URL fast path for streaming summaries. Added the same PDF interception that:
   - Downloads and converts the PDF (same as above)
   - Fires the `onExtracted` hook and writes input metadata
   - Builds the prompt via `buildUrlPrompt()`
   - Calls `summarizeExtractedUrl()` to stream the LLM summary
   - Returns metrics

Both functions bypass `runUrlFlow()` entirely for PDFs, since `runUrlFlow` calls `fetchHtmlDocument` which rejects `application/pdf` content-type responses.

#### Tests

**File: `tests/content.url.test.ts`**

Added 10 test cases:

- `isPdfUrl()`: `.pdf` extension, case-insensitive, query params, fragments, arxiv special case, non-PDF rejection, false positive rejection, malformed URLs
- `shouldPreferUrlMode()`: PDF URL assertions

---

## 2. Problems Encountered & Resolutions

### Problem 1: Extension Shows "Extracting video transcript…" for PDFs

**Symptom:** First test showed "Extracting video transcript…" status text and 401 errors.

**Cause:** There were two places in `background.ts` that set status text for URL-mode extraction. The initial change only updated one location (the main summarize flow). A second location in the chat/agent extraction path (line ~860) still showed "Extracting video transcript…".

**Fix:** Updated both status message locations to check `isPdfUrl(tab.url)` and show "Downloading PDF…" for PDFs.

### Problem 2: 401 Unauthorized — Daemon Token Mismatch

**Symptom:** Extension showed "401 Unauthorized" error when trying to summarize.

**Cause:** The Chrome extension and daemon authenticate via a shared Bearer token stored in `~/.summarize/daemon.json`. After reinstalling the extension, it generated a new token, but the daemon was still running with the old token.

**Resolution steps:**

1. Removed and reinstalled the Chrome extension
2. Copied the new token from the extension settings page
3. Updated `~/.summarize/daemon.json` with the new token value
4. Restarted the daemon (which required killing the old process first — see Problem 3)

### Problem 3: EADDRINUSE — Old Daemon Process Lingering

**Symptom:** `daemon restart` or `daemon run` failed with `listen EADDRINUSE: address already in use 127.0.0.1:8787`.

**Cause:** The old daemon process was still running and holding port 8787.

**Resolution:**

1. Found the PID: `netstat -ano | grep 8787` → identified PID
2. Killed it: `taskkill //F //PID <pid>`
3. Started fresh: `node dist/cli.js daemon run`

Note: On Windows/Git Bash, `taskkill` flags use `//` prefix (e.g., `//F //PID`) because `/` is interpreted as a path separator.

### Problem 4: "Auth: failed" After Token Update

**Symptom:** After updating `daemon.json` with the new token and running `daemon restart`, the daemon log showed `Auth: failed` for extension requests.

**Cause:** `daemon restart` spawned a new process, but the old process (with the old token in memory) was still the one bound to port 8787. The new process failed silently or exited due to EADDRINUSE.

**Fix:** Manually killed the old process via PID, then started the daemon in the foreground to confirm it was using the correct token. Verified with `curl -H "Authorization: Bearer <token>" http://127.0.0.1:8787/v1/ping` returning `{"ok":true}`.

### Problem 5: "Missing uvx/markitdown" — Python Tool Not Installed

**Symptom:** CLI `--extract` on a PDF URL showed "Missing uvx/markitdown" error.

**Cause:** The `uvx` command (from the `uv` Python package manager) was not installed. `uvx` is needed to run `markitdown`, the Python tool that converts PDFs to markdown text.

**Fix:** `pip install uv` which installs both `uv` and `uvx` into the Python Scripts directory.

### Problem 6: uvx Not Found Despite Being Installed (Windows .exe Issue)

**Symptom:** After installing `uv`, the CLI still couldn't find `uvx`. The `resolveExecutableInPath()` function in `src/run/env.ts` searched PATH for `uvx` but couldn't find it.

**Cause:** On Windows, the executable is `uvx.exe`, but `resolveExecutableInPath()` looks for the exact name without appending `.exe`. This is a pre-existing bug in the codebase on Windows.

**Workaround:** Set the `UVX_PATH` environment variable to the full path of `uvx.exe`:

```bash
export UVX_PATH="/path/to/uvx.exe"
```

The `hasUvxCli()` function checks `UVX_PATH` first before falling back to PATH lookup, so this bypasses the bug.

### Problem 7: Daemon Didn't Have UVX_PATH

**Symptom:** Extension showed "Downloading PDF…" status (proving the fast path routing worked), but the summary was nonsensical ("S M O", 3 words, 23 chars).

**Cause:** The daemon process didn't have `UVX_PATH` in its environment. The daemon reads environment variables from `~/.summarize/daemon.json`'s `env` section. Without `UVX_PATH`, `hasUvxCli(env)` returned false, so the PDF interception code was skipped, and `runUrlFlow()` tried to fetch the PDF as HTML — which failed silently and produced garbage.

**Fix:** Added `UVX_PATH` to `~/.summarize/daemon.json`:

```json
{
  "env": {
    "UVX_PATH": "/path/to/uvx.exe"
  }
}
```

Then restarted the daemon.

### Problem 8: Cached Bad Result

**Symptom:** After fixing UVX_PATH and restarting the daemon, the extension still returned the old garbage summary immediately.

**Cause:** The SQLite cache had stored the previous failed extraction/summary result.

**Fix:** Deleted the cache files:

```bash
rm ~/.summarize/cache.sqlite ~/.summarize/cache.sqlite-wal ~/.summarize/cache.sqlite-shm
```

### Problem 9: Original Plan's Assumption Was Wrong

**Symptom:** All extension routing was correct (PDF detected, fast path triggered, "Downloading PDF…" shown), but the daemon returned garbage.

**Cause:** The plan stated "The daemon/CLI already fully supports PDF URLs" — this was incorrect. The CLI supports PDFs because it routes them through the **asset pipeline** (`src/run/runner.ts` → `handleFileInput` → `withUrlAsset`), which uses markitdown. But the daemon's `streamSummaryForUrl()` and `extractContentForUrl()` call `runUrlFlow()` directly, which calls `fetchHtmlDocument()`, which rejects `application/pdf` content-type responses.

**Fix:** Added PDF-specific handling in both daemon functions (see "Layer 3" in the changes section above). For PDF URLs, the daemon now downloads the bytes, converts via markitdown, and either returns the extracted content or feeds it directly to the summarization pipeline — completely bypassing `runUrlFlow()`.

### Problem 10: TypeScript Type Errors in ExtractedLinkContent

**Symptom:** Build failed with `TS2353: Object literal may only specify known properties, and 'text' does not exist in type 'ExtractedLinkContent'`.

**Cause:** The `ExtractedLinkContent` interface (defined in `packages/core/src/content/link-preview/content/types.ts`) uses `content` as the field name for the main text, not `text`. Additionally, the type has ~20 required fields including all transcript-related fields, description, siteName, totalCharacters, mediaDurationSeconds, etc.

**Fix:** Used the existing `ExtractedLinkContent` construction in `streamSummaryForVisiblePage()` (same file, line ~209) as a reference template. Changed `text` → `content`, added all missing required fields (set to `null` for transcript/media fields), and used `satisfies ExtractedLinkContent["diagnostics"]` for the diagnostics object to get type checking.

---

## Architecture Diagram: PDF Flow Through Extension

```
User clicks Summarize on arxiv.org/pdf/2507.11538
  │
  ▼
background.ts: isPdfUrl(tab.url) → true
  │
  ▼
wantsUrlFastPath = true (bypasses content script DOM extraction)
  │
  ▼
POST /v1/summarize { url: "https://arxiv.org/pdf/2507.11538" }
  │
  ▼
daemon/summarize.ts: streamSummaryForUrl()
  │
  ├─ isPdfUrl(url) && hasUvxCli(env) → true
  │
  ▼
fetch(url) → download PDF bytes (1.2 MB)
  │
  ▼
uvx markitdown → convert PDF to markdown text (8.5k words)
  │
  ▼
buildUrlPrompt() → summarizeExtractedUrl() → LLM streaming
  │
  ▼
SSE events → extension panel → rendered summary
```
