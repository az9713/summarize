# Developer Guide for Summarize

**Target Audience**: This guide is written for developers with C/C++/Java experience who are **new to TypeScript, Node.js, pnpm, and web development**. We assume nothing about your knowledge of the JavaScript/TypeScript ecosystem and explain every step explicitly.

---

## Table of Contents

1. [Prerequisites & Environment Setup](#part-1-prerequisites--environment-setup)
2. [Understanding the Codebase](#part-2-understanding-the-codebase)
3. [Daily Development Workflow](#part-3-daily-development-workflow)
4. [Key Concepts for New Web Developers](#part-4-key-concepts-for-new-web-developers)
5. [Making Changes](#part-5-making-changes)
6. [Debugging](#part-6-debugging)
7. [Git Workflow & Contributing](#part-7-git-workflow--contributing)
8. [Glossary](#part-8-glossary)

---

## Part 1: Prerequisites & Environment Setup

### 1.1 Understanding the Technology Stack

Before diving in, let's understand the tools you'll be using. If you're coming from C/C++/Java, think of these analogies:

#### **Node.js** - The Runtime Environment
- **What it is**: A runtime that executes JavaScript/TypeScript code outside the browser
- **Analogy**: Like the JVM for Java, or the C++ runtime for compiled C++ programs
- **What it does**: Provides APIs for file I/O, networking, system calls, etc.
- **Version needed**: 22 or higher (this project requires modern Node.js features)

#### **TypeScript** - The Programming Language
- **What it is**: A typed superset of JavaScript (JavaScript + static types)
- **Analogy**: Like what C++ is to C, or what Kotlin is to Java
- **What it does**: Adds type checking, interfaces, enums, and other features to JavaScript
- **Key difference**: TypeScript compiles to JavaScript (not machine code)

#### **pnpm** - The Package Manager
- **What it is**: A fast, disk-efficient package manager for Node.js projects
- **Analogy**: Like Maven/Gradle for Java, pip for Python, or apt/yum for Linux
- **What it does**: Downloads dependencies, manages versions, links workspace packages
- **Why pnpm?**: Faster and more space-efficient than npm (uses hard links, not copies)

#### **Monorepo** - Multi-Package Repository
- **What it is**: Multiple related packages stored in a single Git repository
- **Analogy**: Like a Maven multi-module project
- **What it does**: Allows sharing code between packages while maintaining separate versioning
- **In this project**: The main CLI package and `@steipete/summarize-core` library

#### **npm/npx** - Node Package Tools
- **npm**: The default Node Package Manager (comes with Node.js)
- **npx**: Executes packages without installing them globally
- **Note**: We use pnpm instead of npm, but npx is still useful for one-off commands

#### **ESM vs CommonJS** - Module Systems
- **CommonJS (CJS)**: The original Node.js module system
  - Syntax: `const foo = require('./bar')` and `module.exports = foo`
- **ESM (ECMAScript Modules)**: The modern, standardized module system
  - Syntax: `import { foo } from './bar.js'` and `export { foo }`
- **This project uses ESM** (indicated by `"type": "module"` in package.json)
- **Important**: ESM imports require file extensions (.js), even when importing .ts files

---

### 1.2 Install Required Software

#### **macOS**

1. **Install Node.js using nvm (Node Version Manager)**
   ```bash
   # Install nvm
   curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

   # Restart your terminal or source your profile
   source ~/.bashrc  # or ~/.zshrc

   # Install Node.js 22
   nvm install 22
   nvm use 22
   nvm alias default 22

   # Verify installation
   node --version  # Should show v22.x.x
   ```

2. **Enable Corepack for pnpm**
   ```bash
   corepack enable

   # Verify pnpm is available
   pnpm --version  # Should show 10.25.0 or higher
   ```

3. **Install Git** (if not already installed)
   ```bash
   # Check if Git is installed
   git --version

   # If not, install via Homebrew
   brew install git
   ```

4. **Install VS Code** (recommended)
   ```bash
   # Via Homebrew
   brew install --cask visual-studio-code

   # Install TypeScript extension
   # Open VS Code, press Cmd+Shift+X, search for "TypeScript"
   # The built-in TypeScript support is usually sufficient
   ```

---

#### **Linux (Ubuntu/Debian)**

1. **Install Node.js using nvm**
   ```bash
   # Install nvm
   curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

   # Restart terminal or source profile
   source ~/.bashrc

   # Install Node.js 22
   nvm install 22
   nvm use 22
   nvm alias default 22

   # Verify
   node --version  # Should show v22.x.x
   ```

2. **Enable Corepack**
   ```bash
   corepack enable
   pnpm --version  # Should show 10.25.0+
   ```

3. **Install Git**
   ```bash
   sudo apt update
   sudo apt install git
   git --version
   ```

4. **Install VS Code**
   ```bash
   # Download from https://code.visualstudio.com/
   # Or via snap
   sudo snap install code --classic
   ```

---

#### **Windows**

1. **Install Node.js using nvm-windows**
   ```powershell
   # Download nvm-windows from:
   # https://github.com/coreybutler/nvm-windows/releases

   # After installation, open PowerShell as Administrator
   nvm install 22
   nvm use 22

   # Verify
   node --version  # Should show v22.x.x
   ```

2. **Enable Corepack**
   ```powershell
   corepack enable
   pnpm --version  # Should show 10.25.0+
   ```

3. **Install Git**
   ```powershell
   # Download from https://git-scm.com/download/win
   # Or via Chocolatey
   choco install git
   ```

4. **Install VS Code**
   ```powershell
   # Download from https://code.visualstudio.com/
   # Or via Chocolatey
   choco install vscode
   ```

---

### 1.3 Clone and Install

```bash
# Clone the repository (replace with actual URL)
git clone https://github.com/steipete/summarize.git
cd summarize

# Install all dependencies
pnpm install
```

**What does `pnpm install` do?**
1. Reads `pnpm-lock.yaml` (the lockfile that pins exact dependency versions)
2. Downloads all dependencies listed in `package.json` into `node_modules/`
3. Links workspace packages (e.g., `@steipete/summarize-core`) so they're accessible
4. Creates symlinks between packages for local development

**Expected output**: You'll see a progress bar downloading packages. It may take 1-3 minutes.

---

### 1.4 Verify the Setup

Run these commands to ensure everything is working:

```bash
# Build the entire project
pnpm build
# Expected: TypeScript compilation, esbuild bundling, no errors
# Output: dist/ directories with compiled JavaScript

# Run tests
pnpm test
# Expected: All tests pass (green checkmarks)

# Run the full quality gate (formatting + linting + tests with coverage)
pnpm check
# Expected: All checks pass
# This is what CI runs, so if this passes locally, CI should pass too
```

**If any command fails**: Check the error message carefully. Common issues:
- Node version too old (must be 22+)
- Missing dependencies (re-run `pnpm install`)
- Permission issues (use sudo only if necessary, avoid it if possible)

---

## Part 2: Understanding the Codebase

### 2.1 Directory Map

Here's the complete project structure:

```
summarize/
├── .github/              # GitHub Actions CI/CD workflows
│   └── workflows/
│       ├── ci.yml        # Main CI pipeline (lint, test, build)
│       └── pages.yml     # GitHub Pages deployment
├── apps/                 # Applications built on top of the core library
│   └── chrome-extension/ # Browser extension (Chrome/Firefox)
│       ├── src/          # Extension source code (Preact components)
│       ├── wxt.config.ts # WXT build configuration
│       └── package.json  # Extension dependencies
├── docs/                 # Documentation
│   └── DEVELOPER_GUIDE.md # This file
├── packages/             # Monorepo packages
│   └── core/             # @steipete/summarize-core library
│       ├── src/          # Core source code
│       │   ├── content/  # Content extraction logic
│       │   ├── prompts/  # LLM prompt templates
│       │   └── index.ts  # Main library exports
│       ├── package.json  # Core package metadata
│       └── tsconfig.build.json
├── scripts/              # Build and utility scripts
│   ├── build-cli.mjs     # CLI bundler (esbuild)
│   └── release.sh        # Release automation
├── src/                  # Main CLI source code
│   ├── cli.ts            # CLI entry point
│   ├── cli-main.ts       # Main CLI logic
│   ├── config.ts         # Configuration management
│   ├── flags.ts          # CLI flag parsers
│   ├── llm/              # LLM integration
│   │   ├── providers/    # Provider-specific implementations
│   │   │   ├── anthropic.ts  # Claude API
│   │   │   ├── openai.ts     # OpenAI/compatible APIs
│   │   │   └── google.ts     # Gemini API
│   │   ├── generate-text.ts  # Main LLM interface
│   │   └── model-id.ts       # Model ID parsing
│   ├── daemon/           # Background daemon for browser extension
│   ├── content/          # Content extraction
│   ├── prompts/          # Prompt generation
│   ├── run/              # Command runners
│   └── tty/              # Terminal UI (progress bars, colors)
├── tests/                # Test files
│   ├── setup.ts          # Test configuration (disables Whisper)
│   └── *.test.ts         # Test suites
├── .oxlintrc.json        # Lint configuration (oxlint)
├── .oxfmtrc.jsonc        # Format configuration (oxfmt)
├── package.json          # Root package metadata
├── pnpm-workspace.yaml   # Workspace configuration
├── pnpm-lock.yaml        # Dependency lockfile (DO NOT EDIT)
├── tsconfig.base.json    # Base TypeScript config
├── tsconfig.build.json   # Build-specific TypeScript config
└── vitest.config.ts      # Test configuration
```

**Key directories you'll work in most often:**
- `src/` - Main CLI code
- `packages/core/src/` - Reusable library code
- `tests/` - Test files
- `docs/` - Documentation

---

### 2.2 How TypeScript Files Become Runnable

If you're used to compiled languages, this might be confusing. Here's the build pipeline:

```
TypeScript (.ts files)
    ↓
[tsc compiler]
    ↓
JavaScript (.js files in dist/)
    ↓
[esbuild bundler] (for CLI only)
    ↓
Single bundled dist/cli.js
    ↓
[package.json "bin" field]
    ↓
"summarize" command in terminal
```

**Step-by-step breakdown:**

1. **Development mode (no build needed)**:
   ```bash
   pnpm summarize "https://example.com"
   ```
   - Uses `tsx` (TypeScript executor) to run `src/cli.ts` directly
   - No compilation step, instant feedback
   - How? The `summarize` script in package.json is: `"tsx src/cli.ts"`

2. **Build mode** (for production/distribution):
   ```bash
   pnpm build
   ```
   - **Step 1**: `tsc -p tsconfig.build.json` compiles TypeScript to JavaScript
     - Input: `src/**/*.ts`
     - Output: `dist/esm/**/*.js` (ESM format)
     - Also generates: `dist/types/**/*.d.ts` (type declarations)
   - **Step 2**: `esbuild` bundles the CLI into a single file
     - Input: `dist/esm/cli.js` + all dependencies
     - Output: `dist/cli.js` (single executable file with a shebang)
   - **Step 3**: The `bin` field in package.json maps commands:
     ```json
     "bin": {
       "summarize": "./dist/cli.js",
       "summarizer": "./dist/cli.js"
     }
     ```
   - After `pnpm install -g`, you can run `summarize` from anywhere

**Important notes:**
- During development, always use `pnpm summarize` or `pnpm s` (no build needed)
- Only build when testing the production bundle or before releasing
- The core library (`packages/core`) must build first (other packages depend on it)

---

### 2.3 The Module System

TypeScript uses the ESM (ECMAScript Modules) system. Here's what you need to know:

#### **Basic ESM Imports**

```typescript
// Named imports
import { foo, bar } from "./utils.js";

// Default import
import config from "./config.js";

// Import everything
import * as helpers from "./helpers.js";

// Side-effect import (runs code but doesn't import anything)
import "./polyfill.js";
```

**CRITICAL: Always use `.js` extensions, even when importing `.ts` files!**
```typescript
// CORRECT
import { parse } from "./parser.js";  // Imports parser.ts

// WRONG (will fail at runtime)
import { parse } from "./parser";
import { parse } from "./parser.ts";
```

**Why?** Node.js ESM requires explicit file extensions. TypeScript strips types but preserves the `.js` extension in the output.

---

#### **Workspace Imports**

This project is a monorepo with multiple packages. Packages can import from each other:

```typescript
// Importing from the core package
import { extractContent } from "@steipete/summarize-core/content";
import { buildPrompt } from "@steipete/summarize-core/prompts";
```

**How does this work?**
1. `pnpm-workspace.yaml` defines workspace packages:
   ```yaml
   packages:
     - .
     - packages/*
     - apps/*
   ```
2. Dependencies in package.json reference workspace packages:
   ```json
   "dependencies": {
     "@steipete/summarize-core": "workspace:*"
   }
   ```
3. pnpm creates symlinks: `node_modules/@steipete/summarize-core` → `packages/core`

---

#### **Path Aliases in Tests**

Tests use special path aliases configured in `vitest.config.ts`:

```typescript
resolve: {
  alias: [
    {
      find: /^@steipete\/summarize-core$/,
      replacement: resolve(rootDir, "packages/core/src/index.ts"),
    },
    // ... more aliases
  ],
}
```

**Why?** Tests run against TypeScript source files directly (no build step), so they need to resolve imports to `.ts` files instead of compiled `.js` files.

---

### 2.4 Package.json Explained

Let's dissect the root `package.json`:

```json
{
  "name": "@steipete/summarize",
  // ^ Package name (scoped to @steipete org on npm)

  "version": "0.11.2",
  // ^ Semantic version (MAJOR.MINOR.PATCH)

  "type": "module",
  // ^ CRITICAL: Uses ESM, not CommonJS
  //   Without this, Node.js treats .js files as CommonJS

  "main": "./dist/esm/index.js",
  // ^ Entry point when someone does: require('@steipete/summarize')
  //   (Not used much since we're ESM)

  "module": "./dist/esm/index.js",
  // ^ Entry point for ESM-aware bundlers

  "types": "./dist/types/index.d.ts",
  // ^ TypeScript type definitions entry point

  "exports": {
    // ^ Modern package entry points (preferred over "main")
    ".": {
      "types": "./dist/types/index.d.ts",
      "import": "./dist/esm/index.js"
    },
    "./content": {
      // ^ Allows: import { x } from '@steipete/summarize/content'
      "types": "./dist/types/content/index.d.ts",
      "import": "./dist/esm/content/index.js"
    }
  },

  "bin": {
    // ^ Executable commands installed with the package
    "summarize": "./dist/cli.js",
    "summarizer": "./dist/cli.js"
  },

  "scripts": {
    // ^ Custom commands (run with `pnpm <script-name>`)
    "build": "pnpm clean && pnpm -C packages/core build && pnpm build:lib && pnpm build:cli",
    "test": "vitest run",
    "summarize": "tsx src/cli.ts"
  },

  "dependencies": {
    // ^ Runtime dependencies (needed to run the app)
    "commander": "^14.0.3",
    "@steipete/summarize-core": "workspace:*"
  },

  "devDependencies": {
    // ^ Development-only dependencies (needed for building/testing)
    "typescript": "^5.9.3",
    "vitest": "^4.0.18"
  },

  "engines": {
    // ^ Required Node.js version
    "node": ">=22"
  },

  "packageManager": "pnpm@10.25.0+sha512.5e82639..."
  // ^ Enforces pnpm version (Corepack uses this)
}
```

**Dependency version prefixes:**
- `^1.2.3` - Compatible with 1.x.x (allows 1.2.4, 1.3.0, but not 2.0.0)
- `~1.2.3` - Compatible with 1.2.x (allows 1.2.4, but not 1.3.0)
- `1.2.3` - Exact version only
- `workspace:*` - Use the local workspace version

---

## Part 3: Daily Development Workflow

### 3.1 Running the CLI During Development

**Most common workflow: Use tsx to run TypeScript directly (no build needed)**

```bash
# Full syntax
pnpm summarize "https://example.com"

# Shorthand (defined in package.json scripts)
pnpm s "https://example.com"

# With flags
pnpm summarize "https://example.com" --model anthropic/claude-sonnet-4-0 --verbose

# Multiple inputs
pnpm summarize "url1" "url2" "path/to/file.pdf"

# Help
pnpm summarize --help
```

**What happens behind the scenes:**
1. pnpm runs the `summarize` script from package.json
2. That script is: `tsx src/cli.ts`
3. `tsx` executes TypeScript directly (no compilation)
4. Your code runs with any arguments you pass

**Advantages of tsx development:**
- Instant feedback (no build step)
- See changes immediately
- Full TypeScript error checking

---

### 3.2 Building

**When to build:**
- Before releasing a new version
- When testing the production bundle
- When CI runs (automated)

**Build commands:**

```bash
# Full build (recommended)
pnpm build
# What it does:
# 1. Clean dist/ directories
# 2. Build packages/core (tsc)
# 3. Build main library (tsc)
# 4. Bundle CLI (esbuild)

# Build only TypeScript (no CLI bundle)
pnpm build:lib

# Build only CLI bundle (assumes TypeScript is built)
pnpm build:cli

# Clean build artifacts
pnpm clean
```

**Build order matters!**
```
packages/core → src/ (TypeScript) → dist/cli.js (esbuild bundle)
```

The core package must build first because the main package depends on it.

---

### 3.3 Testing

**Test framework: Vitest** (similar to Jest, but faster and ESM-native)

#### **Run all tests**
```bash
pnpm test
# Runs all tests once and exits
```

#### **Run specific test file**
```bash
pnpm vitest run tests/article.test.ts
```

#### **Run tests matching a pattern**
```bash
# Run all tests with "extract" in their name
pnpm vitest run -t "extract"

# Run all tests in files matching a glob
pnpm vitest run tests/llm*.test.ts
```

#### **Watch mode (re-run tests on file changes)**
```bash
pnpm vitest --watch

# Or shorthand
pnpm vitest
```

**What happens in watch mode:**
1. Vitest runs all tests
2. Watches for file changes
3. Re-runs only affected tests when you save a file
4. Press `q` to quit, `a` to re-run all tests

#### **Coverage report**
```bash
pnpm test:coverage
```

**What it generates:**
- Terminal summary (% covered per file)
- `coverage/` directory with HTML report
- Open `coverage/index.html` in a browser for detailed visualization

**Coverage thresholds:**
- Branches: 75%
- Functions: 75%
- Lines: 75%
- Statements: 75%

If coverage drops below these thresholds, the build fails.

#### **Understanding setup.ts**

The file `tests/setup.ts` contains:
```typescript
process.env.SUMMARIZE_DISABLE_LOCAL_WHISPER_CPP = "1";
```

**What this does:** Disables the Whisper.cpp local binary during tests.

**Why?** Whisper.cpp is used for audio transcription but:
- Requires downloading a large binary (~100MB)
- Slows down test runs significantly
- Not needed for most tests (unit tests mock the transcription)

**When you need it:** Integration tests for audio transcription use a different mechanism.

---

### 3.4 Linting and Formatting

This project uses **oxlint** and **oxfmt** (Rust-based tools, much faster than ESLint/Prettier).

#### **Check for lint errors**
```bash
pnpm lint
```

**What it checks:**
- TypeScript type-aware rules
- Only 2 rules enabled (see `.oxlintrc.json`):
  - `typescript/no-floating-promises` - Catches missing `await` on Promises
  - `typescript/no-misused-promises` - Catches Promises used in wrong contexts

**Why so few rules?** Philosophy: Use TypeScript's type system for correctness, keep lint rules minimal and focused on real bugs.

#### **Auto-fix lint errors + format code**
```bash
pnpm lint:fix
```

**What it does:**
1. Runs oxlint with `--fix` flag (auto-fixes lint errors)
2. Runs oxfmt to format code

#### **Format only**
```bash
pnpm format
```

**What it does:** Runs oxfmt (formats code according to `.oxfmtrc.jsonc`)

#### **Check formatting (CI mode)**
```bash
pnpm format:check
```

**What it does:** Checks if code is formatted, exits with error if not (doesn't modify files)

**Use this before committing** to ensure CI won't fail on formatting issues.

---

### 3.5 Type Checking

TypeScript can check types without compiling the code:

```bash
pnpm typecheck
```

**What it does:** Runs `tsc --noEmit` (type-check only, no output files)

**When to use:**
- Quick type check without full build
- Validate types before committing
- Debugging type errors

**Common type errors:**
```typescript
// Error: Type 'string | undefined' is not assignable to type 'string'
const name: string = config.name;  // config.name might be undefined

// Fix: Handle undefined case
const name: string = config.name ?? "default";

// Error: Argument of type 'number' is not assignable to parameter of type 'string'
parseUrl(123);

// Fix: Convert to string
parseUrl(String(123));
```

---

### 3.6 The Quality Gate

Before every commit, run:

```bash
pnpm check
```

**What it does:**
```bash
pnpm format:check  # Check code formatting
&&
pnpm lint          # Check lint rules
&&
pnpm test:coverage # Run tests with coverage
```

**All three must pass** for the quality gate to pass.

**This is what CI runs**, so if `pnpm check` passes locally, CI should pass too.

**Typical workflow:**
1. Make changes
2. Run `pnpm check`
3. If formatting fails: `pnpm lint:fix` → re-run `pnpm check`
4. If lint fails: Fix the issue → re-run `pnpm check`
5. If tests fail: Fix the tests → re-run `pnpm check`
6. Once `pnpm check` passes: Commit and push

---

## Part 4: Key Concepts for New Web Developers

### 4.1 Async/Await and Promises

**This is the #1 concept C/C++/Java developers struggle with.**

#### **What is a Promise?**

A `Promise` is a placeholder for a value that will be available in the future.

**Analogy:**
- Java: `Future<T>` or `CompletableFuture<T>`
- C++: `std::future<T>`
- Go: Channels

**Promise states:**
1. **Pending**: Operation hasn't finished yet
2. **Fulfilled**: Operation succeeded, value is available
3. **Rejected**: Operation failed, error is available

#### **Example: Synchronous vs Asynchronous**

**Synchronous (blocking, like C/Java):**
```typescript
// BAD: This doesn't exist in Node.js (no blocking file read)
const data = fs.readFileSync("file.txt");  // Blocks until file is read
console.log(data);
```

**Asynchronous (non-blocking, the Node.js way):**
```typescript
// Returns a Promise immediately
const dataPromise = fs.promises.readFile("file.txt");

// dataPromise is NOT the data, it's a Promise that will eventually contain the data
console.log(dataPromise);  // Promise { <pending> }

// To get the data, use await
const data = await fs.promises.readFile("file.txt");
console.log(data);  // Actual file contents
```

#### **The async/await Pattern**

```typescript
// Marking a function as "async" means it returns a Promise
async function fetchUser(id: number): Promise<User> {
  // await pauses execution until the Promise resolves
  const response = await fetch(`https://api.example.com/users/${id}`);
  const user = await response.json();
  return user;
}

// Calling an async function returns a Promise
const userPromise = fetchUser(123);  // Promise { <pending> }

// You must await it to get the value
const user = await fetchUser(123);  // User object
```

**Rule: You can only use `await` inside an `async` function.**

```typescript
// CORRECT
async function main() {
  const user = await fetchUser(123);
  console.log(user);
}

// WRONG: await outside async function
function main() {
  const user = await fetchUser(123);  // SyntaxError!
}
```

#### **Why the Lint Rules Matter**

**Rule: `typescript/no-floating-promises`**

```typescript
// BAD: Promise is created but never awaited (fire-and-forget)
async function saveUser(user: User) {
  database.save(user);  // Returns a Promise, but we don't await it!
  console.log("User saved");  // This runs BEFORE the save finishes!
}

// GOOD: Await the Promise
async function saveUser(user: User) {
  await database.save(user);  // Wait for save to complete
  console.log("User saved");  // Now this runs AFTER the save
}
```

**Common mistake:**
```typescript
// BAD: forEach doesn't wait for async functions
users.forEach(async (user) => {
  await saveUser(user);  // This doesn't work as expected!
});
console.log("All users saved");  // Runs immediately, users aren't saved yet!

// GOOD: Use Promise.all with map
await Promise.all(users.map(user => saveUser(user)));
console.log("All users saved");  // Now this is accurate
```

#### **Error Handling with try/catch**

```typescript
async function fetchUser(id: number): Promise<User> {
  try {
    const response = await fetch(`https://api.example.com/users/${id}`);

    if (!response.ok) {
      throw new Error(`HTTP error: ${response.status}`);
    }

    const user = await response.json();
    return user;
  } catch (error) {
    console.error("Failed to fetch user:", error);
    throw error;  // Re-throw or handle gracefully
  }
}
```

**Key differences from Java:**
- JavaScript has one `catch` block (no multiple catch clauses like Java)
- `error` is type `unknown` (not `Error`), you must type-check it
- No checked exceptions (TypeScript doesn't enforce try/catch)

---

### 4.2 Fetch API

The Fetch API is Node.js/browser's standard HTTP client.

**Analogy:**
- Java: `HttpClient`, `HttpURLConnection`
- Python: `requests.get()`
- curl: The curl command-line tool

#### **Basic GET request**

```typescript
const response = await fetch("https://api.example.com/data");

// response is a Response object
console.log(response.status);      // 200
console.log(response.statusText);  // "OK"
console.log(response.ok);          // true (status 200-299)

// Parse JSON body
const data = await response.json();

// Or get text
const text = await response.text();
```

#### **POST request with JSON**

```typescript
const response = await fetch("https://api.example.com/users", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": "Bearer token123",
  },
  body: JSON.stringify({
    name: "John",
    email: "john@example.com",
  }),
});

const user = await response.json();
```

#### **Error handling**

**CRITICAL: fetch() does NOT throw on HTTP errors (4xx, 5xx)!**

```typescript
// BAD: This won't catch HTTP 404/500 errors
try {
  const response = await fetch("https://api.example.com/data");
  const data = await response.json();  // Will fail on 404!
} catch (error) {
  console.error("Request failed:", error);  // Only catches network errors
}

// GOOD: Check response.ok
const response = await fetch("https://api.example.com/data");

if (!response.ok) {
  throw new Error(`HTTP ${response.status}: ${response.statusText}`);
}

const data = await response.json();
```

#### **Streaming responses**

```typescript
const response = await fetch("https://api.example.com/stream");

// Get readable stream
const reader = response.body?.getReader();
if (!reader) throw new Error("No body");

while (true) {
  const { done, value } = await reader.read();
  if (done) break;

  // value is Uint8Array (raw bytes)
  console.log("Chunk:", new TextDecoder().decode(value));
}
```

**Used in this project for:** LLM streaming responses (Server-Sent Events)

---

### 4.3 Server-Sent Events (SSE)

SSE is a one-way communication protocol: server → client over HTTP.

**Analogy:**
- Like a long-lived HTTP response that sends multiple messages
- Simpler than WebSockets (which is bidirectional)
- Used by OpenAI/Anthropic for streaming LLM responses

#### **SSE Protocol**

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream

data: {"chunk": "Hello"}

data: {"chunk": " world"}

data: [DONE]
```

Each message is prefixed with `data:` and separated by blank lines.

#### **How this project uses SSE**

**Daemon → Browser Extension:**
1. User clicks extension icon
2. Extension sends request to daemon (local HTTP server on port 39278)
3. Daemon streams summary chunks back via SSE
4. Extension updates UI with each chunk

**Example from `src/daemon/server.ts`:**
```typescript
// Set up SSE response headers
res.writeHead(200, {
  "Content-Type": "text/event-stream",
  "Cache-Control": "no-cache",
  "Connection": "keep-alive",
});

// Send chunks
res.write(`data: ${JSON.stringify({ text: "Hello" })}\n\n`);
res.write(`data: ${JSON.stringify({ text: " world" })}\n\n`);
res.write(`data: [DONE]\n\n`);
res.end();
```

---

### 4.4 The Event Loop

Node.js is **single-threaded** but **non-blocking**.

**Analogy:**
- Java: One thread per request (blocking I/O)
- Node.js: One thread handles all requests (non-blocking I/O)

#### **How it works:**

```
┌───────────────────────────┐
│         Event Loop         │
│                           │
│  1. Check for events      │
│  2. Execute callbacks     │
│  3. Repeat                │
└───────────────────────────┘
```

**Example:**

```typescript
console.log("1. Start");

setTimeout(() => {
  console.log("3. Timeout callback");
}, 0);

console.log("2. End");

// Output:
// 1. Start
// 2. End
// 3. Timeout callback
```

**What happened:**
1. `console.log("1. Start")` runs immediately
2. `setTimeout()` schedules a callback (doesn't run yet)
3. `console.log("2. End")` runs immediately
4. Event loop picks up timeout callback and runs it

**Key insight:** Even `setTimeout(..., 0)` doesn't run immediately—it runs on the next event loop iteration.

#### **Why async/await is necessary**

```typescript
// This doesn't work (file isn't read yet when we log)
fs.promises.readFile("file.txt");
console.log("File contents:", ???);  // File isn't ready yet!

// This works (await pauses until file is read)
const contents = await fs.promises.readFile("file.txt");
console.log("File contents:", contents);
```

**Practical impact:**
- Don't block the event loop with CPU-heavy operations (use worker threads instead)
- I/O operations are fast because they don't block
- You can handle thousands of concurrent requests with one thread

---

## Part 5: Making Changes

### 5.1 Adding a New LLM Provider

Let's say you want to add support for a new LLM provider called "NewProvider".

**Step 1: Create provider file**

```bash
# Create new provider implementation
touch src/llm/providers/newprovider.ts
```

**Step 2: Implement the provider**

```typescript
// src/llm/providers/newprovider.ts
import type { GenerateTextParams, GenerateTextResult } from "./types.js";

export async function generateTextNewProvider(
  params: GenerateTextParams
): Promise<GenerateTextResult> {
  const { model, prompt, apiKey, maxTokens, temperature } = params;

  // Get API key from environment or params
  const key = apiKey ?? process.env.NEWPROVIDER_API_KEY;
  if (!key) {
    throw new Error("Missing NEWPROVIDER_API_KEY");
  }

  // Make API request
  const response = await fetch("https://api.newprovider.com/v1/generate", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${key}`,
    },
    body: JSON.stringify({
      model,
      prompt,
      max_tokens: maxTokens,
      temperature,
    }),
  });

  if (!response.ok) {
    throw new Error(`NewProvider API error: ${response.status}`);
  }

  const data = await response.json();

  return {
    text: data.output,
    usage: {
      promptTokens: data.usage.prompt_tokens,
      completionTokens: data.usage.completion_tokens,
      totalTokens: data.usage.total_tokens,
    },
  };
}
```

**Step 3: Add provider to type definitions**

```typescript
// src/llm/model-id.ts
export type LlmProvider =
  | "xai"
  | "openai"
  | "google"
  | "anthropic"
  | "zai"
  | "nvidia"
  | "newprovider";  // Add this

const PROVIDERS: LlmProvider[] = [
  "xai",
  "openai",
  "google",
  "anthropic",
  "zai",
  "nvidia",
  "newprovider",  // Add this
];
```

**Step 4: Add routing in generate-text.ts**

```typescript
// src/llm/generate-text.ts
import { generateTextNewProvider } from "./providers/newprovider.js";

export async function generateText(params: GenerateTextParams): Promise<GenerateTextResult> {
  const { provider } = parseGatewayStyleModelId(params.model);

  switch (provider) {
    case "xai":
      return generateTextXai(params);
    case "openai":
      return generateTextOpenAI(params);
    case "google":
      return generateTextGoogle(params);
    case "anthropic":
      return generateTextAnthropic(params);
    case "newprovider":
      return generateTextNewProvider(params);  // Add this
    default:
      throw new Error(`Unsupported provider: ${provider}`);
  }
}
```

**Step 5: Add environment variable handling**

```typescript
// src/config.ts (add to env key validation)
export function validateEnvKeys(): void {
  const keys = [
    "OPENAI_API_KEY",
    "ANTHROPIC_API_KEY",
    "GOOGLE_API_KEY",
    "NEWPROVIDER_API_KEY",  // Add this
  ];

  // ... validation logic
}
```

**Step 6: Write tests**

```typescript
// tests/llm-newprovider.test.ts
import { describe, it, expect, vi } from "vitest";
import { generateTextNewProvider } from "../src/llm/providers/newprovider.js";

describe("NewProvider LLM", () => {
  it("should generate text", async () => {
    // Mock fetch
    global.fetch = vi.fn().mockResolvedValue({
      ok: true,
      json: async () => ({
        output: "Generated text",
        usage: {
          prompt_tokens: 10,
          completion_tokens: 20,
          total_tokens: 30,
        },
      }),
    });

    const result = await generateTextNewProvider({
      model: "newprovider-model-1",
      prompt: "Test prompt",
      apiKey: "test-key",
      maxTokens: 100,
      temperature: 0.7,
    });

    expect(result.text).toBe("Generated text");
    expect(result.usage.totalTokens).toBe(30);
  });

  it("should throw on missing API key", async () => {
    delete process.env.NEWPROVIDER_API_KEY;

    await expect(
      generateTextNewProvider({
        model: "newprovider-model-1",
        prompt: "Test",
      })
    ).rejects.toThrow("Missing NEWPROVIDER_API_KEY");
  });
});
```

**Step 7: Update documentation**

Add NewProvider to the README.md list of supported providers.

**Step 8: Test end-to-end**

```bash
# Set API key
export NEWPROVIDER_API_KEY="your-key-here"

# Run CLI
pnpm summarize "https://example.com" --model newprovider/model-name

# Verify tests pass
pnpm test
```

---

### 5.2 Adding a New CLI Flag

Let's add a `--max-retries` flag to control LLM retry behavior.

**Step 1: Add parser in flags.ts**

```typescript
// src/flags.ts

// Add constants
const MIN_RETRIES = 0;
const MAX_RETRIES = 10;

// Add parser function
export function parseRetriesArg(raw: string): number {
  const normalized = raw.trim();

  if (!normalized) {
    throw new Error(`Unsupported --max-retries: ${raw}`);
  }

  const numeric = Number(normalized);

  if (!Number.isFinite(numeric) || !Number.isInteger(numeric)) {
    throw new Error(`Unsupported --max-retries: ${raw}`);
  }

  if (numeric < MIN_RETRIES || numeric > MAX_RETRIES) {
    throw new Error(`Unsupported --max-retries: ${raw} (range ${MIN_RETRIES}-${MAX_RETRIES})`);
  }

  return numeric;
}
```

**Step 2: Add to Commander program**

Find where the CLI program is built (usually in `src/run/help.ts` or similar):

```typescript
// src/run/help.ts
program
  .option("--model <id>", "LLM model to use")
  .option("--max-retries <n>", "Maximum retry attempts (0-10)", "3")  // Add this
  .option("--verbose", "Enable verbose logging");
```

**Step 3: Parse the flag**

```typescript
// src/cli-main.ts (or wherever flags are parsed)
import { parseRetriesArg } from "./flags.js";

const options = program.opts();

const maxRetries = options.maxRetries
  ? parseRetriesArg(options.maxRetries)
  : 3;  // Default value
```

**Step 4: Thread through to runner**

```typescript
// Pass to the function that needs it
await summarizeUrl(url, {
  model: options.model,
  maxRetries,  // Add this
  verbose: options.verbose,
});
```

**Step 5: Use in implementation**

```typescript
// src/llm/generate-text.ts
export async function generateText(
  params: GenerateTextParams,
  maxRetries = 3
): Promise<GenerateTextResult> {
  let lastError: Error | null = null;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await actuallyGenerateText(params);
    } catch (error) {
      lastError = error as Error;
      console.warn(`Attempt ${attempt + 1} failed:`, error);
    }
  }

  throw lastError;
}
```

**Step 6: Write tests**

```typescript
// tests/flags.test.ts
import { describe, it, expect } from "vitest";
import { parseRetriesArg } from "../src/flags.js";

describe("parseRetriesArg", () => {
  it("should parse valid numbers", () => {
    expect(parseRetriesArg("0")).toBe(0);
    expect(parseRetriesArg("5")).toBe(5);
    expect(parseRetriesArg("10")).toBe(10);
  });

  it("should reject out-of-range values", () => {
    expect(() => parseRetriesArg("-1")).toThrow();
    expect(() => parseRetriesArg("11")).toThrow();
  });

  it("should reject non-numeric values", () => {
    expect(() => parseRetriesArg("abc")).toThrow();
    expect(() => parseRetriesArg("3.5")).toThrow();
  });
});
```

**Step 7: Update help text**

The Commander option description is the help text:
```bash
pnpm summarize --help
# Should show:
#   --max-retries <n>  Maximum retry attempts (0-10) (default: "3")
```

---

### 5.3 Modifying Content Extraction

The content extraction pipeline is in `packages/core/src/content/`.

**Architecture:**
```
URL/File
  ↓
[fetch/read]
  ↓
HTML/PDF/Audio
  ↓
[extract-*.ts]
  ↓
Clean text/markdown
```

**Key files:**
- `packages/core/src/content/index.ts` - Main entry point
- `packages/core/src/content/extract-html.ts` - HTML extraction (uses Readability)
- `packages/core/src/content/extract-pdf.ts` - PDF extraction
- `packages/core/src/content/extract-youtube.ts` - YouTube transcripts

**Example: Improve HTML extraction**

```typescript
// packages/core/src/content/extract-html.ts
import { Readability } from "@mozilla/readability";
import { JSDOM } from "jsdom";

export function extractFromHtml(html: string, url: string): string {
  const dom = new JSDOM(html, { url });
  const reader = new Readability(dom.window.document);
  const article = reader.parse();

  if (!article) {
    throw new Error("Failed to extract article content");
  }

  // NEW: Add custom cleaning logic
  let content = article.textContent;

  // Remove excessive whitespace
  content = content.replace(/\n{3,}/g, "\n\n");

  // Remove "Read more" links
  content = content.replace(/Read more\.{3}/g, "");

  return content;
}
```

**Test your changes:**

```bash
# Run content extraction tests
pnpm vitest run tests/content.test.ts

# Test end-to-end
pnpm summarize "https://example.com" --verbose
```

---

### 5.4 Working on the Browser Extension

The browser extension is in `apps/chrome-extension/`.

**Tech stack:**
- **WXT**: Extension framework (handles Chrome/Firefox differences)
- **Preact**: Lightweight React alternative
- **Zag.js**: Headless UI components
- **TypeScript**: Type-safe development

**Development workflow:**

```bash
# Build for Chrome
pnpm -C apps/chrome-extension build

# Build for Firefox
pnpm -C apps/chrome-extension build:firefox

# Watch mode (rebuild on changes)
pnpm -C apps/chrome-extension dev

# Firefox watch mode
pnpm -C apps/chrome-extension dev:firefox
```

**Load unpacked extension in Chrome:**

1. Build the extension:
   ```bash
   pnpm -C apps/chrome-extension build
   ```

2. Open Chrome and go to: `chrome://extensions/`

3. Enable "Developer mode" (top-right toggle)

4. Click "Load unpacked"

5. Select: `apps/chrome-extension/.output/chrome-mv3`

6. The extension icon appears in your toolbar

7. Make changes, rebuild, and click the reload icon on the extension card

**Load in Firefox:**

1. Build for Firefox:
   ```bash
   pnpm -C apps/chrome-extension build:firefox
   ```

2. Open Firefox and go to: `about:debugging#/runtime/this-firefox`

3. Click "Load Temporary Add-on"

4. Select any file in: `apps/chrome-extension/.output/firefox-mv2`

5. Extension loads (but only until you close Firefox)

**Run E2E tests:**

```bash
# Chrome E2E tests
pnpm -C apps/chrome-extension test:chrome

# Firefox E2E tests (requires special flag)
pnpm -C apps/chrome-extension test:firefox:force
```

**Extension architecture:**

```
apps/chrome-extension/
├── entrypoints/
│   ├── background.ts      # Service worker (MV3)
│   ├── content.ts         # Content script (injected into pages)
│   └── popup/
│       ├── index.html     # Popup HTML
│       └── App.tsx        # Popup React component
├── components/            # Reusable UI components
├── lib/                   # Utilities
└── wxt.config.ts          # WXT configuration
```

**How the extension works:**

1. User clicks extension icon
2. Popup opens (`popup/App.tsx`)
3. Popup sends message to background script
4. Background script sends request to daemon (http://localhost:39278)
5. Daemon streams summary chunks via SSE
6. Background script forwards chunks to popup
7. Popup updates UI with streaming text

---

## Part 6: Debugging

### 6.1 Using --verbose

Enable detailed logging:

```bash
pnpm summarize "https://example.com" --verbose
```

**What it shows:**
- HTTP requests and responses
- Content extraction steps
- Token counts
- LLM API calls
- Timing information

**Example output:**
```
[INFO] Fetching URL: https://example.com
[DEBUG] Response status: 200 OK
[DEBUG] Content-Type: text/html; charset=utf-8
[INFO] Extracted 15,234 characters
[DEBUG] Using model: anthropic/claude-sonnet-4-0
[DEBUG] Prompt tokens: 3,456
[DEBUG] Completion tokens: 234
[INFO] Summary generated in 2.3s
```

---

### 6.2 Using --json for Diagnostics

Output structured JSON instead of human-readable text:

```bash
pnpm summarize "https://example.com" --json
```

**Output:**
```json
{
  "url": "https://example.com",
  "title": "Example Article",
  "summary": "This is the summary...",
  "metadata": {
    "model": "anthropic/claude-sonnet-4-0",
    "tokensUsed": 3690,
    "duration": 2.3
  }
}
```

**Use cases:**
- Piping to jq for filtering
- Integration with other tools
- Automated testing

**Example with jq:**
```bash
pnpm summarize "https://example.com" --json | jq '.summary'
```

---

### 6.3 Daemon Logs

The daemon (background service for browser extension) writes logs to disk.

**Log location:**
```bash
# macOS/Linux
~/.summarize/logs/daemon.log

# Windows
%USERPROFILE%\.summarize\logs\daemon.log
```

**Check daemon status:**
```bash
summarize daemon status
```

**View logs:**
```bash
# macOS/Linux
tail -f ~/.summarize/logs/daemon.log

# Windows PowerShell
Get-Content -Path "$env:USERPROFILE\.summarize\logs\daemon.log" -Wait
```

**Common issues:**
- Port 39278 already in use → Kill conflicting process
- Permission errors → Check file permissions on ~/.summarize/
- Extension can't connect → Check if daemon is running

---

### 6.4 VS Code Debugging

Set up debugging to step through TypeScript code.

**Step 1: Create launch.json**

Create `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug CLI",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "pnpm",
      "runtimeArgs": ["tsx", "src/cli.ts"],
      "args": ["https://example.com", "--verbose"],
      "console": "integratedTerminal",
      "skipFiles": ["<node_internals>/**"],
      "envFile": "${workspaceFolder}/.env"
    },
    {
      "name": "Debug Current Test",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "pnpm",
      "runtimeArgs": ["vitest", "run", "${file}"],
      "console": "integratedTerminal",
      "skipFiles": ["<node_internals>/**"]
    }
  ]
}
```

**Step 2: Set breakpoints**

1. Open a TypeScript file (e.g., `src/cli-main.ts`)
2. Click in the gutter (left of line numbers) to set a red dot breakpoint
3. Press F5 or click "Run and Debug" → "Debug CLI"

**Step 3: Debug controls**

- **F5**: Continue
- **F10**: Step over (next line)
- **F11**: Step into (enter function)
- **Shift+F11**: Step out (exit function)
- **Shift+F5**: Stop debugging

**Debugging tests:**

1. Open a test file (e.g., `tests/article.test.ts`)
2. Set breakpoint in test
3. Select "Debug Current Test" configuration
4. Press F5

**Variables panel:**
- Shows local variables, function arguments, and closure variables
- Hover over variables in code to see values
- Add expressions to "Watch" panel

---

## Part 7: Git Workflow & Contributing

### 7.1 Commit Convention

This project uses **Conventional Commits**: `type: message`

**Commit types:**

| Type | Description | Example |
|------|-------------|---------|
| `feat` | New feature | `feat: add NewProvider LLM support` |
| `fix` | Bug fix | `fix: handle undefined in extractContent` |
| `docs` | Documentation only | `docs: update developer guide` |
| `refactor` | Code refactoring (no behavior change) | `refactor: simplify flag parsing` |
| `test` | Add or update tests | `test: add coverage for retry logic` |
| `chore` | Maintenance (deps, build, etc.) | `chore: update dependencies` |
| `perf` | Performance improvement | `perf: optimize HTML extraction` |
| `style` | Code style (formatting, no logic change) | `style: fix indentation` |

**Format:**
```
type: short description

Optional longer description explaining why this change is needed.
Can be multiple paragraphs.

Fixes #123
```

**Examples:**

Good commits:
```
feat: add --max-retries flag for LLM calls

Allows users to control retry behavior when LLM API calls fail.
Default is 3 retries, configurable from 0-10.

Fixes #456
```

```
fix: prevent crash on malformed JSON response

The JSON parser didn't handle null responses from some LLM providers.
Now gracefully falls back to empty string.
```

Bad commits:
```
fixed stuff
```

```
WIP
```

```
asdf
```

**Why this matters:**
- Clear commit history makes debugging easier
- Automatic changelog generation
- Easy to understand what each commit does

---

### 7.2 Branch Strategy

**Main branches:**
- `main` - Stable, deployable code
- Feature branches - Your work-in-progress

**Workflow:**

1. **Create feature branch**
   ```bash
   git checkout -b feature/add-newprovider
   ```

2. **Make changes and commit**
   ```bash
   git add src/llm/providers/newprovider.ts
   git commit -m "feat: add NewProvider LLM support"
   ```

3. **Push to remote**
   ```bash
   git push -u origin feature/add-newprovider
   ```

4. **Create Pull Request on GitHub**
   - Go to repository on GitHub
   - Click "Pull Requests" → "New Pull Request"
   - Select your branch
   - Fill in description
   - Submit for review

5. **Address review feedback**
   ```bash
   # Make changes
   git add .
   git commit -m "fix: address review feedback"
   git push
   ```

6. **Merge** (after approval)
   - Squash and merge (preferred for clean history)
   - Or merge commit (preserves all commits)

**Branch naming:**
- `feature/description` - New features
- `fix/description` - Bug fixes
- `docs/description` - Documentation
- `refactor/description` - Code refactoring

---

### 7.3 CI Pipeline

GitHub Actions runs on every push and pull request.

**What CI runs** (see `.github/workflows/ci.yml`):

```yaml
jobs:
  test:
    strategy:
      matrix:
        node: [22, 24]  # Test on multiple Node versions
    steps:
      - Install dependencies (pnpm install)
      - Lint (pnpm lint)
      - Tests with coverage (pnpm test:coverage)
      - Build (pnpm build)
      - Pack verification (ensure dist/ contains expected files)

  extension-e2e:
    steps:
      - Install Playwright
      - Build extension
      - Run E2E tests
```

**CI must pass before merging a PR.**

**Common CI failures:**

1. **Lint errors**
   - Fix: `pnpm lint:fix` → commit → push

2. **Test failures**
   - Fix the tests locally
   - Ensure `pnpm test` passes before pushing

3. **Coverage below threshold**
   - Add more tests to increase coverage
   - Or adjust thresholds in `vitest.config.ts` (requires justification)

4. **Build errors**
   - Fix TypeScript errors
   - Ensure `pnpm build` succeeds locally

5. **Pack verification fails**
   - Ensure dist/ directory is built correctly
   - Check package.json "files" field includes all necessary files

**View CI logs:**
1. Go to GitHub repository
2. Click "Actions" tab
3. Click on the failing workflow run
4. Click on the failing job
5. Expand the failing step to see logs

---

## Part 8: Glossary

**ANSI** - Terminal escape codes for colors/formatting (e.g., `\x1b[31m` = red text)

**Barrel file** - An index.ts that re-exports from other files (e.g., `export * from './foo.js'`)

**CJS (CommonJS)** - Original Node.js module system (`require`/`module.exports`)

**Corepack** - Node.js tool for managing package managers (pnpm, yarn, npm)

**Daemon** - Background process that runs continuously (e.g., the summarize daemon for browser extension)

**ESM (ECMAScript Modules)** - Modern JavaScript module system (`import`/`export`)

**esbuild** - Fast JavaScript bundler written in Go (used to bundle CLI)

**JIT (Just-In-Time)** - Compilation at runtime (TypeScript → JavaScript via tsx)

**MV3 (Manifest V3)** - Latest Chrome extension format (replaces V2)

**Monorepo** - Single repository containing multiple packages

**Node.js** - JavaScript runtime built on Chrome's V8 engine

**npm** - Node Package Manager (default, comes with Node.js)

**npx** - Execute npm packages without installing globally

**nvm** - Node Version Manager (switch between Node.js versions)

**oxfmt** - Fast code formatter (Rust-based, replaces Prettier)

**oxlint** - Fast linter (Rust-based, replaces ESLint)

**pnpm** - Fast, disk-efficient package manager (alternative to npm/yarn)

**Preact** - Lightweight React alternative (3KB, used in browser extension)

**Promise** - JavaScript async primitive (like Future in Java)

**Shebang** - `#!/usr/bin/env node` at top of file (makes it executable)

**SSE (Server-Sent Events)** - One-way HTTP streaming (server → client)

**Tree-shaking** - Removing unused code during bundling (dead code elimination)

**tsc** - TypeScript compiler

**tsx** - TypeScript executor (runs .ts files directly, no build needed)

**TTY (Teletype)** - Terminal/console (e.g., `process.stdout.isTTY` checks if running in terminal)

**Vitest** - Fast test framework (like Jest but faster, ESM-native)

**Workspace** - Monorepo package (defined in `pnpm-workspace.yaml`)

**WXT** - Browser extension framework (handles Chrome/Firefox differences)

**YAML** - Human-readable config format (used in pnpm-workspace.yaml, CI workflows)

---

## Quick Reference Card

**Most common commands:**

```bash
# Development
pnpm install              # Install dependencies
pnpm s "url"              # Run CLI (short form)
pnpm summarize "url"      # Run CLI (long form)
pnpm build                # Full build
pnpm test                 # Run tests
pnpm check                # Quality gate (format + lint + test)

# Code quality
pnpm lint                 # Check for errors
pnpm lint:fix             # Fix errors + format
pnpm format               # Format code
pnpm typecheck            # Check types

# Debugging
pnpm summarize "url" --verbose  # Detailed logs
pnpm summarize "url" --json     # JSON output
pnpm vitest --watch             # Watch mode tests

# Extension
pnpm -C apps/chrome-extension build          # Build Chrome
pnpm -C apps/chrome-extension build:firefox  # Build Firefox
pnpm -C apps/chrome-extension test:chrome    # E2E tests
```

**File extensions:**
- `.ts` - TypeScript source
- `.js` - JavaScript (compiled or source)
- `.d.ts` - TypeScript type declarations
- `.test.ts` - Test files
- `.json` - JSON config
- `.yaml`/`.yml` - YAML config

**Environment variables:**

```bash
# LLM API keys
export OPENAI_API_KEY="sk-..."
export ANTHROPIC_API_KEY="sk-ant-..."
export GOOGLE_API_KEY="..."

# Development
export SUMMARIZE_DISABLE_LOCAL_WHISPER_CPP=1  # Disable Whisper in tests
export NODE_ENV=development
```

**Paths:**
- Source: `src/`, `packages/core/src/`
- Tests: `tests/`
- Build output: `dist/`, `packages/core/dist/`
- Extension build: `apps/chrome-extension/.output/`
- Config: `~/.summarize/` (daemon state, logs, cache)

---

## Getting Help

**Resources:**
- **README.md** - User-facing documentation
- **AGENTS.md** - AI agent guidelines
- **RELEASING.md** - Release process
- **This guide** - Developer onboarding

**Troubleshooting:**
1. Check error message carefully
2. Search project issues on GitHub
3. Run `pnpm check` to catch common issues
4. Use `--verbose` flag for detailed logs
5. Check daemon logs if extension issues

**Common issues:**

| Problem | Solution |
|---------|----------|
| `pnpm: command not found` | Run `corepack enable` |
| `Node version too old` | Install Node 22+ via nvm |
| `Tests failing` | Check `tests/setup.ts` disables Whisper |
| `Build errors` | Run `pnpm clean && pnpm build` |
| `Extension not loading` | Check build output in `.output/` |
| `Type errors` | Run `pnpm typecheck` to see all errors |
| `Lint fails on CI` | Run `pnpm lint:fix` locally |

---

**Welcome to the project! Happy coding!** 🚀
