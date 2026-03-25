# Figma ↔ deco CMS — Bidirectional Design & Code Sync

## Goal

Enable two flows between Figma and deco CMS:
1. **Design → Code**: Take a Figma design, use Claude AI to convert it to a Preact/TSX section, and save it to a deco CMS site
2. **Code → Design**: Take coded sections from deco CMS and render them visually in Figma as editable designs

---

## Architecture Options Comparison

There are **5 distinct architectural approaches**. They range from zero-code config to full custom plugin development.

---

### ⭐ Option E: Official Claude Code Figma Plugin (RECOMMENDED)

```
Claude Code CLI                Figma Remote MCP              deco Admin MCP
┌──────────────┐              ┌──────────────────┐          ┌─────────────┐
│ Claude Code  │──MCP──►      │ mcp.figma.com    │          │ patchFile   │
│ + Figma      │              │                  │          │ listFiles   │
│   Plugin     │              │ get_file         │          │ read        │
│              │              │ get_file_nodes   │          │             │
│ Skills:      │              │ get_image        │          │             │
│ /implement   │              │ get_styles       │          │             │
│ /generate    │              │ generate_design ◄├──────────│             │
│ /code-connect│              │ create_file      │          │             │
└──────────────┘              └──────────────────┘          └─────────────┘
                                     │                            │
                              Figma Remote MCP             deco Admin MCP
                              (read AND write!)            (read/write)
```

**How it works:**
- Install the **official Figma plugin for Claude Code** — made by Figma, 70k+ installs
- Uses Figma's **Remote MCP server** (`mcp.figma.com/mcp`) — no local server or desktop app needed
- **Bidirectional**: can both read designs AND push code back to Figma as editable layers
- Comes with built-in **skills** (slash commands) for common design↔code workflows
- Combined with deco admin MCP tools, enables full Design↔Code↔deco CMS pipeline

**Setup:**
```bash
# Step 1: Install the Figma plugin in Claude Code
claude plugin install figma

# Step 2: Authenticate (opens browser for Figma OAuth)
# Follow the prompts — Claude Code handles the rest

# Alternative: Add remote MCP server manually
claude mcp add --transport http figma https://mcp.figma.com/mcp
```

**Built-in Skills (slash commands):**

| Skill | What it does |
|-------|-------------|
| `/implement-design` | Reads a Figma frame URL → generates production code matching the design |
| `/create-design-system-rules` | Analyzes your codebase → generates design system conventions for consistent output |
| `/code-connect-components` | Maps Figma components to your code components via Code Connect |
| `/figma-generate-design` | **Code→Figma**: Captures your running UI → converts to editable Figma layers |
| `/figma-generate-library` | Creates Figma component library from your codebase (variables, themes) |
| `/figma-create-new-file` | Creates a new blank Figma file |

**Two-Way Communication ("Code to Canvas"):**
- Announced Feb 2026 by Figma — Claude Code can **push UIs to Figma**
- Say "Send this to Figma" → captures the live browser state → converts to editable Figma layers
- Designer can then annotate, adjust, and explore variants in Figma
- Changes in Figma can be read back and applied to code

**Pros:**
- ✅ **Fully bidirectional** — both Design→Code AND Code→Design
- ✅ **Zero custom code** — just install plugin + configure
- ✅ Official, maintained by Figma (70k+ installs)
- ✅ Built-in skills for common workflows
- ✅ Remote MCP — no Figma Desktop app required
- ✅ OAuth auth (no API key management)
- ✅ Code Connect integration for component mapping
- ✅ Works with claude.ai connector AND Claude Code CLI
- ✅ Free with Claude Pro/Max/Team/Enterprise

**Cons:**
- Developer-focused (requires Claude Code CLI)
- `generate_figma_design` tool may have inconsistent availability between claude.ai and Claude Code CLI
- Less control over the UX compared to a custom plugin
- Designers need Claude Code access to use it directly

**Best for:** Fastest path to full bidirectional integration with zero code

---

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
- Plugin can be published to Figma Community for distribution
- Works in both Figma Desktop and Web
- Designer-friendly (no CLI needed)

**Cons:**
- Most development effort (~2 weeks for full MVP)
- Must build and maintain the plugin UI, auth flow, message protocol
- Requires 3 new API routes in admin repo

**Best for:** Production-grade product with designer-friendly UX, team-wide adoption

---

### Option B: Figma Dev Mode MCP Server + Claude Code

```
Claude Code CLI                Figma MCP Server           deco Admin MCP
┌──────────┐                  ┌────────────────┐         ┌─────────────┐
│ Claude   │──MCP──►          │ get_file       │         │ patchFile   │
│ Code     │                  │ get_file_nodes │         │ listFiles   │
│ (Opus)   │──MCP──►          │ get_image      │         │ read        │
└──────────┘                  └────────────────┘         └─────────────┘
                              Figma REST API              deco Admin MCP
                              (read-only)                 (read/write)
```

