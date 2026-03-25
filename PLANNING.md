# Figma Plugin for deco CMS — Bidirectional Design ↔ Code Sync

## Context

A Figma plugin that enables two flows:
1. **Design → Code**: Select a Figma design, use Claude AI (Opus) to convert it to a Preact/TSX section, and save it to a deco CMS site
2. **Code → Design**: Browse coded sections from deco CMS and render them as frames in Figma

**Feasibility**: Fully feasible. The deco admin already has all the backend building blocks — MCP tools for file operations, screenshot capture via Browserless, Claude AI integration via the chat API, and section creation conventions. We need a Figma plugin + a few thin API bridge routes in the admin repo.

---

## Architecture Overview

```
Figma Plugin                    deco Admin Server              deco Site
┌──────────┐  ┌──────────┐    ┌─────────────────┐            ┌──────────┐
│ Main     │  │ UI       │    │ /api/figma/*     │            │ /live/   │
│ Thread   │◄►│ Thread   │───►│ design-to-code   │──Claude──► │ previews │
│ (code.ts)│  │ (React)  │    │ sections         │            │          │
│          │  │          │◄───│ screenshot       │◄Browserless│          │
└──────────┘  └──────────┘    └─────────────────┘            └──────────┘
     │                              │
     │ Figma API                    │ ctx.invoke (existing MCP tools)
     │ (read/write nodes)           │
                               ┌─────────────────┐
                               │ patchFile, read  │
                               │ listFiles, etc.  │
                               └─────────────────┘
```

### How Figma Plugins Work

Figma plugins have **two isolated threads** that communicate via `postMessage`:

| Thread | Access | No Access |
|--------|--------|-----------|
| **Main thread** (`code.ts`) | Figma Plugin API (read/write nodes, export images) | Network requests |
| **UI thread** (`ui.html` + React) | `fetch()` to external APIs, DOM rendering | Figma document |

This means: all network calls go through the UI thread, all Figma operations go through the main thread.

---

## Part 1: Figma Plugin (this repo)

### Directory Structure

```
figma-deco-plugin/
  manifest.json           # Figma plugin manifest (name, id, main, ui)
  code.ts                 # Main thread — selection handling, PNG export, frame creation
  ui.html                 # UI entry point (loads bundled React app)
  src/
    App.tsx               # React app root with tab navigation
    views/
      Auth.tsx            # API key login + site selector
      DesignToCode.tsx    # Flow 1: select frame → AI → save section
      CodeToDesign.tsx    # Flow 2: browse sections → import to Figma
    api/
      decoApi.ts          # HTTP client for /api/figma/* endpoints
    types.ts              # Shared message types between main ↔ UI threads
  package.json            # Dependencies: react, esbuild/bun for bundling
  tsconfig.json           # TypeScript config with @figma/plugin-typings
```

### Main Thread (`code.ts`) — Key Logic

```typescript
// Show plugin UI
figma.showUI(__html__, { width: 400, height: 600 });

// Listen for selection changes → export frame as PNG
figma.on('selectionchange', () => {
  const node = figma.currentPage.selection[0];
  if (!node) return;

  // Export selected frame as PNG for Claude vision input
  node.exportAsync({ format: 'PNG', constraint: { type: 'SCALE', value: 2 } })
    .then(bytes => {
      figma.ui.postMessage({
        type: 'selection',
        name: node.name,
        bytes: bytes,
        width: node.width,
        height: node.height,
      });
    });
});

// Handle messages from UI thread
figma.ui.onmessage = (msg) => {
  // Code → Design: create a Figma frame from a screenshot
  if (msg.type === 'create-frame') {
    const frame = figma.createFrame();
    frame.name = msg.name;
    frame.resize(msg.width, msg.height);
    const image = figma.createImage(new Uint8Array(msg.imageBytes));
    frame.fills = [{
      type: 'IMAGE',
      imageHash: image.hash,
      scaleMode: 'FILL',
    }];
    // Store deco metadata for future sync
    frame.setPluginData('decoSection', JSON.stringify({
      site: msg.site,
      resolveType: msg.resolveType,
    }));
    // Center in viewport
    figma.viewport.scrollAndZoomIntoView([frame]);
    figma.notify(`Imported: ${msg.name}`);
  }
};
```

### Message Protocol (`src/types.ts`)

```typescript
// Main thread → UI thread
type MainToUI =
  | { type: 'selection'; name: string; bytes: Uint8Array; width: number; height: number }
  | { type: 'frame-created'; frameId: string }
  | { type: 'error'; message: string };

// UI thread → Main thread
type UIToMain =
  | { type: 'export-selection' }
  | { type: 'create-frame'; name: string; imageBytes: ArrayBuffer; width: number; height: number; site: string; resolveType: string };
```

### Authentication

- Reuse deco admin's existing `api_key` table — users generate API keys in admin settings
- Plugin stores key in `figma.clientStorage.setAsync('decoApiKey', key)` (persistent per-user)
- All API calls include `Authorization: Bearer {key}` header
- The existing `withAuth.ts` middleware in the admin repo already validates bearer tokens

