# WebPilot — How It Works, Technologies, and Work Order

> **Purpose:** This document explains, end to end, what WebPilot is made of, which
> technologies keep it running, and the exact order of operations from a
> natural-language prompt to a completed download. It is the single file a
> contributor or reviewer should read to understand the system's moving parts and
> their sequence.

---

## 1. What Is WebPilot?

WebPilot is a **local-first, privacy-respecting AI browser agent**. You give it a
natural-language task — e.g. *"download 20 sunset wallpapers"* or *"extract all
product prices from https://example.com"* — and it:

1. **Plans** a sequence of structured browser actions.
2. **Drives** a real browser (Playwright / Chromium, Firefox, or WebKit).
3. **Observes** each page and **extracts** structured content (images, links, text, tables).
4. **Ranks** candidates semantically (optional local embeddings) or lexically (always).
5. **Downloads** the best matches with SHA-256 hashing, MIME validation, and
   deduplication.
6. **Streams** every step as typed events over WebSockets for observability.

Everything runs **on the user's machine**. No cloud calls, no telemetry, no data leaves the device.

---

## 2. Technologies & Libraries

### 2.1 Runtime & Build Tooling

| Technology | Version | Role |
|---|---|---|
| **Bun** | 1.4.0 | JavaScript/TypeScript runtime; package manager; test runner |
| **Turborepo** | 2.10 | Monorepo build orchestrator — runs typecheck/test/build in dependency order |
| **TypeScript** | 7.0 | Strict type system (`noUncheckedIndexedAccess`, `verbatimModuleSyntax`, etc.) |

### 2.2 Monorepo Layout

Bun workspaces + Turborepo manage the monorepo:

```
WebPilot/
├── apps/
│   ├── api/          Elysia.js HTTP + WebSocket server (API scaffold)
│   ├── web/          SolidStart dashboard shell (static preview)
│   └── test-site/    Local fixture site for integration/e2e tests
├── packages/         All shared, reusable libraries
│   ├── shared/
│   ├── schemas/
│   ├── browser-core/
│   ├── extraction-core/
│   ├── download-core/
│   ├── ai-core/
│   └── agent-core/
├── tests/
│   ├── unit/         Vitest, pure logic
│   └── integration/  Bun test, real browser + fixture site
├── models/           Local ML model weights (ONNX formats)
└── data/             SQLite DB, downloads, screenshots, sessions
```

### 2.3 Browser Automation

| Technology | Version | Role |
|---|---|---|
| **Playwright** | 1.63 | Controls Chromium/Firefox/WebKit; DOM scanning via `page.evaluate()` |

### 2.4 AI / Local Inference

| Technology | Version | Role |
|---|---|---|
| **@huggingface/transformers** | 4.3 | Runs quantized ONNX models **in-process on CPU** — Qwen2.5-0.5B for planning, MiniLM-L6-v2 for embeddings |

### 2.5 Schema & Data Validation

| Technology | Version | Role |
|---|---|---|
| **Zod** | 4.6 | All data contracts are Zod schemas — type-safe at runtime |

### 2.6 API & Database

| Technology | Version | Role |
|---|---|---|
| **Elysia** | latest | Web framework for the API server (HTTP + WebSocket) |
| **Drizzle** | 0.31 | Type-safe SQL toolkit / ORM for SQLite persistence |

### 2.7 Testing

| Technology | Version | Role |
|---|---|---|
| **Vitest** | 5.0 | Unit tests (pure logic) |
| **Bun test** | — | Integration tests (real browser) |
| **Playwright test** | 1.63 | E2E tests (dashboard + API) |

### 2.8 Frontend (Scaffold)

| Technology | Version | Role |
|---|---|---|
| **SolidStart** | latest | Dashboard shell (static preview) |

## 3. Architecture Overview

### 3.1 Layered Dependency Graph

