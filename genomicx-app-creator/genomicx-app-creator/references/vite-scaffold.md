# Vite + React Scaffold Reference

Template files for a GenomicX Vite+React application. Replace `{TOOLNAME}` (lowercase), `{ToolName}` (PascalCase), `{TOOL_DESCRIPTION}`, `{TOOL_TAGLINE}`, `{HOMEPAGE_URL}`, and `{REPO_URL}` throughout.

---

## package.json

```json
{
  "name": "{TOOLNAME}",
  "version": "0.1.0",
  "description": "{TOOL_DESCRIPTION}",
  "type": "module",
  "license": "GPL-3.0-only",
  "author": "Nabil-Fareed Alikhan <nabil@happykhan.com> (https://www.happykhan.com)",
  "homepage": "{HOMEPAGE_URL}",
  "repository": {
    "type": "git",
    "url": "{REPO_URL}.git"
  },
  "bugs": {
    "url": "{REPO_URL}/issues"
  },
  "keywords": [
    "{TOOLNAME}",
    "bioinformatics",
    "genomics",
    "webassembly"
  ],
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "lint": "eslint .",
    "preview": "vite preview",
    "test": "vitest run",
    "test:watch": "vitest",
    "check": "vitest run && eslint . && npm run build"
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "@eslint/js": "^9.13.0",
    "@testing-library/jest-dom": "^6.9.1",
    "@testing-library/react": "^16.3.2",
    "@types/react": "^18.3.12",
    "@types/react-dom": "^18.3.1",
    "@vitejs/plugin-react": "^4.3.3",
    "eslint": "^9.13.0",
    "eslint-plugin-react-hooks": "^5.0.0",
    "eslint-plugin-react-refresh": "^0.4.14",
    "globals": "^15.11.0",
    "jsdom": "^24.1.3",
    "typescript": "~5.6.2",
    "typescript-eslint": "^8.11.0",
    "vite": "^5.4.10",
    "vitest": "^4.0.18"
  }
}
```

**Notes:**
- Add tool-specific dependencies (e.g. `@biowasm/aioli`, `patristic`, `file-saver`) as needed.
- Keep React 18.x for Vite apps (not React 19).

---

## vite.config.ts

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  build: {
    target: 'es2022',
  },
  esbuild: {
    target: 'es2022',
    keepNames: true,
  },
  optimizeDeps: {
    esbuildOptions: {
      target: 'es2022',
      keepNames: true,
    },
  },
})
```

---

## tsconfig.json

```json
{
  "files": [],
  "references": [
    { "path": "./tsconfig.app.json" },
    { "path": "./tsconfig.node.json" }
  ]
}
```

## tsconfig.app.json

```json
{
  "compilerOptions": {
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.app.tsbuildinfo",
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "Bundler",
    "allowImportingTsExtensions": true,
    "isolatedModules": true,
    "moduleDetection": "force",
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedSideEffectImports": true
  },
  "include": ["src"]
}
```

## tsconfig.node.json

```json
{
  "compilerOptions": {
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.node.tsbuildinfo",
    "target": "ES2022",
    "lib": ["ES2023"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "Bundler",
    "allowImportingTsExtensions": true,
    "isolatedModules": true,
    "moduleDetection": "force",
    "noEmit": true,
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedSideEffectImports": true
  },
  "include": ["vite.config.ts"]
}
```

---

## eslint.config.js

```javascript
import js from '@eslint/js'
import globals from 'globals'
import reactHooks from 'eslint-plugin-react-hooks'
import reactRefresh from 'eslint-plugin-react-refresh'
import tseslint from 'typescript-eslint'

export default tseslint.config(
  { ignores: ['dist', '.pixi', 'public'] },
  {
    extends: [js.configs.recommended, ...tseslint.configs.recommended],
    files: ['**/*.{ts,tsx}'],
    languageOptions: {
      ecmaVersion: 2020,
      globals: globals.browser,
    },
    plugins: {
      'react-hooks': reactHooks,
      'react-refresh': reactRefresh,
    },
    rules: {
      ...reactHooks.configs.recommended.rules,
      'react-refresh/only-export-components': [
        'warn',
        { allowConstantExport: true },
      ],
    },
  },
)
```

---

## index.html

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta name="description" content="{TOOL_DESCRIPTION}. No data leaves your machine." />
    <meta name="author" content="Nabil-Fareed Alikhan" />

    <!-- Open Graph -->
    <meta property="og:title" content="{TOOLNAME} — {TOOL_TAGLINE}" />
    <meta property="og:description" content="{TOOL_DESCRIPTION} entirely in your browser. No server required, no data uploaded." />
    <meta property="og:type" content="website" />

    <!-- Twitter Card -->
    <meta name="twitter:card" content="summary" />
    <meta name="twitter:title" content="{TOOLNAME} — {TOOL_TAGLINE}" />
    <meta name="twitter:description" content="{TOOL_DESCRIPTION} entirely in your browser." />
    <meta name="twitter:creator" content="@happy_khan" />

    <title>{TOOLNAME} — {TOOL_TAGLINE}</title>
    <link rel="preconnect" href="https://fonts.googleapis.com" />
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&family=JetBrains+Mono:wght@400;500&display=swap" rel="stylesheet" />
    <!-- Add WASM/CDN script tags here as needed -->
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

---

## src/main.tsx

```typescript
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import './index.css'
import App from './App.tsx'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>,
)
```

---

## src/index.css

Copy the full GX design token system from `references/design-tokens.md` (the "Full CSS" section for Vite apps).

---

## src/App.tsx

```typescript
import { useState, useEffect, useCallback } from 'react'
import { FileUpload } from './components/FileUpload'
import { LogConsole } from './components/LogConsole'
import { AboutPage } from './components/AboutPage'
import { run{ToolName} } from './{toolname}/pipeline'
import type { {ToolName}Result } from './{toolname}/types'
import './App.css'

