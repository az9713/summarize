# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Summarize** (`@steipete/summarize`) is a link/document summarization tool shipping as a CLI, a Chrome/Firefox browser extension, and a local daemon. It extracts content from URLs, files, PDFs, media, YouTube, podcasts, and RSS, then generates streaming Markdown summaries via multiple LLM providers.

## Monorepo Structure

pnpm workspace with lockstep versioning. Three workspace roots:

- **Root** (`@steipete/summarize`) — CLI + daemon + terminal UX. Entry point: `src/cli.ts`
- **`packages/core`** (`@steipete/summarize-core`) — Library surface for programmatic use. Content extraction, prompts, shared contracts. No CLI entrypoints.
- **`apps/chrome-extension`** — Preact + WXT browser extension (Chrome MV3 / Firefox MV3)

Publish order: core first, then CLI. Import from apps should prefer `@steipete/summarize-core` to avoid pulling CLI-only deps.

## Commands

```bash
# Install
pnpm install

# Build (core first, then lib + CLI)
pnpm build

# Quality gate (format + lint + test with coverage)
pnpm check

# Individual steps
pnpm format:check          # oxfmt
pnpm lint                  # oxlint (type-aware)
pnpm lint:fix              # oxlint fix + format
pnpm test                  # vitest run
pnpm test:coverage         # vitest with v8 coverage (75% threshold)

# Run CLI during development (via tsx)
pnpm summarize <args>
pnpm s <args>              # shorthand alias

# Extension
pnpm -C apps/chrome-extension build           # Chrome
pnpm -C apps/chrome-extension build:firefox    # Firefox
pnpm -C apps/chrome-extension test:chrome      # E2E (Playwright, chromium)

# Daemon
pnpm summarize daemon restart
pnpm summarize daemon status

# Run a single test file
pnpm vitest run tests/article.test.ts
# Run tests matching a pattern
pnpm vitest run -t "pattern"

# Type checking
pnpm typecheck
```

## Architecture

### CLI Execution Flow

```
src/cli.ts → src/cli-main.ts → src/run/runner.ts
  ├─ Preflight: help, daemon, slides, transcriber routing
  ├─ Commander program setup + flag parsing (src/flags.ts)
  ├─ Config: loadSummarizeConfig() from src/config.ts
  ├─ Input resolution: URL, file, or stdin
  ├─ Model selection: buildAutoModelAttempts() in src/model-auto.ts
  ├─ Flow dispatch:
  │   ├─ URL → src/run/flows/url/flow.ts (runUrlFlow)
  │   └─ File → src/run/flows/asset/ (asset pipeline)
  ├─ Summary engine: src/run/summary-engine.ts (createSummaryEngine)
  ├─ LLM call: src/llm/generate-text.ts → providers/{openai,anthropic,google}.ts
  └─ Output rendering: src/tty/ (themes, markdown, progress)
```

### Key Modules (src/)

| Path | Purpose |
|------|---------|
| `run/runner.ts` | Main CLI orchestration |
| `run/summary-engine.ts` | Summary generation pipeline, streaming + non-streaming |
| `run/flows/url/` | URL processing: extract → markdown → summarize → output |
| `run/flows/asset/` | File processing: detect → extract → preprocess → media → summarize |
| `llm/generate-text.ts` | Text generation: generateTextWithModelId / streamTextWithModelId |
| `llm/model-id.ts` | Model ID parsing: `provider/model` format → ParsedModelId |
| `llm/providers/` | Per-provider implementations (OpenAI, Anthropic, Google, xAI, Z.AI, NVIDIA) |
| `config.ts` | Config loading + merging. Precedence: CLI flags > env > ~/.summarize/config.json > defaults |
| `flags.ts` | CLI flag parsers (length, format, modes, durations) |
| `cache.ts` | SQLite cache: kinds = extract, summary, transcript, chat, slides. TTL-based. |
| `model-auto.ts` | Auto model selection: matches content kind + token bands → candidate list |
| `daemon/server.ts` | HTTP server on 127.0.0.1:8787, SSE streaming, Bearer token auth |
| `daemon/agent.ts` | Agent responses for browser extension chat (8 tools) |
| `tty/theme.ts` | Theme system: aurora (default), ember, moss, mono |
| `slides/` | Video slide extraction via ffmpeg scene detection + OCR |
| `shared/sse-events.ts` | SSE event types: meta, status, chunk, slides, metrics, done, error |

### Core Library (packages/core/src/)

