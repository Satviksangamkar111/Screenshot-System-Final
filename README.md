# UI Documentation Engine

Automates browser exploration of enterprise web applications (SAP Fiori, UI5, Web Dynpro, WebGUI) to produce Word documents with captured evidence. Discovers and interacts with every control, takes screenshots, and generates two summaries: a **General Summary** (deterministic, no API key needed) and an **AI Summary** (LLM-powered, evidence-validated).

---

## Table of Contents

- [About](#about)
- [Installation](#installation)
- [Usage](#usage)
- [Configuration](#configuration)
- [Architecture](#architecture)
- [Testing & Build](#testing--build)
- [Contributing](#contributing)
- [License](#license)

---

## About

**Problem:** Documenting enterprise application UIs is manual, slow, and error-prone. Screenshots go stale, and nobody knows what changed between versions.

**Solution:** Feed this tool a URL. It opens a browser, discovers every documentable control, fills test data, captures evidence, and assembles a Word document with two analyses side by side.

**For whom:**
- QA teams comparing old vs. new application versions
- Technical writers documenting SAP Fiori or Web Dynpro apps
- Operators who need precise screenshots with no vision-model hallucination

**Key features:**
- **Evidence-based, not pixel-based.** No vision models. The LLM sees structured semantic data (facts, patterns, control tree), never raw screenshots or JSON dumps.
- **End users need nothing installed.** Just open the web URL. Node.js + Chrome/Edge run only on the server machine.
- **Two independent summaries.** General Summary (zero network calls) always works. AI Summary (LLM) validates every claim against real evidence before showing it.
- **Five technology adapters.** Recognizes UI5 → web components → Web Dynpro → WebGUI → generic ARIA/DOM, tried in priority order.
- **Manual mode.** Operator fills fields by hand through a live-streamed view; the engine still handles discovery, screenshots, and assembly.
- **CLI or web.** Paste URLs in a browser, or run repeatable captures from YAML config files.

---

## Installation

### End Users

**Required:** A web browser (Chrome, Edge, Safari, Firefox) and network access to the server.

**That's it.** End users run nothing locally. No Node.js, npm, or browser driver installation needed. Just open the server's URL and paste application URLs.

### Server Setup

**Server machine only** (the machine running the engine):

| Requirement | Why |
|---|---|
| **Node.js ≥ 20** | Runs the engine and server |
| **Google Chrome or Microsoft Edge** | Driven directly over Chrome DevTools Protocol; no binary download |
| **LLM API key** *(optional)* | Only for AI Summary. Deterministic General Summary works without one |

**Once deployed as a self-contained package, the server requires no manual setup either** — just run the executable or container and point browsers at it.

### Setup

```bash
git clone https://github.com/Satviksangamkar111/Screenshot-System-Final.git
cd Screenshot-System-Final
npm install
npm run build
```

Verify the build succeeded:

```bash
npm run typecheck
```

---

## Usage

### Quick start (web UI)

```bash
npm run serve
```

Open `http://localhost:5173` in your browser. Paste one or both application URLs and click **Generate Document**.

### Command line (CLI)

For repeatable, unattended runs, use a YAML config file.

Sign in once per version:

```bash
npm run login -- --app <app-name> --version-id new
```

Capture each version:

```bash
npm run capture -- --app <app-name> --version-id old
npm run capture -- --app <app-name> --version-id new
```

Build the document:

```bash
npm run assemble -- --app <app-name>
```

Or all at once:

```bash
npm run run -- --app <app-name>
```

**Useful flags:**
- `--headed` — Watch the browser (default is headless)
- `--verbose` — Log per-control activity
- `--out <path>` — Save document to a specific path

### Output

Both modes produce:
- `output/runs/<runId>/` — capture data (trace.json, screenshots)
- `output/jobs/<jobId>/` — final Word document

---

## Configuration

### LLM keys (optional, for AI Summary)

Create `.env` in the project root (never committed to git):

```env
GROQ_API_KEY=gsk_...
GROQ_API_KEY_2=gsk_...        # optional second key for rotation
GROQ_API_KEY_3=gsk_...        # optional third key
GEMINI_API_KEY=AIzaSy...      # optional fallback
```

Model selection (defaults are current Groq recommendations):

```env
GROQ_MODELS=openai/gpt-oss-120b,qwen/qwen3.8-27b,qwen/qwen3.6-27b,openai/gpt-oss-20b
GEMINI_MODEL=gemini-3.6-flash
```

Without any key, the tool runs fully offline — only the General Summary is generated.

### Application YAML (`config/apps/<app>.yaml`)

For CLI runs. Example:

```yaml
name: my-app
description: A Fiori application
versions:
  old:
    url: https://app.example.com/old
    requiresAuth: true
  new:
    url: https://app.example.com/new
    requiresAuth: true

safety:
  denyLabels:
    - Save
    - Submit
    - Delete
  allowLabels:
    - Search
    - Execute

budgets:
  appReadyTimeoutMs: 180000  # 3 minutes
  maxControlsPerPage: ~      # unlimited by default
  maxDepth: 5
  maxPages: 50

testdata:
  Customer:
    - "CUST-001"
    - "CUST-002"
  Date:
    - today
    - "+30d"
```

### Optional config files

| File | Purpose |
|---|---|
| `config/lexicon.yaml` | Canonical label names (e.g. map UI5 id suffixes to user-friendly names) |
| `config/testdata.yaml` | Default dummy values by label or control kind |

---

## Architecture

This section describes the engine internals. Skip if you only need to use the tool.

### Two-stage pipeline

```
Capture phase (browser automation)
  └─→ trace.json + screenshots

Assembly phase (pure function)
  └─→ General Summary (deterministic)
  └─→ AI Summary (LLM, evidence-validated)
  └─→ .docx document
```

**Capture** drives a real Chrome/Edge browser directly over Chrome DevTools Protocol (no Playwright, no Puppeteer). It:
1. Discovers every interactive control using five technology adapters (UI5 → web components → Web Dynpro → WebGUI → ARIA fallback)
2. Interacts with each (filling fields, opening dropdowns, taking screenshots)
3. Branches on dialogs with 2+ alternatives (documents all paths)
4. Writes `trace.json` and screenshot files per version

**Assembly** reads `trace.json` (no browser needed) and:
1. Builds a semantic model (Page → Section → Container → Control tree)
2. Derives facts and patterns for the **General Summary** (fully deterministic, zero network calls)
3. If LLM keys are configured, sends the structured model (not screenshots) for the **AI Summary**, validates every cited evidence reference, and drops any unsupported claim

### Five technology adapters (priority order)

1. **UI5 adapter** — reads `sap.ui.core.Element.registry` for live controls, covers both classic (`sap.m.*`) and MDC building blocks (`sap.ui.mdc.*`)
2. **Web Components adapter** — finds `ui5-*` custom elements, including inside shadow roots
3. **Web Dynpro adapter** — parses `[lsdata]` attribute for SAP Web Dynpro ABAP (Lightspeed renderer)
4. **WebGUI adapter** — detects `/its/webgui` URLs and ITS-specific CSS classes
5. **ARIA/DOM adapter** — generic fallback; covers standard HTML roles and elements

Each adapter exposes `detect()`, `probe()`, `classify()` methods. Controls are claimed by the first adapter that recognizes them; later adapters skip already-claimed controls.

### Discovery and interaction

The **Exploration Engine** (`state/explorer.ts`) runs a recursive loop:

1. **Readiness check** — Wait for controls or dialogs to exist, nothing busy, control count stable (up to 3 minutes by default)
2. **Discover** — Walk the current page state, classify each control by kind (input, select, date, valueHelp, checkbox, etc.)
3. **Interact** — For each control, dispatch the right handler (fill field, open dropdown, click button)
4. **Capture** — Take a full-page screenshot at the point of interest (opened dialog, chosen value, etc.)
5. **Recurse** — If a control revealed new UI, explore that branch; on dialogs with 2+ choices, branch for each one

**Loop detection:** If the page fingerprint repeats, stop. Prevents infinite traversal of pages with persistent state.

### Control classification and points

A control becomes a **documentation point** (gets a label + screenshot) only if it's worth seeing:

**Included in documentation:**
- Dropdown/select, calendar, value-help lookup
- Multi-select, checkbox, radio, switch
- File upload
- Button that reveals new UI (opens dialog, navigates page)

**Excluded:**
- Plain text input (filled, not screenshotted)
- Read-only fields (not interactive)
- Pure action button (Save, Submit) — nothing visibly changes
- Button that hides/collapses (detected by fingerprint diff: collapse drops visible-control count below 60%)

### Evidence and validation

All evidence is **structured**, never pixels:

- `trace.json` contains: control tree (kind, label, section, container), interaction events, screenshots paths
- AI Summary LLM receives: semantic model, derived facts, detected patterns, diffs
- Every AI claim is validated against real evidence before display; unsupported claims are dropped

### General Summary (deterministic)

Built from `trace.json` alone, zero network calls:

1. **Single version** → facts about structure, interaction behavior, patterns
2. **Two versions** → categorized changes (ADDED, REMOVED, CHANGED, COMPOSITION_CHANGE, UNRESOLVED) plus overall level (no_change, minor, moderate, major)
3. **Multiset matching** — controls matched by kind + label, never position; handles rearrangements correctly
4. **Junk-label handling** — meaningless labels (".", bare icons) are anonymized but still counted structurally

### AI Summary (LLM over structured evidence)

```
UiDocumentationModel + facts + patterns + diff
  ↓
ai-context.ts (builds structured prompt)
  ↓
Groq (key rotation) → Gemini (fallback)
  ↓
ai-summary.ts (parses, validates every evidence ref)
  ↓
AiSummaryResult → rendered in UI + appended to .docx
```

- **Graceful degradation:** No key set, every provider down, or malformed response → falls back to General Summary automatically
- **Key rotation:** Three Groq keys rotated on auth/rate-limit/5xx errors; Gemini fallback once every model exhausts every key
- **Evidence validation:** Points citing evidence IDs that don't exist in the catalog are dropped before display

### Browser automation (CDP)

No Playwright, no Puppeteer. The engine speaks **Chrome DevTools Protocol** directly:

- `automation/chrome-launcher.ts` — Finds and launches Chrome/Edge, returns the CDP WebSocket
- `automation/cdp-client.ts` / `cdp-session.ts` — Protocol transport, per-target sessions
- `automation/page-shim.ts` / `locator-shim.ts` — Small Playwright-shaped API (`page.locator(...).click()`) over raw CDP
- `automation/shadow-pierce.ts` — Shadow-DOM helpers for web components

Locator syntax (`locator-shim.ts`) implements `:visible`, `:has-text()`, `:text-is()` pseudo-classes (standard CDP `querySelector` rejects them as invalid). Web component internals (calendar picker, value-help button) are clicked by `shadowPierceOrF4`: try shadow querySelector first, fall back to F4 (SAP's universal "open picker" shortcut).

### Shared web server

The HTTP server (`src/server/`) handles both web UI and live view:

- `app.ts` — HTTP routes, static serving
- `jobs.ts` — Job lifecycle (sign-in, capture, assemble, summaries)
- `remoteControl.ts` — CDP screencast feed for live view during sign-in or manual mode
- `userId.ts` — Per-browser session isolation via HttpOnly cookies

Each browser gets its own session cookie; sessions are stored per origin (e.g., two versions on the same host share one sign-in).

### Manual data-entry mode

Operator fills fields themselves via a streamed live view; engine handles discovery, sequencing, screenshots, assembly. Pauses at each control, waits for **Submit** (accept typed value) or **Skip** (leave empty). Output is structurally identical to automatic mode.

### Data model

All core types in `src/types.ts`; AI Summary's model in `src/doc-intelligence/model.ts`.

Key types:
- `ControlKind` — Every classified type (input, select, date, valueHelp, checkbox, revealButton, actionButton, tab, readonly)
- `ControlDescriptor` — A control before interaction (id, kind, label, selectors, section, container)
- `Evidence` — One screenshot point (label, paths, interaction type, section/container identity)
- `RunTrace` — Full capture↔assembly contract serialized to `trace.json`
- `UiDocumentationModel` — AI Summary tree (Page → Section → Container → Control)

---

## Testing & Build

### Build

```bash
npm run build
```

### Type checking

```bash
npm run typecheck
```

No automated tests yet (see [Contributing](#contributing)).

---

## Contributing

Bug reports and feature requests: open an issue on [GitHub](https://github.com/Satviksangamkar111/Screenshot-System-Final/issues).

**Development guidelines:**
- Fork the repo, create a feature branch
- Follow the existing code style (active voice in comments, concise variable names)
- Type-check: `npm run typecheck` must pass
- Test manually against a Fiori app before opening a PR

---

## License

MIT. See [LICENSE](LICENSE) for details.

---

## Troubleshooting

| Symptom | Solution |
|---|---|
| Document has few/no points, screenshots look empty | App was still loading. Raise `budgets.appReadyTimeoutMs` (default 3 min) and re-run |
| Web component not discovered | Verify it's a `ui5-*` tag; if nested in a shadow root, `walkShadowElements` finds it only if the host is reachable from light DOM |
| Picker doesn't open on a web component | `shadowPierceOrF4` tries shadow querySelector first, then F4. Fully custom pickers may need a new case in `interaction/handlers/open-overlay.ts` |
| General Summary tab is empty / "no summary returned" | Server running old code. `index.html` is read per request, but `jobs.ts` lives in the running process. **Restart the server** |
| AI Summary reads identically to General | LLM path fell back. Check server log for reason (usually no key set or all providers failing) |
| AI Summary tab shows an error | Read the message: "No LLM API keys configured" (set `.env`), or "All LLM providers failed" followed by each provider's error |
| `Groq 404: model ... does not exist` | That model was retired. Update `GROQ_MODELS` in `.env` to a current model id |