**How it works:**
- Use the [Figma Dev Mode MCP server](https://developers.figma.com/docs/figma-mcp-server/) (`figma-developer-mcp`)
- Paste Figma URL in Claude Code → reads design → generates TSX → saves to deco CMS
- **Read-only** — Code→Design NOT possible

**Setup:**
```bash
claude mcp add --transport http figma https://mcp.figma.com/mcp
# Or via npx for local server:
# npx figma-developer-mcp --figma-api-key=YOUR_TOKEN
```

**Pros:** Zero code, official, stable, full design context
**Cons:** Read-only (no Code→Design), requires Dev Mode (paid), developers only

**Best for:** Quick Design→Code if you only need one direction

---

### Option C: TalkToFigma MCP (Community) — Full Read/Write via WebSocket

```
Claude Code CLI          WebSocket Bridge        Figma Desktop App
┌──────────┐            ┌───────────────┐       ┌─────────────────┐
│ Claude   │──MCP──►    │ Local WS      │◄─────►│ TalkToFigma     │
│ Code     │            │ Server        │       │ Plugin (running) │
└──────────┘            └───────────────┘       └─────────────────┘
```

**How it works:**
- Uses [TalkToFigma](https://github.com/grab/cursor-talk-to-figma-mcp) (by Grab) — Figma plugin + WebSocket bridge
- Claude can **both read AND write** designs: create frames, text, shapes, modify styles
- Combined with deco admin MCP for full bidirectional flow

**Setup:**
1. Install TalkToFigma plugin in Figma Desktop
2. Run bridge: `bunx cursor-talk-to-figma-mcp@latest`
3. Configure in Claude Code MCP config

**Pros:** Full read/write, free, open-source
**Cons:** Requires Figma Desktop open, community-maintained, basic shapes only (not full component structures), developer-only

**Best for:** Developer power-users who want full canvas control from CLI

---

### Option D: Framelink MCP (LLM-Optimized Read-Only)

```
Claude Code CLI              Framelink MCP             deco Admin MCP
┌──────────┐                ┌────────────────┐        ┌─────────────┐
│ Claude   │──MCP──►        │ get_figma_data │        │ patchFile   │
│ Code     │                │ download_images│        │ read        │
└──────────┘                └────────────────┘        └─────────────┘
```

**How it works:**
- Uses [Framelink MCP](https://github.com/GLips/Figma-Context-MCP) — strips unnecessary Figma metadata for cleaner LLM input
- Two tools: `get_figma_data` + `download_figma_images`
- Read-only, optimized for code generation accuracy

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

**Pros:** LLM-optimized output (better accuracy), lightweight, free
**Cons:** Read-only, community-maintained, no Code→Design

**Best for:** Best code generation accuracy for Design→Code

---

## Comparison Matrix

| Capability | E: Claude Figma Plugin ⭐ | A: Custom Plugin | B: Figma MCP | C: TalkToFigma | D: Framelink |
|---|---|---|---|---|---|
| **Design → Code** | ✅ + skills | ✅ Full | ✅ Full | ✅ Full | ✅ Optimized |
| **Code → Design** | ✅ Editable layers | ✅ Screenshots | ❌ | ✅ Basic shapes | ❌ |
| **Build effort** | 🟢 Zero | 🔴 High (~2 weeks) | 🟢 Zero | 🟡 Low | 🟢 Zero |
| **User** | Developers | Designers + Devs | Developers | Developers | Developers |
| **Figma Desktop required** | No (remote MCP) | No | No | **Yes** | No |
| **Write to Figma** | ✅ Editable layers | ✅ Screenshot frames | ❌ | ✅ Shapes/text | ❌ |
| **Built-in skills** | ✅ 6 skills | ❌ | ❌ | ❌ | ❌ |
| **Code Connect** | ✅ Native | ❌ | ❌ | ❌ | ❌ |
| **Design System gen** | ✅ `/generate-library` | ❌ | ❌ | ❌ | ❌ |
| **Official/Stable** | ✅ Figma official | You maintain | ✅ Figma official | ⚠️ Community | ⚠️ Community |
| **Auth** | OAuth (automatic) | API key | OAuth or token | None needed | Token |
| **Designer-friendly UI** | ❌ CLI only | ✅ Figma panel | ❌ | ❌ | ❌ |
| **Distribution** | Plugin install | Figma Community | CLI config | Local setup | CLI config |

---

## Recommended Strategy

### Phase 1: Immediate (Option E — Zero Code) ⭐

**Start here today.** Install the official Figma plugin for Claude Code:

```bash
# Install the Figma plugin
claude plugin install figma

# Authenticate (opens browser)
# Done!
```

Then you can immediately:

1. **Design → Code → deco CMS:**
   ```
   "Read the design at [Figma URL] and create a deco section from it,
    save it to sections/Hero.tsx on my-site"
   ```
   Claude will use Figma MCP to read the design + deco admin MCP to save the section.

2. **Code → Figma:**
   ```
   "Generate a Figma design from the section at sections/Hero.tsx"
   ```
   Claude uses `generate_figma_design` to push editable layers to Figma.

3. **Design System Sync:**
   ```
   "/create-design-system-rules" → generates conventions from your codebase
   "/figma-generate-library" → creates Figma component library from code
   ```

### Phase 2: Custom Plugin (Option A) — For Designer UX

Build the custom Figma plugin **only if** you need:
- Designer-friendly UI (no CLI) — point-and-click in Figma
- Section gallery with visual previews
- Approval workflow before saving to deco CMS
- Team-wide distribution via Figma Community

### Phase 3 (Optional): Power Users (Option C)

Add TalkToFigma MCP for developers who want granular canvas control (create specific shapes, modify individual nodes).

---

## Phase 1 Detailed Setup: Claude Code + Figma Plugin + deco Admin

### Step 1: Install Figma plugin in Claude Code
```bash
claude plugin install figma
# Follow OAuth prompts to authenticate with Figma
```

### Step 2: Ensure deco admin MCP is configured
The admin MCP server should already be available at your site's endpoint. Verify with:
```bash
claude mcp list
# Should show "figma" server
```

### Step 3: Use it
```bash
# Design → Code
claude "Read this Figma design https://figma.com/design/ABC/file?node-id=1-2
        and create a Preact section for my deco site 'my-site'.
        Use TailwindCSS and match the design exactly."

# Code → Figma
claude "/figma-generate-design"
# Then: "Convert sections/Hero.tsx into a Figma design"

# Design System
claude "/create-design-system-rules"
claude "/figma-generate-library"

# Component Mapping
claude "/code-connect-components"
```

### Step 4: Verify
1. Check that the generated TSX file is saved to your site's `/sections/` directory
2. Check that the Figma file has new editable layers matching your coded section
3. Verify the design system rules match your project's conventions

---

## Phase 2: Custom Plugin Architecture (if needed)

### Plugin Structure (this repo)
```
figma-deco-plugin/
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
```

### Backend Bridge Routes (in `deco-sites/admin` repo)

Three new API routes:

#### Route 1: `POST /api/figma/design-to-code`
- **Input**: Design image (base64) + Figma node metadata + target site
- **Process**: Fetch site theme → load section conventions → call Claude API → save via `patchFile`
- **Reuses**: `routes/api/chat.ts`, `agents/config.ts`, `actions/daemon/fs/patchFile.ts`

#### Route 2: `GET /api/figma/sections?site={site}`
- **Input**: Site name
- **Process**: List `/sections/*.tsx` → read source → extract Props + component name
- **Reuses**: `loaders/decopilot/listFiles.ts`, `loaders/daemon/fs/read.ts`

#### Route 3: `POST /api/figma/section-screenshot`
- **Input**: Site name, section resolveType, viewport dimensions
- **Process**: Preview URL → Browserless Chrome screenshot → return image
- **Reuses**: `loaders/decopilot/getWebsiteScreenshot.ts`

### Auth
- Reuse admin's `api_key` table + `withAuth.ts` middleware
- Store key in `figma.clientStorage` (persistent per-user)

---

## Reference Files in Admin Repo (`deco-sites/admin`)

| File | What to Reuse |
|------|---------------|
| `routes/api/chat.ts` | Claude AI integration pattern (MCP client, AI SDK) |
| `agents/config.ts` | Section creation conventions/system prompts |
| `actions/daemon/fs/patchFile.ts` | File write via MCP tool |
| `loaders/decopilot/listFiles.ts` | List files on a deco site |
| `loaders/daemon/fs/read.ts` | Read file contents |
| `loaders/decopilot/getWebsiteScreenshot.ts` | Browserless Chrome screenshots |
| `middlewares/withAuth.ts` | API key / bearer token auth |
| `sdk/chatConfig.ts` | `MESH_API_KEY`, `MESH_BASE_URL` config |

---

## Sources

- [Figma Plugin for Claude Code](https://claude.com/plugins/figma) — Official plugin page
- [Figma Connector for Claude](https://claude.com/connectors/figma) — claude.ai native connector
- [Claude Code to Figma: Code to Canvas](https://www.figma.com/blog/introducing-claude-code-to-figma/) — Figma blog (Feb 2026)
- [Claude Code + Figma Guide](https://uxplanet.org/claude-code-figma-f647facbe181) — UX Planet walkthrough
- [Anthropic Official Plugins Directory](https://github.com/anthropics/claude-plugins-official)
- [Figma MCP Server Guide](https://help.figma.com/hc/en-us/articles/32132100833559-Guide-to-the-Figma-MCP-server)
- [Figma MCP Remote Server Setup](https://developers.figma.com/docs/figma-mcp-server/remote-server-installation/)
- [TalkToFigma MCP (Grab)](https://github.com/grab/cursor-talk-to-figma-mcp)
- [Framelink Figma MCP](https://github.com/GLips/Figma-Context-MCP)
- [Figma MCP + Cursor Tutorial](https://www.builder.io/blog/cursor-figma-mcp-server)
- [Claude Code Figma Skills](https://mcpservers.org/claude-skills/figma/create-design-system-rules)