| Path | Purpose |
|------|---------|
| `content/link-preview/client.ts` | createLinkPreviewClient() — content extraction orchestrator |
| `content/link-preview/content/` | HTML parsing (Cheerio + JSDOM + @mozilla/readability + sanitize-html) |
| `content/transcript/` | Transcript providers: YouTube web API, Apify, yt-dlp + Whisper |
| `prompts/link-summary.ts` | buildLinkSummaryPrompt() — parameterized summary prompt builder |
| `prompts/summary-lengths.ts` | Length presets: short/medium/long/xl/xxl with char targets |
| `openai/` | OpenAI-compatible API utilities |
| `shared/contracts.ts` | Shared type contracts (SummaryLength, etc.) |

### Daemon Server (src/daemon/)

HTTP server on 127.0.0.1:8787. Key routes:

| Route | Method | Purpose |
|-------|--------|---------|
| `/health` | GET | Health check (no auth) |
| `/v1/ping` | GET | Auth verification (Bearer token) |
| `/v1/summarize` | POST | Start summarization (returns session ID) |
| `/v1/summarize/{id}/events` | GET | SSE stream for session |
| `/v1/agent` | POST | Agent chat (SSE or JSON) |
| `/v1/tools` | GET | Available tools (yt-dlp, ffmpeg, tesseract) |
| `/v1/models` | GET | Available LLM models |
| `/v1/summarize/{id}/slides` | GET | Cached slides for session |

Sessions: 30 min max lifetime, 2000 event buffer. Platform services: launchd (macOS), systemd (Linux), schtasks (Windows).

### Extension (apps/chrome-extension/src/)

Preact 11 + WXT. Side panel UI → Background service worker → Daemon HTTP/SSE.

Message flow: `panel:summarize` → background → `POST /v1/summarize` → SSE events → `bg:summarize-stream` → panel UI.

### LLM Provider System

Model IDs use gateway format: `provider/model` (e.g., `openai/gpt-5-mini`). Parsed by `parseGatewayStyleModelId()` in `src/llm/model-id.ts`. Providers: openai, anthropic, google, xai, zai, nvidia. OpenRouter supported via `openrouter/` prefix or `forceOpenRouter`. CLI providers: claude, codex, gemini, agent.

Auto-selection (`--model auto`): `buildAutoModelAttempts()` in `src/model-auto.ts` builds ordered candidate list by content kind (text, website, youtube, image, video, file) and token count bands. Each candidate tried in order until one succeeds.

## Testing

- Framework: Vitest 4 with v8 coverage
- Tests live in `tests/**/*.test.ts`
- Setup: `tests/setup.ts` (disables local Whisper in tests via `SUMMARIZE_DISABLE_LOCAL_WHISPER_CPP=1`)
- Coverage thresholds: 75% across branches, functions, lines, statements
- Excluded from coverage: daemon (integration-tested), slides extraction, OS/browser integration, barrel/type files
- Extension E2E: Playwright (chromium only; Firefox unreliable due to `moz-extension://` limitations)
- vitest.config.ts aliases map `@steipete/summarize-core/*` to source for tests

## Tooling

- **Formatter:** oxfmt (not Prettier) — run `pnpm format`
- **Linter:** oxlint with TypeScript type-aware mode — only two rules: `no-floating-promises` and `no-misused-promises` (catches missing `await`)
- **TypeScript:** ES2023 target, NodeNext modules, strict mode
- **Node.js:** >=22 required
- **Package manager:** pnpm 10.25.0+
- **Build:** tsc for library, esbuild for CLI bundle, WXT for extension

## Multi-Agent Awareness

Multiple agents often work in this folder. If you see files/changes you don't recognize, ignore them and list them at the end. Never commit in `vendor/summarize` — treat it as read-only.

## Rebuild After Changes

When modifying extension or daemon code, run both in order:
1. `pnpm -C apps/chrome-extension build`
2. `pnpm summarize daemon restart`

## Commits

Use Conventional Commits: `type: message` (feat, fix, docs, refactor, test, chore, etc.)

## Key Patterns

- **Config precedence:** CLI flags > env vars > `~/.summarize/config.json` (JSON5) > defaults
- **Streaming:** SSE for daemon↔extension, AsyncIterable for CLI LLM output
- **Caching:** SQLite at `~/.summarize/cache/`, media cache separate (2GB, 7-day TTL)
- **Error handling:** Typed error classes, friendly messages, verbose mode for full stacks
- **ESM:** All files use ESM imports with `.js` extensions (even for `.ts` source files)
- **Workspace imports:** `@steipete/summarize-core` resolves to `packages/core`