```
apps/api ──► @webpilot/agent-core ──► @webpilot/browser-core ──► @webpilot/extraction-core
                  │                          │                          │
                  │                          │                          └──► @webpilot/shared
                  │                          │
                  │                          └──► @webpilot/download-core ──► @webpilot/shared
                  │                                       │
                  ├──► @webpilot/ai-core ──► @webpilot/shared
                  │                               │
                  ├──► @webpilot/schemas ─────────┘
                  ├──► @webpilot/shared
                  └──► @webpilot/download-core

apps/web ──► (static shell, no live agent connection yet)
```

**Key principle:** `@webpilot/schemas` is the **contract layer**. It defines
Zod schemas for every data type that crosses a boundary (actions, events, errors,
page state, agent state, task plans, download manifests). `schemas` depends on
nothing inside the monorepo — it is the foundation.

`@webpilot/shared` sits one layer above and provides cross-cutting utilities
(configuration, logging, cancellation, event bus, path safety, hashing, MIME
detection, URLs, text). It only depends on `schemas`.

### 3.2 The "Always-a-Plan" Guarantee

The planning layer has **two planners**:

1. **AI Planner** (`ModelManager` + `parseModelOutput`) — tries Qwen2.5-0.5B
   running in-process via `@huggingface/transformers`. If the model is enabled but
   unavailable (weights not downloaded, import fails), it returns `null` and the
   system transparently falls back.
2. **Rule Planner** (`planWithRules`) — a deterministic regex-based intent parser
   that is **always available**, requires no model, and can produce a valid plan
   from any prompt.

Both produce a `TaskPlan` (goal + ordered action steps) validated against the
same `ActionSchema` before execution.

### 3.3 Package Tour

#### `@webpilot/shared` — Cross-cutting utilities

Configuration, logging, cancellation, event bus, path safety, hashing, MIME
detection, URL helpers, and text utilities. Every other package depends on this.

#### `@webpilot/schemas` — The Contract Layer

Pure Zod schemas + TypeScript types for every data type that crosses a boundary:
actions, events, errors, page state, agent state, task plans, download manifests.
No I/O logic. Nothing in the monorepo depends *upward* from `schemas`.

#### `@webpilot/extraction-core` — Deterministic DOM Extraction

Self-contained scripts (serialized into the browser page by Playwright) that scan
for images, links, buttons, inputs, text, tables, metadata, and JSON-LD. The
`Extractor` class validates results through Zod schemas and adds a `sourcePage`
audit trail.

#### `@webpilot/browser-core` — Playwright Lifecycle

`BrowserManager` owns the browser + isolated in-memory context. `BrowserController`
translates validated `Action` objects into Playwright calls. `locator.ts` provides
resilient element resolution (testid → role → label → placeholder → text → css).

#### `@webpilot/download-core` — Safe Downloading

`Downloader` class: fetch → size check → MIME sniff → SHA-256 → dedupe → save →
manifest update. `verify.ts` checks magic bytes. `naming.ts` produces `tree-001.jpg`
style filenames. `dedupe.ts` provides content-addressed deduplication.
`manifest.ts` writes `manifest.json` atomically.

#### `@webpilot/ai-core` — Local AI

`ModelManager` — lazy-loads Qwen2.5-0.5B (ONNX, q4, CPU). Offline-first.
`EmbeddingManager` — lazy-loads MiniLM-L6-v2 for semantic ranking.
`rule-planner.ts` — deterministic intent parser (fallback).
`parse-plan.ts` — JSON extraction + validation from LLM output.
`prompts.ts` — system prompt + context summary builder.
`ranking.ts` — hybrid lexical + semantic scoring: `score = (1-w)·lexical + w·semantic`.

#### `@webpilot/agent-core` — The Agent Engine

`AgentEngine` — the main loop: plan → validate → execute → observe → evaluate,
with replanning on failure. `validator.ts` — second gate (runtime policy + permissions).
`observer.ts` — compact page state → AI context. `replanner.ts` — recovery strategies.
`state.ts` — immutable state transitions.

#### `apps/api` — The API Server (Elysia)

