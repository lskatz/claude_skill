---
name: genomicx-app-creator
description: Create new GenomicX browser-based bioinformatics applications following established conventions. Use when scaffolding a new genomicx tool, creating a new bioinformatics web app, or adding a tool to the genomicx suite. Triggers on "new genomicx app", "create genomicx tool", "scaffold bioinformatics webapp", "new browser tool", or when building any planned tool (specx, genetrax, pmlstx, impressx, consensusx).
---

# GenomicX App Creator

## Architecture Overview

GenomicX apps are **client-side bioinformatics tools** running WASM-compiled tools in the browser. Privacy-first: no server-side data processing, all computation happens locally.

**Two framework options:**

| | Vite + React 18 | Next.js 15 |
|---|---|---|
| **Use when** | Single-page tool, simple state | Multi-page, SSR, complex state across routes |
| **Mature examples** | mlstx, mashtreewebx | ronaQC |
| **CSS** | Plain CSS + GX design tokens | Tailwind + GX tokens via CSS vars |
| **Testing** | vitest | vitest (with jsdom + v8 coverage) |
| **React** | 18.x | 19.x |

> **Note:** brigx is an older Next.js app that predates the GX design system. Do not use it as a reference for new apps. Use ronaQC as the sole Next.js reference.

**Standard UI flow:** FileUpload -> Processing (progress bar + log) -> Results -> Export

## Framework Decision Guide

Choose **Vite** if:
- Single analysis workflow (upload -> run -> view results)
- No server-side rendering needed
- Simple routing (tab-based, not URL-based)

Choose **Next.js** if:
- Multiple distinct pages (e.g. import, report, control, help)
- Need SSR/SSG for SEO or pre-rendering
- Want Tailwind CSS
- Complex state shared across route segments (Context + useReducer)

## Scaffolding Workflow

### Step 1: Initialize project

```bash
mkdir {toolname} && cd {toolname}
npm init -y
```

### Step 2: Choose framework and read scaffold reference

- **Vite:** Read `references/vite-scaffold.md`
- **Next.js:** Read `references/nextjs-scaffold.md`

Copy all template files from the chosen reference, replacing placeholders:
- `{TOOLNAME}` — lowercase tool name (e.g. `specx`)
- `{ToolName}` — PascalCase (e.g. `Specx`)
- `{TOOL_DESCRIPTION}` — one-sentence description
- `{TOOL_TAGLINE}` — short tagline (e.g. "Rapid Speciation & Quality Control")
- `{HOMEPAGE_URL}` — `https://{toolname}.vercel.app`
- `{REPO_URL}` — `https://github.com/genomicx/{toolname}` or `https://github.com/happykhan/{toolname}`

### Step 3: Install dependencies

```bash
npm install
```

### Step 4: Copy design tokens

- **Vite:** Copy full GX token CSS from `references/design-tokens.md` into `src/index.css`
- **Next.js:** Use the trimmed token set in `app/globals.css` (Tailwind config bridges the gap via `var()` references). See `references/nextjs-scaffold.md` for the exact globals.css template.

### Step 5: Create domain module structure

```
src/{toolname}/          # Vite
  types.ts               # Exported interfaces and defaults
  pipeline.ts            # Main pipeline orchestration

lib/                     # Next.js
  types.ts
  pipeline.ts
  context.tsx            # React context + useReducer for state
```

### Step 6: Create standard UI components

**Vite minimum set:**
- `FileUpload` — drag-and-drop file input with file list
- `LogConsole` — auto-scrolling log with copy button and line numbers
- `AboutPage` — privacy note with shield icon, author info, references

**Next.js minimum set:**
- `FileDropZone` — drag-and-drop with visual drag-over feedback, keyboard accessible
- `ProcessingProgress` — spinner + progress bar with aria attributes
- `Alert` — info/warning/error variants with icons
- `Navigation` — responsive with mobile hamburger menu, pill-style active links
- `Footer` — standard GenomicX footer

### Step 7: Wire up WASM tools

Three integration patterns:

**Pattern 1: biowasm CDN script tag** (simplest, for tools available on biowasm.com)
```html
<script src="https://biowasm.com/cdn/v3/aioli.js"></script>
```
```typescript
declare const Aioli: any
const cli = await new Aioli(['minimap2/2.22'])
```

**Pattern 2: npm import** (Next.js, avoids SSR issues)
```typescript
const Aioli = (await import('@biowasm/aioli')).default
const cli = await new Aioli(['samtools/1.10', 'ivar/1.3.1'])
```

