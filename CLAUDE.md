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

`src/cli.ts` → `src/cli-main.ts` → `src/run/runner.ts` → content extraction → LLM summarization → streaming Markdown output.

### Key Modules (src/)

| Path | Purpose |
|------|---------|
| `run/runner.ts` | Main CLI orchestration |
| `run/summary-engine.ts` | Summary generation pipeline |
| `run/flows/` | Processing flows for different input types |
| `llm/providers/` | LLM integrations (OpenAI, Anthropic, Google, CLI-based) |
| `llm/generate-text.ts` | Text generation with provider dispatch |
| `config.ts` | Config parsing (CLI flags > env vars > ~/.summarize/config.json > defaults) |
| `flags.ts` | CLI flag definitions |
| `cache.ts` | SQLite cache (extract, summary, transcript, chat, slides) |
| `daemon/` | Local HTTP server for browser extension integration |
| `tty/` | Terminal UI: themes (aurora/ember/moss/mono), spinners, markdown rendering |
| `model-auto.ts` | Auto model selection with fallback chain |
| `slides/` | Video slide extraction + OCR |

### Core Library (packages/core/src/)

| Path | Purpose |
|------|---------|
| `content/link-preview/` | Content extraction orchestration |
| `content/link-preview/content/` | HTML parsing, article extraction (Readability + Cheerio) |
| `content/transcript/` | Transcript handling (YouTube, podcasts) |
| `prompts/` | Summary prompt templates and length presets |
| `openai/` | OpenAI-compatible API utilities |

### Extension (apps/chrome-extension/src/)

Preact 11 + WXT. Side panel UI with headless components via @zag-js. Connects to local daemon for streaming results and media tool access.

## Testing

- Framework: Vitest 4 with v8 coverage
- Tests live in `tests/**/*.test.ts`
- Setup: `tests/setup.ts` (disables local Whisper in tests)
- Coverage thresholds: 75% across branches, functions, lines, statements
- Excluded from coverage: daemon (integration-tested), slides extraction, OS/browser integration, barrel/type files
- Extension E2E: Playwright (chromium only; Firefox is unreliable due to `moz-extension://` limitations)

## Tooling

- **Formatter:** oxfmt (not Prettier) — run `pnpm format`
- **Linter:** oxlint with TypeScript type-aware mode — only two rules enabled: `no-floating-promises` and `no-misused-promises`
- **TypeScript:** ES2023 target, NodeNext modules, strict mode
- **Node.js:** >=22 required
- **Package manager:** pnpm 10.25.0+

## Multi-Agent Awareness

Multiple agents often work in this folder. If you see files/changes you don't recognize, ignore them and list them at the end. Never commit in `vendor/summarize` — treat it as read-only.

## Rebuild After Changes

When modifying extension or daemon code, run both in order:
1. `pnpm -C apps/chrome-extension build`
2. `pnpm summarize daemon restart`

## Commits

Use Conventional Commits: `type: message` (feat, fix, docs, refactor, test, chore, etc.)