`server.ts` / `app.ts` — lifecycle + routes + WebSocket `/ws/tasks/:taskId`.
`agent/service.ts` — task queue (single-run), engine construction, event channel,
permission broker, download sync to SQLite. `routes/` — tasks, downloads, system.

#### `apps/web` — SolidStart Dashboard (scaffold)

Static accessible shell. Live task connection is planned but not wired yet.

---

## 4. How It Works — The Agent Loop

### 4.1 The Complete Cycle

```
User submits task
  │
  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    AGENT LOOP (runs repeatedly)                     │
│                                                                     │
│  1. UNDERSTAND  2. PLAN  3. VALIDATE  4. EXECUTE  5. OBSERVE  6. EVALUATE │
│                                                                     │
│  observe page →   generate plan →   schema+policy  →  run →  re-observe →  done? │
│  (counts only)    AI or rules       validation       Playwright            │
│                                                                     │
│  ↓                                                                 │
│  If action fails → replan: retry / degrade locator / abort        │
│                                                                     │
│  Loop continues while: steps < maxSteps AND time < maxTime AND    │
│  not cancelled AND not complete                                    │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 The Six Stages

#### Stage 1: Understand (Observation)

At the top of every loop iteration, the engine observes the current browser page:

```typescript
const pageState = await this.safeObserve();     // → BrowserController.observe()
const contextSummary = observePage(this.state.goal, pageState, this.state);
```

- `safeObserve()` → `BrowserManager.getPageState()` → `Extractor.observe()`
  → runs `scanPageStateInPage()` inside the browser via `page.evaluate()`.
- Result: URL, title, counts (links/images/buttons/inputs/forms/headings),
  and a short list (max 40) of interactive controls — **never raw HTML**.
- `observePage()` packages this into a compact context summary sent to the AI
  model and emitted as `PAGE_CHANGED` events.

#### Stage 2: Plan (AI or Rules)

```typescript
const plan = await this.planFromStrategy(prompt, contextSummary);
```

1. If `planQueue` has leftover steps from a previous AI plan, **reuse them**.
2. If `ModelManager` is enabled → try AI planning with Qwen2.5-0.5B (30s timeout).
   `parseModelOutput()` extracts JSON, validates against `ActionSchema`, falls
   back to rules on any failure.
3. If AI fails or is unavailable → emit `AI_UNAVAILABLE`, call `planWithRules()`.

**Rule-based planning** (`rule-planner.ts`): regex-based intent parser. Extracts
URLs, parses "download N images of X" / "extract Y from URL", falls back to
navigate + extract text. Always available, no model needed.

#### Stage 3: Validate (Schema + Policy + Permission)

Each planned action goes through three gates:

```typescript
const validation = validateAction(action);
if (validation.needsPermission) {
  const granted = await this.requestPermission(validation);
  if (!granted) return false;
}
```

- **Gate 1 (Schema):** Zod `ActionSchema` rejects unknown types, bad fields,
  oversized strings. `HttpUrlSchema` only accepts `http:`/`https:`.
- **Gate 2 (Policy):** `assertRuntimePolicy()` — download count ≤ maxDownloads,
  hosts in allowlist, URLs are http(s).
- **Gate 3 (Permission):** `permissionFor()` — form submissions always need
  approval; downloads >10 or when `requireDownloadConfirmation` is set; clicks on
    elements named "buy"/"delete"/"subscribe" etc.

#### Stage 4: Execute

```typescript
switch (action.type) {
  case 'navigate': → BrowserController.navigate() → page.goto()
  case 'click':    → BrowserController.click()  → resolveLocator() → locator.click()
  case 'type':     → BrowserController.type()   → resolveLocator() → locator.type() + Enter
  case 'press':    → BrowserController.press()  → locator.press(key)
  case 'scroll':   → BrowserController.scroll() → page.evaluate() scroll
  case 'extract':  → executeExtract() → Extractor.extract() → rankItems()
  case 'download': → executeDownload() → rankDownloadCandidates() → Downloader.downloadMany()
  case 'wait':     → BrowserController.wait() → sleep() or waitForSelector()
}
```

- Each action has a **60-second timeout** via `withTimeout()` from `shared/time.ts`.
- `click` and `type` use `resolveLocator()` which tries multiple strategies
  (testid → role → label → placeholder → text → css) and returns the first match.
- `extract` delegates to the `Extractor` class (`scanImagesInPage`, `scanLinksInPage`,
  etc., running in-browser), then **optionally** ranks results using
  `EmbeddingManager` (semantic) + `lexicalSimilarity` (lexical) in a hybrid score.
- `download` selects top-N candidates from state, computes semantic scores if
  embeddings are available, then calls `Downloader.downloadMany()` which batches
  the actual fetches with per-file size, MIME, and dedup checks.

#### Stage 5: Observe (Re-observation After Execution)

After execution, `advanceState()` updates agent state **immutably**:
- Records the action taken and its result summary.
- Updates `currentUrl`, `pageTitle`.
- Stores extracted candidates (for `extract` actions).
- Increments the step counter.

#### Stage 6: Evaluate (Completion Check)

- **`download`**: complete when `completed >= action.count`, or when no steps
  remain and ≥1 file was downloaded.
- **`extract`**: only completes the task if there are no more pending steps
  (a downstream `download` step means extraction alone doesn't finish).
- **All other actions**: not terminal — the loop continues.

### 4.3 Self-Healing & Error Recovery

When a step fails, `replanForFailure()` picks a recovery strategy:

| Error Code | Strategy | Rationale |
|---|---|---|
| `SELECTOR_NOT_FOUND` / `ACTION_TIMEOUT` (click/type) | `retry_alternative` | Degrade locator (testid→role→text→css), retry |
| `NAVIGATION_FAILED` / `NETWORK_FAILED` / `ACTION_TIMEOUT` | `retry_same` | Looks transient; retry same action |
| `AI_INVALID_JSON` / `ACTION_INVALID` | `retry_alternative` | Planner should try a different action |
| Any non-recoverable code, or 3+ attempts | `abort` | Stop the task |

The **locator degradation ladder** is defined in `degradeLocator()`:

```
testid → role → text → css → (give up)
```

---

## 5. Work Order — End-to-End Execution Flow

This section traces a complete task from prompt to persisted result. Example:
**"download 5 sunset wallpapers"**.

### Step 1: Task Creation (via API)

```
HTTP POST /tasks { prompt: "download 5 sunset wallpapers", autoRun: true }
  │
  ▼
