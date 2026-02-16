# Summarize 📝 — Chrome Side Panel + CLI

> **This repository is a clone of [https://github.com/steipete/summarize](https://github.com/steipete/summarize).**
> The original project is created and maintained by [Peter Steinberger (@steipete)](https://github.com/steipete).
> This clone is for study, documentation, and local development purposes.
> For the latest updates, issues, and contributions, please refer to the [original repository](https://github.com/steipete/summarize).

---

![GitHub Repo Banner](https://ghrb.waren.build/banner?header=Summarize%F0%9F%93%9D&subheader=Chrome+Side+Panel+%2B+CLI&bg=f3f4f6&color=1f2937&support=true)

<!-- Created with GitHub Repo Banner by Waren Gonzaga: https://ghrb.waren.build -->

Fast summaries from URLs, files, and media. Works in the terminal, a Chrome Side Panel and Firefox Sidebar.

**0.11.0 preview (unreleased):** this README reflects the upcoming release.

## 0.11.0 preview highlights (most interesting first)

- Chrome Side Panel **chat** (streaming agent + history) inside the sidebar.
- **YouTube slides**: screenshots + OCR + transcript cards, timestamped seek, OCR/Transcript toggle.
- Media-aware summaries: auto‑detect video/audio vs page content.
- Streaming Markdown + metrics + cache‑aware status.
- CLI supports URLs, files, podcasts, YouTube, audio/video, PDFs.

## Feature overview

- URLs, files, and media: web pages, PDFs, images, audio/video, YouTube, podcasts, RSS.
- Slide extraction for video sources (YouTube/direct media) with OCR + timestamped cards.
- Transcript-first media flow: published transcripts when available, Whisper fallback when not. Audio/video pipeline: `yt-dlp` (download) → `ffmpeg` (transcode/split) → `whisper.cpp` or cloud Whisper (speech→text) → LLM summary. All optional — only needed for audio/video content.
- Streaming output with Markdown rendering, metrics, and cache-aware status.
- Local, paid, and free models: OpenAI‑compatible local endpoints, paid providers, plus an OpenRouter free preset.
- Output modes: Markdown/text, JSON diagnostics, extract-only, metrics, timing, and cost estimates.
- Smart default: if content is shorter than the requested length, we return it as-is (use `--force-summary` to override).

## Get the extension (recommended)

![Summarize extension screenshot](docs/assets/summarize-extension.png)

One‑click summarizer for the current tab. Chrome Side Panel + Firefox Sidebar + local daemon for streaming Markdown.

**Chrome Web Store:** [Summarize Side Panel](https://chromewebstore.google.com/detail/summarize/cejgnmmhbbpdmjnfppjdfkocebngehfg)

YouTube slide screenshots (from the browser):

![Summarize YouTube slide screenshots](docs/assets/youtube-slides.png)

### Beginner quickstart (extension)

1. Install the CLI (choose one):
   - **npm** (cross‑platform): `npm i -g @steipete/summarize`
   - **Homebrew** (macOS arm64): `brew install steipete/tap/summarize`
2. Install the extension (Chrome Web Store link above) and open the Side Panel.
3. The panel shows a token + install command. Run it in Terminal:
   - `summarize daemon install --token <TOKEN>`

### The Daemon — What, Why, and How

#### What is it?

The daemon is a lightweight local HTTP server that runs in the background on your machine. It listens on `127.0.0.1:8787` (localhost only — never exposed to the network) and acts as the bridge between the browser extension and all the heavy processing that browsers can't do themselves.

The implementation lives in [`src/daemon/server.ts`](src/daemon/server.ts). The entry point is `runDaemonServer()` which creates a Node.js `http.Server`:

```typescript
// src/daemon/server.ts:452-468
export async function runDaemonServer({
  env, fetchImpl, config,
  port = config.port ?? DAEMON_PORT_DEFAULT,  // default: 8787
  signal, onListening, onSessionEvent,
}: { ... }): Promise<void> {
```

#### Why is it needed?

Chrome extensions run in a strict sandbox. They **cannot**:

1. **Run local binaries** — `yt-dlp` (YouTube audio download), `ffmpeg` (video processing, slide extraction), `tesseract` (OCR), `whisper.cpp` (audio transcription) are all command-line tools that only run natively on your OS
2. **Access the filesystem** — SQLite caching, saving slide images, reading local files
3. **Make unrestricted API calls** — CORS policies block direct browser-to-LLM-provider calls
4. **Run long computations** — Chrome kills idle service workers after ~30 seconds

The daemon solves all of these by running as a normal Node.js process on your machine:

```
Without daemon:  Extension --X--> LLM APIs        (blocked by CORS)
                 Extension --X--> yt-dlp, ffmpeg   (can't run binaries)
                 Extension --X--> filesystem        (sandboxed)

With daemon:     Extension --HTTP--> Daemon ---> LLM APIs      (no CORS issue)
                                     Daemon ---> yt-dlp, ffmpeg (native access)
                                     Daemon ---> SQLite cache   (filesystem)
                                     Daemon ---> slide images   (filesystem)
```

#### How does it work?

**Authentication:** Every `/v1/*` request requires a Bearer token. The token is generated when you install the daemon and shared with the extension. See [`src/daemon/server.ts:512-517`](src/daemon/server.ts):

```typescript
const token = readBearerToken(req);
const authed = token && token === config.token;
if (pathname.startsWith("/v1/") && !authed) {
  json(res, 401, { ok: false, error: "unauthorized" }, cors);
  return;
}
```

**HTTP API routes** (defined in [`src/daemon/server.ts`](src/daemon/server.ts)):

| Route | Method | Purpose |
|-------|--------|---------|
| `/health` | GET | Health check — no auth required. Returns `{ok, pid, version}` |
| `/v1/ping` | GET | Auth verification — confirms Bearer token is valid |
| `/v1/summarize` | POST | Start a summarization. Accepts `{url, text, title, model, length, ...}`. Returns `{ok, id}` with a session ID |
| `/v1/summarize/{id}/events` | GET | **SSE stream** — subscribe to real-time summary events for a session |
| `/v1/agent` | POST | Chat/agent requests from the extension (SSE or JSON response) |
| `/v1/tools` | GET | Reports which local tools are available (`yt-dlp`, `ffmpeg`, `tesseract`) |
| `/v1/models` | GET | Lists available LLM models based on configured API keys |
| `/v1/logs` | GET | Tail daemon logs (for troubleshooting) |
| `/v1/processes` | GET | List active/completed processing tasks |
| `/v1/summarize/{id}/slides` | GET | Fetch extracted slide images for a session |
| `/v1/slides/{sourceId}/{index}` | GET | Stable URL for individual slide images (survives session cleanup) |

**Streaming via SSE (Server-Sent Events):** When the extension requests a summary, the daemon creates a session and streams results back in real-time using SSE. Events are defined in [`src/shared/sse-events.ts`](src/shared/sse-events.ts):

```typescript
// src/shared/sse-events.ts:32-40
export type SseEvent =
  | { event: "meta";    data: { model, modelLabel, inputSummary, summaryFromCache } }
  | { event: "status";  data: { text: string } }           // "Extracting content..."
  | { event: "chunk";   data: { text: string } }           // streaming summary text
  | { event: "slides";  data: { sourceUrl, slides[] } }    // extracted slide images
  | { event: "metrics"; data: { elapsedMs, summary } }     // timing/cost info
  | { event: "done";    data: {} }                          // session complete
  | { event: "error";   data: { message: string } };       // something went wrong
```

Sessions are buffered in memory (max 2000 events or 512KB per session, 30-minute lifetime) so clients can reconnect without losing data. See [`src/daemon/server.ts:234-238`](src/daemon/server.ts):

```typescript
const MAX_SESSION_BUFFER_EVENTS = 2000;
const MAX_SESSION_BUFFER_BYTES = 512 * 1024;
const MAX_SESSION_LIFETIME_MS = 30 * 60_000;
```

**Typical request flow:**

1. Extension sends `POST /v1/summarize` with `{url: "https://example.com", model: "auto", length: "medium"}`
2. Daemon creates a session (UUID), responds `{ok: true, id: "abc-123"}`
3. Extension subscribes to `GET /v1/summarize/abc-123/events` (SSE)
4. Daemon extracts content (using core library's `createLinkPreviewClient`)
5. Daemon calls LLM via `generateTextWithModelId` / `streamTextWithModelId`
6. Daemon pushes events to all subscribed clients: `meta` → `status` → `chunk` (repeated) → `metrics` → `done`

**Platform service management:** The daemon registers as an auto-starting system service so it's always ready when you open the extension. The implementation is in [`src/daemon/cli.ts`](src/daemon/cli.ts) with platform-specific backends:

- **macOS:** `launchd` — see [`src/daemon/launchd.ts`](src/daemon/launchd.ts) (label: `com.steipete.summarize.daemon`)
- **Linux:** `systemd` user service — see [`src/daemon/systemd.ts`](src/daemon/systemd.ts)
- **Windows:** Scheduled Task — see [`src/daemon/schtasks.ts`](src/daemon/schtasks.ts) (task: `Summarize Daemon`)

**Daemon commands:**

```bash
summarize daemon install --token <TOKEN>   # Register + start as system service
summarize daemon install --token <TOKEN> --dev  # Use local source (for developers)
summarize daemon status                    # Check if running
summarize daemon restart                   # Restart after code changes
summarize daemon stop                      # Stop the service
summarize daemon uninstall                 # Remove autostart (keeps config)
```

**Configuration:** Stored in `~/.summarize/daemon.json` with the token, port, and a snapshot of environment variables (so the daemon has access to API keys even when started by the system service manager).

If you only want the **CLI**, you can skip the daemon install entirely — the CLI runs everything directly in-process.

Notes:

- Summarization only runs when the Side Panel is open.
- Auto mode summarizes on navigation (incl. SPAs); otherwise use the button.
- Tip: configure `free` via `summarize refresh-free` (needs `OPENROUTER_API_KEY`). Add `--set-default` to set model=`free`.

More:

- Step-by-step install: [apps/chrome-extension/README.md](apps/chrome-extension/README.md)
- Architecture + troubleshooting: [docs/chrome-extension.md](docs/chrome-extension.md)
- Firefox compatibility notes: [apps/chrome-extension/docs/firefox.md](apps/chrome-extension/docs/firefox.md)

### Slides (extension)

- Select **Video + Slides** in the Summarize picker.
- Slides render at the top; expand to full‑width cards with timestamps.
- Click a slide to seek the video; toggle **Transcript/OCR** when OCR is significant.
- Requirements: `yt-dlp` + `ffmpeg` for extraction; `tesseract` for OCR. Missing tools show an in‑panel notice.

### Advanced (unpacked / dev)

1. Build + load the extension (unpacked):
   - Chrome: `pnpm -C apps/chrome-extension build`
     - `chrome://extensions` → Developer mode → Load unpacked
     - Pick: `apps/chrome-extension/.output/chrome-mv3`
   - Firefox: `pnpm -C apps/chrome-extension build:firefox`
     - `about:debugging#/runtime/this-firefox` → Load Temporary Add-on
     - Pick: `apps/chrome-extension/.output/firefox-mv3/manifest.json`
2. Open Side Panel/Sidebar → copy token.
3. Install daemon in dev mode:
   - `pnpm summarize daemon install --token <TOKEN> --dev`

## CLI

![Summarize CLI screenshot](docs/assets/summarize-cli.png)

### Install

Requires Node 22+.

- npx (no install):

```bash
npx -y @steipete/summarize "https://example.com"
```

- npm (global):

```bash
npm i -g @steipete/summarize
```

- npm (library / minimal deps):

```bash
npm i @steipete/summarize-core
```

```ts
import { createLinkPreviewClient } from "@steipete/summarize-core/content";
```

- Homebrew (custom tap):

```bash
brew install steipete/tap/summarize
```

Apple Silicon only (arm64).

### CLI vs extension

- **CLI only:** just install via npm/Homebrew and run `summarize ...` (no daemon needed).
- **Chrome/Firefox extension:** install the CLI **and** run `summarize daemon install --token <TOKEN>` so the Side Panel can stream results and use local tools.

> **Extension + YouTube:** The extension fully supports YouTube video summarization. Navigate to any YouTube video and click Summarize — the daemon extracts the transcript server-side (via YouTube web API, yt-dlp, or Whisper) and summarizes it.
>
> **Extension limitation:** The extension cannot read PDFs rendered in the browser's built-in PDF viewer (e.g., `arxiv.org/pdf/...` links) or other embedded binary files. The PDF viewer is a separate browser component whose content is not exposed to the DOM. Workaround: use the HTML version of the page (e.g., `arxiv.org/abs/...` instead of `arxiv.org/pdf/...`), or use the CLI for PDFs and media.

### Quickstart

```bash
summarize "https://example.com"
```

### Inputs

URLs or local paths:

```bash
summarize "/path/to/file.pdf" --model google/gemini-3-flash-preview
summarize "https://example.com/report.pdf" --model google/gemini-3-flash-preview
summarize "/path/to/audio.mp3"
summarize "/path/to/video.mp4"
```

Stdin (pipe content using `-`):

```bash
echo "content" | summarize -
pbpaste | summarize -
# binary stdin also works (PDF/image/audio/video bytes)
cat /path/to/file.pdf | summarize -
```

**Notes:**

- Stdin has a 50MB size limit
- The `-` argument tells summarize to read from standard input
- Text stdin is treated as UTF-8 text (whitespace-only input is rejected as empty)
- Binary stdin is preserved as raw bytes and file type is auto-detected when possible
- Useful for piping clipboard content or command output

YouTube (supports `youtube.com` and `youtu.be`):

```bash
summarize "https://youtu.be/dQw4w9WgXcQ" --youtube auto
```

Podcast RSS (transcribes latest enclosure):

```bash
summarize "https://feeds.npr.org/500005/podcast.xml"
```

Apple Podcasts episode page:

```bash
summarize "https://podcasts.apple.com/us/podcast/2424-jelly-roll/id360084272?i=1000740717432"
```

Spotify episode page (best-effort; may fail for exclusives):

```bash
summarize "https://open.spotify.com/episode/5auotqWAXhhKyb9ymCuBJY"
```

### Output length

`--length` controls how much output we ask for (guideline), not a hard cap.

```bash
summarize "https://example.com" --length long
summarize "https://example.com" --length 20k
```

- Presets: `short|medium|long|xl|xxl`
- Character targets: `1500`, `20k`, `20000`
- Optional hard cap: `--max-output-tokens <count>` (e.g. `2000`, `2k`)
  - Provider/model APIs still enforce their own maximum output limits.
  - If omitted, no max token parameter is sent (provider default).
  - Prefer `--length` unless you need a hard cap.
- Short content: when extracted content is shorter than the requested length, the CLI returns the content as-is.
  - Override with `--force-summary` to always run the LLM.
- Minimums: `--length` numeric values must be >= 50 chars; `--max-output-tokens` must be >= 16.
- Preset targets (source of truth: `packages/core/src/prompts/summary-lengths.ts`):
  - short: target ~900 chars (range 600-1,200)
  - medium: target ~1,800 chars (range 1,200-2,500)
  - long: target ~4,200 chars (range 2,500-6,000)
  - xl: target ~9,000 chars (range 6,000-14,000)
  - xxl: target ~17,000 chars (range 14,000-22,000)

### What file types work?

Best effort and provider-dependent. These usually work well:

- `text/*` and common structured text (`.txt`, `.md`, `.json`, `.yaml`, `.xml`, ...)
  - Text-like files are inlined into the prompt for better provider compatibility.
- PDFs: `application/pdf` (provider support varies; Google is the most reliable here)
- Images: `image/jpeg`, `image/png`, `image/webp`, `image/gif`
- Audio/Video: `audio/*`, `video/*` (local audio/video files MP3/WAV/M4A/OGG/FLAC/MP4/MOV/WEBM automatically transcribed, when supported by the model)

Notes:

- If a provider rejects a media type, the CLI fails fast with a friendly message.
- xAI models do not support attaching generic files (like PDFs) via the AI SDK; use Google/OpenAI/Anthropic for those.

### Model ids

Use gateway-style ids: `<provider>/<model>`.

Examples:

- `openai/gpt-5-mini`
- `anthropic/claude-sonnet-4-5`
- `xai/grok-4-fast-non-reasoning`
- `google/gemini-3-flash-preview`
- `zai/glm-4.7`
- `openrouter/openai/gpt-5-mini` (force OpenRouter)

Note: some models/providers do not support streaming or certain file media types. When that happens, the CLI prints a friendly error (or auto-disables streaming for that model when supported by the provider).

### Limits

- Text inputs over 10 MB are rejected before tokenization.
- Text prompts are preflighted against the model input limit (LiteLLM catalog), using a GPT tokenizer.

### Common flags

```bash
summarize <input> [flags]
```

Use `summarize --help` or `summarize help` for the full help text.

- `--model <provider/model>`: which model to use (defaults to `auto`)
- `--model auto`: automatic model selection + fallback (default)
- `--model <name>`: use a config-defined model (see Configuration)
- `--timeout <duration>`: `30s`, `2m`, `5000ms` (default `2m`)
- `--retries <count>`: LLM retry attempts on timeout (default `1`)
- `--length short|medium|long|xl|xxl|s|m|l|<chars>`
- `--language, --lang <language>`: output language (`auto` = match source)
- `--max-output-tokens <count>`: hard cap for LLM output tokens
- `--cli [provider]`: use a CLI provider (`--model cli/<provider>`). Supports `claude`, `gemini`, `codex`, `agent`. If omitted, uses auto selection with CLI enabled.
- `--stream auto|on|off`: stream LLM output (`auto` = TTY only; disabled in `--json` mode)
- `--plain`: keep raw output (no ANSI/OSC Markdown rendering)
- `--no-color`: disable ANSI colors
- `--theme <name>`: CLI theme (`aurora`, `ember`, `moss`, `mono`)
- `--format md|text`: website/file content format (default `text`)
- `--markdown-mode off|auto|llm|readability`: HTML -> Markdown mode (default `readability`)
- `--preprocess off|auto|always`: controls `uvx markitdown` usage (default `auto`)
  - Install `uvx`: `brew install uv` (or https://astral.sh/uv/)
- `--extract`: print extracted content and exit (URLs only; stdin `-` is not supported)
  - Deprecated alias: `--extract-only`
- `--slides`: extract slides for YouTube/direct video URLs and render them inline in the summary narrative (auto-renders inline in supported terminals)
- `--slides-ocr`: run OCR on extracted slides (requires `tesseract`)
- `--slides-dir <dir>`: base output dir for slide images (default `./slides`)
- `--slides-scene-threshold <value>`: scene detection threshold (0.1-1.0)
- `--slides-max <count>`: maximum slides to extract (default `6`)
- `--slides-min-duration <seconds>`: minimum seconds between slides
- `--json`: machine-readable output with diagnostics, prompt, `metrics`, and optional summary
- `--verbose`: debug/diagnostics on stderr
- `--metrics off|on|detailed`: metrics output (default `on`)

### Coding CLIs (Codex, Claude, Gemini, Agent)

Summarize can use common coding CLIs as local model backends:

- `codex` -> `--cli codex` / `--model cli/codex/<model>`
- `claude` -> `--cli claude` / `--model cli/claude/<model>`
- `gemini` -> `--cli gemini` / `--model cli/gemini/<model>`
- `agent` (Cursor Agent CLI) -> `--cli agent` / `--model cli/agent/<model>`

Requirements:

- Binary installed and on `PATH` (or set `CODEX_PATH`, `CLAUDE_PATH`, `GEMINI_PATH`, `AGENT_PATH`)
- Provider authenticated (`codex login`, `claude auth`, `gemini` login flow, `agent login` or `CURSOR_API_KEY`)

Quick smoke test:

```bash
printf "Summarize CLI smoke input.\nOne short paragraph. Reply can be brief.\n" >/tmp/summarize-cli-smoke.txt

summarize --cli codex --plain --timeout 2m /tmp/summarize-cli-smoke.txt
summarize --cli claude --plain --timeout 2m /tmp/summarize-cli-smoke.txt
summarize --cli gemini --plain --timeout 2m /tmp/summarize-cli-smoke.txt
summarize --cli agent --plain --timeout 2m /tmp/summarize-cli-smoke.txt
```

Set explicit CLI allowlist/order:

```json
{
  "cli": { "enabled": ["codex", "claude", "gemini", "agent"] }
}
```

Configure implicit auto CLI fallback:

```json
{
  "cli": {
    "autoFallback": {
      "enabled": true,
      "onlyWhenNoApiKeys": true,
      "order": ["claude", "gemini", "codex", "agent"]
    }
  }
}
```

More details: [`docs/cli.md`](docs/cli.md)

### Auto model ordering

`--model auto` builds candidate attempts from built-in rules (or your `model.rules` overrides).
CLI attempts are prepended when:

- `cli.enabled` is set (explicit allowlist/order), or
- implicit auto selection is active and `cli.autoFallback` is enabled.

Default fallback behavior: only when no API keys are configured, order `claude, gemini, codex, agent`, and remember/prioritize last successful provider (`~/.summarize/cli-state.json`).

Set explicit CLI attempts:

```json
{
  "cli": { "enabled": ["gemini"] }
}
```

Disable implicit auto CLI fallback:

```json
{
  "cli": { "autoFallback": { "enabled": false } }
}
```

Note: explicit `--model auto` does not trigger implicit auto CLI fallback unless `cli.enabled` is set.

### Website extraction (Firecrawl + Markdown)

Non-YouTube URLs go through a fetch -> extract pipeline. When direct fetch/extraction is blocked or too thin,
`--firecrawl auto` can fall back to Firecrawl (if configured).

- `--firecrawl off|auto|always` (default `auto`)
- `--extract --format md|text` (default `text`; if `--format` is omitted, `--extract` defaults to `md` for non-YouTube URLs)
- `--markdown-mode off|auto|llm|readability` (default `readability`)
  - `auto`: use an LLM converter when configured; may fall back to `uvx markitdown`
  - `llm`: force LLM conversion (requires a configured model key)
  - `off`: disable LLM conversion (still may return Firecrawl Markdown when configured)
- Plain-text mode: use `--format text`.

### YouTube transcripts

`--youtube auto` tries best-effort web transcript endpoints first. When captions are not available, it falls back to:

1. Apify (if `APIFY_API_TOKEN` is set): uses a scraping actor (`faVsWy9VTSNVIhWpR`)
2. yt-dlp + Whisper (if `yt-dlp` is available): downloads audio, then transcribes with local `whisper.cpp` when installed
   (preferred), otherwise falls back to OpenAI (`OPENAI_API_KEY`) or FAL (`FAL_KEY`)

Environment variables for yt-dlp mode:

- `YT_DLP_PATH` - optional path to yt-dlp binary (otherwise `yt-dlp` is resolved via `PATH`)
- `SUMMARIZE_WHISPER_CPP_MODEL_PATH` - optional override for the local `whisper.cpp` model file
- `SUMMARIZE_WHISPER_CPP_BINARY` - optional override for the local binary (default: `whisper-cli`)
- `SUMMARIZE_DISABLE_LOCAL_WHISPER_CPP=1` - disable local whisper.cpp (force remote)
- `OPENAI_API_KEY` - OpenAI Whisper transcription
- `OPENAI_WHISPER_BASE_URL` - optional OpenAI-compatible Whisper endpoint override
- `FAL_KEY` - FAL AI Whisper fallback

Apify costs money but tends to be more reliable when captions exist.

### Slide extraction (YouTube + direct video URLs)

Extract slide screenshots (scene detection via `ffmpeg`) and optional OCR:

```bash
summarize "https://www.youtube.com/watch?v=..." --slides
summarize "https://www.youtube.com/watch?v=..." --slides --slides-ocr
```

Outputs are written under `./slides/<sourceId>/` (or `--slides-dir`). OCR results are included in JSON output
(`--json`) and stored in `slides.json` inside the slide directory. When scene detection is too sparse, the
extractor also samples at a fixed interval to improve coverage.
When using `--slides`, supported terminals (kitty/iTerm/Konsole) render inline thumbnails automatically inside the
summary narrative (the model inserts `[slide:N]` markers). Timestamp links are clickable when the terminal supports
OSC-8 (YouTube/Vimeo/Loom/Dropbox). If inline images are unsupported, Summarize prints a note with the on-disk
slide directory.

Use `--slides --extract` to print the full timed transcript and insert slide images inline at matching timestamps.

Format the extracted transcript as Markdown (headings + paragraphs) via an LLM:

```bash
summarize "https://www.youtube.com/watch?v=..." --extract --format md --markdown-mode llm
```

### Media transcription (Whisper)

Local audio/video files are transcribed first, then summarized. `--video-mode transcript` forces
direct media URLs (and embedded media) through Whisper first. Prefers local `whisper.cpp` when available; otherwise requires
`OPENAI_API_KEY` or `FAL_KEY`.

### Local ONNX transcription (Parakeet/Canary)

Summarize can use NVIDIA Parakeet/Canary ONNX models via a local CLI you provide. Auto selection (default) prefers ONNX when configured.

- Setup helper: `summarize transcriber setup`
- Install `sherpa-onnx` from upstream binaries/build (Homebrew may not have a formula)
- Auto selection: set `SUMMARIZE_ONNX_PARAKEET_CMD` or `SUMMARIZE_ONNX_CANARY_CMD` (no flag needed)
- Force a model: `--transcriber parakeet|canary|whisper|auto`
- Docs: `docs/nvidia-onnx-transcription.md`

### Verified podcast services (2025-12-25)

Run: `summarize <url>`

- Apple Podcasts
- Spotify
- Amazon Music / Audible podcast pages
- Podbean
- Podchaser
- RSS feeds (Podcasting 2.0 transcripts when available)
- Embedded YouTube podcast pages (e.g. JREPodcast)

Transcription: prefers local `whisper.cpp` when installed; otherwise uses OpenAI Whisper or FAL when keys are set.

### Translation paths

`--language/--lang` controls the output language of the summary (and other LLM-generated text). Default is `auto`.

When the input is audio/video, the CLI needs a transcript first. The transcript comes from one of these paths:

1. Existing transcript (preferred)
   - YouTube: uses `youtubei` / `captionTracks` when available.
   - Podcasts: uses Podcasting 2.0 RSS `<podcast:transcript>` (JSON/VTT) when the feed publishes it.
2. Whisper transcription (fallback)
   - YouTube: falls back to yt-dlp (audio download) + Whisper transcription when configured; Apify is a last resort.
   - Prefers local `whisper.cpp` when installed + model available.
   - Otherwise uses cloud Whisper (OpenAI `OPENAI_API_KEY`) or FAL (`FAL_KEY`).

For direct media URLs, use `--video-mode transcript` to force transcribe -> summarize:

```bash
summarize https://example.com/file.mp4 --video-mode transcript --lang en
```

### Configuration

Single config location:

- `~/.summarize/config.json`

Supported keys today:

```json
{
  "model": { "id": "openai/gpt-5-mini" },
  "env": { "OPENAI_API_KEY": "sk-..." },
  "ui": { "theme": "ember" }
}
```

Shorthand (equivalent):

```json
{
  "model": "openai/gpt-5-mini"
}
```

Also supported:

- `model: { "mode": "auto" }` (automatic model selection + fallback; see [docs/model-auto.md](docs/model-auto.md))
- `model.rules` (customize candidates / ordering)
- `models` (define presets selectable via `--model <preset>`)
- `env` (generic env var defaults; process env still wins)
- `apiKeys` (legacy shortcut, mapped to env names; prefer `env` for new configs)
- `cache.media` (media download cache: TTL 7 days, 2048 MB cap by default; `--no-media-cache` disables)
- `media.videoMode: "auto"|"transcript"|"understand"`
- `slides.enabled` / `slides.max` / `slides.ocr` / `slides.dir` (defaults for `--slides`)
- `ui.theme: "aurora"|"ember"|"moss"|"mono"`
- `openai.useChatCompletions: true` (force OpenAI-compatible chat completions)

Note: the config is parsed leniently (JSON5), but comments are not allowed. Unknown keys are ignored.

Media cache defaults:

```json
{
  "cache": {
    "media": { "enabled": true, "ttlDays": 7, "maxMb": 2048, "verify": "size" }
  }
}
```

Note: `--no-cache` bypasses summary caching only (LLM output). Extract/transcript caches still apply. Use `--no-media-cache` to skip media files.

Precedence:

1. `--model`
2. `SUMMARIZE_MODEL`
3. `~/.summarize/config.json`
4. default (`auto`)

Theme precedence:

1. `--theme`
2. `SUMMARIZE_THEME`
3. `~/.summarize/config.json` (`ui.theme`)
4. default (`aurora`)

Environment variable precedence:

1. process env
2. `~/.summarize/config.json` (`env`)
3. `~/.summarize/config.json` (`apiKeys`, legacy)

### Environment variables

Set the key matching your chosen `--model`:

- Optional fallback defaults can be stored in config:
  - `~/.summarize/config.json` -> `"env": { "OPENAI_API_KEY": "sk-..." }`
  - process env always takes precedence
  - legacy `"apiKeys"` still works (mapped to env names)

- `OPENAI_API_KEY` (for `openai/...`)
- `NVIDIA_API_KEY` (for `nvidia/...`)
- `ANTHROPIC_API_KEY` (for `anthropic/...`)
- `XAI_API_KEY` (for `xai/...`)
- `Z_AI_API_KEY` (for `zai/...`; supports `ZAI_API_KEY` alias)
- `GEMINI_API_KEY` (for `google/...`)
  - also accepts `GOOGLE_GENERATIVE_AI_API_KEY` and `GOOGLE_API_KEY` as aliases

OpenAI-compatible chat completions toggle:

- `OPENAI_USE_CHAT_COMPLETIONS=1` (or set `openai.useChatCompletions` in config)

UI theme:

- `SUMMARIZE_THEME=aurora|ember|moss|mono`
- `SUMMARIZE_TRUECOLOR=1` (force 24-bit ANSI)
- `SUMMARIZE_NO_TRUECOLOR=1` (disable 24-bit ANSI)

OpenRouter (OpenAI-compatible):

- Set `OPENROUTER_API_KEY=...`
- Prefer forcing OpenRouter per model id: `--model openrouter/<author>/<slug>`
- Built-in preset: `--model free` (uses a default set of OpenRouter `:free` models)

### `summarize refresh-free`

Quick start: make free the default (keep `auto` available)

```bash
summarize refresh-free --set-default
summarize "https://example.com"
summarize "https://example.com" --model auto
```

Regenerates the `free` preset (`models.free` in `~/.summarize/config.json`) by:

- Fetching OpenRouter `/models`, filtering `:free`
- Skipping models that look very small (<27B by default) based on the model id/name
- Testing which ones return non-empty text (concurrency 4, timeout 10s)
- Picking a mix of smart-ish (bigger `context_length` / output cap) and fast models
- Refining timings and writing the sorted list back

If `--model free` stops working, run:

```bash
summarize refresh-free
```

Flags:

- `--runs 2` (default): extra timing runs per selected model (total runs = 1 + runs)
- `--smart 3` (default): how many smart-first picks (rest filled by fastest)
- `--min-params 27b` (default): ignore models with inferred size smaller than N billion parameters
- `--max-age-days 180` (default): ignore models older than N days (set 0 to disable)
- `--set-default`: also sets `"model": "free"` in `~/.summarize/config.json`

Example:

```bash
OPENROUTER_API_KEY=sk-or-... summarize "https://example.com" --model openrouter/meta-llama/llama-3.1-8b-instruct:free
OPENROUTER_API_KEY=sk-or-... summarize "https://example.com" --model openrouter/minimax/minimax-m2.5
```

If your OpenRouter account enforces an allowed-provider list, make sure at least one provider
is allowed for the selected model. When routing fails, `summarize` prints the exact providers to allow.

Legacy: `OPENAI_BASE_URL=https://openrouter.ai/api/v1` (and either `OPENAI_API_KEY` or `OPENROUTER_API_KEY`) also works.

NVIDIA API Catalog (OpenAI-compatible; free credits):

- Set `NVIDIA_API_KEY=...`
- Optional: `NVIDIA_BASE_URL=https://integrate.api.nvidia.com/v1`
- Credits: API Catalog trial starts with 1000 free API credits on signup (up to 5000 total via “Request More” in the API Catalog profile)
- Pick a model id from `/v1/models` (examples: fast `stepfun-ai/step-3.5-flash`, strong but slower `z-ai/glm5`)

```bash
export NVIDIA_API_KEY="nvapi-..."
summarize "https://example.com" --model nvidia/stepfun-ai/step-3.5-flash
```

Z.AI (OpenAI-compatible):

- `Z_AI_API_KEY=...` (or `ZAI_API_KEY=...`)
- Optional base URL override: `Z_AI_BASE_URL=...`

Optional services:

- `FIRECRAWL_API_KEY` (website extraction fallback)
- `YT_DLP_PATH` (path to yt-dlp binary for audio extraction)
- `FAL_KEY` (FAL AI API key for audio transcription via Whisper)
- `APIFY_API_TOKEN` (YouTube transcript fallback)

### Model limits

The CLI uses the LiteLLM model catalog for model limits (like max output tokens):

- Downloaded from: `https://raw.githubusercontent.com/BerriAI/litellm/main/model_prices_and_context_window.json`
- Cached at: `~/.summarize/cache/`

### Library usage (optional)

Recommended (minimal deps):

- `@steipete/summarize-core/content`
- `@steipete/summarize-core/prompts`

Compatibility (pulls in CLI deps):

- `@steipete/summarize/content`
- `@steipete/summarize/prompts`

### Development

```bash
pnpm install
pnpm check
```

## Documentation

### For Users

- **[User Guide](docs/USER_GUIDE.md)** — Step-by-step guide with 12 practical use cases. Start here if you're new to Summarize.

### For Developers

- **[Developer Guide](docs/DEVELOPER_GUIDE.md)** — Complete onboarding guide for developers new to TypeScript/Node.js. Covers environment setup, daily workflow, key concepts (async/await, SSE, fetch), and how to make changes.
- **[Architecture](docs/ARCHITECTURE.md)** — Comprehensive architecture documentation with ASCII diagrams showing system overview, execution flows, daemon protocol, extension messaging, LLM provider system, and caching.

### Reference

- [CLI providers and config](docs/cli.md)
- [Auto model rules](docs/model-auto.md)
- [Website extraction](docs/website.md)
- [YouTube handling](docs/youtube.md)
- [Media pipeline](docs/media.md)
- [Config schema and precedence](docs/config.md)
- [Chrome extension architecture](docs/chrome-extension.md)
- [NVIDIA ONNX transcription](docs/nvidia-onnx-transcription.md)
- [Docs index](docs/README.md)

## Troubleshooting

- "Receiving end does not exist": Chrome did not inject the content script yet.
  - Extension details -> Site access -> On all sites (or allow this domain)
  - Reload the tab once.
- "Failed to fetch" / daemon unreachable:
  - `summarize daemon status`
  - Logs: `~/.summarize/logs/daemon.err.log`
- More troubleshooting: see [User Guide - Troubleshooting](docs/USER_GUIDE.md#part-8-troubleshooting)

License: MIT
