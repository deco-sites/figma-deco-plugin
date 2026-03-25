# Figma ↔ deco CMS — Bidirectional Design & Code Sync

## Goal

Enable two flows between Figma and deco CMS:
1. **Design → Code**: Take a Figma design, use Claude AI to convert it to a Preact/TSX section, and save it to a deco CMS site
2. **Code → Design**: Browse coded sections from deco CMS and render them visually in Figma

---

## Architecture Options Comparison

There are **4 distinct architectural approaches** to build this integration. Each has different trade-offs in terms of complexity, capabilities, and user experience.

### Option A: Custom Figma Plugin + Admin Bridge Routes

```
Figma Plugin (custom)          deco Admin Server              deco Site
┌──────────┐  ┌──────────┐   ┌─────────────────┐            ┌──────────┐
│ Main     │  │ UI       │   │ /api/figma/*     │            │ /live/   │
│ Thread   │◄►│ Thread   │──►│ design-to-code   │──Claude──► │ previews │
│ (code.ts)│  │ (React)  │   │ sections         │            │          │
│          │  │          │◄──│ screenshot       │◄Browserless│          │
└──────────┘  └──────────┘   └─────────────────┘            └──────────┘
```

**How it works:**
- Build a custom Figma plugin from scratch (this repo)
- Plugin runs inside Figma with two threads: main thread (Figma API) + UI thread (network)
- Plugin exports selected frames as PNG → sends to admin server → Claude generates TSX → saves via MCP tools
- For Code→Design: admin server screenshots sections via Browserless → plugin creates Figma frames

**Pros:**
- Full control over UX and functionality
- Deep Figma integration (read/write nodes, access variables/tokens, store metadata in `pluginData`)
- No dependency on third-party MCP servers
- Plugin can be published to Figma Community for distribution
- Works in both Figma Desktop and Web

**Cons:**
- Most development effort (~2 weeks for full MVP)
- Must build and maintain the plugin UI, auth flow, message protocol
- Requires 3 new API routes in admin repo
- Users must install the plugin manually

**Best for:** Production-grade product, maximum control, team-wide adoption

---

### Option B: Figma MCP Server + Claude Code (No Plugin Needed)

```
Developer Workflow (no Figma plugin)

Claude Code CLI                Figma MCP Server           deco Admin MCP
┌──────────┐                  ┌────────────────┐         ┌─────────────┐
│ Claude   │──MCP──►          │ get_file       │         │ patchFile   │
│ Code     │                  │ get_file_nodes │         │ listFiles   │
│ (Opus)   │──MCP──►          │ get_image      │         │ read        │
│          │                  │ get_styles     │         │             │
└──────────┘                  └────────────────┘         └─────────────┘
                                     │                         │
                              Figma REST API            deco Admin MCP
                              (read-only)               (read/write)
```