---

## Part 2: Backend Bridge Routes (in deco admin repo: `deco-sites/admin`)

Three new API routes to add to the admin codebase. These are thin bridges that compose existing infrastructure.

### Route 1: `POST /api/figma/design-to-code`

Converts a Figma design image into a Preact/TSX section using Claude AI.

**Request:**
```json
{
  "image": "<base64 PNG>",
  "nodeName": "Hero Section",
  "nodeMetadata": { "textContent": [...], "colors": [...], "fonts": [...] },
  "sitename": "my-site",
  "env": "main",
  "sectionName": "HeroSection"
}
```

**Process:**
1. Fetch site theme via existing `getTheme` loader → provides TailwindCSS design tokens
2. Load section conventions from `agents/config.ts` (lines 151-225) → system prompt for Claude
3. Call Claude API using the same pattern as `routes/api/chat.ts`:
   - Uses `@ai-sdk/openai-compatible` + `MESH_API_KEY` env var
   - Sends image as vision input + node metadata for precision
4. Claude generates a complete Preact/TSX section with `Props` interface
5. Save via `ctx.invoke["deco-sites/admin"].actions.daemon.fs.patchFile` → writes to `/sections/{SectionName}.tsx`
6. Return section path + preview URL

**Response:**
```json
{
  "sectionPath": "sections/HeroSection.tsx",
  "previewUrl": "https://my-site.deco.site/live/previews/...",
  "code": "// generated TSX source"
}
```

**Reuses from admin repo:**
- `routes/api/chat.ts` — Claude AI integration pattern (MCP client + AI SDK)
- `components/spaces/siteEditor/extensions/Deco/agents/config.ts` — section creation conventions
- `actions/daemon/fs/patchFile.ts` — file write via MCP tool
- `sdk/chatConfig.ts` — `MESH_API_KEY`, `MESH_BASE_URL` config

### Route 2: `GET /api/figma/sections?site={site}&env={env}`

Lists all sections available on a deco site with metadata.

**Process:**
1. Call `loaders/decopilot/listFiles.ts` to list files matching `/sections/*.tsx`
2. For each file, read source via `loaders/daemon/fs/read.ts`
3. Parse each file to extract: component name, `Props` interface fields, exports
4. Return section catalog

**Response:**
```json
{
  "sections": [
    {
      "name": "HeroSection",
      "path": "sections/HeroSection.tsx",
      "resolveType": "site/sections/HeroSection.tsx",
      "props": ["title: string", "subtitle?: string", "image: ImageWidget"]
    }
  ]
}
```

**Reuses:**
- `loaders/decopilot/listFiles.ts` — file listing
- `loaders/daemon/fs/read.ts` — file reading

### Route 3: `POST /api/figma/section-screenshot`

Generates a screenshot of a rendered section.

**Request:**
```json
{
  "sitename": "my-site",
  "resolveType": "site/sections/HeroSection.tsx",
  "props": { "title": "Hello World" },
  "width": 1440,
  "height": 900,
  "format": "png"
}
```

**Process:**
1. Construct preview URL: `https://{site-domain}/live/previews/{resolveType}` with base64-encoded props
2. Call `getWebsiteScreenshot` loader (uses Browserless Chrome headless browser)
3. Return screenshot URL or bytes

**Reuses:**
- `loaders/decopilot/getWebsiteScreenshot.ts` — Browserless Chrome screenshot capture
- Preview URL system from `components/spaces/siteEditor/extensions/CMS/views/Preview.tsx`

---

## Part 3: Detailed Flows

### Flow 1 — Design → Code (Figma → deco CMS)

```
User selects frame in Figma
        ↓
Main thread exports as PNG (2x) + extracts node metadata
        ↓
UI thread receives image via postMessage
        ↓
UI sends POST /api/figma/design-to-code
  (image + metadata + site name + section name)
        ↓
Admin server fetches site theme (TailwindCSS tokens)
        ↓
Admin server calls Claude with:
  - Design image (vision)
  - Node metadata (text, colors, fonts, spacing)
  - Section conventions (Props interface, widget types, Preact patterns)
  - Site theme (design tokens)
        ↓
Claude generates Preact/TSX section code
        ↓
Admin server saves to /sections/{Name}.tsx via patchFile
        ↓
Returns preview URL + code to plugin
        ↓
Plugin shows rendered preview for user approval
        ↓
User can iterate (send feedback → Claude refines)
```

### Flow 2 — Code → Design (deco CMS → Figma)

```
User opens "Import from Code" tab
        ↓
Plugin calls GET /api/figma/sections?site=mysite
        ↓
Admin returns list of sections with metadata
        ↓
Plugin displays section gallery with cards
        ↓
User clicks a section card
        ↓
Plugin calls POST /api/figma/section-screenshot
  (resolveType + viewport dimensions)
        ↓
Admin renders section via Browserless → returns screenshot
        ↓
Plugin sends screenshot to main thread via postMessage
        ↓
Main thread creates Figma frame:
  - Sets frame name = section name
  - Sets frame size = viewport dimensions
  - Fills with screenshot image
  - Stores deco metadata in pluginData (for future sync)
        ↓
Frame appears in Figma canvas
```