AgentService.createTask()
  ├── Generate sequential ID: task-0001 (used as download dir name)
  ├── Store in SQLite: { id, status: "queued", prompt }
  ├── Assign to queue (single-run: one browser at a time)
  └── Emit: TASK_QUEUED
```

### Step 2: Run Begins (AgentEngine.run)

```
AgentService.runTask("task-0001", "download 5 sunset wallpapers")
  ├── Create CancellationToken (for cancellation)
  ├── Create EventBus<AgentEvent> — subscribers: SQLite store, WebSocket broadcaster, logger
  ├── Build BrowserManager (Playwright: Chromium headless) → launch()
  ├── Build BrowserController (wraps BrowserManager)
  ├── Build Downloader (downloadsRoot → data/downloads/)
  ├── Build AgentEngine (wires all + ModelManager + EmbeddingManager)
  ├── Emit: TASK_STARTED
  └── Enter agentLoop("download 5 sunset wallpapers")
```

### Step 3: Agent Loop — Plan & Navigate

```
agentLoop iteration 1:
  [1] UNDERSTAND → safeObserve() → empty page state (nothing loaded yet)
  [2] PLAN → planQueue empty → try AI (Qwen2.5-0.5B) → parseModelOutput → fallback to planWithRules()
  [3-6] Execute navigate → observe → evaluate → false → loop continues