**How it works:**
- Use the [official Figma MCP server](https://www.figma.com/blog/introducing-figma-mcp-server/) (`figma-developer-mcp`) — no plugin needed
- Configure it alongside the deco admin MCP server in Claude Code's MCP config
- Developer pastes a Figma frame URL in Claude Code → Claude reads the design via Figma MCP → generates TSX → saves to deco CMS via admin MCP tools
- For Code→Design: **NOT POSSIBLE** — Figma MCP is read-only, cannot create frames in Figma

**Setup:**
```json
// .claude/mcp.json
{
  "mcpServers": {
    "figma": {
      "command": "npx",
      "args": ["-y", "figma-developer-mcp", "--figma-api-key=YOUR_TOKEN"]
    }
  }
}
```

**Pros:**
- Zero code to build for the Design→Code direction
- Uses official, Figma-maintained MCP server (stable, supported)
- Developer-friendly CLI workflow (no context switching to a plugin UI)
- Gets full design context: styles, components, variants, design tokens
- Can also read FigJam and Make files
- Works with Figma's [remote MCP server](https://help.figma.com/hc/en-us/articles/32132100833559-Guide-to-the-Figma-MCP-server) (no desktop app required)

**Cons:**
- **Read-only** — cannot create or modify Figma designs (Code→Design is impossible)
- Requires Figma Dev Mode (paid feature on Professional+ plans)
- Developer-only workflow (designers can't use it without Claude Code)
- No visual plugin UI in Figma
- Large files can hit Figma REST API rate limits

**Best for:** Quick Design→Code MVP for developers, zero-build prototype

---

### Option C: TalkToFigma MCP (Community) + Claude Code — Full Read/Write

```
Developer Workflow (with Figma plugin companion)

Claude Code CLI          WebSocket Bridge        Figma Desktop App
┌──────────┐            ┌───────────────┐       ┌─────────────────┐
│ Claude   │──MCP──►    │ Local WS      │◄─────►│ TalkToFigma     │
│ Code     │            │ Server        │       │ Plugin (running) │
│ (Opus)   │            │ (port 3055)   │       │                 │
└──────────┘            └───────────────┘       └─────────────────┘
     │                                                  │
     │ also connects to                          Figma Plugin API
     │ deco Admin MCP                            (full read/write)
```

**How it works:**
- Uses the [TalkToFigma community project](https://github.com/grab/cursor-talk-to-figma-mcp) (by Grab)
- A Figma plugin runs inside Figma Desktop and communicates via WebSocket to a local bridge server
- The bridge exposes MCP tools that Claude Code can use
- Claude can **both read AND write** Figma designs: create frames, text, shapes, modify styles, layouts, colors
- Combined with deco admin MCP: full bidirectional Design↔Code flow entirely from Claude Code

**Setup:**
1. Install TalkToFigma plugin in Figma Desktop
2. Run WebSocket bridge: `bunx cursor-talk-to-figma-mcp@latest`
3. Configure in Claude Code MCP config
4. Keep Figma Desktop open with the plugin connected

**Pros:**
- **Full read/write** — can create Figma frames from code (Code→Design works!)
- No custom plugin to build (uses existing community plugin)
- All orchestration happens through Claude Code (single interface)
- Can scan text nodes, set text, modify layouts, padding, colors, corner radius
- Free, open-source, works with any Figma plan

**Cons:**
- **Requires Figma Desktop app open** with the plugin running (no web-only workflow)
- Community-maintained, not official Figma — could break with Figma updates
- WebSocket bridge adds architectural complexity
- Created frames are basic (rectangles, text, shapes) — not high-fidelity component structures
- No distribution to non-developer users (requires local setup)
- Designer-unfriendly (requires CLI knowledge)

**Best for:** Developer power-user workflow with full bidirectional capability

---

### Option D: Framelink MCP + Admin MCP (Optimized Read-Only)

```
Claude Code CLI              Framelink MCP             deco Admin MCP
┌──────────┐                ┌────────────────┐        ┌─────────────┐
│ Claude   │──MCP──►        │ get_figma_data │        │ patchFile   │
│ Code     │                │ download_images│        │ listFiles   │
│ (Opus)   │──MCP──►        │                │        │ read        │
└──────────┘                └────────────────┘        └─────────────┘
                                   │                        │
                            Figma REST API            deco Admin MCP
                            (simplified output)       (read/write)
```

**How it works:**
- Uses [Framelink's MCP server](https://github.com/GLips/Figma-Context-MCP) (`@anthropic-ai/framelink-figma-mcp`)
- Similar to Option B but **optimized for LLM consumption** — strips unnecessary metadata, provides cleaner layout/style data
- Two tools: `get_figma_data` (structure + styling) and `download_figma_images` (image assets)
- Combined with deco admin MCP for saving generated sections

**Setup:**
```json
{
  "mcpServers": {
    "framelink-figma": {
      "command": "npx",
      "args": ["-y", "@anthropic-ai/framelink-figma-mcp"],
      "env": { "FIGMA_ACCESS_TOKEN": "YOUR_TOKEN" }
    }
  }
}
```

**Pros:**
- Simplified, LLM-optimized output (better code generation accuracy)
- Open-source, free, works with any Figma plan
- Lighter weight than official Figma MCP
- Can generate design tokens (colors, typography) as JSON
- No Figma Desktop app required

**Cons:**
- **Read-only** — same limitation as Option B, no Code→Design
- Less design context than official Figma MCP (no component variants, no Code Connect)
- Community-maintained
- May not recognize custom component libraries

**Best for:** Lightweight Design→Code with better LLM accuracy

---

## Comparison Matrix

| Capability | A: Custom Plugin | B: Official Figma MCP | C: TalkToFigma MCP | D: Framelink MCP |
|---|---|---|---|---|
| **Design → Code** | ✅ Full | ✅ Full | ✅ Full | ✅ Full |
| **Code → Design** | ✅ Screenshots as frames | ❌ Not possible | ✅ Creates actual nodes | ❌ Not possible |
| **Build effort** | 🔴 High (~2 weeks) | 🟢 Zero (config only) | 🟡 Low (config + setup) | 🟢 Zero (config only) |
| **User** | Designers + Developers | Developers only | Developers only | Developers only |
| **Figma Desktop required** | No (web + desktop) | No | **Yes** | No |
| **Figma plan required** | Any | Dev Mode (paid) | Any | Any |
| **Write to Figma** | ✅ (frames from screenshots) | ❌ | ✅ (actual shapes/text) | ❌ |
| **Distribution** | Figma Community plugin | CLI config | Local setup | CLI config |
| **Design token extraction** | ✅ Via Plugin API variables | ✅ Via REST API | ✅ Limited | ✅ Optimized |
| **Maintenance** | You maintain | Figma maintains | Community | Community |
| **Visual UI in Figma** | ✅ Custom UI panel | ❌ | ❌ | ❌ |
| **Real-time design access** | ✅ Live document | ❌ Snapshot | ✅ Live document | ❌ Snapshot |
| **Preview before saving** | ✅ In-plugin iframe | ❌ | ❌ | ❌ |
| **Official/Stable** | You control | ✅ Figma official | ⚠️ Community | ⚠️ Community |

---

## Recommended Strategy: Hybrid (B or D now → A later)

### Phase 1: Immediate / Zero-Build (Options B or D)

Start with a **zero-code approach** using MCP servers with Claude Code:

1. Configure the **Figma MCP server** (official or Framelink) in your Claude Code setup
2. Configure the **deco admin MCP server** (already exists at `/mcp/messages`)
3. Use Claude Code to: paste a Figma URL → read design → generate section → save to deco CMS
4. This gives you Design→Code **today** with no code to write

**Recommended MCP config for Claude Code:**
```json
{
  "mcpServers": {
    "figma": {
      "command": "npx",
      "args": ["-y", "figma-developer-mcp", "--figma-api-key=YOUR_FIGMA_TOKEN"]
    }
  }
}
```

### Phase 2: Full Product (Option A)

Build the custom Figma plugin for:
- Designer-friendly UI (no CLI needed)
- Code→Design flow (import sections as visual frames)
- Team-wide distribution via Figma Community
- Preview and approval workflow before saving
- Iterative refinement with Claude

### Phase 2.5 (Optional): Add TalkToFigma (Option C) for power users

For developers who want full bidirectional control from Claude Code:
- Add the TalkToFigma MCP alongside the others
- Enables creating actual Figma nodes (not just screenshot frames) from code

---

## Phase 1 Implementation: MCP Config (Zero Code)

Just add this to your project's `.claude/mcp.json`:

```json
{
  "mcpServers": {
    "figma": {
      "command": "npx",
      "args": ["-y", "figma-developer-mcp", "--figma-api-key=YOUR_FIGMA_TOKEN"]
    }
  }
}
```

Then in Claude Code, you can say:
> "Read the design at https://figma.com/design/ABC123/my-file?node-id=123-456 and create a deco section from it"

Claude will:
1. Use `get_file_nodes` to read the design structure
2. Use `get_image` to export a visual reference
3. Generate a Preact/TSX section matching the design
4. Use deco admin's `patchFile` MCP tool to save it

---

## Phase 2 Implementation: Custom Plugin (Full Plan)

### Plugin Architecture

```
figma-deco-plugin/          (this repo)
  manifest.json             # Figma plugin manifest
  code.ts                   # Main thread — selection, export, frame creation
  ui.html                   # UI shell
  src/
    App.tsx                 # React app root with tab navigation
    views/
      Auth.tsx              # API key login + site selector
      DesignToCode.tsx      # Flow 1: select frame → AI → save section
      CodeToDesign.tsx      # Flow 2: browse sections → import to Figma
    api/
      decoApi.ts            # HTTP client for /api/figma/* endpoints
    types.ts                # Shared message types between threads
  package.json
  tsconfig.json
```

### Backend Bridge Routes (in `deco-sites/admin` repo)

Three new API routes:

#### Route 1: `POST /api/figma/design-to-code`
- **Input**: Design image (base64) + Figma node metadata + target site
- **Process**: Fetch site theme → load section conventions → call Claude API → save via `patchFile`
- **Reuses**: `routes/api/chat.ts` (AI pattern), `agents/config.ts` (conventions), `actions/daemon/fs/patchFile.ts`

#### Route 2: `GET /api/figma/sections?site={site}`
- **Input**: Site name
- **Process**: List `/sections/*.tsx` → read source → extract Props + component name
- **Reuses**: `loaders/decopilot/listFiles.ts`, `loaders/daemon/fs/read.ts`

#### Route 3: `POST /api/figma/section-screenshot`
- **Input**: Site name, section resolveType, viewport dimensions
- **Process**: Construct preview URL → capture via Browserless Chrome → return image
- **Reuses**: `loaders/decopilot/getWebsiteScreenshot.ts`

### Flow Details

**Design → Code:**
1. User selects a frame in Figma
2. Plugin exports as PNG (2x) + extracts node metadata (text, colors, fonts)
3. UI sends to `POST /api/figma/design-to-code`
4. Claude generates Preact/TSX section with Props interface
5. Server saves to `/sections/{Name}.tsx` via `patchFile`
6. Plugin shows rendered preview for approval
7. User can iterate (feedback → Claude refines)

**Code → Design:**
1. User opens "Import" tab → plugin calls `GET /api/figma/sections`
2. Plugin displays section gallery with lazy-loaded thumbnails
3. User clicks section → `POST /api/figma/section-screenshot`
4. Plugin creates Figma frame with screenshot as fill + deco metadata in `pluginData`

### Authentication
- Reuse admin's `api_key` table (user generates key in admin settings)
- Store in `figma.clientStorage` (persistent per-user)
- Bearer token auth via existing `withAuth.ts` middleware

---

## Reference Files in Admin Repo (`deco-sites/admin`)

| File | What to Reuse |
|------|---------------|
| `routes/api/chat.ts` | Claude AI integration pattern (MCP client, AI SDK, streaming) |
| `components/spaces/siteEditor/extensions/Deco/agents/config.ts` | Section creation conventions/system prompts |
| `actions/daemon/fs/patchFile.ts` | File write mechanism via MCP tool |
| `loaders/decopilot/listFiles.ts` | List files on a deco site |
| `loaders/daemon/fs/read.ts` | Read file contents from a deco site |
| `loaders/decopilot/getWebsiteScreenshot.ts` | Browserless Chrome screenshot capture |
| `middlewares/withAuth.ts` | API key / bearer token auth validation |
| `sdk/chatConfig.ts` | `MESH_API_KEY`, `MESH_BASE_URL` config |
| `components/spaces/siteEditor/extensions/CMS/views/Preview.tsx` | Preview URL construction |

---

## Verification Plan

| Test | Steps | Expected Result |
|------|-------|-----------------|
| Phase 1: MCP config | Add Figma MCP to `.claude/mcp.json` → paste Figma URL in Claude Code | Claude reads design and generates section code |
| Phase 2: Plugin loads | Install plugin in Figma dev mode | UI renders with Auth/Design→Code/Code→Design tabs |
| Phase 2: Auth | Enter API key → select site | Authenticated API calls succeed |
| Phase 2: Design→Code | Select hero frame → click Convert | TSX file created in site's `/sections/` |
| Phase 2: Code→Design | Browse sections → click Import | Frame created in Figma with screenshot |
| Phase 2: Round-trip | Export → import generated section | Visual fidelity comparison passes |

---

## Sources

- [Official Figma MCP Server Blog Post](https://www.figma.com/blog/introducing-figma-mcp-server/)
- [Figma MCP Server Guide](https://help.figma.com/hc/en-us/articles/32132100833559-Guide-to-the-Figma-MCP-server)
- [Figma MCP Server Developer Docs](https://developers.figma.com/docs/figma-mcp-server/)
- [TalkToFigma MCP (Grab)](https://github.com/grab/cursor-talk-to-figma-mcp)
- [Framelink Figma MCP](https://github.com/GLips/Figma-Context-MCP)
- [figma-developer-mcp on npm](https://www.npmjs.com/package/figma-developer-mcp)
- [TalkToFigma Figma Community Plugin](https://www.figma.com/community/plugin/1485687494525374295/talk-to-figma-mcp-plugin)
- [Builder.io: Design to Code with Figma MCP](https://www.builder.io/blog/figma-mcp-server)
