# Summarize User Guide

Welcome! This guide will help you get started with Summarize, a tool that creates concise summaries from URLs, files, and media using AI. Whether you're catching up on articles, learning from videos, or processing documents, Summarize can help.

## Part 1: What is Summarize?

Summarize is a tool that takes content and produces a concise summary using artificial intelligence. Think of it as having a smart assistant read something for you and tell you the key points.

### What can it summarize?

- Web pages and articles
- YouTube videos (including transcripts)
- PDF documents
- Images with text
- Audio files (podcasts, recordings, lectures)
- Video files
- Podcast episodes (from Apple Podcasts, Spotify, RSS feeds)

### How do you use it?

Summarize works in two ways:

1. **Command-line tool (CLI)**: Run commands in your Terminal or Command Prompt
2. **Browser extension**: Click a button in Chrome or Firefox to summarize the current page

### Why use Summarize instead of asking an AI chatbot directly?

You might wonder: "Why not just paste a URL into Claude, ChatGPT, or Claude Code and ask it to summarize?" For simple web pages, that works fine. Summarize is purpose-built for cases where general AI tools fall short:

| Capability | Summarize | AI chatbot (Claude Code, ChatGPT, etc.) |
|---|---|---|
| **YouTube videos** | Extracts transcript via yt-dlp + Whisper, then summarizes | Cannot transcribe video audio |
| **Podcasts / audio files** | Downloads and transcribes audio, then summarizes | Cannot process audio |
| **PDFs** | Direct PDF text extraction and parsing | Limited or no PDF parsing from URLs |
| **Video slide extraction** | Scene detection (ffmpeg) + OCR (tesseract) | Not available |
| **Caching** | SQLite cache — same URL twice is instant and free | Every request costs tokens |
| **Cost tracking** | Shows exact token usage and cost per summary | No per-request cost visibility |
| **Batch processing** | `summarize "url1" "url2" "url3"` in one command | One at a time |
| **Browser extension** | One-click summarize while browsing | Must switch to chat window |
| **Multiple providers** | Switch between OpenAI, Google, Anthropic, xAI, free models | Locked to one provider |
| **Length control** | Tuned presets: short/medium/long/xl/xxl | Prompt-dependent, less consistent |

**Use an AI chatbot when:**
- You need a one-off summary of a simple web page
- You're already in a chat session and want a quick answer
- You want to ask follow-up questions about the content

**Use Summarize when:**
- The content is YouTube, podcasts, audio, video, or PDFs
- You summarize URLs repeatedly (caching saves time and money)
- You want cost control or need to compare models
- You're browsing and want one-click summaries (extension)
- You need batch/scripting workflows

### What AI models does it use?

Summarize can work with multiple AI providers:

- **OpenAI** (GPT-4, GPT-5, etc.)
- **Google** (Gemini models - great for PDFs)
- **Anthropic** (Claude models)
- **xAI** (Grok models)
- **OpenRouter** (access to many models, including free options)
- **Local CLI providers** (Claude CLI, Gemini CLI, Codex)

You can use paid API services or free options. Don't worry - we'll walk through setting up a free option below.

---

## Part 2: Installation

### What you'll need

Before starting, you need **Node.js version 22 or higher** installed on your computer. Node.js is a platform that lets you run JavaScript programs (like Summarize) on your computer.

#### Installing Node.js