```

### Step 4: Agent Loop — Type & Search

```
agentLoop iteration 2:
  [1] UNDERSTAND → safeObserve() → search page with 50 images, search-input found
  [2] PLAN → planQueue has step → pop { type: "sunset wallpapers", target: search-input, submit: true }
  [3] VALIDATE → permissionFor → form_submission → emit PERMISSION_REQUESTED
  [4] Execute → resolveLocator("search-input") → type + Enter → navigate to results
```

### Step 5: Agent Loop — Extract Images

```
agentLoop iteration 3:
  [1-2] UNDERSTAND → 500 images on results page; PLAN → extract images with query "sunset wallpaper"
  [4] Execute → Extractor.extractImages() → scanImagesInPage() (in-browser) → 500 images
  ├── rankItems() → computeSemanticScores() (MiniLM if available) → rankImages(semanticWeight=0.4, topK=50)
  ├── state.candidates = top 50, state.selected = top 10
  └── emit: ITEMS_RANKED, ITEM_FOUND (×5)
```

### Step 6: Agent Loop — Download

```
agentLoop iteration 4:
  [1-2] PLAN → pop { type: "download", count: 5 }
  [3] VALIDATE → count=5 ≤ maxDownloads(100) → no permission needed (count < 10)
  [4] Execute → Downloader.downloadMany():
  For each candidate (stop at count=5):
    ├── Fetch over HTTP (host in allowlist ✓)
    ├── verifyPayload() → sniffType() magic bytes → "image/jpeg"
    ├── sha256Hex(payload)
    ├── DedupeIndex.find(hash) → skip if duplicate
    ├── buildFilename("sunset-wallpaper", 001, "jpg")
    ├── writeFile → data/downloads/task-0001/sunset-wallpaper-001.jpg
    ├── appendManifestFile → manifest.json
    └── Returns: { completed: 5, duplicates: 1 }
  [5-6] Observe → Evaluate → completed(5) >= count(5) → TRUE → TASK COMPLETE
```

### Step 7: Finalization

```
AgentService.runTask() finally block:
  ├── broker.denyAll("task-0001") → deny pending permissions
  ├── syncDownloads → read manifest.json → SQLite
  ├── store.updateTask: status=completed, stepCount=4, downloadsCount=5
  └── broadcaster.publish → all WebSocket clients updated
```

### Step 8: Client Receives Everything

Dashboard over `GET /ws/tasks/task-0001`:

```
1. { kind: "hello", taskId }
2. { kind: "task",  task: { ... } }
3. { kind: "event", event: { type: "TASK_QUEUED" } }
4. { kind: "event", event: { type: "TASK_STARTED" } }
5. { kind: "event", event: { type: "AI_THINKING" / "AI_UNAVAILABLE" } }
6. { kind: "event", event: { type: "PLAN_CREATED", data: { steps, source } } }
... ACTION_PLANNED, ACTION_STARTED, ACTION_COMPLETED events ...
... ITEMS_RANKED, DOWNLOAD_STARTED, DOWNLOAD_COMPLETED ...
N. { kind: "event", event: { type: "TASK_COMPLETED", data: { steps: 4, downloaded: 5 } } }
```

Events are also persisted to SQLite via `TaskChannel`, so a WebSocket
reconnection replays the full timeline from `?since=<sequence>`.

### Step 9: Artifacts on Disk

```
data/
├── webpilot.db
└── downloads/
    └── task-0001/
        ├── manifest.json           ← full provenance for every file
        ├── .dedupe.json            ← content hash index (hidden)
        ├── sunset-wallpaper-001.jpg
        ├── ...-002.jpg
        ├── ...-003.jpg
        ├── ...-004.jpg
        └── ...-005.jpg