---

## Part 4: Implementation Order

### Step 1: Bridge Routes in Admin Repo (`deco-sites/admin`)
1. Create `routes/api/figma/design-to-code.ts`
2. Create `routes/api/figma/sections.ts`
3. Create `routes/api/figma/section-screenshot.ts`
4. Test all 3 routes with curl/httpie against local admin dev server

### Step 2: Plugin Scaffold (this repo)
1. Create `manifest.json` with Figma plugin config
2. Create `code.ts` with selection listener + frame creation
3. Create `ui.html` shell
4. Set up build: esbuild or Bun to bundle TypeScript/React for Figma
5. Install `@figma/plugin-typings` for type safety
6. React UI with 3 tabs: Auth | Design→Code | Code→Design

### Step 3: Auth Flow
1. API key input form in Auth tab
2. Store in `figma.clientStorage`
3. Site selector dropdown (fetches user's sites from admin API)
4. Test: enter key → verify requests reach admin server authenticated

### Step 4: Design → Code Flow
1. Selection change listener → PNG export in main thread
2. Design preview + section name input in UI
3. Call `design-to-code` route → show loading state
4. Display generated code + live preview
5. "Refine" button for iterative Claude feedback loop
6. "Save" button to confirm

### Step 5: Code → Design Flow
1. Section gallery grid with lazy-loaded thumbnails
2. Screenshot fetching from `section-screenshot` route
3. Click to import → frame creation in Figma
4. `pluginData` metadata for future re-sync detection

---

## Verification Plan

| Test | Steps | Expected Result |
|------|-------|-----------------|
| Plugin loads | Install via Figma dev mode → run plugin | UI renders with 3 tabs |
| Auth works | Enter API key → select site | Authenticated API calls succeed |
| Design → Code | Select hero frame → click "Convert" | TSX file created in site's `/sections/` dir, preview renders |
| Code → Design | Open Import tab → browse sections | Gallery loads with thumbnails |
| Import to Figma | Click section → "Import" | Frame created in Figma with correct dimensions |
| Round-trip | Export design → import generated section | Visual fidelity comparison passes |

---

## Reference Files in Admin Repo (`deco-sites/admin`)

These existing files should be studied and reused when implementing the bridge routes:

| File | What to Reuse |
|------|---------------|
| `routes/api/chat.ts` | Claude AI integration pattern (MCP client, AI SDK, streaming) |
| `components/spaces/siteEditor/extensions/Deco/agents/config.ts` | Section creation conventions/system prompts (lines 151-225) |
| `actions/daemon/fs/patchFile.ts` | File write mechanism via MCP tool |
| `loaders/decopilot/listFiles.ts` | List files on a deco site |
| `loaders/daemon/fs/read.ts` | Read file contents from a deco site |
| `loaders/decopilot/getWebsiteScreenshot.ts` | Browserless Chrome screenshot capture |
| `middlewares/withAuth.ts` | API key / bearer token auth validation |
| `sdk/chatConfig.ts` | `MESH_API_KEY`, `MESH_BASE_URL`, `STUDIO_BASE_URL` config |
| `components/spaces/siteEditor/extensions/CMS/views/Preview.tsx` | Preview URL construction pattern |
| `components/ui/Embedded.tsx` | iframe-based section rendering |

---

## Tech Stack

**Plugin:**
- TypeScript + React (bundled for Figma runtime)
- `@figma/plugin-typings` for Figma API types
- esbuild or Bun for bundling
- TailwindCSS for plugin UI styling

**Admin bridge routes:**
- Deno 2.x (Fresh framework)
- `@ai-sdk/openai-compatible` + `ai` SDK for Claude integration
- Existing MCP tools via `ctx.invoke`
- Browserless Chrome for screenshots

---

## Key Technical Notes

1. **Figma plugin sandbox**: Main thread has NO network access. All HTTP calls must go through the UI thread via `postMessage` relay.

2. **Image format**: Export Figma frames as PNG at 2x scale for Claude vision input. Claude handles high-res images well and this preserves detail.

3. **Section conventions**: deco sections must export a default Preact component + `Props` interface. They use widget types like `ImageWidget`, `TextArea`, `Color` from `@deco/apps/website`. The agent config already documents these conventions.

4. **Preview rendering**: Sections are previewed by navigating to `/live/previews/{resolveType}` with props base64-encoded in the query string. The admin's `Embedded` component handles this in iframes.

5. **Screenshot capture**: Uses Browserless Chrome (`https://chrome.browserless.io`) with configurable viewport, format, and quality. Requires `BROWSERLESS_TOKEN` env var on the admin server.

6. **Auth**: The admin's `withAuth.ts` middleware supports bearer token auth via the `api_key` Supabase table. No new auth system needed.