type Theme = 'light' | 'dark'
type View = 'analysis' | 'about'

function App() {
  const [files, setFiles] = useState<File[]>([])
  const [running, setRunning] = useState(false)
  const [progress, setProgress] = useState('')
  const [progressPct, setProgressPct] = useState(0)
  const [error, setError] = useState('')
  const [logLines, setLogLines] = useState<string[]>([])
  const [result, setResult] = useState<{ToolName}Result | null>(null)

  const [theme, setTheme] = useState<Theme>(() => {
    return (localStorage.getItem('gx-theme') as Theme) || 'dark'
  })

  const [currentView, setCurrentView] = useState<View>('analysis')

  useEffect(() => {
    document.documentElement.setAttribute('data-theme', theme)
    localStorage.setItem('gx-theme', theme)
  }, [theme])

  const handleRun = useCallback(async () => {
    if (files.length === 0) return

    setRunning(true)
    setError('')
    setResult(null)
    setLogLines([])
    setProgress('Starting...')
    setProgressPct(0)

    try {
      const res = await run{ToolName}(
        files,
        (msg, pct) => {
          setProgress(msg)
          setProgressPct(pct)
        },
        (msg) => {
          setLogLines((prev) => [...prev, msg])
        },
      )
      setResult(res)
      setProgress('')
    } catch (err) {
      setError(err instanceof Error ? err.message : String(err))
    } finally {
      setRunning(false)
    }
  }, [files])

  const canRun = files.length > 0 && !running

  return (
    <div className="app">
      <header className="app-header">
        <div className="header-top">
          <h1>{TOOLNAME}</h1>
          <button
            className="theme-toggle"
            onClick={() =>
              setTheme((t) => (t === 'light' ? 'dark' : 'light'))
            }
            aria-label="Toggle theme"
          >
            {theme === 'light' ? '\u263E' : '\u2600'}
          </button>
        </div>
        <p className="subtitle">{TOOL_TAGLINE}</p>
        <nav className="tab-bar">
          <button
            className={`tab ${currentView === 'analysis' ? 'tab-active' : ''}`}
            onClick={() => setCurrentView('analysis')}
          >
            Analysis
          </button>
          <button
            className={`tab ${currentView === 'about' ? 'tab-active' : ''}`}
            onClick={() => setCurrentView('about')}
          >
            About
          </button>
        </nav>
      </header>

      <main className="app-main">
        {currentView === 'analysis' ? (
          <>
            <div className="controls">
              <FileUpload
                files={files}
                onFilesChange={setFiles}
                disabled={running}
              />
              {/* Add tool-specific options component here */}
              <button
                className="run-button"
                onClick={handleRun}
                disabled={!canRun}
              >
                {running ? 'Running...' : 'Run Analysis'}
              </button>
            </div>

            {running && (
              <section className="progress" aria-live="polite">
                <div
                  className="progress-bar"
                  role="progressbar"
                  aria-valuenow={Math.round(progressPct)}
                  aria-valuemin={0}
                  aria-valuemax={100}
                >
                  <div
                    className="progress-fill"
                    style={{ width: `${progressPct}%` }}
                  />
                </div>
                <p className="progress-text">{progress}</p>
              </section>
            )}

            {error && (
              <section className="error" role="alert">
                <p>{error}</p>
              </section>
            )}

            {/* Add result rendering components here */}

            {logLines.length > 0 && <LogConsole lines={logLines} />}
          </>
        ) : (
          <AboutPage />
        )}
      </main>

      <footer className="app-footer">
        <div className="footer-inner">
          <span>GenomicX &mdash; open-source bioinformatics for the browser</span>
          <div className="footer-links">
            <a href="https://github.com/genomicx" target="_blank" rel="noopener noreferrer">GitHub</a>
            <a href="https://genomicx.vercel.app/about" target="_blank" rel="noopener noreferrer">Mission</a>
            <a href="https://www.happykhan.com/" target="_blank" rel="noopener noreferrer">Nabil-Fareed Alikhan</a>
          </div>
        </div>
      </footer>
    </div>
  )
}