```

---

## 6. Key Guarantees & Design Principles

| Principle | How It's Enforced |
|---|---|
| **Always a valid plan** | AI → `parseModelOutput` → rule fallback. Two layers. |
| **Safety first** | Schema (Zod) → runtime policy → permission gate (human approval) |
| **Never trust the server** | MIME sniffing (magic bytes), SHA-256 hashing, `safeJoin` |
| **No data leaves device** | All AI on CPU in-process; `allowModelDownload=false` by default |
| **Self-healing** | `replanForFailure()` — retry/replan/abort based on error codes |
| **Full observability** | 24 typed `AgentEvent` types, event bus, WebSocket, SQLite |
| **Cooperative cancellation** | `CancellationToken` checked every loop iteration |
| **Graceful degradation** | AI off → rules; embeddings missing → lexical; browser error → safe state |

---

## 7. Environment Configuration

All settings are `WEBPILOT_*` env vars with safe defaults (see `.env.example`).

| Variable | Default | Controls |
|---|---|---|
| `WEBPILOT_AI_ENABLED` | `true` | Whether to try the local LLM planner first |
| `WEBPILOT_AI_ALLOW_MODEL_DOWNLOAD` | `false` | Whether to download model weights from HF Hub |
| `WEBPILOT_AI_MODEL_ID` | `onnx-community/Qwen2.5-0.5B-Instruct` | Planner model |
| `WEBPILOT_AI_QUANTIZATION` | `q4` | Quantization level |
| `WEBPILOT_AI_MAX_NEW_TOKENS` | `512` | Max output tokens |
| `WEBPILOT_EMBEDDING_MODEL_ID` | `Xenova/all-MiniLM-L6-v2` | Embedding model |
| `WEBPILOT_BROWSER` | `chromium` | `chromium` / `firefox` / `webkit` |
| `WEBPILOT_HEADLESS` | `true` | Run browser headlessly |
| `WEBPILOT_MAX_AGENT_STEPS` | `30` | Max steps per task |
| `WEBPILOT_MAX_DOWNLOADS` | `100` | Max files per task |
| `WEBPILOT_MAX_FILE_SIZE_MB` | `25` | Per-file size cap |
| `WEBPILOT_MAX_TASK_TIME_MS` | `600000` (10 min) | Wall-clock cap |
| `WEBPILOT_DOWNLOAD_ALLOWLIST` | *(empty)* | Host allowlist (empty = open) |
| `WEBPILOT_REQUIRE_DOWNLOAD_CONFIRMATION` | `false` | Always ask before downloading |
| `WEBPILOT_LOG_LEVEL` | `info` | `debug` / `info` / `warn` / `error` |

---

## 8. Event Types Reference

The 24 `AgentEventType` values emitted on the event bus (and streamed over
WebSocket):

| Event | When Emitted |
|---|---|
| `TASK_QUEUED` | Task created, waiting for browser |
| `TASK_STARTED` | Browser launched, loop begins |
| `AI_THINKING` | Model is generating a plan |
| `AI_UNAVAILABLE` | Model unavailable, falling back to rules |
| `PLAN_CREATED` | A plan was produced (AI or rules) |
| `ACTION_PLANNED` | An action was validated, about to run |
| `ACTION_STARTED` | Action execution begins |
| `ACTION_COMPLETED` | Action succeeded |
| `ACTION_FAILED` | Action threw an error |
| `ACTION_REJECTED` | Action failed schema/policy validation |
| `PAGE_CHANGED` | Browser navigation detected |
| `ITEM_FOUND` | An extracted/ranked item (top results, ×5) |
| `ITEMS_RANKED` | Batch of items ranked by relevance |
| `DOWNLOAD_STARTED` | Download attempt begins |
| `DOWNLOAD_COMPLETED` | Download batch finished |
| `DOWNLOAD_FAILED` | Individual download failed |
| `PERMISSION_REQUESTED` | Sensitive action needs human approval |
| `PERMISSION_GRANTED` | User approved the action |
| `PERMISSION_DENIED` | User denied the action |
| `TASK_COMPLETED` | All goals achieved |
| `TASK_FAILED` | Unrecoverable error |
| `TASK_CANCELLED` | Cancelled by user |
| `LOG` | Diagnostic message |
| `BROWSER_STATUS` | Browser lifecycle change |

---

## 9. Error Codes & Recovery

The 24 `ErrorCode` values and how `replanForFailure()` handles them:

| Code | Recoverable | Recovery Strategy |
|---|---|---|
| `SELECTOR_NOT_FOUND` | ✅ | Degrade locator, retry |
| `NAVIGATION_FAILED` | ✅ (unless DNS/connection) | Retry same |
| `NETWORK_FAILED` | ✅ | Retry same |
| `ACTION_TIMEOUT` | ✅ | Retry same / degrade locator |
| `AI_TIMEOUT` | ✅ | Retry AI, then fall back to rules |
| `AI_INVALID_JSON` | ✅ | Fall back to rule parser |
| `AI_MODEL_UNAVAILABLE` | ❌ | Use rule planner (no retry) |
| `DOWNLOAD_FAILED` | ✅ | Try next candidate |
| `MIME_MISMATCH` | ✅ | Fix extension, retry |
| `DUPLICATE_FILE` | ✅ | Skip, try next |
| `FILE_TOO_LARGE` | ❌ | Skip file, try next |
| `LIMIT_EXCEEDED` | ❌ | Abort |
| `UNSAFE_URL` | ❌ | Reject action |
| `PERMISSION_DENIED` | ❌ | Stop task |
| `TASK_CANCELLED` | ❌ | Stop immediately |
| `TASK_TIMEOUT` | ❌ | Stop task |
| `CONFIG_INVALID` | ❌ | Abort |
| `BROWSER_LAUNCH_FAILED` | ❌ | Abort |
| `EXTRACTION_FAILED` | ✅ | Retry or skip |
| `ACTION_INVALID` | ✅ | Planner tries again |
| `ACTION_NOT_PERMITTED` | ❌ | Stop task |
| `UNKNOWN` | ✅ | Retry once, then abort |

---

## 10. Running WebPilot

### Setup

```bash
git clone https://github.com/missarii/Web-Pilot.git
cd Web-Pilot
bun install
cp .env.example .env
```

### Commands

```bash
bun run api              # Start the API server (http://127.0.0.1:8787)
bun run test-site        # Start fixture test site (http://127.0.0.1:3001/test-site/)
bun run dev              # Turbo: run all dev scripts in parallel
bun run typecheck        # Turbo: typecheck all packages
bun run test:unit        # Vitest unit tests
bun run test:integration # Bun test — real browser + fixture site
bun run test:e2e         # Playwright E2E tests (install browsers first)
bun run build            # Turbo: build all packages
```

### CI

`.github/workflows/ci.yml` runs `bun install --frozen-lockfile`, then
`bun run typecheck` and `bun run test:unit`. Integration, e2e, browser install,
and model downloads are deliberately out of scope.

---

## 11. Testing Strategy

| Layer | Tool | What It Tests | Location |
|---|---|---|---|
| Unit | Vitest | Pure logic: rule planner, lexical similarity, config, naming, dedupe | `tests/unit/` |
| Integration | Bun test | Real Playwright browser + fixture site: navigation, extraction, downloads | `tests/integration/` |
| E2E | Playwright | Dashboard + API: create task, receive events, download files | `tests/e2e/` |

---

## 12. Where to Go Next

| Want to… | Go to |
|---|---|
| Add a new action type | `@webpilot/schemas/src/actions.ts` (extend `ActionSchema`) |
| Change what data is extracted | `@webpilot/extraction-core/src/scripts/` (DOM scanning) |
| Improve planning | `@webpilot/ai-core/src/rule-planner.ts` or `prompts.ts` |
| Add a download safety check | `@webpilot/download-core/src/verify.ts` |
| Change the agent loop | `@webpilot/agent-core/src/engine.ts` |
| Add an API endpoint | `apps/api/src/routes/` |
| Build the dashboard | `apps/web/` (SolidStart) |
| Understand the existing architecture | `docs/architecture.md`