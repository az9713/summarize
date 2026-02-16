# Summarize - Architecture Documentation

**Version:** 0.11.2
**Last Updated:** February 15, 2026

This document provides a comprehensive architectural overview of the Summarize project, designed to help developers (especially those from C/C++/Java backgrounds) understand the TypeScript/web-based implementation.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Monorepo Structure](#2-monorepo-structure)
3. [CLI Execution Flow](#3-cli-execution-flow-detailed)
4. [Content Extraction Pipeline](#4-content-extraction-pipeline)
5. [Processing Flows](#5-processing-flows)
6. [Daemon Architecture](#6-daemon-architecture)
7. [Browser Extension Architecture](#7-browser-extension-architecture)
8. [LLM Provider System](#8-llm-provider-system)
9. [Caching System](#9-caching-system)
10. [Data Flow Summary](#10-data-flow-summary)
11. [SSE Event Protocol](#11-sse-event-protocol)
12. [Configuration System](#12-configuration-system)

---

## 1. System Overview

The Summarize project consists of three main components that work together to extract and summarize web content using various LLM providers.

```
                          SUMMARIZE SYSTEM OVERVIEW

┌─────────────────────────────────────────────────────────────────────────┐
│                         EXTERNAL LLM PROVIDERS                          │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌──────┐ ┌──────────┐   │
│  │ OpenAI │ │Anthropic│ │ Google │ │  xAI   │ │ NVIDIA│ │OpenRouter│   │
│  │  GPT   │ │ Claude  │ │ Gemini │ │  Grok  │ │  AI   │ │  (proxy) │   │
│  └────┬───┘ └───┬────┘ └───┬────┘ └───┬────┘ └───┬──┘ └────┬─────┘   │
│       └─────────┴──────────┴──────────┴──────────┴──────────┘         │
└───────────────────────────────────────────┬─────────────────────────────┘
                                            │ HTTPS API Calls
                                            V
┌─────────────────────────────────────────────────────────────────────────┐
│                        APPLICATION COMPONENTS                           │
│                                                                         │
│  ┌──────────────────┐      ┌──────────────────┐      ┌──────────────┐ │
│  │   CLI TOOL       │      │  DAEMON SERVER   │      │   BROWSER    │ │
│  │  (Command Line)  │<---->│  127.0.0.1:8787  │<---->│  EXTENSION   │ │
│  │                  │ HTTP │                  │ HTTP │              │ │
│  │ - summarize cmd  │      │ - HTTP Server    │      │ - Side Panel │ │
│  │ - stdin/file     │      │ - SSE Streaming  │      │ - Content    │ │
│  │ - terminal UX    │      │ - Bearer Auth    │      │   Scripts    │ │
│  │ - config mgmt    │      │ - Session Mgmt   │      │ - Background │ │
│  └────────┬─────────┘      └────────┬─────────┘      └──────┬───────┘ │
│           │                         │                        │         │
│           └─────────────────────────┴────────────────────────┘         │
│                                     │                                  │
│                                     V                                  │
│                        ┌────────────────────────┐                      │
│                        │    CORE LIBRARY        │                      │
│                        │  @steipete/core        │                      │
│                        │                        │                      │
│                        │ - Content Extraction   │                      │
│                        │ - Link Preview Client  │                      │
│                        │ - Transcript Providers │                      │
│                        │ - Shared Contracts     │                      │
│                        │ - Prompt Templates     │                      │
│                        └────────────────────────┘                      │
└─────────────────────────────────────────────────────────────────────────┘

Legend:
  <---> = Bidirectional communication (HTTP/HTTPS)
  ----> = Unidirectional dependency
  [ ]   = Component/Module
```

### Key Communication Paths

1. **CLI → Core**: Direct function calls (in-process)
2. **CLI → Daemon**: HTTP requests to localhost:8787 (when daemon mode active)
3. **Extension → Daemon**: HTTP + Server-Sent Events (SSE) for streaming
4. **All → LLM Providers**: HTTPS REST/streaming API calls

### Component Responsibilities

- **CLI**: User-facing terminal tool, file/URL input handling, formatted output
- **Daemon**: Background HTTP server, concurrent session management, service lifecycle
- **Extension**: Browser integration, page content extraction, UI rendering
- **Core**: Reusable logic, content extraction, provider abstraction

---

## 2. Monorepo Structure

The project uses **pnpm workspaces** to manage a monorepo with three main packages.

```
PROJECT ROOT (@steipete/summarize)
│
├─ package.json              (Root package: CLI + daemon + terminal UX)
├─ pnpm-workspace.yaml       (Workspace configuration)
├─ src/                      (Root package source code)
│  ├─ cli.ts                 (Entry point: process.argv → runCliMain)
│  ├─ cli-main.ts            (Main CLI orchestrator)
│  ├─ run/                   (Execution flows and engines)
│  │  ├─ runner.ts           (Top-level CLI runner)
│  │  ├─ summary-engine.ts   (Summary generation engine)
│  │  └─ flows/              (URL and asset processing flows)
│  ├─ daemon/                (HTTP server and service management)
│  │  ├─ server.ts           (HTTP/SSE server implementation)
│  │  ├─ launchd.ts          (macOS service manager)
│  │  ├─ systemd.ts          (Linux service manager)
│  │  └─ schtasks.ts         (Windows service manager)
│  ├─ llm/                   (LLM provider integration)
│  │  ├─ generate-text.ts    (Unified LLM interface)
│  │  ├─ model-id.ts         (Model ID parsing/normalization)
│  │  └─ providers/          (OpenAI, Anthropic, Google, etc.)
│  ├─ config.ts              (Configuration loading/merging)
│  ├─ cache.ts               (SQLite-based cache store)
│  └─ tty/                   (Terminal themes and progress UI)
│
├─ packages/
│  └─ core/                  (@steipete/summarize-core)
│     ├─ package.json        (Core library package)
│     └─ src/
│        ├─ content/         (Content extraction)
│        │  ├─ index.ts      (Public exports)
│        │  ├─ link-preview/
│        │  │  ├─ client.ts  (createLinkPreviewClient factory)
│        │  │  └─ content/
│        │  │     ├─ fetcher.ts    (HTML fetching)
│        │  │     ├─ readability.ts (Mozilla Readability)
│        │  │     └─ youtube.ts    (YouTube transcripts)
│        │  └─ transcript/
│        │     ├─ providers/
│        │     │  ├─ youtube/  (YouTubeI API, Apify, yt-dlp)
│        │     │  └─ podcast/  (RSS, Apple, Spotify)
│        │     └─ index.ts
│        └─ prompts/         (LLM prompt templates)
│
└─ apps/
   └─ chrome-extension/      (@steipete/summarize-chrome-extension)
      ├─ package.json        (Extension package - private)
      ├─ wxt.config.ts       (WXT build configuration)
      └─ src/
         ├─ entrypoints/
         │  ├─ background.ts       (Service worker: message routing)
         │  ├─ extract.content.ts  (Content script: page extraction)
         │  └─ sidepanel/
         │     └─ main.ts          (Preact UI entry point)
         └─ lib/               (Shared extension utilities)

DEPENDENCY DIRECTION:
  CLI/Daemon ──────> Core (uses)
  Extension  ──────> Daemon (HTTP client)
  Daemon     ──────> Core (uses)

All three depend on Core, but Core has no dependencies on the others.
```

### Build System

```
Root Package:
  pnpm build → clean → build core → build lib (tsc) → build CLI (esbuild)

Core Package:
  pnpm build → tsc -p tsconfig.build.json

Extension:
  pnpm build → wxt build (produces Chrome MV3 and Firefox MV3)
  pnpm build:firefox → BROWSER=firefox wxt build
```

### Key Files by Concern

| Concern | Root | Core | Extension |
|---------|------|------|-----------|
| Entry Point | `src/cli.ts` | `src/index.ts` | `src/entrypoints/background.ts` |
| Config | `src/config.ts` | N/A | `src/lib/settings.ts` |
| HTTP Server | `src/daemon/server.ts` | N/A | N/A |
| Content Extraction | N/A | `src/content/link-preview/client.ts` | `src/entrypoints/extract.content.ts` |
| LLM Calls | `src/llm/generate-text.ts` | N/A | N/A |

---

## 3. CLI Execution Flow (Detailed)

This section traces the complete path from `summarize <url>` to final output.

```
COMMAND: summarize https://example.com
│
V
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. ENTRY POINT: src/cli.ts (lines 1-18)                                │
│                                                                         │
│    import { runCliMain } from "./cli-main.js"                          │
│                                                                         │
│    runCliMain({                                                         │
│      argv: process.argv.slice(2),  // ["https://example.com"]         │
│      env: process.env,                                                  │
│      fetch: globalThis.fetch,                                           │
│      stdout/stderr: process streams                                     │
│    })                                                                   │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 2. CLI MAIN: src/cli-main.ts (runCliMain function)                     │
│                                                                         │
│    A. Load .env from cwd (lines 58-66)                                 │
│       ├─ Parse .env file (custom parser, lines 26-56)                  │
│       └─ Merge with process.env                                         │
│                                                                         │
│    B. Setup pipe error handlers (lines 15-24)                          │
│       └─ Ignore EPIPE errors (broken pipe when piped to head, etc.)    │
│                                                                         │
│    C. Detect verbose mode from argv (lines 127-131)                    │
│       └─ Check for --verbose, --debug flags                            │
│                                                                         │
│    D. Call runCli(argv, { env, fetch, stdout, stderr })                │
│       └─ Imported from "./run.js" → src/run.ts → src/run/runner.ts    │
│                                                                         │
│    E. Error handling (lines 136-154)                                   │
│       ├─ Verbose: print full stack trace                               │
│       └─ Non-verbose: strip ANSI codes, print message only             │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 3. RUNNER: src/run/runner.ts (runCli function)                         │
│                                                                         │
│    A. Preflight Routing (check for special commands first)             │
│       ├─ --help           → show help and exit                         │
│       ├─ daemon <command> → src/daemon/cli.ts                          │
│       ├─ slides <url>     → src/slides/cli.ts                          │
│       └─ transcriber      → src/transcriber-cli.ts                     │
│                                                                         │
│    B. Commander.js Program Setup                                       │
│       ├─ Define all CLI flags (see src/flags.ts)                       │
│       │  └─ --model, --length, --extract, --json, etc.                │
│       ├─ Parse argv                                                     │
│       └─ Extract parsed flags and positional args                      │
│                                                                         │
│    C. Load Configuration: src/config.ts (loadSummarizeConfig)          │
│       └─ (See Section 12 for detailed config hierarchy)                │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 4. INPUT RESOLUTION                                                     │
│                                                                         │
│    Determine source:                                                    │
│    ├─ URL provided?        → URL flow                                  │
│    ├─ File path provided?  → Asset flow                                │
│    └─ No args?             → Read from stdin                           │
│                                                                         │
│    Parse input to extract:                                             │
│    ├─ Content kind (website, youtube, video, file, etc.)               │
│    └─ Estimated token count (for model selection)                      │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 5. MODEL SELECTION: src/model-auto.ts                                  │
│                                                                         │
│    Function: buildAutoModelAttempts({ kind, promptTokens, ... })       │
│                                                                         │
│    If --model=auto (default):                                          │
│    ├─ Load auto rules from config (config.model.auto)                  │
│    ├─ Match rules by content kind: ["text", "website", "youtube",      │
│    │                                 "image", "video", "file"]          │
│    ├─ Check token bands (if defined)                                   │
│    │  └─ Example: { token: { max: 50000 }, candidates: ["gpt-5-mini"]}│
│    ├─ Build candidate list (ordered by preference)                     │
│    │  └─ Default rules: video → Gemini, text → GPT-5-mini             │
│    └─ Return attempt list with fallback chain                          │
│                                                                         │
│    Else (explicit model):                                              │
│    └─ Parse model ID: parseGatewayStyleModelId(rawModelId)             │
│       └─ (See Section 8 for model ID format)                           │
│                                                                         │
│    Output: AutoModelAttempt[]                                          │
│    ├─ Each attempt has: userModelId, llmModelId, transport, requiredEnv│
│    └─ Attempts are tried in order until one succeeds                   │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 6. FLOW DISPATCH                                                        │
│                                                                         │
│    If URL input:                                                        │
│    └─> src/run/flows/url/flow.ts (runUrlFlow)                          │
│                                                                         │
│    If File/Asset input:                                                │
│    └─> src/run/flows/asset/ (multiple files, asset flow)               │
│                                                                         │
│    Both flows use:                                                      │
│    └─> src/run/summary-engine.ts (createSummaryEngine)                 │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 7. SUMMARY ENGINE: src/run/summary-engine.ts                           │
│                                                                         │
│    createSummaryEngine({                                               │
│      content: extractedContent,                                         │
│      model: selectedModel,                                              │
│      flags: cliFlags,                                                   │
│      cache: cacheState,                                                 │
│      ...                                                                │
│    })                                                                   │
│                                                                         │
│    A. Check cache first (if enabled)                                   │
│       └─ cache.store.getText("summary", cacheKey)                      │
│                                                                         │
│    B. Build prompt (from packages/core/src/prompts/)                   │
│       └─ Include: content, length instructions, format preferences     │
│                                                                         │
│    C. Dispatch to LLM                                                   │
│       └─> src/llm/generate-text.ts                                     │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 8. LLM CALL: src/llm/generate-text.ts                                  │
│                                                                         │
│    A. Choose function based on streaming preference:                   │
│       ├─ streamTextWithModelId(modelId, prompt, options)               │
│       └─ generateTextWithModelId(modelId, prompt, options)             │
│                                                                         │
│    B. Parse model ID:                                                   │
│       parseGatewayStyleModelId(modelId)                                 │
│       └─ Extract provider and model name                               │
│                                                                         │
│    C. Dispatch to provider-specific implementation:                    │
│       ├─ "openai"     → src/llm/providers/openai.ts                    │
│       ├─ "anthropic"  → src/llm/providers/anthropic.ts                 │
│       ├─ "google"     → src/llm/providers/google.ts                    │
│       ├─ "xai"        → src/llm/providers/openai.ts (OpenAI-compatible)│
│       ├─ "nvidia"     → src/llm/providers/openai.ts (OpenAI-compatible)│
│       └─ "zai"        → src/llm/providers/openai.ts (OpenAI-compatible)│
│                                                                         │
│    D. Provider sends HTTPS request to API                              │
│       └─ Returns text stream or complete response                      │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 9. OUTPUT RENDERING: src/tty/                                          │
│                                                                         │
│    A. Apply theme (src/tty/theme.ts)                                   │
│       ├─ Themes: aurora, ember, moss, mono                             │
│       └─ Respect NO_COLOR env var                                      │
│                                                                         │
│    B. Format output based on flags:                                    │
│       ├─ --json           → JSON structure                             │
│       ├─ --extract        → Raw extracted content                      │
│       ├─ --format=markdown→ Markdown conversion                        │
│       └─ default          → Themed terminal output                     │
│                                                                         │
│    C. Write to stdout                                                   │
│       └─ Stream chunks if streaming enabled                            │
│                                                                         │
│    D. Show metrics (if verbose)                                        │
│       └─ Elapsed time, tokens used, cost estimate                      │
└─────────────────────────────────────────────────────────────────────────┘

EXECUTION COMPLETE
```

### Key Decision Points

1. **Preflight Check** (runner.ts): Is this a special command (daemon, slides, help)?
2. **Input Type** (runner.ts): URL vs File vs Stdin?
3. **Model Selection** (model-auto.ts): Auto rules vs explicit model?
4. **Cache Hit** (cache.ts): Do we have a cached summary?
5. **Streaming** (generate-text.ts): Stream tokens or wait for complete response?

### Error Propagation

```
LLM Provider Error
       │
       V
   (thrown exception)
       │
       V
   generateTextWithModelId (catches, wraps in LlmError)
       │
       V
   summary-engine.ts (tries fallback model if available)
       │
       V
   flow.ts (catches, formats error message)
       │
       V
   cli-main.ts (catches, strips ANSI if not verbose, writes to stderr)
       │
       V
   process.exitCode = 1
```

---

## 4. Content Extraction Pipeline

This pipeline converts URLs and media to clean, structured text suitable for LLM processing.

```
CONTENT EXTRACTION ARCHITECTURE

INPUT: URL (https://example.com/article)
│
V
┌─────────────────────────────────────────────────────────────────────────┐
│ CLIENT FACTORY: packages/core/src/content/link-preview/client.ts       │
│                                                                         │
│   createLinkPreviewClient({                                            │
│     fetch: globalThis.fetch,                                            │
│     env: process.env,                                                   │
│     scrapeWithFirecrawl: firecrawlClient,   // Optional                │
│     apifyApiToken: "...",                    // For YouTube fallback   │
│     ytDlpPath: "/usr/bin/yt-dlp",            // For media downloads    │
│     transcription: { ... },                  // Whisper config         │
│     transcriptCache: cacheStore,                                        │
│     mediaCache: mediaCacheStore,                                        │
│     onProgress: (event) => { ... }                                      │
│   })                                                                    │
│                                                                         │
│   Returns: LinkPreviewClient                                           │
│     └─ fetchLinkContent(url, options)                                  │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ FETCH COORDINATOR: packages/core/src/content/link-preview/content/     │
│                    index.ts (fetchLinkContent)                          │
│                                                                         │
│   A. URL Classification                                                │
│      ├─ YouTube URL?       → YouTube flow                              │
│      ├─ Twitter/X URL?     → Twitter flow                              │
│      ├─ Podcast URL?       → Podcast RSS flow                          │
│      └─ Generic website    → HTML flow                                 │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
                ┌────────────┴───────────┐
                V                        V
        ┌──────────────┐        ┌──────────────┐
        │ YOUTUBE FLOW │        │  HTML FLOW   │
        └──────┬───────┘        └──────┬───────┘
               V                        V
┌──────────────────────────┐   ┌────────────────────────┐
│ YOUTUBE TRANSCRIPT       │   │ HTML FETCHING          │
│ packages/core/src/       │   │ .../content/fetcher.ts │
│ content/transcript/      │   │                        │
│ providers/youtube/       │   │ 1. Fetch HTML          │
│                          │   │    ├─ Set User-Agent   │
│ Attempt chain:           │   │    └─ Timeout handling │
│ 1. YouTubeI API          │   │                        │
│    (youtube/api.ts)      │   │ 2. Parse with Cheerio  │
│    └─ Official endpoint  │   │    └─ Extract metadata │
│                          │   │                        │
│ 2. Caption Tracks        │   │ 3. Check for blocking  │
│    (youtube/captions.ts) │   │    └─ If blocked:      │
│    └─ Embedded in HTML   │   │       Firecrawl mode?  │
│                          │   └────────┬───────────────┘
│ 3. Apify YouTube Scraper │            V
│    (youtube/apify.ts)    │   ┌────────────────────────┐
│    └─ API token required │   │ READABILITY            │
│                          │   │ .../content/           │
│ 4. yt-dlp + Whisper      │   │ readability.ts         │
│    (youtube/yt-dlp.ts)   │   │                        │
│    ├─ Download audio     │   │ 1. Create JSDOM        │
│    └─ Transcribe locally │   │    └─ DOM API compat   │
│                          │   │                        │
│ If all fail:             │   │ 2. @mozilla/           │
│ └─ Extract from HTML     │   │    readability         │
│    description           │   │    └─ Extract article  │
└──────────┬───────────────┘   │       content          │
           │                   │                        │
           │                   │ 3. Clean HTML          │
           │                   │    (sanitize-html)     │
           V                   │    └─ Remove scripts,  │
┌────────────────────────┐     │       styles, ads      │
│ MEDIA TRANSCRIPTION    │     └────────┬───────────────┘
│ packages/core/src/     │              │
│ content/transcript/    │              V
│                        │     ┌────────────────────────┐
│ For audio/video URLs:  │     │ CONTENT CLEANER        │
│                        │     │ .../content/cleaner.ts │
│ 1. Check cache         │     │                        │
│    └─ mediaCache.get() │     │ 1. Collapse whitespace │
│                        │     │ 2. Remove empty paras  │
│ 2. Download media      │     │ 3. Normalize encoding  │
│    └─ Stream to disk   │     │ 4. Extract text nodes  │
│                        │     └────────┬───────────────┘
│ 3. Transcribe:         │              │
│    ├─ Local whisper.cpp│              │
│    │  (if available)   │              V
│    ├─ OpenAI Whisper  │     ┌────────────────────────┐
│    │  (API)            │     │ STRUCTURED OUTPUT      │
│    └─ FAL AI           │     │                        │
│       (API fallback)   │     │ ExtractedLinkContent:  │
│                        │     │ {                      │
│ 4. Save to cache       │     │   url: string          │
│    └─ mediaCache.set() │     │   title: string?       │
└────────────────────────┘     │   description: string? │
                               │   content: string      │
                               │   contentType: "html"  │
                               │   source: "readability"│
                               │   transcript?: {       │
                               │     text: string       │
                               │     source: "youtubei" │
                               │     timedText?: [...] │
                               │   }                    │
                               │   media?: {            │
                               │     hasVideo: bool     │
                               │     hasAudio: bool     │
                               │     duration: number   │
                               │   }                    │
                               │ }                      │
                               └────────────────────────┘

SPECIAL FLOWS:

┌────────────────────────────────────────────────────────────────────────┐
│ FIRECRAWL FALLBACK (when enabled)                                     │
│ packages/core/src/content/link-preview/content/firecrawl.ts           │
│                                                                        │
│ Used when:                                                             │
│ ├─ Site blocks user-agents                                            │
│ ├─ JavaScript-heavy SPA (requires rendering)                          │
│ └─ --firecrawl=always flag set                                        │
│                                                                        │
│ API call: POST https://api.firecrawl.dev/v1/scrape                    │
│ ├─ Returns rendered markdown                                          │
│ └─ Bypasses most blocking techniques                                  │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│ PODCAST RSS FLOW                                                       │
│ packages/core/src/content/transcript/providers/podcast/               │
│                                                                        │
│ 1. Detect podcast URL (Apple Podcasts, Spotify, RSS feed)             │
│ 2. Fetch RSS/JSON feed                                                │
│    └─ Parse with cheerio/xml2js                                       │
│ 3. Extract episode metadata                                           │
│ 4. Check for embedded transcript                                      │
│    └─ podcast:transcript tag (Podcast 2.0 namespace)                  │
│ 5. If no transcript: download audio + transcribe                      │
│    └─ Same media transcription pipeline as above                      │
└────────────────────────────────────────────────────────────────────────┘
```

### Extraction Priority

For maximum reliability, the pipeline attempts multiple methods in order:

**YouTube Videos:**
1. YouTubeI API (official, fast, free)
2. Embedded caption tracks (parsed from HTML)
3. Apify YouTube Scraper (requires API token, robust)
4. yt-dlp + local Whisper (slow, requires binaries, works offline)
5. Fallback: HTML description field

**General Websites:**
1. Direct HTML fetch + Readability
2. Firecrawl (if enabled and site is blocked)

**Media Files (audio/video):**
1. Check transcript cache
2. Local whisper.cpp (if available)
3. OpenAI Whisper API
4. FAL AI Whisper API (fallback)

### Key Dependencies

- **@mozilla/readability**: Article extraction (MIT license, Firefox Reader View algorithm)
- **cheerio**: Fast HTML parsing (jQuery-like API)
- **jsdom**: Full DOM implementation for Readability
- **sanitize-html**: XSS prevention and HTML cleaning

---

## 5. Processing Flows

The system has two main processing pipelines depending on input type.

### URL Flow

```
URL FLOW: src/run/flows/url/flow.ts (runUrlFlow)

INPUT: URL string
│
V
┌─────────────────────────────────────────────────────────────────────────┐
│ PHASE 1: EXTRACTION                                                     │
│ File: src/run/flows/url/extract.ts                                     │
│                                                                         │
│ 1. Initialize link preview client                                      │
│    └─ createLinkPreviewClient({ fetch, env, ... })                     │
│                                                                         │
│ 2. Show progress UI                                                     │
│    ├─ Spinner: "Fetching website (connecting)..."                      │
│    └─ OSC 9; escape codes for iTerm2/Windows Terminal                  │
│                                                                         │
│ 3. Call client.fetchLinkContent(url, options)                          │
│    └─ See Section 4 for extraction pipeline details                    │
│                                                                         │
│ 4. Handle extraction result:                                           │
│    ├─ Success: ExtractedLinkContent                                    │
│    └─ Error: format and display error message                          │
│                                                                         │
│ 5. Update spinner: "Fetched website (123 KB, 5,678 words)"             │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ PHASE 2: MARKDOWN CONVERSION (Optional)                                │
│ File: src/run/flows/url/markdown.ts                                    │
│                                                                         │
│ If --format=markdown or --markdown flag:                               │
│                                                                         │
│ A. For HTML content:                                                    │
│    └─> src/llm/html-to-markdown.ts                                     │
│       ├─ Use LLM to convert HTML → clean markdown                      │
│       └─ Model: small/fast model (e.g., gpt-5-mini)                    │
│                                                                         │
│ B. For transcripts (YouTube, podcasts):                                │
│    └─> src/llm/transcript-to-markdown.ts                               │
│       ├─ Format timed text into readable structure                     │
│       └─ Add section headers, clean up timestamps                      │
│                                                                         │
│ If --extract mode:                                                      │
│    └─ Skip to output (no summarization)                                │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ PHASE 3: SUMMARIZATION                                                  │
│ File: src/run/flows/url/summary.ts (summarizeExtractedUrl)             │
│                                                                         │
│ 1. Build cache key (src/cache.ts: buildExtractCacheKey)                │
│    ├─ Hash: URL + model + length + format preferences                  │
│    └─ Check cache: cacheState.store.getText("summary", key)            │
│                                                                         │
│ 2. If cache hit:                                                        │
│    ├─ Return cached summary                                             │
│    └─ Mark as "from cache" in metadata                                 │
│                                                                         │
│ 3. If cache miss:                                                       │
│    └─> src/run/summary-engine.ts (createSummaryEngine)                 │
│       │                                                                 │
│       ├─ Build prompt from template                                    │
│       │  (packages/core/src/prompts/)                                  │
│       │  ├─ Include content (HTML or transcript)                       │
│       │  ├─ Add length instruction (short/medium/long)                 │
│       │  └─ Add format preferences (bullets, prose, etc.)              │
│       │                                                                 │
│       ├─ Dispatch to LLM (src/llm/generate-text.ts)                    │
│       │  └─ Stream or non-streaming based on flags                     │
│       │                                                                 │
│       └─ Save to cache: cacheState.store.setText("summary", key, text) │
│                                                                         │
│ 4. Return summary + metadata                                           │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ PHASE 4: OUTPUT                                                         │
│ File: src/run/flows/url/summary.ts (outputExtractedUrl)                │
│                                                                         │
│ 1. Format summary based on flags:                                      │
│    ├─ --json         → JSON with metadata                              │
│    ├─ --plain        → Raw text (no colors)                            │
│    └─ default        → Themed output with headers                      │
│                                                                         │
│ 2. Apply theme (src/tty/theme.ts)                                      │
│    └─ Colors, formatting, box drawing                                  │
│                                                                         │
│ 3. Stream to stdout                                                     │
│    ├─ If streaming: write chunks as received                           │
│    └─ If complete: write all at once                                   │
│                                                                         │
│ 4. Show metrics (if verbose):                                          │
│    ├─ Elapsed time                                                      │
│    ├─ Tokens: prompt=X, completion=Y, total=Z                          │
│    └─ Estimated cost: $0.0123                                          │
└─────────────────────────────────────────────────────────────────────────┘

OUTPUT: Rendered summary to stdout
```

### Asset Flow (Files)

```
ASSET FLOW: src/run/flows/asset/

INPUT: File path (e.g., video.mp4, image.png, document.pdf)
│
V
┌─────────────────────────────────────────────────────────────────────────┐
│ PHASE 1: INPUT DETECTION                                               │
│ File: src/run/flows/asset/input.ts                                     │
│                                                                         │
│ 1. Determine file type:                                                │
│    └─ Use file-type package (magic number detection)                   │
│       ├─ video/* → Video file                                          │
│       ├─ audio/* → Audio file                                          │
│       ├─ image/* → Image file                                          │
│       └─ Other   → Try as text                                         │
│                                                                         │
│ 2. Validate file:                                                       │
│    ├─ Exists and readable?                                             │
│    ├─ Size within limits?                                              │
│    └─ Supported media type?                                            │
│       (src/run/attachments.ts: assertAssetMediaTypeSupported)          │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ PHASE 2: EXTRACTION                                                     │
│ File: src/run/flows/asset/extract.ts                                   │
│                                                                         │
│ Route by type:                                                          │
│                                                                         │
│ ┌─ Video/Audio ─────────────────────────────────────────┐              │
│ │  1. Check if model supports native video/audio       │              │
│ │     └─ Gemini models can process video directly      │              │
│ │                                                        │              │
│ │  2. If not supported OR transcript mode:              │              │
│ │     └─> MEDIA TRANSCRIPTION PIPELINE                  │              │
│ └───────────────────────────────────────────────────────┘              │
│                                                                         │
│ ┌─ Image ──────────────────────────────────────────────┐               │
│ │  1. Check if model supports vision                    │               │
│ │     └─ GPT-5, Claude Opus 4, Gemini support images    │               │
│ │                                                        │               │
│ │  2. Read image file                                    │               │
│ │  3. Convert to base64 or URL                           │               │
│ │  4. Prepare as LLM attachment                          │               │
│ └───────────────────────────────────────────────────────┘               │
│                                                                         │
│ ┌─ Text/Document ─────────────────────────────────────┐                │
│ │  1. Read file content                                │                │
│ │  2. Detect encoding (UTF-8, UTF-16, etc.)            │                │
│ │  3. Parse if structured (JSON, XML, CSV, etc.)       │                │
│ │  4. Return as text content                           │                │
│ └──────────────────────────────────────────────────────┘                │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ MEDIA TRANSCRIPTION PIPELINE                                            │
│ File: src/run/flows/asset/media.ts                                     │
│                                                                         │
│ 1. Check media cache:                                                   │
│    └─ mediaCacheState.get(filePath, fileHash)                          │
│                                                                         │
│ 2. If cache miss, transcribe:                                          │
│    ├─ Attempt 1: Local whisper.cpp (if installed)                      │
│    │  └─ Subprocess: whisper --model base audio.mp3                    │
│    │                                                                    │
│    ├─ Attempt 2: OpenAI Whisper API                                    │
│    │  └─ POST https://api.openai.com/v1/audio/transcriptions           │
│    │     Body: { file: audio.mp3, model: "whisper-1" }                 │
│    │                                                                    │
│    └─ Attempt 3: FAL AI Whisper (fallback)                             │
│       └─ POST https://fal.run/fal-ai/wizper                            │
│                                                                         │
│ 3. Save to media cache:                                                 │
│    └─ mediaCacheState.set(filePath, transcript, ttl)                   │
│                                                                         │
│ 4. Return transcript with metadata:                                    │
│    └─ { text, source: "whisper", duration, language }                  │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ PHASE 3: PREPROCESSING                                                  │
│ File: src/run/flows/asset/preprocess.ts                                │
│                                                                         │
│ 1. Clean and normalize extracted content                               │
│ 2. Apply content-specific transformations:                             │
│    ├─ Transcripts: add speaker labels, clean timestamps                │
│    ├─ Images: extract text via OCR if needed                           │
│    └─ Documents: preserve structure (headers, lists, etc.)             │
│                                                                         │
│ 3. Prepare for LLM input:                                               │
│    └─ Format as markdown or plain text                                 │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ PHASE 4: SUMMARIZATION                                                  │
│ (Same as URL Flow Phase 3 - uses summary-engine.ts)                    │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ PHASE 5: OUTPUT                                                         │
│ File: src/run/flows/asset/output.ts                                    │
│ (Same as URL Flow Phase 4 - themed output to stdout)                   │
└─────────────────────────────────────────────────────────────────────────┘

OUTPUT: Rendered summary to stdout
```

### Media Policy Decision Tree

```
Media file detected (src/run/flows/asset/media-policy.ts)
│
V
Does model support native media? (e.g., Gemini with video)
├─ YES: Use direct attachment (faster, more accurate)
│        └─> Send file directly to LLM
│
└─ NO:  Transcribe first
        ├─ Check cache
        ├─ Transcribe (whisper.cpp → OpenAI → FAL)
        └─> Send transcript text to LLM
```

---

## 6. Daemon Architecture

The daemon is a local HTTP server that provides a persistent background service for the CLI and browser extension.

```
DAEMON SERVER ARCHITECTURE
Server: 127.0.0.1:8787 (localhost only, no external access)

┌─────────────────────────────────────────────────────────────────────────┐
│ DAEMON ENTRY POINTS                                                     │
│                                                                         │
│ CLI: summarize daemon <command>                                         │
│  └─> src/daemon/cli.ts (daemon subcommand handler)                     │
│      ├─ start    → Start server + install service                      │
│      ├─ stop     → Stop server                                         │
│      ├─ restart  → Stop + Start                                        │
│      ├─ status   → Check if running                                    │
│      ├─ logs     → Tail log files                                      │
│      └─ uninstall→ Remove service configuration                        │
│                                                                         │
│ Programmatic: import { startDaemonServer } from "./daemon/server.js"   │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ HTTP SERVER: src/daemon/server.ts (startDaemonServer function)         │
│                                                                         │
│ Server: node:http (Node.js built-in HTTP server)                       │
│ Binding: 127.0.0.1:8787                                                 │
│ Protocol: HTTP/1.1                                                      │
│                                                                         │
│ Security:                                                               │
│ ├─ Bearer Token Authentication (Authorization: Bearer <token>)         │
│ │  └─ Token stored in ~/.summarize/daemon-token                        │
│ ├─ Localhost-only binding (blocks external access)                     │
│ └─ CORS headers for browser extension origin                           │
│                                                                         │
│ Lifecycle:                                                              │
│ ├─ Startup: Load config, initialize cache, bind socket                 │
│ ├─ Graceful shutdown: Wait for sessions to complete                    │
│ └─ Signal handling: SIGTERM, SIGINT                                    │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ HTTP ROUTES                                                             │
│                                                                         │
│ ┌─ OPTIONS * ────────────────────────────────────────────┐             │
│ │  Preflight CORS response (for browser extension)       │             │
│ │  └─ Headers: allow-origin, allow-credentials, etc.     │             │
│ └─────────────────────────────────────────────────────────┘             │
│                                                                         │
│ ┌─ GET /health ──────────────────────────────────────────┐             │
│ │  Health check (no auth required)                       │             │
│ │  Response: { ok: true, version: "0.11.2" }             │             │
│ └─────────────────────────────────────────────────────────┘             │
│                                                                         │
│ ┌─ POST /v1/ping ────────────────────────────────────────┐             │
│ │  Auth check (requires bearer token)                    │             │
│ │  Response: { ok: true }                                │             │
│ └─────────────────────────────────────────────────────────┘             │
│                                                                         │
│ ┌─ POST /v1/summarize ───────────────────────────────────┐             │
│ │  Start a new summarization session                     │             │
│ │  Request: { url, title?, model?, length?, ... }        │             │
│ │  Response: { id: "session-uuid", url: "/v1/..." }      │             │
│ │  └─ Creates session, returns SSE stream URL            │             │
│ └─────────────────────────────────────────────────────────┘             │
│                                                                         │
│ ┌─ GET /v1/summarize/{id}/events ───────────────────────┐              │
│ │  SSE event stream for a session                        │              │
│ │  Content-Type: text/event-stream                       │              │
│ │  Events: meta, status, chunk, metrics, done, error     │              │
│ │  └─ Long-lived connection, server pushes events        │              │
│ └─────────────────────────────────────────────────────────┘             │
│                                                                         │
│ ┌─ POST /v1/agent ───────────────────────────────────────┐             │
│ │  Agent/chat completions (conversational AI)            │             │
│ │  Request: { messages[], tools[], ... }                 │             │
│ │  Response: SSE stream of assistant messages            │             │
│ └─────────────────────────────────────────────────────────┘             │
│                                                                         │
│ ┌─ GET /v1/tools ────────────────────────────────────────┐             │
│ │  List available tools for agent mode                   │             │
│ │  Response: [{ name, description, parameters }, ...]    │             │
│ └─────────────────────────────────────────────────────────┘             │
│                                                                         │
│ ┌─ GET /v1/models ───────────────────────────────────────┐             │
│ │  List available LLM models                             │             │
│ │  Response: [{ id, label, provider, ... }, ...]         │             │
│ └─────────────────────────────────────────────────────────┘             │
│                                                                         │
│ ┌─ GET /v1/processes ────────────────────────────────────┐             │
│ │  List running background processes                     │             │
│ │  Response: [{ id, command, status, ... }, ...]         │             │
│ └─────────────────────────────────────────────────────────┘             │
└─────────────────────────────────────────────────────────────────────────┘

SESSION LIFECYCLE:

1. Client POSTs to /v1/summarize
   ├─ Server creates session with unique ID
   ├─ Spawns background task to perform summarization
   └─ Returns session ID immediately

2. Client GETs /v1/summarize/{id}/events
   ├─ Server opens SSE stream
   ├─ Buffers events if client not connected yet
   └─ Pushes events as they occur

3. Background task emits events:
   ├─ meta:    { model, modelLabel, inputSummary }
   ├─ status:  { text: "Fetching website..." }
   ├─ chunk:   { text: "Summary text..." } (multiple)
   ├─ metrics: { elapsedMs, summary, details }
   └─ done:    {} (session complete)

4. On error:
   └─ error:   { message: "Error description" }

5. Session cleanup (after 30 minutes or on error)
   └─ Remove from sessions Map

SESSION STORAGE:

Session = {
  id: string (UUID)
  createdAtMs: number
  done: boolean
  buffer: SseEvent[]        // Buffered events (max 2000)
  bufferBytes: number       // Total bytes buffered
  clients: Set<Response>    // Connected SSE clients
  lastMeta: { model, modelLabel, ... }
}

sessions: Map<string, Session>

SSE EVENT FORMAT (text/event-stream):

event: meta
data: {"model":"openai/gpt-5-mini","modelLabel":"GPT-5 Mini"}

event: chunk
data: {"text":"The article discusses..."}

event: done
data: {}

(blank line separates events)
```

### Platform Service Managers

The daemon can be installed as a system service that starts automatically.

```
PLATFORM SERVICE MANAGERS

┌──────────────────────────────────────────────────────────────────────┐
│ macOS: launchd (src/daemon/launchd.ts)                               │
│                                                                      │
│ Plist file: ~/Library/LaunchAgents/com.steipete.summarize.plist     │
│                                                                      │
│ <?xml version="1.0" encoding="UTF-8"?>                              │
│ <plist version="1.0">                                                │
│ <dict>                                                               │
│   <key>Label</key>                                                   │
│   <string>com.steipete.summarize</string>                            │
│   <key>ProgramArguments</key>                                        │
│   <array>                                                            │
│     <string>/usr/local/bin/summarize</string>                        │
│     <string>daemon</string>                                          │
│     <string>start</string>                                           │
│     <string>--no-install</string>                                    │
│   </array>                                                           │
│   <key>RunAtLoad</key><true/>                                        │
│   <key>KeepAlive</key><true/>                                        │
│   <key>StandardOutPath</key>                                         │
│   <string>/Users/.../.summarize/logs/daemon.log</string>             │
│   <key>StandardErrorPath</key>                                       │
│   <string>/Users/.../.summarize/logs/daemon-error.log</string>       │
│ </dict>                                                              │
│ </plist>                                                             │
│                                                                      │
│ Commands:                                                            │
│ ├─ launchctl load ~/Library/LaunchAgents/...plist                   │
│ ├─ launchctl unload ~/Library/LaunchAgents/...plist                 │
│ └─ launchctl list | grep summarize                                  │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│ Linux: systemd (src/daemon/systemd.ts)                               │
│                                                                      │
│ Unit file: ~/.config/systemd/user/summarize-daemon.service          │
│                                                                      │
│ [Unit]                                                               │
│ Description=Summarize Daemon                                         │
│ After=network.target                                                 │
│                                                                      │
│ [Service]                                                            │
│ Type=simple                                                          │
│ ExecStart=/usr/local/bin/summarize daemon start --no-install         │
│ Restart=on-failure                                                   │
│ StandardOutput=append:/home/.../.summarize/logs/daemon.log           │
│ StandardError=append:/home/.../.summarize/logs/daemon-error.log      │
│                                                                      │
│ [Install]                                                            │
│ WantedBy=default.target                                              │
│                                                                      │
│ Commands:                                                            │
│ ├─ systemctl --user enable summarize-daemon                         │
│ ├─ systemctl --user start summarize-daemon                          │
│ ├─ systemctl --user status summarize-daemon                         │
│ └─ systemctl --user stop summarize-daemon                           │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│ Windows: Task Scheduler (src/daemon/schtasks.ts)                    │
│                                                                      │
│ Task name: SummarizeDaemon                                           │
│                                                                      │
│ XML configuration created via schtasks command:                      │
│                                                                      │
│ schtasks /create /tn "SummarizeDaemon" \                             │
│   /tr "C:\...\summarize.exe daemon start --no-install" \             │
│   /sc onlogon /rl highest                                            │
│                                                                      │
│ Logging:                                                             │
│ ├─ stdout → %USERPROFILE%\.summarize\logs\daemon.log                │
│ └─ stderr → %USERPROFILE%\.summarize\logs\daemon-error.log          │
│                                                                      │
│ Commands:                                                            │
│ ├─ schtasks /run /tn "SummarizeDaemon"                               │
│ ├─ schtasks /end /tn "SummarizeDaemon"                               │
│ ├─ schtasks /query /tn "SummarizeDaemon"                             │
│ └─ schtasks /delete /tn "SummarizeDaemon"                            │
└──────────────────────────────────────────────────────────────────────┘
```

### Daemon Configuration

The daemon loads its configuration from the same sources as the CLI (see Section 12), but runs with elevated privileges and persists across terminal sessions.

**Key Paths:**
- Token: `~/.summarize/daemon-token` (auto-generated UUID)
- Logs: `~/.summarize/logs/daemon.log`, `daemon-error.log`
- PID file: `~/.summarize/daemon.pid`
- Cache: `~/.summarize/cache/` (shared with CLI)

---

## 7. Browser Extension Architecture

The browser extension provides a side panel UI for summarizing web pages without switching to the terminal.

```
BROWSER EXTENSION ARCHITECTURE
Build system: WXT (https://wxt.dev) - Web Extension Toolkit

┌─────────────────────────────────────────────────────────────────────────┐
│ EXTENSION COMPONENTS                                                    │
│                                                                         │
│  ┌──────────────────────┐       ┌──────────────────────┐              │
│  │  SIDE PANEL (UI)     │       │  BACKGROUND          │              │
│  │  Preact App          │<─────>│  SERVICE WORKER      │              │
│  │                      │ msgs  │                      │              │
│  │ src/entrypoints/     │       │ src/entrypoints/     │              │
│  │ sidepanel/main.ts    │       │ background.ts        │              │
│  │                      │       │                      │              │
│  │ UI Components:       │       │ Message Router       │              │
│  │ - Summary display    │       │ - panel <-> daemon   │              │
│  │ - Model selector     │       │ - content <-> panel  │              │
│  │ - Settings panel     │       │ - State management   │              │
│  │ - Chat interface     │       │                      │              │
│  │ - Slides gallery     │       │ HTTP Client:         │              │
│  │                      │       │ - SSE stream reader  │              │
│  └──────┬───────────────┘       └───────┬──────────────┘              │
│         │                               │                             │
│         └───────────┬───────────────────┘                             │
│                     │                                                  │
│          ┌──────────V──────────┐                                       │
│          │  CONTENT SCRIPTS    │                                       │
│          │                     │                                       │
│          │ extract.content.ts  │ (Injected into web pages)            │
│          │ - Read page content │                                       │
│          │ - Extract text      │                                       │
│          │ - Detect media      │                                       │
│          │                     │                                       │
│          │ hover.content.ts    │                                       │
│          │ - Link hover popup  │                                       │
│          └─────────────────────┘                                       │
└─────────────────────────────────────────────────────────────────────────┘

MESSAGE FLOW:

User clicks "Summarize" button in side panel
│
V
┌─────────────────────────────────────────────────────────────────────────┐
│ SIDE PANEL (src/entrypoints/sidepanel/main.ts)                         │
│                                                                         │
│ 1. User interaction triggers onSummarize()                             │
│                                                                         │
│ 2. Send message to background:                                         │
│    browser.runtime.sendMessage({                                       │
│      type: "panel:summarize",                                          │
│      refresh: false,                                                    │
│      inputMode: "page"  // or "video"                                  │
│    })                                                                   │
│                                                                         │
│ 3. Listen for response messages:                                       │
│    browser.runtime.onMessage.addListener((msg) => {                    │
│      switch (msg.type) {                                               │
│        case "ui:state":   updateUiState(msg.state)                     │
│        case "ui:status":  showStatus(msg.status)                       │
│        case "run:start":  beginStreamingUi(msg.run)                    │
│        case "run:error":  showError(msg.message)                       │
│      }                                                                  │
│    })                                                                   │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ BACKGROUND SERVICE WORKER (src/entrypoints/background.ts)              │
│                                                                         │
│ 1. Receive message from side panel                                     │
│    onMessage("panel:summarize")                                        │
│                                                                         │
│ 2. Query current tab for content:                                      │
│    const [tab] = await browser.tabs.query({ active: true })            │
│    const response = await browser.tabs.sendMessage(tab.id, {           │
│      type: "extract",                                                   │
│      maxChars: 100000                                                   │
│    })                                                                   │
│    └─> Goes to content script (extract.content.ts)                     │
│                                                                         │
│ 3. Content script extracts page content:                               │
│    ├─ Use @mozilla/readability for article text                        │
│    ├─ Detect media (video, audio elements)                             │
│    └─ Return { ok: true, url, title, text, media }                     │
│                                                                         │
│ 4. Build request payload (src/lib/daemon-payload.ts):                  │
│    const payload = buildSummarizeRequestBody({                         │
│      url: response.url,                                                 │
│      title: response.title,                                             │
│      content: response.text,                                            │
│      model: settings.model,                                             │
│      length: settings.length,                                           │
│      inputMode: "page"                                                  │
│    })                                                                   │
│                                                                         │
│ 5. POST to daemon:                                                      │
│    const res = await fetch("http://127.0.0.1:8787/v1/summarize", {     │
│      method: "POST",                                                    │
│      headers: {                                                         │
│        "authorization": `Bearer ${token}`,                              │
│        "content-type": "application/json"                               │
│      },                                                                 │
│      body: JSON.stringify(payload)                                     │
│    })                                                                   │
│    const { id } = await res.json()  // session ID                      │
│                                                                         │
│ 6. Notify panel that stream is starting:                               │
│    browser.runtime.sendMessage({                                       │
│      type: "run:start",                                                 │
│      run: { id, url, title, model, reason: "user request" }            │
│    })                                                                   │
│                                                                         │
│ 7. Open SSE stream:                                                     │
│    const eventSource = new EventSource(                                │
│      `http://127.0.0.1:8787/v1/summarize/${id}/events`,                │
│      { headers: { authorization: `Bearer ${token}` } }                  │
│    )                                                                    │
│                                                                         │
│ 8. Forward events to panel:                                            │
│    eventSource.addEventListener("meta", (e) => {                        │
│      const data = JSON.parse(e.data)                                   │
│      // forward to panel                                               │
│    })                                                                   │
│    eventSource.addEventListener("chunk", (e) => {                       │
│      const { text } = JSON.parse(e.data)                               │
│      // append to summary in panel                                     │
│    })                                                                   │
│    eventSource.addEventListener("done", () => {                         │
│      // close stream, mark complete                                    │
│    })                                                                   │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ SIDE PANEL (Update UI with streaming chunks)                           │
│                                                                         │
│ Stream controller (src/entrypoints/sidepanel/stream-controller.ts):    │
│                                                                         │
│ 1. Receive "chunk" messages from background                            │
│ 2. Append text to summary display (incremental rendering)              │
│ 3. Render markdown with markdown-it library                            │
│ 4. Auto-scroll to bottom as content arrives                            │
│ 5. On "done": show metrics, enable export/copy buttons                 │
└─────────────────────────────────────────────────────────────────────────┘

COMPLETE: Summary displayed in side panel
```

### Extension Message Types

```typescript
// Panel → Background
type PanelToBg =
  | { type: "panel:ready" }
  | { type: "panel:summarize"; refresh?: boolean; inputMode?: "page" | "video" }
  | { type: "panel:agent"; requestId: string; messages: Message[]; tools: string[] }
  | { type: "panel:ping" }
  | { type: "panel:closed" }

// Background → Panel
type BgToPanel =
  | { type: "ui:state"; state: UiState }
  | { type: "ui:status"; status: string }
  | { type: "run:start"; run: { id, url, title, model } }
  | { type: "run:error"; message: string }
  | { type: "agent:chunk"; requestId: string; text: string }

// Background → Content Script
type BgToContent =
  | { type: "extract"; maxChars: number }
  | { type: "seek"; seconds: number }

// Content Script → Background
type ContentToBg =
  | { ok: true; url: string; title: string; text: string; media?: MediaInfo }
  | { ok: false; error: string }
```

### WXT Build Output

```
Chrome MV3 (.output/chrome-mv3/):
├─ manifest.json (version 3)
├─ background.js (service worker)
├─ sidepanel.html + sidepanel.js
├─ content-scripts/
│  ├─ extract.js
│  └─ hover.js
└─ icons/

Firefox MV3 (.output/firefox-mv3/):
├─ manifest.json (version 3, Firefox-compatible)
├─ background.js
├─ sidepanel.html + sidepanel.js
└─ ... (same structure)

Build command maps:
  pnpm build        → Chrome MV3
  pnpm build:firefox → Firefox MV3
```

### Extension Dependencies

- **Preact**: Lightweight React alternative (3KB)
- **@zag-js/preact**: Headless UI components (select, checkbox)
- **markdown-it**: Markdown rendering in panel
- **@mozilla/readability**: Article extraction (same as CLI)

---

## 8. LLM Provider System

The LLM provider system abstracts multiple AI services behind a unified interface.

```
LLM PROVIDER DISPATCH SYSTEM

MODEL ID FORMAT: <provider>/<model>
Examples:
  openai/gpt-5-mini
  anthropic/claude-opus-4-6
  google/gemini-3-flash-preview
  xai/grok-4-fast-non-reasoning
  zai/llama-4-70b-instruct
  nvidia/nvidia/llama-4-nemotron-70b-instruct

┌─────────────────────────────────────────────────────────────────────────┐
│ ENTRY POINT: src/llm/generate-text.ts                                  │
│                                                                         │
│ export async function generateTextWithModelId(                         │
│   modelId: string,                                                      │
│   prompt: string | Message[],                                          │
│   options: GenerateOptions                                             │
│ ): Promise<GenerateResult>                                             │
│                                                                         │
│ export async function* streamTextWithModelId(                          │
│   modelId: string,                                                      │
│   prompt: string | Message[],                                          │
│   options: GenerateOptions                                             │
│ ): AsyncGenerator<string>                                              │
│                                                                         │
│ Common options:                                                         │
│ ├─ temperature: number (0-2, default 1)                                │
│ ├─ maxTokens: number (output limit)                                    │
│ ├─ stopSequences: string[]                                             │
│ ├─ attachments?: Attachment[] (images, files)                          │
│ ├─ tools?: Tool[] (function calling)                                   │
│ └─ fetch: typeof fetch (injectable for testing)                        │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ MODEL ID PARSER: src/llm/model-id.ts                                   │
│                                                                         │
│ export function parseGatewayStyleModelId(raw: string): ParsedModelId   │
│                                                                         │
│ Input:  "openai/gpt-5-mini"                                            │
│ Output: {                                                               │
│   provider: "openai",                                                   │
│   model: "gpt-5-mini",                                                  │
│   canonical: "openai/gpt-5-mini"                                       │
│ }                                                                       │
│                                                                         │
│ Normalization rules:                                                    │
│ 1. Lowercase everything                                                 │
│ 2. Anthropic aliases: "claude-sonnet-4" → "claude-sonnet-4-0"          │
│ 3. Grok aliases: "grok-4-1-fast" → "grok-4-fast-non-reasoning"         │
│ 4. Infer provider if missing:                                          │
│    ├─ "grok-*"   → xai/                                                │
│    ├─ "gemini-*" → google/                                             │
│    ├─ "claude-*" → anthropic/                                          │
│    └─ other      → openai/ (default)                                   │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ PROVIDER DISPATCH (src/llm/generate-text.ts)                           │
│                                                                         │
│ switch (parsed.provider) {                                             │
│   case "openai":                                                        │
│   case "xai":                                                           │
│   case "nvidia":                                                        │
│   case "zai":                                                           │
│     └─> src/llm/providers/openai.ts                                    │
│         (All use OpenAI-compatible API)                                │
│                                                                         │
│   case "anthropic":                                                     │
│     └─> src/llm/providers/anthropic.ts                                 │
│                                                                         │
│   case "google":                                                        │
│     └─> src/llm/providers/google.ts                                    │
│ }                                                                       │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
                ┌────────────┴──────────┬─────────────┐
                V                       V             V
┌──────────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ OPENAI PROVIDER      │  │ ANTHROPIC        │  │ GOOGLE           │
│ src/llm/providers/   │  │ PROVIDER         │  │ PROVIDER         │
│ openai.ts            │  │ anthropic.ts     │  │ google.ts        │
│                      │  │                  │  │                  │
│ Providers using      │  │ Claude Messages  │  │ Gemini Generate  │
│ this implementation: │  │ API              │  │ Content API      │
│ - OpenAI             │  │                  │  │                  │
│ - xAI (Grok)         │  │ POST /v1/        │  │ POST /v1beta/    │
│ - NVIDIA             │  │ messages         │  │ models/{model}:  │
│ - Z.AI               │  │                  │  │ generateContent  │
│                      │  │ Request:         │  │                  │
│ POST /v1/chat/       │  │ {                │  │ Request:         │
│ completions          │  │   model,         │  │ {                │
│                      │  │   messages: [{   │  │   contents: [{   │
│ Request:             │  │     role,        │  │     role,        │
│ {                    │  │     content      │  │     parts: [{    │
│   model,             │  │   }],            │  │       text       │
│   messages: [{       │  │   max_tokens,    │  │     }]           │
│     role,            │  │   temperature,   │  │   }],            │
│     content          │  │   stream         │  │   generationConfi│
│   }],                │  │ }                │  │   g: {           │
│   max_tokens,        │  │                  │  │     temperature, │
│   temperature,       │  │ Streaming:       │  │     maxOutput    │
│   stream             │  │ - SSE format     │  │     Tokens       │
│ }                    │  │ - event: message_│  │   }              │
│                      │  │   start, content_│  │ }                │
│ Streaming:           │  │   block_delta,   │  │                  │
│ - SSE format         │  │   message_stop   │  │ Streaming:       │
│ - data: {choices}    │  │                  │  │ - Newline-       │
│                      │  │ Base URL:        │  │   delimited JSON │
│ Base URLs:           │  │ - Anthropic:     │  │                  │
│ - OpenAI:            │  │   api.anthropic. │  │ Base URL:        │
│   api.openai.com     │  │   com            │  │ - Google:        │
│ - xAI:               │  │                  │  │   generative     │
│   api.x.ai           │  │ Auth:            │  │   languageapi    │
│ - NVIDIA:            │  │ - x-api-key:     │  │   .googleapis.   │
│   integrate.api.     │  │   <key>          │  │   com            │
│   nvidia.com         │  │ - anthropic-     │  │                  │
│ - Z.AI:              │  │   version:       │  │ Auth:            │
│   api.z.ai           │  │   2024-07-25     │  │ - x-goog-api-key:│
│                      │  │                  │  │   <key>          │
│ Auth:                │  └──────────────────┘  └──────────────────┘
│ - Authorization:     │
│   Bearer <api-key>   │
└──────────────────────┘

API KEY RESOLUTION (from environment):

Provider    | Primary Env Var      | Config Fallback
----------- | -------------------- | -------------------------
OpenAI      | OPENAI_API_KEY       | config.apiKeys.openai
Anthropic   | ANTHROPIC_API_KEY    | config.apiKeys.anthropic
Google      | GEMINI_API_KEY       | config.apiKeys.google
            | GOOGLE_API_KEY       |
xAI         | XAI_API_KEY          | config.apiKeys.xai
NVIDIA      | NVIDIA_API_KEY       | config.apiKeys.nvidia
Z.AI        | Z_AI_API_KEY         | config.apiKeys.zai
OpenRouter  | OPENROUTER_API_KEY   | config.apiKeys.openrouter
            | (proxy to any model) |
```

### CLI Provider System (Local Binaries)

For models accessible via local CLI tools (Claude Desktop, Codex, Gemini CLI, Cursor Agent):

```
CLI PROVIDER DISPATCH: src/llm/cli.ts

Supported CLI providers:
├─ claude  (Claude Desktop CLI)
├─ codex   (Codex CLI by OpenAI)
├─ gemini  (Gemini CLI by Google)
└─ agent   (Cursor Agent)

Configuration (config.cli):
{
  enabled: ["claude", "gemini"],
  claude: {
    binary: "/usr/local/bin/claude",
    extraArgs: ["--model", "opus-4"],
    model: "opus-4"
  },
  autoFallback: {
    enabled: true,
    onlyWhenNoApiKeys: true,
    order: ["claude", "gemini", "codex"]
  }
}

Execution:
1. Spawn subprocess with prompt piped to stdin
2. Capture stdout (summary) and stderr (logs)
3. Wait for exit code
4. Return text or throw error

Auto fallback chain:
  Native API failed → Try CLI providers in order → Fail if all exhausted
```

### Auto Model Selection

```
AUTO MODEL SELECTION: src/model-auto.ts (buildAutoModelAttempts)

Input: {
  kind: "video" | "website" | "youtube" | "text" | "image" | "file"
  promptTokens: 12000
  desiredOutputTokens: 2048
  requiresVideoUnderstanding: true
}

Rules (config.model.auto):
[
  {
    when: ["video"],
    candidates: ["google/gemini-3-flash-preview"]
  },
  {
    when: ["text", "website"],
    bands: [
      { token: { max: 50000 }, candidates: ["openai/gpt-5-mini"] },
      { token: { min: 50000 }, candidates: ["openai/gpt-5"] }
    ]
  }
]

Selection algorithm:
1. Filter rules by content kind
2. Check token bands (if defined)
3. Build candidate list (in order)
4. Expand each candidate to:
   ├─ Native provider attempt (if API key available)
   ├─ OpenRouter attempt (if API key available)
   └─ CLI attempt (if enabled and binary available)
5. Return ordered attempt list

Example output:
[
  {
    transport: "native",
    userModelId: "google/gemini-3-flash-preview",
    llmModelId: "gemini-3-flash-preview",
    requiredEnv: "GEMINI_API_KEY"
  },
  {
    transport: "cli",
    userModelId: "google/gemini-3-flash-preview",
    llmModelId: null,
    requiredEnv: "CLI_GEMINI"
  }
]

Fallback logic:
  Try attempts[0] → if fails, try attempts[1] → ... → fail if all exhausted
```

---

## 9. Caching System

The caching system uses SQLite to store extracted content, summaries, and transcripts.

```
CACHING ARCHITECTURE

Storage: SQLite database (node:sqlite or bun:sqlite)
Location: ~/.summarize/cache/cache.db

┌─────────────────────────────────────────────────────────────────────────┐
│ CACHE STORE: src/cache.ts                                              │
│                                                                         │
│ export type CacheKind =                                                │
│   | "extract"     // Extracted article content                         │
│   | "summary"     // Generated summaries                               │
│   | "transcript"  // Video/audio transcripts                           │
│   | "chat"        // Chat/agent conversation history                   │
│   | "slides"      // Slide extraction results                          │
│                                                                         │
│ Database schema:                                                        │
│                                                                         │
│ CREATE TABLE cache (                                                    │
│   kind TEXT NOT NULL,                                                   │
│   key TEXT NOT NULL,                                                    │
│   value TEXT NOT NULL,                                                  │
│   expires_at INTEGER,                                                   │
│   size_bytes INTEGER NOT NULL,                                          │
│   created_at INTEGER NOT NULL,                                          │
│   PRIMARY KEY (kind, key)                                               │
│ );                                                                      │
│                                                                         │
│ CREATE INDEX idx_expires_at ON cache(expires_at);                      │
│ CREATE INDEX idx_kind ON cache(kind);                                  │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ CACHE STATE: src/run/cache-state.ts                                    │
│                                                                         │
│ export type CacheState = {                                             │
│   mode: "default" | "bypass"                                           │
│   store: CacheStore | null                                             │
│   ttlMs: number                                                         │
│   maxBytes: number                                                      │
│   path: string | null                                                  │
│ }                                                                       │
│                                                                         │
│ Configuration precedence:                                              │
│ 1. CLI flag: --no-cache (sets mode to "bypass")                        │
│ 2. Config file: config.cache.enabled (boolean)                         │
│ 3. Default: enabled                                                     │
│                                                                         │
│ TTL (Time To Live):                                                     │
│ ├─ Default: 30 days                                                     │
│ ├─ Config: config.cache.ttlDays                                        │
│ └─ Calculation: ttlMs = ttlDays * 24 * 60 * 60 * 1000                  │
│                                                                         │
│ Max size:                                                               │
│ ├─ Default: 512 MB                                                      │
│ └─ Config: config.cache.maxMb                                          │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ CACHE OPERATIONS                                                        │
│                                                                         │
│ Get:                                                                    │
│   cacheStore.getText(kind, key)                                        │
│   └─> SELECT value FROM cache WHERE kind=? AND key=?                   │
│       AND (expires_at IS NULL OR expires_at > ?)                       │
│                                                                         │
│ Set:                                                                    │
│   cacheStore.setText(kind, key, value, ttlMs)                          │
│   └─> INSERT OR REPLACE INTO cache                                     │
│       (kind, key, value, expires_at, size_bytes, created_at)           │
│       VALUES (?, ?, ?, ?, ?, ?)                                        │
│                                                                         │
│ Cleanup (on startup):                                                   │
│   DELETE FROM cache WHERE expires_at IS NOT NULL AND expires_at < ?    │
│                                                                         │
│ Size enforcement:                                                       │
│   If total size > maxBytes:                                            │
│   └─> DELETE oldest entries until size < maxBytes                      │
│       ORDER BY created_at ASC                                           │
└─────────────────────────────────────────────────────────────────────────┘

CACHE KEY GENERATION:

Extract cache key (buildExtractCacheKey):
  SHA256(url + firecrawlMode + youtubeMode)

Summary cache key (buildSummaryCacheKey):
  SHA256(
    extractCacheKey +
    modelId +
    lengthPreset +
    format +
    language +
    customPrompt
  )

Transcript cache key:
  SHA256(url + source + timestampsEnabled)

Slides cache key:
  SHA256(url + ocrEnabled)
```

### Media Cache (Separate System)

For downloaded media files (audio/video for transcription):

```
MEDIA CACHE: src/run/media-cache-state.ts

Storage: Filesystem (not SQLite)
Location: ~/.summarize/media-cache/

Structure:
  ~/.summarize/media-cache/
  ├─ <hash1>.mp4
  ├─ <hash2>.mp3
  ├─ <hash3>.wav
  └─ metadata.json (index of files)

Configuration:
  config.media.cache: {
    enabled: true,
    maxMb: 2048,          // 2 GB default
    ttlDays: 7,           // 7 days default
    verify: "size"        // "none" | "size" | "hash"
  }

Operations:

  Get:
    1. Compute file hash (SHA256 of URL + options)
    2. Check if file exists
    3. Verify integrity (size or hash check)
    4. Return file path

  Set:
    1. Compute file hash
    2. Download media to temp location
    3. Move to cache directory
    4. Update metadata.json
    5. Enforce size limit (delete oldest if exceeded)

  Cleanup:
    - On startup: delete files older than TTL
    - On size limit: delete oldest files first (LRU)

Verification modes:
  - "none": Trust file exists (fast, risky)
  - "size": Check file size matches expected (default)
  - "hash": Compute SHA256 and verify (slow, secure)
```

### Cache Invalidation

```
Manual cache clearing:
  summarize --no-cache <url>      // Bypass cache for this run
  rm -rf ~/.summarize/cache/      // Delete all cached data

Automatic expiration:
  - On startup: cleanup expired entries
  - On size limit: delete oldest entries (LRU)
  - TTL-based: entries expire after configured days

Cache warming (pre-populate):
  Not currently implemented, but architecture supports it:
    cacheStore.setText("extract", key, content, ttlMs)
```

---

## 10. Data Flow Summary

This section shows a complete end-to-end trace of a URL summarization request.

```
END-TO-END DATA FLOW: Summarize a YouTube video

COMMAND: summarize https://youtube.com/watch?v=abc123

┌─────────────────────────────────────────────────────────────────────────┐
│ 1. CLI ENTRY (src/cli.ts)                                              │
│    Process: node dist/cli.js https://youtube.com/watch?v=abc123        │
│    └─> runCliMain({ argv: ["https://..."], env, fetch, ... })          │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 2. CLI MAIN (src/cli-main.ts)                                          │
│    ├─ Load .env file                                                    │
│    ├─ Setup pipe error handlers                                        │
│    └─> runCli(argv, { env, fetch, stdout, stderr })                    │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 3. RUNNER (src/run/runner.ts)                                          │
│    ├─ Parse flags with Commander.js                                    │
│    ├─ Load config: loadSummarizeConfig()                               │
│    ├─ Detect input type: YouTube URL                                   │
│    └─> Route to URL flow                                               │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 4. CONFIG LOADING (src/config.ts: loadSummarizeConfig)                 │
│    Merge sources (highest to lowest priority):                         │
│    ├─ CLI flags: --model=auto, --length=medium                         │
│    ├─ Environment: OPENAI_API_KEY=sk-..., SUMMARIZE_MODEL=auto         │
│    ├─ Config file: ~/.summarize/config.json                            │
│    └─ Defaults: model.auto rules, cache.enabled=true                   │
│                                                                         │
│    Result: SummarizeConfig object                                      │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 5. CACHE INITIALIZATION (src/run/cache-state.ts)                       │
│    ├─ Open SQLite: ~/.summarize/cache/cache.db                         │
│    ├─ Run cleanup: DELETE expired entries                              │
│    └─ Return CacheState { mode: "default", store, ttlMs, ... }         │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 6. MODEL SELECTION (src/model-auto.ts)                                 │
│    Input: { kind: "youtube", promptTokens: null, ... }                 │
│    ├─ Match auto rules for "youtube"                                   │
│    ├─ Default rule: ["openai/gpt-5-mini"]                              │
│    └─ Build attempt list:                                              │
│       [{ transport: "native", userModelId: "openai/gpt-5-mini",        │
│          llmModelId: "gpt-5-mini", requiredEnv: "OPENAI_API_KEY" }]    │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 7. URL FLOW (src/run/flows/url/flow.ts: runUrlFlow)                    │
│    ├─ Show spinner: "Fetching website (connecting)..."                 │
│    └─> Extract content                                                 │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 8. CONTENT EXTRACTION (packages/core/src/content/link-preview/)        │
│    ├─ createLinkPreviewClient({ fetch, env, ... })                     │
│    └─> client.fetchLinkContent("https://youtube.com/watch?v=abc123")   │
│        │                                                                │
│        ├─ Detect YouTube URL                                           │
│        └─> YouTube transcript flow                                     │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 9. YOUTUBE TRANSCRIPT (.../content/transcript/providers/youtube/)      │
│    Attempt 1: YouTubeI API                                             │
│    └─> GET https://www.youtube.com/watch?v=abc123                      │
│        Parse HTML for ytInitialPlayerResponse                          │
│        Extract captions URL from captionTracks                         │
│        └─> GET <captions URL>                                          │
│            Parse TimedText XML                                          │
│            └─ SUCCESS: return transcript                                │
│                                                                         │
│    (If Attempt 1 fails, try Apify, yt-dlp, etc.)                       │
│                                                                         │
│    Result: {                                                            │
│      transcript: {                                                      │
│        text: "Welcome to the video. Today we'll...",                   │
│        source: "youtubei",                                              │
│        timedText: [                                                     │
│          { start: 0, duration: 3.2, text: "Welcome to the video." },   │
│          ...                                                            │
│        ]                                                                │
│      }                                                                  │
│    }                                                                    │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 10. EXTRACTION COMPLETE (back to URL flow)                             │
│     ├─ Update spinner: "Fetched website (transcript, 12,345 words)"    │
│     └─> Check extract mode flag                                        │
│         ├─ If --extract: output transcript and exit                    │
│         └─ Else: continue to summarization                             │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 11. SUMMARIZATION (src/run/flows/url/summary.ts)                       │
│     A. Build cache key:                                                │
│        extractKey = SHA256("youtube.com/watch?v=abc123" + opts)        │
│        summaryKey = SHA256(extractKey + "gpt-5-mini" + "medium")       │
│                                                                         │
│     B. Check cache:                                                     │
│        cached = cacheStore.getText("summary", summaryKey)              │
│        └─ MISS (first time summarizing this video)                     │
│                                                                         │
│     C. Create summary engine:                                           │
│        └─> src/run/summary-engine.ts                                   │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 12. SUMMARY ENGINE (src/run/summary-engine.ts)                         │
│     A. Build prompt:                                                    │
│        Template from: packages/core/src/prompts/                       │
│        ├─ System: "You are a helpful assistant..."                     │
│        ├─ Content: <full transcript text>                              │
│        └─ Instructions: "Summarize in medium length, use bullets..."   │
│                                                                         │
│     B. Dispatch to LLM:                                                 │
│        └─> src/llm/generate-text.ts                                    │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 13. LLM CALL (src/llm/generate-text.ts)                                │
│     A. Parse model ID:                                                  │
│        parseGatewayStyleModelId("openai/gpt-5-mini")                   │
│        └─ { provider: "openai", model: "gpt-5-mini" }                  │
│                                                                         │
│     B. Select streaming or non-streaming:                              │
│        flags.streamingEnabled = true (default)                         │
│        └─> streamTextWithModelId()                                     │
│                                                                         │
│     C. Dispatch to provider:                                            │
│        └─> src/llm/providers/openai.ts                                 │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 14. OPENAI PROVIDER (src/llm/providers/openai.ts)                      │
│     A. Build request:                                                   │
│        POST https://api.openai.com/v1/chat/completions                 │
│        Headers: {                                                       │
│          "authorization": "Bearer sk-...",                              │
│          "content-type": "application/json"                             │
│        }                                                                │
│        Body: {                                                          │
│          model: "gpt-5-mini",                                           │
│          messages: [{ role: "user", content: "<prompt>" }],            │
│          stream: true,                                                  │
│          temperature: 1.0                                               │
│        }                                                                │
│                                                                         │
│     B. Send request via fetch()                                         │
│                                                                         │
│     C. Parse SSE stream:                                                │
│        data: {"choices":[{"delta":{"content":"This video"}}]}          │
│        data: {"choices":[{"delta":{"content":" discusses"}}]}          │
│        ...                                                              │
│        data: [DONE]                                                     │
│                                                                         │
│     D. Yield text chunks:                                               │
│        yield "This video"                                               │
│        yield " discusses"                                               │
│        ...                                                              │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 15. OUTPUT RENDERING (src/tty/)                                        │
│     A. Apply theme:                                                     │
│        theme = resolveThemeNameFromSources({ env })                    │
│        └─ "aurora" (default)                                            │
│                                                                         │
│     B. Stream chunks to stdout:                                         │
│        for await (const chunk of llmStream) {                           │
│          stdout.write(theme.apply(chunk))                               │
│        }                                                                │
│                                                                         │
│     C. Show metrics (if verbose):                                       │
│        ├─ Elapsed time: 12.3s                                          │
│        ├─ Tokens: prompt=12,345, completion=234, total=12,579          │
│        └─ Cost: $0.0062                                                 │
└────────────────────────────┬────────────────────────────────────────────┘
                             V
┌─────────────────────────────────────────────────────────────────────────┐
│ 16. CACHE SAVE (back to summary-engine.ts)                             │
│     cacheStore.setText(                                                │
│       "summary",                                                        │
│       summaryKey,                                                       │
│       fullSummaryText,                                                  │
│       30 * 24 * 60 * 60 * 1000  // 30 days TTL                         │
│     )                                                                   │
│                                                                         │
│     SQLite:                                                             │
│     INSERT OR REPLACE INTO cache                                        │
│       (kind, key, value, expires_at, size_bytes, created_at)           │
│     VALUES                                                              │
│       ("summary", "<hash>", "<text>", <timestamp>, 1234, <now>)        │
└─────────────────────────────────────────────────────────────────────────┘

COMPLETE: Summary displayed in terminal with theme colors
Next run: Cache hit, instant result (no LLM call)
```

### File and Module References

```
Flow step → Key files

1. Entry              → src/cli.ts (lines 1-18)
2. Main orchestrator  → src/cli-main.ts (runCliMain, lines 115-156)
3. Runner             → src/run/runner.ts (runCli)
4. Config             → src/config.ts (loadSummarizeConfig)
5. Cache init         → src/run/cache-state.ts (createCacheStateFromConfig)
6. Model selection    → src/model-auto.ts (buildAutoModelAttempts)
7. URL flow           → src/run/flows/url/flow.ts (runUrlFlow)
8. Extraction         → packages/core/src/content/link-preview/client.ts
9. YouTube transcript → packages/core/src/content/transcript/providers/youtube/
10. Extract complete  → src/run/flows/url/extract.ts
11. Summarization     → src/run/flows/url/summary.ts (summarizeExtractedUrl)
12. Summary engine    → src/run/summary-engine.ts (createSummaryEngine)
13. LLM dispatch      → src/llm/generate-text.ts (streamTextWithModelId)
14. Provider          → src/llm/providers/openai.ts
15. Output            → src/tty/theme.ts, src/tty/spinner.ts
16. Cache save        → src/cache.ts (CacheStore.setText)
```

---

## 11. SSE Event Protocol

The Server-Sent Events protocol enables real-time streaming from the daemon to clients (CLI or extension).

```
SERVER-SENT EVENTS (SSE) PROTOCOL

Protocol: text/event-stream (HTTP streaming)
Direction: Daemon (server) → Client (browser extension or CLI)
Connection: Long-lived (up to 30 minutes per session)

┌─────────────────────────────────────────────────────────────────────────┐
│ EVENT TYPES (src/shared/sse-events.ts)                                 │
│                                                                         │
│ export type SseEvent =                                                 │
│   | { event: "meta"; data: SseMetaData }                               │
│   | { event: "status"; data: { text: string } }                        │
│   | { event: "chunk"; data: { text: string } }                         │
│   | { event: "slides"; data: SseSlidesData }                           │
│   | { event: "metrics"; data: SseMetricsData }                         │
│   | { event: "done"; data: {} }                                        │
│   | { event: "error"; data: { message: string } }                      │
└─────────────────────────────────────────────────────────────────────────┘

EVENT DETAILS:

┌─────────────────────────────────────────────────────────────────────────┐
│ 1. META EVENT (sent first, once per session)                           │
│                                                                         │
│    Purpose: Provide session metadata before content starts             │
│                                                                         │
│    Payload: SseMetaData {                                              │
│      model: string | null                                              │
│        // Canonical model ID (e.g., "openai/gpt-5-mini")               │
│                                                                         │
│      modelLabel: string | null                                         │
│        // Human-readable model name (e.g., "GPT-5 Mini")               │
│                                                                         │
│      inputSummary: string | null                                       │
│        // Brief description of input (e.g., "12,345 words, YouTube")   │
│                                                                         │
│      summaryFromCache: boolean | null                                  │
│        // true if summary retrieved from cache (no LLM call)           │
│    }                                                                    │
│                                                                         │
│    Wire format:                                                         │
│      event: meta                                                        │
│      data: {"model":"openai/gpt-5-mini","modelLabel":"GPT-5 Mini",...} │
│      (blank line)                                                       │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ 2. STATUS EVENT (progress updates, multiple allowed)                   │
│                                                                         │
│    Purpose: Show what the system is doing (extraction, processing)     │
│                                                                         │
│    Payload: { text: string }                                           │
│      // Examples:                                                       │
│      // - "Fetching website..."                                        │
│      // - "Extracting transcript..."                                   │
│      // - "Generating summary..."                                      │
│                                                                         │
│    Wire format:                                                         │
│      event: status                                                      │
│      data: {"text":"Fetching website..."}                              │
│      (blank line)                                                       │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ 3. CHUNK EVENT (summary text, multiple allowed)                        │
│                                                                         │
│    Purpose: Stream summary text as it's generated by the LLM           │
│                                                                         │
│    Payload: { text: string }                                           │
│      // Incremental text chunk (e.g., "The video ", "discusses ", ...) │
│                                                                         │
│    Client behavior:                                                     │
│      ├─ Append each chunk to existing text                             │
│      ├─ Render progressively (show partial summary as it arrives)      │
│      └─ No need to wait for "done" to start displaying                 │
│                                                                         │
│    Wire format:                                                         │
│      event: chunk                                                       │
│      data: {"text":"The video discusses "}                             │
│      (blank line)                                                       │
│                                                                         │
│      event: chunk                                                       │
│      data: {"text":"three main topics: "}                              │
│      (blank line)                                                       │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ 4. SLIDES EVENT (for presentations/videos, optional)                   │
│                                                                         │
│    Purpose: Provide extracted slides from a video or presentation      │
│                                                                         │
│    Payload: SseSlidesData {                                            │
│      sourceUrl: string                                                 │
│        // Original URL of the content                                  │
│                                                                         │
│      sourceId: string                                                  │
│        // Unique ID (e.g., YouTube video ID)                           │
│                                                                         │
│      sourceKind: string                                                │
│        // Type of content (e.g., "youtube")                            │
│                                                                         │
│      ocrAvailable: boolean                                             │
│        // Whether OCR text extraction was performed                    │
│                                                                         │
│      slides: Array<{                                                   │
│        index: number                                                   │
│          // Slide number (0-based)                                     │
│                                                                         │
│        timestamp: number                                               │
│          // Time in video (seconds)                                    │
│                                                                         │
│        imageUrl: string                                                │
│          // URL to slide image (local file:// or http://)             │
│                                                                         │
│        ocrText?: string | null                                         │
│          // Extracted text from slide (if ocrAvailable)                │
│                                                                         │
│        ocrConfidence?: number | null                                   │
│          // OCR confidence score (0-1)                                 │
│      }>                                                                 │
│    }                                                                    │
│                                                                         │
│    Wire format:                                                         │
│      event: slides                                                      │
│      data: {"sourceUrl":"https://...","slides":[{...},...]}            │
│      (blank line)                                                       │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ 5. METRICS EVENT (performance data, once at end)                       │
│                                                                         │
│    Purpose: Provide timing and usage statistics                        │
│                                                                         │
│    Payload: SseMetricsData {                                           │
│      elapsedMs: number                                                 │
│        // Total elapsed time (milliseconds)                            │
│                                                                         │
│      summary: string                                                   │
│        // Human-readable summary (e.g., "12.3s, 234 tokens, $0.0012") │
│                                                                         │
│      details: string | null                                            │
│        // Optional detailed breakdown                                  │
│                                                                         │
│      summaryDetailed: string                                           │
│        // Verbose summary with all metrics                             │
│                                                                         │
│      detailsDetailed: string | null                                    │
│        // Verbose details                                              │
│    }                                                                    │
│                                                                         │
│    Wire format:                                                         │
│      event: metrics                                                     │
│      data: {"elapsedMs":12345,"summary":"12.3s, 234 tokens",...}       │
│      (blank line)                                                       │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ 6. DONE EVENT (session complete, sent last)                            │
│                                                                         │
│    Purpose: Signal that the session is complete and stream will close  │
│                                                                         │
│    Payload: {} (empty object)                                          │
│                                                                         │
│    Client behavior:                                                     │
│      ├─ Mark summary as complete                                       │
│      ├─ Enable export/copy buttons                                     │
│      └─ Close SSE connection                                           │
│                                                                         │
│    Wire format:                                                         │
│      event: done                                                        │
│      data: {}                                                           │
│      (blank line)                                                       │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ 7. ERROR EVENT (sent on failure, replaces "done")                      │
│                                                                         │
│    Purpose: Report errors to client                                    │
│                                                                         │
│    Payload: { message: string }                                        │
│      // Error description (e.g., "Failed to extract content")          │
│                                                                         │
│    Client behavior:                                                     │
│      ├─ Display error message                                          │
│      ├─ Mark session as failed                                         │
│      └─ Close SSE connection                                           │
│                                                                         │
│    Wire format:                                                         │
│      event: error                                                       │
│      data: {"message":"Failed to extract content: timeout"}            │
│      (blank line)                                                       │
└─────────────────────────────────────────────────────────────────────────┘

TYPICAL EVENT SEQUENCE:

Successful summarization:
  meta → status → status → chunk → chunk → ... → chunk → metrics → done

Error during extraction:
  meta → status → error

Cached result (instant):
  meta (with summaryFromCache=true) → chunk (single, full summary) → metrics → done

With slides:
  meta → status → chunk → ... → chunk → slides → metrics → done
```

### SSE Stream Example

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

event: meta
data: {"model":"openai/gpt-5-mini","modelLabel":"GPT-5 Mini","inputSummary":"12,345 words, YouTube transcript","summaryFromCache":false}

event: status
data: {"text":"Generating summary..."}

event: chunk
data: {"text":"This video provides "}

event: chunk
data: {"text":"an overview of "}

event: chunk
data: {"text":"web architecture patterns."}

event: chunk
data: {"text":" Key topics include:\n\n"}

event: chunk
data: {"text":"- Client-server architecture\n"}

event: chunk
data: {"text":"- RESTful APIs\n"}

event: chunk
data: {"text":"- Database design\n"}

event: metrics
data: {"elapsedMs":12345,"summary":"12.3s, 234 tokens, $0.0012","details":null,"summaryDetailed":"Elapsed: 12.3s, Prompt tokens: 12345, Completion tokens: 234, Total: 12579, Cost: $0.0012","detailsDetailed":null}

event: done
data: {}

(connection closes)
```

### Client Implementation (Browser Extension)

```javascript
// src/entrypoints/background.ts (simplified)

const eventSource = new EventSource(
  `http://127.0.0.1:8787/v1/summarize/${sessionId}/events`,
  { headers: { authorization: `Bearer ${token}` } }
);

let summaryText = "";

eventSource.addEventListener("meta", (e) => {
  const meta = JSON.parse(e.data);
  sendToPanel({ type: "ui:state", state: { model: meta.model, ... } });
});

eventSource.addEventListener("status", (e) => {
  const { text } = JSON.parse(e.data);
  sendToPanel({ type: "ui:status", status: text });
});

eventSource.addEventListener("chunk", (e) => {
  const { text } = JSON.parse(e.data);
  summaryText += text;
  sendToPanel({ type: "chunk", text });
});

eventSource.addEventListener("done", () => {
  sendToPanel({ type: "complete", summary: summaryText });
  eventSource.close();
});

eventSource.addEventListener("error", (e) => {
  const { message } = JSON.parse(e.data);
  sendToPanel({ type: "error", message });
  eventSource.close();
});
```

---

## 12. Configuration System

The configuration system merges settings from multiple sources with a clear precedence hierarchy.

```
CONFIGURATION HIERARCHY (highest to lowest priority)

1. CLI FLAGS (override everything)
   ├─ --model=openai/gpt-5
   ├─ --length=long
   ├─ --no-cache
   └─ etc.

2. ENVIRONMENT VARIABLES
   ├─ SUMMARIZE_MODEL=openai/gpt-5
   ├─ SUMMARIZE_LENGTH=long
   ├─ OPENAI_API_KEY=sk-...
   └─ etc.

3. CONFIG FILE: ~/.summarize/config.json (JSON5 format)
   {
     model: {
       default: "openai/gpt-5-mini",
       auto: [ /* auto rules */ ]
     },
     cache: {
       enabled: true,
       ttlDays: 30
     }
   }

4. BUILT-IN DEFAULTS (lowest priority)
   ├─ model: "auto"
   ├─ length: "medium"
   ├─ cache: enabled
   └─ etc.

┌─────────────────────────────────────────────────────────────────────────┐
│ CONFIG LOADER: src/config.ts (loadSummarizeConfig)                     │
│                                                                         │
│ export type SummarizeConfig = {                                        │
│   model: ModelConfig                                                   │
│   env: EnvConfig                                                       │
│   cache: CacheConfig                                                   │
│   media: {                                                             │
│     cache: MediaCacheConfig                                            │
│   }                                                                     │
│   slides: SlideSettings                                                │
│   ui: UiConfig                                                         │
│   cli: CliConfig                                                       │
│   apiKeys: ApiKeysConfig                                               │
│   openai: OpenAiConfig                                                 │
│   anthropic: AnthropicConfig                                           │
│   google: GoogleConfig                                                 │
│   xai: XaiConfig                                                       │
│   nvidia: NvidiaConfig                                                 │
│   logging: LoggingConfig                                               │
│ }                                                                       │
└─────────────────────────────────────────────────────────────────────────┘

CONFIG SECTIONS:

┌─────────────────────────────────────────────────────────────────────────┐
│ 1. MODEL CONFIG                                                         │
│                                                                         │
│ model: {                                                                │
│   default: "auto" | "<provider>/<model>"                               │
│     // Default model when not specified                                │
│                                                                         │
│   auto: AutoRule[]                                                     │
│     // Auto-selection rules (see Section 8)                            │
│     // Example:                                                         │
│     [                                                                   │
│       {                                                                 │
│         when: ["video"],                                               │
│         candidates: ["google/gemini-3-flash-preview"]                  │
│       },                                                                │
│       {                                                                 │
│         when: ["text", "website"],                                     │
│         bands: [                                                        │
│           { token: { max: 50000 }, candidates: ["openai/gpt-5-mini"] },│
│           { token: { min: 50000 }, candidates: ["openai/gpt-5"] }      │
│         ]                                                               │
│       }                                                                 │
│     ]                                                                   │
│                                                                         │
│   length: {                                                             │
│     presets: {                                                          │
│       short: { maxCharacters: 500 },                                   │
│       medium: { maxCharacters: 2000 },                                 │
│       long: { maxCharacters: 8000 }                                    │
│     }                                                                   │
│   }                                                                     │
│ }                                                                       │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ 2. ENV CONFIG (custom environment variables)                           │
│                                                                         │
│ env: {                                                                  │
│   // Key-value pairs merged into process.env for subprocesses          │
│   // Useful for setting provider-specific variables                    │
│   "CUSTOM_VAR": "value"                                                 │
│ }                                                                       │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ 3. CACHE CONFIG                                                         │
│                                                                         │
│ cache: {                                                                │
│   enabled: true,                                                        │
│     // Enable/disable all caching (extract, summary, transcript)       │
│                                                                         │
│   ttlDays: 30,                                                          │
│     // Time-to-live for cache entries (days)                           │
│                                                                         │
│   maxMb: 512,                                                           │
│     // Maximum cache size (megabytes)                                  │
│                                                                         │
│   path: "~/.summarize/cache"                                           │
│     // Custom cache directory (optional)                               │
│ }                                                                       │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ 4. MEDIA CACHE CONFIG (separate from main cache)                       │
│                                                                         │
│ media: {                                                                │
│   cache: {                                                              │
│     enabled: true,                                                      │
│     maxMb: 2048,           // 2 GB default (larger than main cache)    │
│     ttlDays: 7,            // Shorter TTL (media files are big)        │
│     path: "~/.summarize/media-cache",                                  │
│     verify: "size"         // "none" | "size" | "hash"                 │
│   }                                                                     │
│ }                                                                       │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ 5. SLIDES CONFIG (for presentation/video slide extraction)             │
│                                                                         │
│ slides: {                                                               │
│   enabled: true,           // Extract slides from videos               │
│   ocrEnabled: false,       // Run OCR on slide images                  │
│   interval: 5,             // Extract one slide every N seconds        │
│   format: "jpg",           // Image format: "jpg" | "png"              │
│   quality: 85,             // JPEG quality (1-100)                     │
│   width: 1280              // Max image width (pixels)                 │
│ }                                                                       │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ 6. UI CONFIG (terminal and extension UI preferences)                   │
│                                                                         │
│ ui: {                                                                   │
│   theme: "aurora",         // "aurora" | "ember" | "moss" | "mono"     │
│   color: true,             // Enable ANSI colors                       │
│   trueColor: "auto",       // "auto" | "always" | "never"              │
│   progressEnabled: true    // Show spinners and progress bars          │
│ }                                                                       │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ 7. CLI CONFIG (CLI provider settings)                                  │
│                                                                         │
│ cli: {                                                                  │
│   enabled: ["claude", "gemini"],                                       │
│     // Which CLI providers to enable                                   │
│                                                                         │
│   claude: {                                                             │
│     binary: "/usr/local/bin/claude",                                   │
│     model: "opus-4",                                                    │
│     extraArgs: ["--verbose"]                                           │
│   },                                                                    │
│                                                                         │
│   gemini: {                                                             │
│     binary: "/usr/local/bin/gemini",                                   │
│     model: "gemini-3-flash"                                            │
│   },                                                                    │
│                                                                         │
│   autoFallback: {                                                       │
│     enabled: true,                                                      │
│       // Try CLI providers if native API fails                         │
│                                                                         │
│     onlyWhenNoApiKeys: true,                                           │
│       // Only use CLI if no API key configured                         │
│                                                                         │
│     order: ["claude", "gemini", "codex"]                               │
│       // Fallback order                                                │
│   }                                                                     │
│ }                                                                       │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ 8. API KEYS CONFIG (alternative to environment variables)              │
│                                                                         │
│ apiKeys: {                                                              │
│   openai: "sk-...",                                                     │
│   anthropic: "sk-ant-...",                                             │
│   google: "AIza...",                                                    │
│   xai: "xai-...",                                                       │
│   nvidia: "nvapi-...",                                                  │
│   zai: "z-...",                                                         │
│   openrouter: "sk-or-...",                                             │
│   apify: "apify_...",                                                   │
│   firecrawl: "fc-...",                                                  │
│   fal: "fal_..."                                                        │
│ }                                                                       │
│                                                                         │
│ Note: Environment variables take precedence over config file keys      │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ 9. PROVIDER-SPECIFIC CONFIGS                                           │
│                                                                         │
│ openai: {                                                               │
│   baseUrl: "https://api.openai.com/v1",                                │
│     // Override API base URL (for proxies)                             │
│                                                                         │
│   useChatCompletions: true,                                            │
│     // Use /chat/completions vs /completions                           │
│                                                                         │
│   whisperUsdPerMinute: 0.006                                           │
│     // Cost estimation for Whisper transcription                       │
│ },                                                                      │
│                                                                         │
│ anthropic: {                                                            │
│   baseUrl: "https://api.anthropic.com"                                 │
│ },                                                                      │
│                                                                         │
│ google: {                                                               │
│   baseUrl: "https://generativelanguage.googleapis.com"                 │
│ },                                                                      │
│                                                                         │
│ xai: {                                                                  │
│   baseUrl: "https://api.x.ai/v1"                                       │
│ },                                                                      │
│                                                                         │
│ nvidia: {                                                               │
│   baseUrl: "https://integrate.api.nvidia.com/v1"                       │
│ }                                                                       │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ 10. LOGGING CONFIG                                                      │
│                                                                         │
│ logging: {                                                              │
│   enabled: false,          // Enable file logging                      │
│   level: "info",           // "debug" | "info" | "warn" | "error"     │
│   format: "json",          // "json" | "pretty"                        │
│   file: "~/.summarize/logs/app.log",                                   │
│   maxMb: 10,               // Max log file size                        │
│   maxFiles: 5              // Max number of rotated log files          │
│ }                                                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

### Example Config File

```json5
// ~/.summarize/config.json

{
  // Model configuration
  model: {
    default: "auto",
    auto: [
      {
        when: ["video", "youtube"],
        candidates: ["google/gemini-3-flash-preview"]
      },
      {
        when: ["text", "website"],
        bands: [
          { token: { max: 50000 }, candidates: ["openai/gpt-5-mini"] },
          { token: { min: 50000 }, candidates: ["openai/gpt-5"] }
        ]
      }
    ],
    length: {
      presets: {
        short: { maxCharacters: 500 },
        medium: { maxCharacters: 2000 },
        long: { maxCharacters: 8000 }
      }
    }
  },

  // Cache settings
  cache: {
    enabled: true,
    ttlDays: 30,
    maxMb: 512
  },

  // Media cache (separate)
  media: {
    cache: {
      enabled: true,
      maxMb: 2048,
      ttlDays: 7,
      verify: "size"
    }
  },

  // Slides extraction
  slides: {
    enabled: true,
    ocrEnabled: false,
    interval: 5
  },

  // UI preferences
  ui: {
    theme: "aurora",
    color: true,
    progressEnabled: true
  },

  // CLI providers
  cli: {
    enabled: ["claude"],
    claude: {
      binary: "/usr/local/bin/claude",
      model: "opus-4"
    },
    autoFallback: {
      enabled: true,
      onlyWhenNoApiKeys: true
    }
  },

  // API keys (alternative to env vars)
  apiKeys: {
    openai: "sk-...",
    anthropic: "sk-ant-..."
  },

  // Provider overrides
  openai: {
    baseUrl: "https://api.openai.com/v1"
  }
}
```

### Config File Location

The config file is loaded from:
1. Custom path (if `--config=/path/to/config.json` flag provided)
2. `~/.summarize/config.json` (default)
3. `$XDG_CONFIG_HOME/summarize/config.json` (Linux/Unix)

The file format is **JSON5**, which supports:
- Comments (`//` and `/* */`)
- Trailing commas
- Unquoted keys
- Single quotes

### Environment Variable Mapping

```
CLI Flag             | Environment Variable        | Config Path
---------------------|-----------------------------|-----------------------
--model=<id>         | SUMMARIZE_MODEL             | model.default
--length=<preset>    | SUMMARIZE_LENGTH            | (no config equivalent)
--no-cache           | SUMMARIZE_CACHE=false       | cache.enabled
--theme=<name>       | SUMMARIZE_THEME             | ui.theme
(API key)            | OPENAI_API_KEY              | apiKeys.openai
(API key)            | ANTHROPIC_API_KEY           | apiKeys.anthropic
(API key)            | GEMINI_API_KEY              | apiKeys.google
```

---

## Conclusion

This architecture document provides a comprehensive overview of the Summarize project's design. The system is built with modularity, extensibility, and developer experience in mind.

### Key Architectural Principles

1. **Separation of Concerns**: Core logic (extraction, prompts) is isolated from CLI/daemon/extension
2. **Dependency Injection**: All external dependencies (fetch, env, cache) are injectable for testing
3. **Provider Abstraction**: LLM providers are swappable via a unified interface
4. **Graceful Degradation**: Multiple fallback strategies (provider → OpenRouter → CLI)
5. **Progressive Enhancement**: Features like caching and slides are optional add-ons

### For New Developers

If you're new to TypeScript/web development and come from C/C++/Java:

- **async/await**: Similar to futures/promises, but with native syntax
- **ESM imports**: JavaScript's module system (import/export vs require)
- **pnpm workspaces**: Monorepo tool (like Bazel or Maven multi-module projects)
- **SQLite**: Embedded database (like SQLite in C, but with Node.js bindings)
- **SSE**: Server-Sent Events (one-way push from server, unlike WebSockets which are bidirectional)
- **Preact**: Lightweight React (virtual DOM UI framework)
- **WXT**: Build tool for browser extensions (compiles TypeScript → multiple browser targets)

### Further Reading

- **Code Style**: See `.oxlintrc.json` for linting rules
- **Testing**: `vitest` test files in `*.test.ts`
- **Build Process**: `scripts/build-cli.mjs` for CLI bundling
- **Extension Build**: `apps/chrome-extension/wxt.config.ts`

For questions or contributions, see the main README.md.

---

**Document Version:** 1.0
**Architecture Version:** 0.11.2
**Last Updated:** February 15, 2026