1. Go to [nodejs.org](https://nodejs.org)
2. Download the "LTS" version (Long Term Support)
3. Run the installer and follow the prompts
4. To verify it's installed, open Terminal (macOS/Linux) or Command Prompt (Windows) and type:
   ```bash
   node --version
   ```
   You should see a version number like `v22.0.0` or higher.

### 2.1 Quick Install (CLI only)

Choose the option that works best for your system:

#### macOS

**Option A: Using Homebrew (recommended for Apple Silicon Macs)**

If you have Homebrew installed:
```bash
brew install steipete/tap/summarize
```

Note: This is currently only available for Apple Silicon (M1/M2/M3) Macs.

**Option B: Using npm (works on any Mac)**

```bash
npm install -g @steipete/summarize
```

The `-g` flag means "global" - it installs Summarize so you can use it from anywhere.

#### Windows

First, make sure you have Node.js 22 or higher installed (see above). Then open Command Prompt or PowerShell and run:

```bash
npm install -g @steipete/summarize
```

#### Linux

First, install Node.js 22 or higher. You can use:
- Your package manager (apt, yum, etc.)
- [Node Version Manager (nvm)](https://github.com/nvm-sh/nvm)

Then install Summarize:

```bash
npm install -g @steipete/summarize
```

### 2.2 Optional Tools (ffmpeg, Whisper, yt-dlp, tesseract)

Summarize works out of the box for web pages and text. For **audio, video, YouTube, and podcasts**, it relies on optional local tools:

| Tool | What it does in Summarize | Required for |
|------|---------------------------|--------------|
| **ffmpeg** | Transcodes audio/video to MP3 for transcription. Splits large files into segments. Detects scene changes to extract slides from videos. `ffprobe` (bundled with ffmpeg) measures media duration for progress bars. | Audio/video transcription, slide extraction |
| **Whisper** (whisper.cpp) | Converts speech to text locally — no API key or cloud service needed. Summarize feeds it audio (via ffmpeg) and gets back a text transcript that the LLM then summarizes. | Free local transcription (alternative: OpenAI/Groq/FAL cloud APIs) |
| **yt-dlp** | Downloads audio/video from YouTube and other sites. Used when YouTube's web transcript API has no captions available. | YouTube videos without captions, slide extraction |
| **tesseract** | OCR — extracts text from slide screenshot images produced by ffmpeg scene detection. | Slide text extraction (optional) |

**The transcription pipeline:**

```
Audio/Video file or YouTube URL
  → yt-dlp downloads media (if URL)
  → ffmpeg transcodes to MP3 + splits if too large
  → Whisper (local) or cloud API converts speech → text transcript
  → LLM summarizes the transcript
```

**Installation:**

```bash
# macOS (Homebrew)
brew install ffmpeg yt-dlp tesseract

# For local Whisper transcription (free, no API key needed):
brew install whisper-cpp
# Or download from: https://github.com/ggerganov/whisper.cpp

# Windows (winget)
winget install ffmpeg
winget install yt-dlp
# Or download from their respective GitHub releases pages

# Linux (apt)
sudo apt install ffmpeg tesseract-ocr
pip install yt-dlp
```

**If you don't install these tools:**
- Web pages, articles, and text files work normally — no extra tools needed
- YouTube videos with existing captions still work (transcript fetched via web API)
- YouTube videos *without* captions, podcasts, and local audio/video files will fail transcription
- Slide extraction won't be available

**Cloud alternatives to local Whisper** (no local install needed, but requires API keys):
- OpenAI Whisper API: set `OPENAI_API_KEY`
- Groq: set `GROQ_API_KEY`
- FAL: set `FAL_KEY`

### 2.3 Verify Installation

After installation, verify it worked by checking the version:

```bash
summarize --version
```

You should see a version number displayed, like `0.11.2`.

To see all available commands and options:

```bash
summarize --help
```

### 2.3 Try Without Installing

If you want to try Summarize before installing it, you can use `npx`:

```bash
npx -y @steipete/summarize "https://example.com"
```

This downloads and runs Summarize temporarily. It's a great way to test it out!

---

## Part 3: Setting Up an API Key

To generate summaries, Summarize needs to communicate with an AI service. Most AI services require an API key - think of it as a password that lets Summarize use the AI on your behalf.

You have several options, from completely free to paid services with more features.

### 3.1 Free Option: OpenRouter

OpenRouter provides access to many AI models, including free options. This is a great way to get started without spending money.

#### Step 1: Create an OpenRouter account

1. Go to [openrouter.ai](https://openrouter.ai)
2. Click "Sign Up" or "Get Started"
3. Create an account (you can use Google, GitHub, or email)

#### Step 2: Get your API key

1. Once logged in, click on your profile or "API Keys"
2. Click "Create Key" or "New API Key"
3. Give it a name like "Summarize CLI"
4. Copy the key - it will look like `sk-or-v1-abc123...`

**Important**: Save this key somewhere safe! You won't be able to see it again.

#### Step 3: Save the key for Summarize

On **macOS or Linux**:

```bash
# Create the config directory
mkdir -p ~/.summarize

# Save your API key (replace YOUR-KEY with your actual key)
echo '{"env":{"OPENROUTER_API_KEY":"sk-or-v1-YOUR-KEY"}}' > ~/.summarize/config.json
```

On **Windows** (PowerShell):

```powershell
# Create the config directory
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.summarize"

# Save your API key (replace YOUR-KEY with your actual key)
'{"env":{"OPENROUTER_API_KEY":"sk-or-v1-YOUR-KEY"}}' | Out-File -FilePath "$env:USERPROFILE\.summarize\config.json" -Encoding utf8
```

#### Step 4: Set up free models

Now configure Summarize to use free models from OpenRouter:

```bash
summarize refresh-free --set-default
```

This command:
- Tests which free models are working
- Picks the best performing ones
- Sets them as your default

#### Step 5: Try it out!

```bash
summarize "https://en.wikipedia.org/wiki/Artificial_intelligence"
```

If you see a summary appear, congratulations! You're all set up with free AI summarization.

### 3.2 OpenAI (Paid)

OpenAI provides GPT models, which are very capable but require payment.

#### Step 1: Create an OpenAI account

1. Go to [platform.openai.com](https://platform.openai.com)
2. Sign up or log in
3. Add a payment method in your account settings

#### Step 2: Get your API key

1. Go to "API Keys" in your account
2. Click "Create new secret key"
3. Give it a name like "Summarize"
4. Copy the key (starts with `sk-...`)

#### Step 3: Save the key

Add it to your config file at `~/.summarize/config.json`:

```json
{
  "env": {
    "OPENAI_API_KEY": "sk-YOUR-KEY-HERE"
  },
  "model": "openai/gpt-5-mini"
}
```

Or set it as an environment variable:

**macOS/Linux:**
```bash
export OPENAI_API_KEY="sk-YOUR-KEY-HERE"
```

**Windows (PowerShell):**
```powershell
$env:OPENAI_API_KEY="sk-YOUR-KEY-HERE"
```

### 3.3 Google Gemini (Generous Free Tier)

Google's Gemini models have a generous free tier and work exceptionally well with PDFs.

#### Step 1: Get a Gemini API key

1. Go to [aistudio.google.com](https://aistudio.google.com)
2. Sign in with your Google account
3. Click "Get API key"
4. Create a new API key

#### Step 2: Save the key

Add it to `~/.summarize/config.json`:

```json
{
  "env": {
    "GEMINI_API_KEY": "YOUR-KEY-HERE"
  },
  "model": "google/gemini-3-flash-preview"
}
```

Or export it as an environment variable:

```bash
export GEMINI_API_KEY="YOUR-KEY-HERE"
```

#### Why use Gemini?

- Excellent at processing PDFs
- Very fast responses
- Generous free tier
- Good at understanding images

### 3.4 Anthropic Claude (Paid)

Claude models are known for their thoughtful, nuanced summaries.

#### Step 1: Get a Claude API key

1. Go to [console.anthropic.com](https://console.anthropic.com)
2. Create an account and add payment
3. Go to "API Keys"
4. Create a new key

#### Step 2: Save the key

Add to `~/.summarize/config.json`:

```json
{
  "env": {
    "ANTHROPIC_API_KEY": "sk-ant-YOUR-KEY-HERE"
  },
  "model": "anthropic/claude-sonnet-4-5"
}
```

### 3.5 Using CLI Providers (No API Key Needed!)

If you have the Claude CLI, Gemini CLI, or Codex installed and authenticated, Summarize can use them directly without needing API keys.

#### Check if you have a CLI provider installed

Try running one of these:
```bash
claude --version
gemini --version
codex --version
```

If any of these work, you can use that provider:

```bash
# Use Claude CLI
summarize "https://example.com" --cli claude

# Use Gemini CLI
summarize "https://example.com" --cli gemini

# Use Codex
summarize "https://example.com" --cli codex
```

#### Automatic CLI fallback

If you have no API keys configured, Summarize will automatically try to use available CLI providers. It will remember which one worked last time and prefer it next time.

### 3.6 How Auto Model Selection Works

When you run Summarize **without specifying `--model` or `--cli`**, it uses **auto-selection** (`--model auto` is the default). Auto-selection picks the best available model based on your API keys and the content type.

**Priority order for websites, YouTube, and text** (under 50k tokens):

| Priority | Model | Required API key |
|----------|-------|------------------|
| 1st | `google/gemini-3-flash-preview` | `GOOGLE_GENERATIVE_AI_API_KEY` |
| 2nd | `openai/gpt-5-mini` | `OPENAI_API_KEY` |
| 3rd | `anthropic/claude-sonnet-4-5` | `ANTHROPIC_API_KEY` |

Summarize tries each candidate in order and uses the **first one you have an API key for**. For example, if you only have `OPENAI_API_KEY` set, it will use `openai/gpt-5-mini`.

**Special cases:**
- **Video understanding**: prefers `google/gemini-3-flash-preview` (best video support)
- **Images**: prefers Google, then OpenAI, then Anthropic
- **PDFs/files**: prefers Google (best PDF support), then OpenAI, then Anthropic
- **Very large content** (200k+ tokens): tries `xai/grok-4-fast-non-reasoning` first (largest context window)

**If no API keys are set at all**, Summarize falls back to CLI providers in this order:
1. `claude` (Claude CLI)
2. `gemini` (Gemini CLI)
3. `codex` (OpenAI Codex CLI)
4. `agent` (Agent CLI)

**To override auto-selection**, use `--model` or `--cli`:

```bash
# Use a specific model
summarize "https://example.com" --model anthropic/claude-sonnet-4-5

# Use a CLI provider
summarize "https://example.com" --cli claude

# Set a default in config so you never need --model
# Add to ~/.summarize/config.json:
# { "model": "openai/gpt-5-mini" }
```

### 3.7 `--model auto` vs `--cli claude` — What's the Difference?

These are **two different ways** to reach an LLM:

| | `--model auto` (default) | `--cli claude` |
|---|---|---|
| **How it calls the LLM** | Direct HTTP API call (e.g., `api.openai.com`, `api.anthropic.com`) | Shells out to the `claude` CLI binary installed on your machine |
| **Authentication** | API key from env var (e.g., `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`) | The Claude CLI's own login session |
| **Which model** | First available by priority (Google → OpenAI → Anthropic) | Claude Sonnet (default for `--cli claude`) |
| **Billing** | Per-token API pricing from your provider | Your Claude subscription plan |

**Example:** If your only API key is `ANTHROPIC_API_KEY`, then both of these commands end up using Claude Sonnet:

```bash
# Auto-selects anthropic/claude-sonnet-4-5 (via direct API, billed per-token)
summarize "https://example.com"

# Uses claude CLI binary (billed via your Claude subscription)
summarize "https://example.com" --cli claude
```

The summary output is essentially the same — same model, same content. The difference is the billing path and how the request reaches Anthropic's servers.

**When to use which:**
- **`--model auto`** (or `--model provider/model`): when you have API keys and want direct API access with per-token billing
- **`--cli claude`**: when you have the Claude CLI installed and want to use your Claude subscription instead of API billing
- Other CLI options: `--cli gemini` (Gemini CLI), `--cli codex` (OpenAI Codex CLI)

---

## Part 4: Quick Start - Your First Summary

Now that you have an API key set up, let's create your first summary!

### Step 1: Open your terminal

- **macOS**: Open Terminal (find it in Applications > Utilities)
- **Windows**: Open Command Prompt or PowerShell (search in Start menu)
- **Linux**: Open your terminal application

### Step 2: Run your first summary

```bash
summarize "https://en.wikipedia.org/wiki/TypeScript"
```

### What happens next?

You'll see a series of steps:

1. **Fetching**: Summarize downloads the web page
2. **Extracting**: It pulls out the main text content
3. **Sending to AI**: The content is sent to your chosen AI model
4. **Streaming response**: The summary appears word by word as the AI writes it

### Understanding the output

After the summary, you'll see some metrics:
- **Tokens**: How much text was processed (input) and generated (output)
- **Time**: How long each step took
- **Cost**: Estimated cost (if using a paid API)
- **Cache**: Whether cached results were used (faster and cheaper!)

---

## Part 5: Educational Use Cases

Here are ten practical ways to use Summarize in real life, with step-by-step examples.

### Use Case 1: Summarize a News Article

Perfect for staying informed without spending too much time reading.

```bash
summarize "https://www.bbc.com/news/technology"
```

**Why this is useful:**
- Quickly catch up on current events
- Get the key facts from long articles
- Save time while staying informed

**Tip:** Use `--length short` for very quick summaries:
```bash
summarize "https://www.bbc.com/news/technology" --length short
```

### Use Case 2: Summarize a YouTube Video

Save time by reading a summary instead of watching a long video.

```bash
summarize "https://www.youtube.com/watch?v=dQw4w9WgXcQ"
```

**What happens:**
1. Summarize extracts the video transcript (either official captions or generated via speech recognition)
2. The transcript is sent to the AI
3. You get a summary of what the video covers

**Why this is useful:**
- Decide if a video is worth watching
- Review key points from educational videos
- Get information from videos when you can't watch with sound

**Note:** This works best with videos that have captions or clear speech.

### Use Case 3: Summarize a PDF Document

Great for research papers, reports, and long documents.

```bash
summarize "/path/to/document.pdf" --model google/gemini-3-flash-preview
```

**Why we use Gemini for PDFs:**
Google's Gemini models have excellent PDF support and can understand document structure, tables, and images within PDFs.

**Examples:**
```bash
# Local file
summarize "~/Documents/research-paper.pdf" --model google/gemini-3-flash-preview

# PDF from the web
summarize "https://example.com/report.pdf" --model google/gemini-3-flash-preview
```

**Tips:**
- For very long PDFs, use `--length long` to get more detail
- Use `--extract` to see the raw text extracted from the PDF

### Use Case 4: Get Different Summary Lengths

Control how detailed your summary should be.

```bash
# Very brief summary (~900 characters)
summarize "https://example.com" --length short

# Standard summary (~1,800 characters)
summarize "https://example.com" --length medium

# Detailed summary (~4,200 characters)
summarize "https://example.com" --length long

# Very detailed (~9,000 characters)
summarize "https://example.com" --length xl

# Extremely detailed (~17,000 characters)
summarize "https://example.com" --length xxl
```

**When to use each:**
- **short**: Quick overview, checking if content is relevant
- **medium**: Default, good for most articles
- **long**: Detailed understanding, educational content
- **xl**: Comprehensive analysis, research papers
- **xxl**: Maximum detail, book chapters, long documents

You can also specify exact character counts:
```bash
summarize "https://example.com" --length 3000
```

### Use Case 5: Summarize a Podcast Episode

Perfect for catching up on podcast episodes when you don't have time to listen.

```bash
# Apple Podcasts
summarize "https://podcasts.apple.com/us/podcast/the-daily/id1200361736"

# Spotify
summarize "https://open.spotify.com/episode/5auotqWAXhhKyb9ymCuBJY"

# RSS feed (summarizes the latest episode)
summarize "https://feeds.npr.org/500005/podcast.xml"
```

**What happens:**
1. Summarize downloads the podcast audio file
2. It transcribes the audio to text (using Whisper AI)
3. The transcript is summarized

**Why this is useful:**
- Decide which episodes to listen to
- Review key points from episodes you've heard
- Get information from podcasts when you can't listen

**Note:** Transcription can take a minute or two for long episodes.

### Use Case 6: Summarize in a Different Language

Get summaries in any language, regardless of the source language.

```bash
# Summary in Spanish
summarize "https://example.com" --lang spanish

# Summary in Japanese
summarize "https://example.com" --lang japanese

# Summary in German
summarize "https://example.com" --lang de

# Summary in French
summarize "https://example.com" --lang fr
```

**Supported formats:**
- Language names: `spanish`, `japanese`, `german`, `french`, `chinese`, etc.
- Short codes: `es`, `ja`, `de`, `fr`, `zh`, etc.
- Auto-detect (default): `--lang auto` (matches source language)

**Use cases:**
- Practice a foreign language
- Get information from foreign language sources in your language
- Share summaries with people who speak different languages

### Use Case 7: Extract Content Without Summarizing

Sometimes you just want the clean text from a cluttered web page.

```bash
# Extract as plain text
summarize "https://example.com" --extract

# Extract as Markdown (preserves formatting)
summarize "https://example.com" --extract --format md
```

**Why this is useful:**
- Remove ads and clutter from articles
- Save articles in a clean format
- Copy text for use elsewhere
- See what content would be summarized before running the AI

**Example:** Extract a recipe without all the blog content:
```bash
summarize "https://foodblog.com/chocolate-cake-recipe" --extract --format md > recipe.md
```

### Use Case 8: Pipe Content from Clipboard

Summarize text you've copied without saving it to a file first.

**macOS:**
```bash
pbpaste | summarize -
```

**Linux:**
```bash
xclip -selection clipboard -o | summarize -
```

**Windows (PowerShell):**
```powershell
Get-Clipboard | summarize -
```

**What the `-` means:**
The dash tells Summarize to read from "standard input" (stdin) - text piped into it from another command.

**Use cases:**
- Summarize text you've copied from anywhere
- Process output from other commands
- Quick summaries without creating files

**Example workflow:**
1. Read an email
2. Copy the entire message
3. Run `pbpaste | summarize -`
4. Get a quick summary of the key points

### Use Case 9: Summarize YouTube Videos with Slides

For presentation or tutorial videos, extract key slides with timestamps.

```bash
# Extract slides and include them in summary
summarize "https://www.youtube.com/watch?v=..." --slides

# Extract slides with text recognition (OCR)
summarize "https://www.youtube.com/watch?v=..." --slides --slides-ocr

# Control how many slides to extract
summarize "https://www.youtube.com/watch?v=..." --slides --slides-max 12
```

**What happens:**
1. Summarize detects scene changes in the video
2. Extracts screenshot images at key moments
3. Optionally runs OCR to extract text from slides
4. Creates a summary that references the slides
5. Saves slide images to a local folder

**Why this is useful:**
- Review presentation slides without watching the video
- Extract diagrams and visual content
- Get timestamped references to important moments
- Create study materials from educational videos

**Requirements:**
- `yt-dlp` (for video download)
- `ffmpeg` (for scene detection)
- `tesseract` (for OCR, optional)

**Note:** Slides are saved in `./slides/<video-id>/` by default. You can view them alongside the summary.

### Use Case 10: Use a Specific AI Model

Choose different models for different tasks based on quality, speed, and cost.

```bash
# Fast and cheap (OpenAI)
summarize "https://example.com" --model openai/gpt-5-mini

# High quality (Anthropic)
summarize "https://example.com" --model anthropic/claude-sonnet-4-5

# Best for PDFs (Google)
summarize "document.pdf" --model google/gemini-3-flash-preview

# Very fast (xAI)
summarize "https://example.com" --model xai/grok-4-fast-non-reasoning

# Free option (OpenRouter)
summarize "https://example.com" --model free
```

**How to choose:**
- **Speed matters**: `openai/gpt-5-mini`, `google/gemini-3-flash-preview`, `xai/grok-4-fast-non-reasoning`
- **Quality matters**: `anthropic/claude-sonnet-4-5`, `openai/gpt-5`
- **PDFs**: `google/gemini-3-flash-preview` (best PDF support)
- **No cost**: `--model free` (free OpenRouter models)
- **Local/private**: `--cli claude` or `--cli gemini` (if you have the CLI installed)

### Use Case 11: Free Summarization Setup

Set up completely free summarization using OpenRouter's free models.

```bash
# Test and configure free models
summarize refresh-free

# Set free models as your default
summarize refresh-free --set-default

# Now use Summarize without specifying --model free
summarize "https://example.com"

# You can still use other models when needed
summarize "https://example.com" --model auto
```

**What `refresh-free` does:**
1. Fetches the list of free models from OpenRouter
2. Tests each model to see if it's working
3. Selects the best performing free models
4. Saves them as the "free" preset in your config

**Why run this:**
- Free models change over time
- Some free models stop working
- This keeps your free option up to date

**Tip:** Run `summarize refresh-free` every few weeks to keep your free models fresh.

### Use Case 12: Summarize Local Audio and Video Files

Process recordings, lectures, meetings, or any local media files.

```bash
# Audio files
summarize "/path/to/meeting-recording.mp3"
summarize "/path/to/interview.wav"
summarize "/path/to/lecture.m4a"

# Video files
summarize "/path/to/conference-talk.mp4"
summarize "/path/to/tutorial.mov"
```

**Supported formats:**
- Audio: MP3, WAV, M4A, OGG, FLAC
- Video: MP4, MOV, WEBM, MKV

**What happens:**
1. Summarize transcribes the audio using Whisper AI
2. The transcript is sent to your chosen AI model
3. You get a summary of what was said

**Use cases:**
- Summarize meeting recordings
- Review lecture content
- Extract key points from interviews
- Process voice memos

**Requirements:**
- Local Whisper (recommended): Install `whisper.cpp`
- Or OpenAI API key (uses OpenAI's Whisper API)
- Or FAL API key (alternative transcription service)

**Tip:** For long recordings, consider using `--length long` to capture more detail.

---

## Part 6: Browser Extension Setup

The browser extension lets you summarize any web page with a single click from Chrome or Firefox.

> **YouTube support:** The extension fully supports YouTube video summarization. Just navigate to any YouTube video and click Summarize — the daemon extracts the transcript server-side using YouTube's web API, yt-dlp, or Whisper, then sends it to the LLM for summarization. No extra setup needed beyond the daemon.
>
> **Limitation — PDFs and binary files:** The extension **cannot** read:
> - PDFs rendered in the browser's built-in PDF viewer (e.g., `arxiv.org/pdf/...` links)
> - Embedded binary files (images, audio, video files)
> - Content behind JavaScript-heavy rendering that doesn't expose text to the DOM
>
> The browser's PDF viewer is a separate component — the extension can only access the HTML DOM, not the PDF content inside it. For YouTube, the URL alone is enough for the daemon to extract the transcript; for PDFs, the extension would need to download and forward the file, which it doesn't currently do.
>
> **Workarounds for PDFs:**
> - Navigate to the HTML version of the page (e.g., use `arxiv.org/abs/...` instead of `arxiv.org/pdf/...`)
> - Use the CLI for PDFs and other non-HTML content:
>   ```bash
>   summarize "https://arxiv.org/pdf/2507.11538" --model google/gemini-3-flash-preview
>   summarize "/path/to/document.pdf" --model google/gemini-3-flash-preview
>   ```

### 6.1 Install the Extension

#### For Chrome

1. Go to the [Chrome Web Store](https://chromewebstore.google.com/detail/summarize/cejgnmmhbbpdmjnfppjdfkocebngehfg)
2. Click "Add to Chrome"
3. Click "Add extension" to confirm

#### For Firefox

1. The Firefox version is currently available as a manual installation
2. See the [Firefox documentation](https://github.com/steipete/summarize/blob/main/apps/chrome-extension/docs/firefox.md) for detailed instructions

### 6.2 Install and Start the Daemon

The browser extension needs a local "daemon" (background service) to work. This is because the extension can't run heavy tools like video downloaders directly in the browser.

#### Why do you need a daemon?

The daemon runs on your computer (localhost only) and:
- Extracts content from web pages
- Downloads and transcribes media files
- Performs OCR on images
- Streams summaries back to the extension

Don't worry - it only runs on your local computer and requires a security token.

#### Step 1: Open the extension

1. Click the Summarize icon in your browser toolbar
2. If needed, pin it so it's always visible
3. Open the Side Panel (Chrome) or Sidebar (Firefox)

#### Step 2: Get your installation token

When you first open the extension, you'll see:
- A unique security token (like `abc123def456`)
- An installation command

#### Step 3: Install the daemon

Copy the command shown in the extension and run it in your terminal:

```bash
summarize daemon install --token abc123def456
```

Replace `abc123def456` with the actual token shown in your extension.

**What this does:**
- Installs a background service
- Configures it to start automatically
- Sets up the security token for communication

**Auto-start configuration:**
- **macOS**: Uses `launchd` (system service manager)
- **Linux**: Uses `systemd` (user service)
- **Windows**: Uses Task Scheduler

#### Step 4: Verify it's running

```bash
summarize daemon status
```

You should see that the daemon is running on port `8787`.

#### If you're running from the local codebase (developers)

If you cloned the Summarize repo and want to use your local version with the extension instead of a globally installed one:

```bash
cd /path/to/summarize

# 1. Install dependencies and build
pnpm install
pnpm build

# 2. Build the Chrome extension
pnpm -C apps/chrome-extension build

# 3. Load it as unpacked in Chrome:
#    - Go to chrome://extensions
#    - Enable "Developer mode" (top-right toggle)
#    - Click "Load unpacked"
#    - Select: apps/chrome-extension/.output/chrome-mv3

# 4. Open the Side Panel, copy the token, install daemon in dev mode:
pnpm summarize daemon install --token <TOKEN> --dev

# 5. Verify
pnpm summarize daemon status
```

The `--dev` flag tells the daemon to run from your local source code so your changes take effect.

After making code changes, rebuild and restart:
```bash
pnpm -C apps/chrome-extension build        # Rebuild extension
# Then click the reload icon on chrome://extensions
pnpm summarize daemon restart              # Restart daemon
```

### 6.3 Using the Extension

Now you're ready to summarize pages from your browser!

#### Basic usage

1. Navigate to any web page
2. Open the Summarize Side Panel
3. Click the "Summarize" button
4. Watch the summary stream in

#### Automatic mode

Enable auto-summarize in the extension settings to get summaries automatically when you navigate to new pages.

#### YouTube videos

When you're on a YouTube page:
1. You'll see a "Video + Slides" option
2. Click it to extract video slides and transcript
3. Click slides to seek to that timestamp in the video
4. Toggle between OCR text and transcript

#### Chat mode

After generating a summary:
1. Type a question in the chat box
2. Ask follow-up questions about the content
3. Get detailed explanations of specific points

**Example questions:**
- "Can you explain the second point in more detail?"
- "What are the main arguments against this?"
- "Summarize this in 3 bullet points"

### 6.4 Extension Settings

Click the settings icon in the extension to configure:
- **Default model**: Which AI model to use
- **Summary length**: Short, medium, long, etc.
- **Auto-summarize**: Automatically summarize new pages
- **Theme**: Choose your preferred color scheme

---

## Part 7: Configuration

Summarize uses a configuration file to store your preferences and API keys.

### 7.1 Config File Location

The configuration file is located at:
- **macOS/Linux**: `~/.summarize/config.json`
- **Windows**: `C:\Users\YourName\.summarize\config.json`

### 7.2 Common Configurations

Here are practical examples of useful configurations:

#### Set a default model

```json
{
  "model": "openai/gpt-5-mini"
}
```

Now you don't need to specify `--model` every time.

#### Store all your API keys

```json
{
  "env": {
    "OPENAI_API_KEY": "sk-...",
    "ANTHROPIC_API_KEY": "sk-ant-...",
    "GEMINI_API_KEY": "...",
    "OPENROUTER_API_KEY": "sk-or-..."
  }
}
```

#### Set default summary length

```json
{
  "model": "openai/gpt-5-mini",
  "length": "long"
}
```

#### Choose a theme

```json
{
  "ui": {
    "theme": "ember"
  }
}
```

Available themes: `aurora`, `ember`, `moss`, `mono`

#### Configure media caching

```json
{
  "cache": {
    "media": {
      "enabled": true,
      "ttlDays": 7,
      "maxMb": 2048
    }
  }
}
```

**What this does:**
- Caches downloaded media files for 7 days
- Limits cache size to 2GB
- Speeds up repeated summaries of the same media

#### Set default language for summaries

```json
{
  "output": {
    "language": "spanish"
  }
}
```

#### Complete example configuration

```json
{
  "model": "openai/gpt-5-mini",
  "env": {
    "OPENAI_API_KEY": "sk-...",
    "GEMINI_API_KEY": "..."
  },
  "output": {
    "language": "auto"
  },
  "ui": {
    "theme": "aurora"
  },
  "cache": {
    "media": {
      "enabled": true,
      "ttlDays": 7,
      "maxMb": 2048
    }
  }
}
```

### 7.3 Editing the Config File

**Method 1: Using a text editor**

1. Open the file in any text editor:
   ```bash
   # macOS
   open ~/.summarize/config.json

   # Linux
   nano ~/.summarize/config.json

   # Windows
   notepad C:\Users\YourName\.summarize\config.json
   ```

2. Make your changes
3. Save the file

**Method 2: Create/edit from command line**

```bash
# Create the directory if it doesn't exist
mkdir -p ~/.summarize

# Edit with your preferred editor
nano ~/.summarize/config.json
```

**Important:** Make sure your JSON is valid! Use a JSON validator if unsure.

---

## Part 8: Troubleshooting

### Problem: "No API key found"

**Solution:**
1. Make sure you've set up an API key (see Part 3)
2. Check that your config file has the right format:
   ```bash
   cat ~/.summarize/config.json
   ```
3. Verify the key name matches your model:
   - OpenAI models need `OPENAI_API_KEY`
   - Gemini models need `GEMINI_API_KEY`
   - etc.

### Problem: "Command not found: summarize"

**Solution:**
1. Make sure you installed globally with `-g`:
   ```bash
   npm install -g @steipete/summarize
   ```
2. Check if npm global bin is in your PATH:
   ```bash
   npm bin -g
   ```
3. Try closing and reopening your terminal

### Problem: Daemon not running (Extension)

**Error:** "Failed to fetch" or "Daemon unreachable"

**Solution:**
1. Check daemon status:
   ```bash
   summarize daemon status
   ```
2. If not running, start it:
   ```bash
   summarize daemon start
   ```
3. Check the logs for errors:
   ```bash
   # macOS/Linux
   cat ~/.summarize/logs/daemon.err.log

   # Windows
   type %USERPROFILE%\.summarize\logs\daemon.err.log
   ```
4. Reinstall if needed:
   ```bash
   summarize daemon install --token YOUR-TOKEN
   ```

### Problem: Extension not working on some pages

**Error:** "Receiving end does not exist"

**Solution:**
1. Open Chrome extension settings
2. Find Summarize
3. Set "Site access" to "On all sites"
4. Reload the tab you want to summarize

### Problem: "Model not available" or rate limits

**Solution:**
1. If using free models, they may be temporarily unavailable
2. Refresh your free models:
   ```bash
   summarize refresh-free
   ```
3. Try a different model:
   ```bash
   summarize "URL" --model auto
   ```
4. Check if you have API credits/quota remaining with your provider

### Problem: Slow summarization

**Possible causes and solutions:**

1. **Large content**: Use `--length short` for faster results
2. **Slow model**: Try a faster model like `openai/gpt-5-mini` or `google/gemini-3-flash-preview`
3. **Transcription**: Audio/video transcription takes time - this is normal
4. **Network**: Check your internet connection

### Problem: Bad or incomplete summaries

**Solution:**
1. Try a different model:
   ```bash
   summarize "URL" --model anthropic/claude-sonnet-4-5
   ```
2. Request a longer summary:
   ```bash
   summarize "URL" --length long
   ```
3. Check if the content was extracted properly:
   ```bash
   summarize "URL" --extract
   ```

### Problem: YouTube video transcription fails

**Solution:**
1. Check if the video has captions (YouTube should work automatically)
2. For videos without captions, you need transcription tools:
   - Install `yt-dlp`: `brew install yt-dlp` (macOS) or see [yt-dlp.org](https://github.com/yt-dlp/yt-dlp)
   - Set up Whisper transcription (see Part 3)
3. Try setting an Apify token if available (more reliable):
   ```json
   {
     "env": {
       "APIFY_API_TOKEN": "your-token"
     }
   }
   ```

### Problem: PDF summarization fails

**Solution:**
1. Use Google's Gemini model (best PDF support):
   ```bash
   summarize "file.pdf" --model google/gemini-3-flash-preview
   ```
2. Check if the PDF is valid:
   ```bash
   summarize "file.pdf" --extract
   ```
3. Some encrypted or scanned PDFs may not work well

### Problem: Browser extension extracts only a few words from a PDF

The browser extension cannot read PDFs rendered in Chrome/Firefox's built-in PDF viewer. The PDF viewer is a separate embedded component — the extension can only access the HTML DOM, not the PDF content inside it.

**Solution:**
1. Navigate to the HTML version of the page (e.g., `arxiv.org/abs/...` instead of `arxiv.org/pdf/...`)
2. Or use the CLI instead:
   ```bash
   summarize "https://arxiv.org/pdf/2507.11538" --model google/gemini-3-flash-preview
   ```

### Getting Help

If you're still having issues:

1. **Check the logs**:
   ```bash
   # CLI verbose output
   summarize "URL" --verbose

   # Daemon logs
   cat ~/.summarize/logs/daemon.err.log
   ```

2. **Search existing issues**: [GitHub Issues](https://github.com/steipete/summarize/issues)

3. **Ask for help**: Create a new issue with:
   - What you're trying to do
   - The exact command you ran
   - The error message
   - Your OS and Node version

---

## Part 9: Command Reference

Here's a quick reference of the most useful commands and flags.

### Basic Command Structure

```bash
summarize <input> [flags]
```

### Input Types

| Input | Example |
|-------|---------|
| Web URL | `"https://example.com"` |
| YouTube | `"https://youtube.com/watch?v=..."` |
| Local file | `"/path/to/file.pdf"` |
| Stdin | `-` (pipe content in) |

### Essential Flags

| Flag | Description | Example |
|------|-------------|---------|
| `--model <id>` | Choose AI model | `--model openai/gpt-5-mini` |
| `--length <size>` | Summary length | `--length short` |
| `--lang <language>` | Output language | `--lang spanish` |
| `--extract` | Extract content only (no summary) | `--extract` |
| `--format <type>` | Output format (text or md) | `--format md` |
| `--help` | Show all available options | `--help` |

### Length Options

| Option | Approx. Characters | Best For |
|--------|-------------------|----------|
| `short` | ~900 | Quick overview |
| `medium` | ~1,800 | Standard articles |
| `long` | ~4,200 | Detailed analysis |
| `xl` | ~9,000 | Research papers |
| `xxl` | ~17,000 | Books, long documents |
| `3000` | Exactly 3000 | Custom length |

### Model Examples

| Provider | Model ID | Best For |
|----------|----------|----------|
| OpenAI | `openai/gpt-5-mini` | Fast, cheap, general use |
| OpenAI | `openai/gpt-5` | Highest quality |
| Google | `google/gemini-3-flash-preview` | PDFs, speed |
| Anthropic | `anthropic/claude-sonnet-4-5` | Nuanced analysis |
| xAI | `xai/grok-4-fast-non-reasoning` | Very fast |
| OpenRouter | `free` | Free models |
| Auto | `auto` (default) | Picks first available by priority: Google → OpenAI → Anthropic → xAI |

### Language Options

| Format | Examples |
|--------|----------|
| Full name | `english`, `spanish`, `japanese`, `german` |
| Short code | `en`, `es`, `ja`, `de` |
| Auto-detect | `auto` (default) |

### YouTube & Media Flags

| Flag | Description |
|------|-------------|
| `--youtube auto` | Enable YouTube transcript extraction |
| `--slides` | Extract video slides |
| `--slides-ocr` | Add OCR text recognition |
| `--slides-max <n>` | Maximum slides to extract |

### Advanced Flags

| Flag | Description |
|------|-------------|
| `--verbose` | Show detailed debug information |
| `--json` | Output as JSON (for scripts) |
| `--plain` | Plain text output (no formatting) |
| `--no-cache` | Don't use cached summaries |
| `--timeout <time>` | Set timeout (e.g., `30s`, `2m`) |
| `--cli <provider>` | Use CLI provider (claude, gemini, codex) |

### Daemon Commands

| Command | Description |
|---------|-------------|
| `summarize daemon install --token <token>` | Install daemon |
| `summarize daemon start` | Start daemon |
| `summarize daemon stop` | Stop daemon |
| `summarize daemon status` | Check daemon status |
| `summarize daemon uninstall` | Remove daemon |

### Utility Commands

| Command | Description |
|---------|-------------|
| `summarize --version` | Show version |
| `summarize --help` | Show help |
| `summarize refresh-free` | Update free models |

### Complete Examples

```bash
# Standard article summary
summarize "https://blog.com/article"

# YouTube video with slides
summarize "https://youtube.com/watch?v=..." --slides --slides-ocr

# PDF with Google (best PDF support)
summarize "paper.pdf" --model google/gemini-3-flash-preview

# Long form, in Spanish
summarize "https://example.com" --length long --lang es

# Extract clean text without summarizing
summarize "https://cluttered-blog.com/recipe" --extract --format md

# Use free models
summarize "https://example.com" --model free

# Pipe clipboard content
pbpaste | summarize -

# Transcribe and summarize audio
summarize "meeting.mp3" --length long

# Use local Claude CLI (no API key needed)
summarize "https://example.com" --cli claude
```

---

## Conclusion

You now have everything you need to start using Summarize! Here's a quick recap:

1. **Install** Summarize via npm or Homebrew
2. **Set up an API key** (free options available via OpenRouter)
3. **Run** `summarize "URL"` to create your first summary
4. **Explore** different models, lengths, and features
5. **Install the extension** for one-click summaries in your browser

### Next Steps

- Experiment with different models to find your favorite
- Try the browser extension for seamless browsing
- Set up your config file for your preferred defaults
- Explore advanced features like slides, multi-language, and media transcription

### Quick Reference Card

```bash
# Most common commands
summarize "https://example.com"                    # Basic summary
summarize "URL" --length short                     # Quick summary
summarize "URL" --model free                       # Use free models
summarize "video.mp4"                              # Transcribe & summarize
summarize "URL" --extract                          # Just extract text
summarize "URL" --lang spanish                     # Summary in Spanish

# Setup
summarize refresh-free --set-default               # Set up free models
summarize daemon install --token TOKEN             # Install extension daemon
```

Happy summarizing!