export default App
```

---

## src/App.css

Complete component styling for Vite apps. All sections use GX design tokens.

```css
.app {
  max-width: var(--gx-max-width);
  margin: 0 auto;
  padding: 2rem 2rem 1rem;
  min-height: 100vh;
}

/* ── Header ─────────────────────────────────── */
.app-header {
  margin-bottom: 2.5rem;
}

.header-top {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.app-header h1 {
  margin: 0;
  font-size: 2.25rem;
  font-weight: 800;
  letter-spacing: -0.03em;
  background: var(--gx-gradient);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.subtitle {
  margin: 0.3rem 0 0;
  color: var(--gx-text-muted);
  font-size: 0.95rem;
  font-weight: 400;
}

/* Theme toggle */
.theme-toggle {
  background: var(--gx-bg-alt);
  border: 1px solid var(--gx-border);
  border-radius: 10px;
  padding: 0.45rem 0.7rem;
  cursor: pointer;
  font-size: 1.15rem;
  line-height: 1;
  transition: all 0.2s;
  color: var(--gx-text);
  box-shadow: var(--gx-shadow);
}

.theme-toggle:hover {
  background: var(--gx-bg-elevated);
  border-color: var(--gx-accent);
  transform: scale(1.05);
}

/* Tab navigation */
.tab-bar {
  display: flex;
  gap: 0;
  margin-top: 1.25rem;
  border-bottom: 2px solid var(--gx-border);
}

.tab {
  padding: 0.6rem 1.5rem;
  background: none;
  border: none;
  border-bottom: 2px solid transparent;
  margin-bottom: -2px;
  cursor: pointer;
  font-family: inherit;
  font-size: 0.9rem;
  font-weight: 500;
  color: var(--gx-text-muted);
  transition: color 0.2s, border-color 0.2s;
}

.tab:hover {
  color: var(--gx-text);
}

.tab-active {
  color: var(--gx-accent);
  border-bottom-color: var(--gx-accent);
  font-weight: 600;
}

/* ── Controls ────────────────────────────────── */
.controls {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1.25rem;
  margin-bottom: 1.5rem;
}

@media (max-width: 768px) {
  .controls {
    grid-template-columns: 1fr;
  }
}

/* ── File upload ─────────────────────────────── */
.file-upload-area {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 180px;
  border: 2px dashed var(--gx-border);
  border-radius: 12px;
  padding: 2rem 1.5rem;
  text-align: center;
  cursor: pointer;
  transition: all 0.25s;
  background: var(--gx-bg-alt);
  box-shadow: var(--gx-shadow);
}

.file-upload-area:hover {
  border-color: var(--gx-accent);
  background: var(--gx-accent-dim);
  box-shadow: var(--gx-shadow-md);
  transform: translateY(-1px);
}

.file-upload-area input[type='file'] {
  display: none;
}

.file-upload-icon {
  width: 40px;
  height: 40px;
  margin-bottom: 0.75rem;
  color: var(--gx-accent);
  opacity: 0.8;
}

.file-upload-label {
  color: var(--gx-text);
  font-size: 0.9rem;
  line-height: 1.5;
}

.file-upload-hint {
  font-size: 0.8rem;
  color: var(--gx-text-muted);
  margin-top: 0.25rem;
}

.file-list {
  list-style: none;
  padding: 0;
  margin: 0.75rem 0 0;
  display: flex;
  flex-wrap: wrap;
  gap: 0.4rem;
  justify-content: center;
  max-height: 5.5rem;
  overflow-y: auto;
}

.file-list li {
  background: var(--gx-tag-bg);
  border: 1px solid var(--gx-tag-border);
  padding: 0.2rem 0.65rem;
  border-radius: 20px;
  font-size: 0.8rem;
  font-family: 'JetBrains Mono', 'Fira Code', monospace;
  color: var(--gx-tag-text);
}

/* ── Buttons ─────────────────────────────────── */
.run-button {
  width: 100%;
  padding: 0.75rem 2rem;
  background: var(--gx-gradient);
  color: white;
  border: none;
  border-radius: 10px;
  font-family: inherit;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
  box-shadow: 0 2px 8px rgba(13, 148, 136, 0.3);
}

.run-button:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 4px 16px rgba(13, 148, 136, 0.4);
}

.run-button:active:not(:disabled) {
  transform: translateY(0);
  box-shadow: 0 1px 4px rgba(13, 148, 136, 0.3);
}

.run-button:disabled {
  background: var(--gx-accent);
  cursor: not-allowed;
  box-shadow: none;
  opacity: 0.5;
}

.export-button {
  padding: 0.45rem 1rem;
  background: var(--gx-bg-alt);
  color: var(--gx-accent);
  border: 1px solid var(--gx-accent);
  border-radius: 8px;
  font-family: inherit;
  font-size: 0.85rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
}

.export-button:hover {
  background: var(--gx-accent);
  color: white;
}

/* ── Progress ────────────────────────────────── */
.progress {
  margin-bottom: 1.5rem;
  background: var(--gx-bg-alt);
  border: 1px solid var(--gx-border);
  border-radius: 12px;
  padding: 1.25rem 1.5rem;
  box-shadow: var(--gx-shadow);
}

.progress-bar {
  width: 100%;
  height: 8px;
  background: var(--gx-bg);
  border-radius: 4px;
  overflow: hidden;
  position: relative;
}

.progress-fill {
  height: 100%;
  background: var(--gx-gradient);
  transition: width 0.4s ease-out;
  border-radius: 4px;
  position: relative;
}

.progress-fill::after {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(
    90deg,
    transparent 0%,
    rgba(255, 255, 255, 0.3) 50%,
    transparent 100%
  );
  animation: shimmer 1.5s infinite;
}

@keyframes shimmer {
  0% { transform: translateX(-100%); }
  100% { transform: translateX(100%); }
}

.progress-text {
  margin: 0.6rem 0 0;
  font-size: 0.8rem;
  color: var(--gx-text-muted);
  font-family: 'JetBrains Mono', 'Fira Code', monospace;
}

/* ── Error ───────────────────────────────────── */
.error {
  background: rgba(239, 68, 68, 0.1);
  border: 1px solid rgba(239, 68, 68, 0.2);
  border-radius: 10px;
  padding: 1rem 1.25rem;
  margin-bottom: 1.5rem;
}

.error p {
  margin: 0;
  color: var(--gx-error);
  font-size: 0.9rem;
}

/* ── Results ─────────────────────────────────── */
.results {
  background: var(--gx-bg-alt);
  border: 1px solid var(--gx-border);
  border-radius: 12px;
  padding: 1.5rem;
  box-shadow: var(--gx-shadow);
}

.results-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 1rem;
}