**Pattern 3: Custom WASM in `public/wasm/`** (for tools not on biowasm)
```html
<script src="/wasm/mash.js"></script>
```
Used by mashtreewebx for the custom Mash WASM build (Emscripten MODULARIZE pattern).

### Step 8: Add to portal

Add an entry to `genomicx.github.io/apps.json` following the format in `assets/apps.json.example`. The `id` field must be **lowercase**.

## Conventions Checklist

### Language & Types
- TypeScript 5.6+ with strict mode enabled
- All domain types in `types.ts` with exported interfaces
- No `any` — use `unknown` + type guards where needed

### File Naming
- PascalCase: React components (`FileUpload.tsx`, `LogConsole.tsx`)
- camelCase: logic modules (`pipeline.ts`, `buildTree.ts`, `parseFasta.ts`)
- `.worker.ts` suffix for Web Workers (use with Comlink for compute-heavy tasks)

### State Management
- React hooks: `useState`, `useCallback`, `useRef`, `useEffect`
- No external state libraries for Vite apps
- Next.js: React context + `useReducer` for complex state, wrapped in `app/providers.tsx`

### Error Handling
- `try/catch` in pipeline functions
- User-visible error state: `const [error, setError] = useState('')`
- Vite: display in `<section className="error" role="alert">`
- Next.js: use `Alert` component with `variant="error"`

### Progress Reporting
- Callback pattern: `(msg: string, pct: number) => void`
- Vite: custom progress bar with `role="progressbar"` and `aria-valuenow`
- Next.js: `ProcessingProgress` component with spinner + progress bar
- Scale sub-task progress to overall range (e.g. 5-70% for alignment)

### Logging
- Prefixed messages: `[Module] message`
- Log callback: `(msg: string) => void`
- `LogConsole` component (Vite) with copy button, line numbers, auto-scroll

### Export
- CSV/JSON/SVG via `file-saver` library
- `saveAs(blob, filename)` pattern

### Theme
- Light/dark via `data-theme` attribute on `<html>`
- Persisted in `localStorage('gx-theme')`
- Default: `'dark'`
- Toggle button: `☾` (light mode) / `☀` (dark mode)

### Metadata
- License: `GPL-3.0-only`
- Author: `Nabil-Fareed Alikhan <nabil@happykhan.com> (https://www.happykhan.com)`
- Twitter: `@happy_khan`
- Deploy: Vercel

## package.json Conventions

```json
{
  "name": "{toolname}",
  "version": "0.1.0",
  "description": "Browser-based {tool description}",
  "type": "module",
  "license": "GPL-3.0-only",
  "author": "Nabil-Fareed Alikhan <nabil@happykhan.com> (https://www.happykhan.com)",
  "homepage": "https://{toolname}.vercel.app",
  "repository": {
    "type": "git",
    "url": "https://github.com/happykhan/{toolname}.git"
  },
  "bugs": {
    "url": "https://github.com/happykhan/{toolname}/issues"
  },
  "keywords": ["{toolname}", "bioinformatics", "genomics", "webassembly"],
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "lint": "eslint .",
    "preview": "vite preview",
    "test": "vitest run",
    "test:watch": "vitest",
    "check": "vitest run && eslint . && npm run build"
  }
}
```

For Next.js apps, replace scripts with:
```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "type-check": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage"
  }
}
```

Tools that need external database/reference data may add helper scripts (e.g. mlstx has `"fetch-db": "npx tsx scripts/fetch_pubmlst.ts"`).

## Portal Registration

Add a new entry to `/home/ubuntu/code/genomicx/genomicx.github.io/apps.json`. See `assets/apps.json.example` for the format.

**Status tracking fields:**
- `scoping` — requirements and design
- `pipeline` — core bioinformatics logic
- `webDev` — UI and frontend implementation
- `benchmarking` — validation against reference tools

Values: `"not-started"`, `"in-progress"`, `"done"`

**Icon options:** `rings`, `dna`, `tree`, `shield`, `microscope`, `zap`, `link`, `wrench`

## Resources

### references/
- `vite-scaffold.md` — Complete Vite + React template (package.json, configs, components, App.css)
- `nextjs-scaffold.md` — Complete Next.js 15 template (package.json, configs, components, providers)
- `design-tokens.md` — Full GX CSS design token system with token reference table

### assets/
- `apps.json.example` — Portal entry format for genomicx.github.io