.results-header h2 {
  margin: 0;
  font-size: 1.2rem;
  font-weight: 700;
  color: var(--gx-text);
}

.results-actions {
  display: flex;
  gap: 0.5rem;
  align-items: center;
}

.results-table-container {
  overflow-x: auto;
  border-radius: 8px;
  border: 1px solid var(--gx-border);
}

.results-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.85rem;
}

.results-table th,
.results-table td {
  padding: 0.55rem 0.75rem;
  text-align: left;
  white-space: nowrap;
}

.results-table th {
  background: var(--gx-bg-elevated);
  font-weight: 600;
  font-size: 0.8rem;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  color: var(--gx-text-muted);
  border-bottom: 2px solid var(--gx-border);
  position: sticky;
  top: 0;
}

.results-table tbody tr {
  border-bottom: 1px solid var(--gx-border);
  transition: background 0.15s;
}

.results-table tbody tr:last-child {
  border-bottom: none;
}

.results-table tbody tr:hover {
  background: var(--gx-bg-elevated);
}

/* ── Log console ─────────────────────────────── */
.log-console {
  background: var(--gx-bg-alt);
  border: 1px solid var(--gx-border);
  border-radius: 12px;
  padding: 1.5rem;
  margin-top: 1.5rem;
  box-shadow: var(--gx-shadow);
}

.log-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 0.75rem;
}

.log-header h2 {
  margin: 0;
  font-size: 1.2rem;
  font-weight: 700;
  color: var(--gx-text);
}

.log-header-right {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.log-count {
  font-size: 0.8rem;
  color: var(--gx-text-muted);
  font-family: 'JetBrains Mono', 'Fira Code', monospace;
}

.copy-log-button {
  padding: 0.3rem 0.7rem;
  font-size: 0.75rem;
  font-weight: 600;
  border: 1px solid var(--gx-border);
  border-radius: 6px;
  background: var(--gx-bg-alt);
  color: var(--gx-text-muted);
  cursor: pointer;
  transition: all 0.2s ease;
}

.copy-log-button:hover {
  background: var(--gx-accent);
  color: white;
  border-color: var(--gx-accent);
}

.log-body {
  max-height: 200px;
  overflow-y: auto;
  background: var(--gx-bg-elevated);
  border: 1px solid var(--gx-border);
  border-radius: 8px;
  padding: 0.75rem;
  font-family: 'JetBrains Mono', 'Fira Code', monospace;
  font-size: 0.78rem;
  line-height: 1.6;
}

.log-line {
  color: var(--gx-text);
  white-space: pre-wrap;
  word-break: break-all;
}

.log-index {
  color: var(--gx-text-muted);
  margin-right: 0.75rem;
  user-select: none;
}

/* ── About page ──────────────────────────────── */
.about-page {
  max-width: 700px;
}

.about-page section {
  background: var(--gx-bg-alt);
  border: 1px solid var(--gx-border);
  border-radius: 12px;
  padding: 1.75rem;
  margin-bottom: 1.5rem;
  box-shadow: var(--gx-shadow);
}

.about-page h2 {
  margin: 0 0 0.75rem;
  font-size: 1.2rem;
  font-weight: 700;
  color: var(--gx-text);
}

.about-page h3 {
  margin: 0 0 0.25rem;
  font-size: 1.1rem;
  font-weight: 600;
  color: var(--gx-text);
}

.about-page p {
  margin: 0 0 0.75rem;
  color: var(--gx-text);
  line-height: 1.7;
}

.about-role {
  color: var(--gx-text-muted) !important;
  font-size: 0.95rem;
}

.about-links {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-top: 1rem;
}

.about-links a {
  display: inline-flex;
  align-items: center;
  gap: 0.4rem;
  color: var(--gx-accent);
  text-decoration: none;
  font-size: 0.85rem;
  font-weight: 500;
  padding: 0.35rem 0.85rem;
  border: 1px solid var(--gx-border);
  border-radius: 20px;
  background: var(--gx-bg-elevated);
  transition: all 0.2s;
}

.about-links a:hover {
  color: white;
  background: var(--gx-accent);
  border-color: var(--gx-accent);
}

.privacy-note {
  display: flex;
  align-items: flex-start;
  gap: 0.75rem;
  background: var(--gx-accent-dim);
  border: 1px solid var(--gx-tag-border);
  border-radius: 10px;
  padding: 1rem 1.25rem;
  margin-top: 0.5rem;
}

.privacy-note svg {
  flex-shrink: 0;
  width: 20px;
  height: 20px;
  color: var(--gx-accent);
  margin-top: 0.1rem;
}

.privacy-note p {
  margin: 0;
  font-size: 0.9rem;
  color: var(--gx-text);
}

/* ── Footer ──────────────────────────────────── */
.app-footer {
  margin-top: auto;
  border-top: 1px solid var(--gx-border);
  padding: 2rem;
  font-size: 0.8rem;
  color: var(--gx-text-muted);
}

.footer-inner {
  max-width: var(--gx-max-width);
  margin: 0 auto;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.footer-links {
  display: flex;
  gap: 1.5rem;
}

.footer-links a {
  color: var(--gx-text-muted);
  font-size: 0.8rem;
}

.footer-links a:hover {
  color: var(--gx-accent);
}
```

Add tool-specific sections as needed (e.g. options panels, tree containers, distance matrix styling).

---

## src/components/FileUpload.tsx

```typescript
import { useCallback } from 'react'

interface FileUploadProps {
  files: File[]
  onFilesChange: (files: File[]) => void
  disabled: boolean
}

export function FileUpload({ files, onFilesChange, disabled }: FileUploadProps) {
  const handleChange = useCallback(
    (e: React.ChangeEvent<HTMLInputElement>) => {
      if (e.target.files) {
        onFilesChange(Array.from(e.target.files))
      }
    },
    [onFilesChange],
  )

  const handleDrop = useCallback(
    (e: React.DragEvent) => {
      e.preventDefault()
      if (e.dataTransfer.files) {
        const accepted = Array.from(e.dataTransfer.files).filter((f) => {
          const name = f.name.replace(/\.gz$/, '')
          return (
            name.endsWith('.fasta') ||
            name.endsWith('.fa') ||
            name.endsWith('.fna') ||
            name.endsWith('.fsa')
          )
        })
        if (accepted.length > 0) {
          onFilesChange(accepted)
        }
      }
    },
    [onFilesChange],
  )

  const handleDragOver = useCallback((e: React.DragEvent) => {
    e.preventDefault()
  }, [])

  return (
    <div
      className="file-upload"
      onDrop={handleDrop}
      onDragOver={handleDragOver}
    >
      <label className="file-upload-area">
        <input
          type="file"
          multiple
          accept=".fasta,.fa,.fna,.fsa,.fasta.gz,.fa.gz,.fna.gz,.fsa.gz,.gz"
          onChange={handleChange}
          disabled={disabled}
          aria-label="Upload genome files"
        />
        <svg
          className="file-upload-icon"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          strokeWidth="1.5"
          strokeLinecap="round"
          strokeLinejoin="round"
        >
          <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4" />
          <polyline points="17 8 12 3 7 8" />
          <line x1="12" y1="3" x2="12" y2="15" />
        </svg>
        {files.length === 0 ? (
          <>
            <div className="file-upload-label">
              Drop files here or click to browse
            </div>
            <div className="file-upload-hint">.fasta, .fa, .fna, .fsa, .gz</div>
          </>
        ) : (
          <>
            <div className="file-upload-label">
              {files.length} file(s) selected
            </div>
            <ul className="file-list">
              {files.map((f) => (
                <li key={f.name}>{f.name}</li>
              ))}
            </ul>
          </>
        )}
      </label>
    </div>
  )
}
```

**Notes:** Adjust `accept` attribute and drop filter for the tool's specific file types.

---

## src/components/LogConsole.tsx

```typescript
import { useRef, useEffect, useCallback, useState } from 'react'

interface LogConsoleProps {
  lines: string[]
}

export function LogConsole({ lines }: LogConsoleProps) {
  const bottomRef = useRef<HTMLDivElement>(null)
  const [copied, setCopied] = useState(false)

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: 'smooth' })
  }, [lines])

  const handleCopy = useCallback(() => {
    const text = lines.join('\n')
    navigator.clipboard.writeText(text).then(() => {
      setCopied(true)
      setTimeout(() => setCopied(false), 2000)
    })
  }, [lines])

  if (lines.length === 0) return null

  return (
    <section className="log-console">
      <div className="log-header">
        <h2>Log</h2>
        <div className="log-header-right">
          <span className="log-count">{lines.length} entries</span>
          <button className="copy-log-button" onClick={handleCopy}>
            {copied ? 'Copied!' : 'Copy Log'}
          </button>
        </div>
      </div>
      <div className="log-body">
        {lines.map((line, i) => (
          <div key={i} className="log-line">
            <span className="log-index">{String(i + 1).padStart(3, ' ')}</span>
            {line}
          </div>
        ))}
        <div ref={bottomRef} />
      </div>
    </section>
  )
}
```

---

## src/components/AboutPage.tsx

```typescript
export function AboutPage() {
  return (
    <div className="about-page">
      <section>
        <h2>About {TOOLNAME}</h2>
        <p>
          {TOOL_DESCRIPTION}
        </p>
        <div className="privacy-note">
          <svg
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            strokeWidth="2"
            strokeLinecap="round"
            strokeLinejoin="round"
          >
            <path d="M12 22s8-4 8-10V5l-8-3-8 3v7c0 6 8 10 8 10z" />
          </svg>
          <p>
            No data leaves your machine — all processing happens client-side
            using WebAssembly.
          </p>
        </div>
      </section>

      <section>
        <h2>References</h2>
        <p>
          {/* Add tool-specific citations here */}
        </p>
      </section>

      <section>
        <h2>About the Author</h2>
        <h3>Nabil-Fareed Alikhan</h3>
        <p className="about-role">
          Senior Bioinformatician, Centre for Genomic Pathogen Surveillance,
          University of Oxford
        </p>
        <p>
          Bioinformatics researcher and software developer specialising in
          microbial genomics. I build widely used open-source tools, publish
          peer-reviewed research, and co-host the MicroBinfie podcast.
        </p>
        <div className="about-links">
          <a href="https://www.happykhan.com" target="_blank" rel="noopener noreferrer">
            happykhan.com
          </a>
          <a href="https://orcid.org/0000-0002-1243-0767" target="_blank" rel="noopener noreferrer">
            ORCID: 0000-0002-1243-0767
          </a>
          <a href="mailto:nabil@happykhan.com">nabil@happykhan.com</a>
          <a href="https://twitter.com/happy_khan" target="_blank" rel="noopener noreferrer">
            Twitter: @happy_khan
          </a>
          <a href="https://mstdn.science/@happykhan" target="_blank" rel="noopener noreferrer">
            Mastodon: @happykhan@mstdn.science
          </a>
        </div>
      </section>
    </div>
  )
}
```

---

## src/{toolname}/types.ts

```typescript
/** Result from the {TOOLNAME} pipeline */
export interface {ToolName}Result {
  // Define tool-specific result fields
}

/** Pipeline options */
export interface {ToolName}Options {
  // Define tool-specific option fields
}

export const DEFAULT_OPTIONS: {ToolName}Options = {
  // Set defaults
}
```

---

## src/{toolname}/pipeline.ts

```typescript
import type { {ToolName}Result } from './types'

type ProgressCallback = (msg: string, pct: number) => void
type LogCallback = (msg: string) => void

/**
 * Run the full {TOOLNAME} pipeline.
 */
export async function run{ToolName}(
  inputFiles: File[],
  onProgress: ProgressCallback,
  onLog: LogCallback,
): Promise<{ToolName}Result> {
  if (inputFiles.length === 0) {
    throw new Error('At least 1 file is required')
  }

  onProgress('Reading files...', 5)
  onLog('[{ToolName}] Starting pipeline...')

  // Step 1: Read files
  for (let i = 0; i < inputFiles.length; i++) {
    const f = inputFiles[i]
    onLog(`[{ToolName}] Reading ${f.name}...`)
    onProgress(
      `Reading files... (${i + 1}/${inputFiles.length})`,
      5 + (i / inputFiles.length) * 20,
    )
  }

  // Step 2: Process with WASM tool
  onProgress('Processing...', 30)
  onLog('[{ToolName}] Running analysis...')

  // TODO: Implement tool-specific pipeline steps

  onProgress('Done!', 100)
  onLog('[{ToolName}] Pipeline complete.')

  return {
    // Return results
  } as {ToolName}Result
}
```
