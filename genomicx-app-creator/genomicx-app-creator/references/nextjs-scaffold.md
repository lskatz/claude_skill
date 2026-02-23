# Next.js 15 Scaffold Reference

Template files for a GenomicX Next.js 15 application. Replace `{TOOLNAME}` (lowercase), `{ToolName}` (PascalCase), `{TOOL_DESCRIPTION}`, `{TOOL_TAGLINE}`, and `{REPO_URL}` throughout.

Use Next.js when the app needs: multi-page routing, SSR/SSG, complex state management across pages, or Tailwind CSS integration. The sole mature Next.js reference is **ronaQC**.

---

## package.json

```json
{
  "name": "{TOOLNAME}",
  "version": "0.1.0",
  "private": true,
  "description": "{TOOL_DESCRIPTION}",
  "license": "GPL-3.0-only",
  "author": "Nabil-Fareed Alikhan <nabil@happykhan.com> (https://www.happykhan.com)",
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "type-check": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage"
  },
  "dependencies": {
    "@biowasm/aioli": "^3.2.1",
    "file-saver": "^2.0.5",
    "next": "^15.1.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "react-hot-toast": "^2.4.1"
  },
  "devDependencies": {
    "@testing-library/dom": "^10.4.1",
    "@testing-library/jest-dom": "^6.6.3",
    "@testing-library/react": "^16.1.0",
    "@testing-library/user-event": "^14.5.2",
    "@types/file-saver": "^2.0.7",
    "@types/node": "^22.10.0",
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "@vitejs/plugin-react": "^4.3.4",
    "@vitest/coverage-v8": "^2.1.9",
    "autoprefixer": "^10.4.20",
    "eslint": "^9.16.0",
    "eslint-config-next": "^15.1.0",
    "jsdom": "^25.0.1",
    "postcss": "^8.4.49",
    "tailwindcss": "^3.4.16",
    "typescript": "^5.7.0",
    "vitest": "^2.1.0"
  }
}
```

**Notes:**
- Next.js apps use React 19.
- Add tool-specific dependencies (e.g. `d3`, `comlink`, `pako`) as needed.

---

## next.config.js

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  webpack: (config, { isServer }) => {
    config.experiments = {
      ...config.experiments,
      asyncWebAssembly: true,
      layers: true,
    };

    if (!isServer) {
      config.output.globalObject = 'self';
    }

    return config;
  },
};

module.exports = nextConfig;
```

---

## tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": {
      "@/*": ["./*"]
    }
  },
  "include": [
    "next-env.d.ts",
    "**/*.ts",
    "**/*.tsx",
    ".next/types/**/*.ts"
  ],
  "exclude": [
    "node_modules",
    "**/__tests__/**",
    "vitest.config.ts",
    "e2e/**"
  ]
}
```

---

## vitest.config.ts

```typescript
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: './lib/test-setup.ts',
    include: ['**/__tests__/**/*.test.{ts,tsx}'],
    coverage: {
      provider: 'v8',
      include: ['lib/**/*.ts', 'lib/**/*.tsx'],
      exclude: [
        'lib/__tests__/**',
        'lib/test-setup.ts',
        'lib/types.ts',
        // Add app-specific exclusions (e.g. 'lib/colorScales.ts')
      ],
      thresholds: {
        lines: 70,
      },
    },
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, '.'),
    },
  },
})
```

---

## .eslintrc.json

```json
{
  "extends": "next/core-web-vitals"
}
```

---

## tailwind.config.ts

```typescript
import type { Config } from 'tailwindcss'

const config: Config = {
  content: [
    './app/**/*.{js,ts,jsx,tsx,mdx}',
    './components/**/*.{js,ts,jsx,tsx,mdx}',
    './lib/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  darkMode: ['selector', '[data-theme="dark"]'],
  theme: {
    extend: {
      colors: {
        gx: {
          bg: 'var(--gx-bg)',
          'bg-alt': 'var(--gx-bg-alt)',
          surface: 'var(--gx-surface)',
          'surface-hover': 'var(--gx-surface-hover)',
          border: 'var(--gx-border)',
          text: 'var(--gx-text)',
          'text-muted': 'var(--gx-text-muted)',
          'text-inverted': 'var(--gx-text-inverted)',
          accent: 'var(--gx-accent)',
          'accent-hover': 'var(--gx-accent-hover)',
          success: 'var(--gx-success)',
          warning: 'var(--gx-warning)',
          error: 'var(--gx-error)',
          info: 'var(--gx-info)',
        },
      },
      fontFamily: {
        sans: ['Inter', 'ui-sans-serif', 'system-ui', 'sans-serif'],
        mono: ['JetBrains Mono', 'ui-monospace', 'SFMono-Regular', 'monospace'],
      },
      boxShadow: {
        card: '0 6px 18px rgba(19, 31, 63, 0.06)',
      },
    },
  },
  plugins: [],
}

export default config
```

---

## app/globals.css

Next.js apps use a trimmed token set (Tailwind handles spacing, radius, shadows, fonts).

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* ─── GenomicX Design Tokens ─── */

:root {
  --gx-bg: #f8fafc;
  --gx-bg-alt: #f1f5f9;
  --gx-surface: #ffffff;
  --gx-surface-hover: #f1f5f9;
  --gx-border: #e2e8f0;
  --gx-text: #0f172a;
  --gx-text-muted: #64748b;
  --gx-text-inverted: #ffffff;
  --gx-accent: #0d9488;
  --gx-accent-hover: #0f766e;
  --gx-success: #16a34a;
  --gx-warning: #d97706;
  --gx-error: #dc2626;
  --gx-info: #2563eb;
}

[data-theme="dark"] {
  --gx-bg: #0f172a;
  --gx-bg-alt: #1e293b;
  --gx-surface: #1e293b;
  --gx-surface-hover: #334155;
  --gx-border: #334155;
  --gx-text: #f1f5f9;
  --gx-text-muted: #94a3b8;
  --gx-text-inverted: #0f172a;
  --gx-accent: #2dd4bf;
  --gx-accent-hover: #14b8a6;
  --gx-success: #22c55e;
  --gx-warning: #fbbf24;
  --gx-error: #f87171;
  --gx-info: #60a5fa;
}

/* ─── Base ─── */

body {
  background: var(--gx-bg);
  color: var(--gx-text);
  font-family: 'Inter', ui-sans-serif, system-ui, sans-serif;
}

/* ─── Component Classes ─── */

@layer components {
  .card {
    @apply rounded-xl border border-gx-border bg-gx-surface shadow-card;
  }

  .btn-primary {
    @apply inline-flex items-center justify-center gap-2 rounded-lg px-5 py-2.5
      text-sm font-semibold bg-gx-accent text-gx-text-inverted
      hover:bg-gx-accent-hover transition-colors duration-150
      focus:outline-none focus:ring-2 focus:ring-gx-accent focus:ring-offset-2
      disabled:opacity-50 disabled:cursor-not-allowed;
  }

  .btn-secondary {
    @apply inline-flex items-center justify-center gap-2 rounded-lg px-5 py-2.5
      text-sm font-semibold border border-gx-border bg-gx-surface text-gx-text
      hover:bg-gx-surface-hover transition-colors duration-150
      focus:outline-none focus:ring-2 focus:ring-gx-accent focus:ring-offset-2
      disabled:opacity-50 disabled:cursor-not-allowed;
  }

  .input-field {
    @apply w-full rounded-lg border border-gx-border bg-gx-surface px-3 py-2
      text-sm text-gx-text placeholder:text-gx-text-muted
      focus:outline-none focus:ring-2 focus:ring-gx-accent focus:border-gx-accent
      transition-colors duration-150;
  }

  .label {
    @apply block text-sm font-medium text-gx-text mb-1;
  }

  .section-title {
    @apply text-2xl font-bold text-gx-text tracking-tight;
  }

  .progress-bar {
    @apply h-2 rounded-full bg-gx-bg-alt overflow-hidden;
  }

  .progress-bar-fill {
    @apply h-full rounded-full bg-gx-accent transition-all duration-300 ease-out;
  }

  .gx-table {
    @apply w-full text-sm;
  }

  .gx-table thead {
    @apply border-b-2 border-gx-border;
  }

  .gx-table th {
    @apply px-4 py-3 text-left text-xs font-semibold uppercase tracking-wider text-gx-text-muted;
  }

  .gx-table td {
    @apply px-4 py-3 font-mono text-sm text-gx-text;
  }

  .gx-table tbody tr {
    @apply border-b border-gx-border hover:bg-gx-surface-hover transition-colors;
  }
}

/* ─── Scrollbar ─── */

::-webkit-scrollbar {
  width: 8px;
  height: 8px;
}

::-webkit-scrollbar-track {
  background: var(--gx-bg-alt);
}

::-webkit-scrollbar-thumb {
  background: var(--gx-border);
  border-radius: 4px;
}

::-webkit-scrollbar-thumb:hover {
  background: var(--gx-text-muted);
}

/* ─── Focus Visible ─── */

*:focus-visible {
  outline: 2px solid var(--gx-accent);
  outline-offset: 2px;
}
```

---

## app/layout.tsx

```typescript
import type { Metadata } from 'next'
import { Inter, JetBrains_Mono } from 'next/font/google'
import './globals.css'
import Navigation from '@/components/Navigation'
import Footer from '@/components/Footer'
import Providers from './providers'

const inter = Inter({
  subsets: ['latin'],
  variable: '--font-sans',
  display: 'swap',
})

const jetbrains = JetBrains_Mono({
  subsets: ['latin'],
  variable: '--font-mono',
  display: 'swap',
})

export const metadata: Metadata = {
  title: {
    default: '{ToolName}',
    template: '%s | {ToolName}',
  },
  description: '{TOOL_DESCRIPTION}',
}

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en" data-theme="dark" suppressHydrationWarning>
      <head>
        <script
          dangerouslySetInnerHTML={{
            __html: `
              (function() {
                var theme = localStorage.getItem('gx-theme') || 'dark';
                document.documentElement.setAttribute('data-theme', theme);
              })();
            `,
          }}
        />
      </head>
      <body className={`${inter.variable} ${jetbrains.variable} font-sans antialiased min-h-screen flex flex-col`}>
        <a
          href="#main-content"
          className="sr-only focus:not-sr-only focus:absolute focus:top-2 focus:left-2 focus:z-[100] focus:rounded-lg focus:bg-gx-accent focus:px-4 focus:py-2 focus:text-gx-text-inverted"
        >
          Skip to content
        </a>
        <Providers>
          <Navigation />
          <main id="main-content" className="mx-auto w-full max-w-5xl flex-1 px-4 py-8">
            {children}
          </main>
          <Footer />
        </Providers>
      </body>
    </html>
  )
}
```

---

## app/providers.tsx

```typescript
'use client'

import { AppProviders } from '@/lib/context'
import { Toaster } from 'react-hot-toast'

export default function Providers({ children }: { children: React.ReactNode }) {
  return (
    <AppProviders>
      <Toaster
        position="top-right"
        toastOptions={{
          style: {
            background: 'var(--gx-surface)',
            color: 'var(--gx-text)',
            border: '1px solid var(--gx-border)',
          },
        }}
      />
      {children}
    </AppProviders>
  )
}
```

---

## app/page.tsx

```typescript
'use client'

import { useState } from 'react'
import FileDropZone from '@/components/FileDropZone'
import ProcessingProgress from '@/components/ProcessingProgress'
import Alert from '@/components/Alert'
import type { ProcessingProgress as ProgressType } from '@/lib/types'

export default function HomePage() {
  const [files, setFiles] = useState<File[]>([])
  const [progress, setProgress] = useState<ProgressType>({
    step: '',
    percent: 0,
    active: false,
  })

  return (
    <div className="space-y-6">
      <h1 className="section-title">{ToolName}</h1>

      <Alert variant="info">
        <p>{TOOL_DESCRIPTION}. All processing happens in your browser — no data leaves your machine.</p>
      </Alert>

      <div className="card p-6 space-y-4">
        <FileDropZone
          accept=".fasta,.fa,.fna,.bam"
          multiple
          onFiles={setFiles}
          label="Drop files here"
          hint="Or click to browse"
        />
      </div>

      <ProcessingProgress progress={progress} />
    </div>
  )
}
```

---

## lib/types.ts

```typescript
/** Processing progress state */
export interface ProcessingProgress {
  step: string
  percent: number
  active: boolean
}

// Add tool-specific types below
```

---

## lib/context.tsx

Context + useReducer pattern for complex cross-page state. Adapt actions and state to your tool.

```typescript
'use client'

import { createContext, useContext, useReducer, type Dispatch, type ReactNode } from 'react'

// --- State types ---

interface AppState {
  // Define your app-wide state here
  files: File[]
}

type AppAction =
  | { type: 'ADD_FILES'; files: File[] }
  | { type: 'CLEAR_FILES' }

// --- Reducer ---

function appReducer(state: AppState, action: AppAction): AppState {
  switch (action.type) {
    case 'ADD_FILES':
      return { ...state, files: [...state.files, ...action.files] }
    case 'CLEAR_FILES':
      return { ...state, files: [] }
    default:
      return state
  }
}

// --- Context ---

const AppStateContext = createContext<AppState | null>(null)
const AppDispatchContext = createContext<Dispatch<AppAction> | null>(null)

export function AppProviders({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(appReducer, { files: [] })

  return (
    <AppStateContext.Provider value={state}>
      <AppDispatchContext.Provider value={dispatch}>
        {children}
      </AppDispatchContext.Provider>
    </AppStateContext.Provider>
  )
}

// --- Custom hooks ---

export function useAppState() {
  const ctx = useContext(AppStateContext)
  if (!ctx) throw new Error('useAppState must be used within AppProviders')
  return ctx
}

export function useAppDispatch() {
  const ctx = useContext(AppDispatchContext)
  if (!ctx) throw new Error('useAppDispatch must be used within AppProviders')
  return ctx
}
```

---

## lib/pipeline.ts

```typescript
import type { ProcessingProgress } from './types'

async function createCLI() {
  const Aioli = (await import('@biowasm/aioli')).default
  return new Aioli(['samtools/1.10'])  // Adjust tools as needed
}

export async function runPipeline(
  file: File,
  onProgress: (p: ProcessingProgress) => void,
): Promise<void> {
  onProgress({ step: 'Initializing...', percent: 5, active: true })

  const CLI = await createCLI()
  const mountedFiles = await CLI.mount([file])
  const mountedPath = mountedFiles[0]

  onProgress({ step: 'Processing...', percent: 30, active: true })

  // TODO: Implement tool-specific pipeline

  void mountedPath

  onProgress({ step: 'Complete', percent: 100, active: false })
}
```

---

## lib/test-setup.ts

```typescript
import '@testing-library/jest-dom'
```

---

## components/Navigation.tsx

Responsive navigation with mobile hamburger menu, pill-style active links, and theme toggle.

```typescript
'use client'

import Link from 'next/link'
import { usePathname } from 'next/navigation'
import { useState, useEffect } from 'react'

type Theme = 'light' | 'dark'

const navLinks = [
  { name: 'Home', href: '/' },
  { name: 'Help', href: '/help' },
]

export default function Navigation() {
  const pathname = usePathname()
  const [mobileOpen, setMobileOpen] = useState(false)
  const [theme, setTheme] = useState<Theme>('dark')

  useEffect(() => {
    const stored = localStorage.getItem('gx-theme') as Theme | null
    if (stored) setTheme(stored)
  }, [])

  const toggleTheme = () => {
    const next = theme === 'light' ? 'dark' : 'light'
    setTheme(next)
    document.documentElement.setAttribute('data-theme', next)
    localStorage.setItem('gx-theme', next)
  }

  return (
    <header className="sticky top-0 z-50 border-b border-gx-border bg-gx-surface/90 backdrop-blur">
      <nav className="mx-auto flex max-w-5xl items-center justify-between px-4 py-3" aria-label="Main navigation">
        <Link href="/" className="text-xl font-bold text-gx-accent hover:text-gx-accent-hover transition-colors">
          {ToolName}
        </Link>

        {/* Desktop nav */}
        <div className="hidden items-center gap-1 md:flex">
          {navLinks.map((link) => {
            const isActive = pathname === link.href
            return (
              <Link
                key={link.name}
                href={link.href}
                className={`rounded-lg px-3 py-2 text-sm font-medium transition-colors ${
                  isActive
                    ? 'bg-gx-accent text-gx-text-inverted'
                    : 'text-gx-text-muted hover:text-gx-text hover:bg-gx-surface-hover'
                }`}
              >
                {link.name}
              </Link>
            )
          })}
          <button
            onClick={toggleTheme}
            className="ml-2 rounded-lg p-2 text-gx-text-muted hover:bg-gx-surface-hover transition-colors"
            aria-label="Toggle theme"
          >
            {theme === 'light' ? '\u263E' : '\u2600'}
          </button>
        </div>

        {/* Mobile hamburger */}
        <div className="flex items-center gap-2 md:hidden">
          <button
            onClick={toggleTheme}
            className="rounded-lg p-2 text-gx-text-muted hover:bg-gx-surface-hover transition-colors"
            aria-label="Toggle theme"
          >
            {theme === 'light' ? '\u263E' : '\u2600'}
          </button>
          <button
            onClick={() => setMobileOpen(!mobileOpen)}
            className="rounded-lg p-2 text-gx-text-muted hover:bg-gx-surface-hover transition-colors"
            aria-label={mobileOpen ? 'Close menu' : 'Open menu'}
            aria-expanded={mobileOpen}
          >
            {mobileOpen ? (
              <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" aria-hidden="true">
                <line x1="18" y1="6" x2="6" y2="18" />
                <line x1="6" y1="6" x2="18" y2="18" />
              </svg>
            ) : (
              <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" aria-hidden="true">
                <line x1="3" y1="6" x2="21" y2="6" />
                <line x1="3" y1="12" x2="21" y2="12" />
                <line x1="3" y1="18" x2="21" y2="18" />
              </svg>
            )}
          </button>
        </div>
      </nav>

      {/* Mobile menu */}
      {mobileOpen && (
        <div className="border-t border-gx-border px-4 pb-4 md:hidden">
          {navLinks.map((link) => {
            const isActive = pathname === link.href
            return (
              <Link
                key={link.name}
                href={link.href}
                onClick={() => setMobileOpen(false)}
                className={`block rounded-lg px-3 py-2 text-sm font-medium transition-colors ${
                  isActive
                    ? 'bg-gx-accent text-gx-text-inverted'
                    : 'text-gx-text-muted hover:text-gx-text hover:bg-gx-surface-hover'
                }`}
              >
                {link.name}
              </Link>
            )
          })}
        </div>
      )}
    </header>
  )
}
```

---

## components/Footer.tsx

```typescript
export default function Footer() {
  return (
    <footer className="mt-auto border-t border-gx-border py-6">
      <div className="mx-auto flex max-w-5xl items-center justify-between px-4 text-xs text-gx-text-muted">
        <span>GenomicX &mdash; open-source bioinformatics for the browser</span>
        <div className="flex gap-4">
          <a href="https://github.com/genomicx" target="_blank" rel="noopener noreferrer" className="hover:text-gx-text">GitHub</a>
          <a href="https://genomicx.vercel.app/about" target="_blank" rel="noopener noreferrer" className="hover:text-gx-text">Mission</a>
          <a href="https://www.happykhan.com/" target="_blank" rel="noopener noreferrer" className="hover:text-gx-text">Nabil-Fareed Alikhan</a>
        </div>
      </div>
    </footer>
  )
}
```

---

## components/FileDropZone.tsx

Accessible drag-and-drop with visual feedback, keyboard support, and input reset.

```typescript
'use client'

import { useCallback, useRef, useState, type DragEvent, type ChangeEvent } from 'react'

interface FileDropZoneProps {
  accept?: string
  multiple?: boolean
  onFiles: (files: File[]) => void
  label: string
  hint?: string
  disabled?: boolean
}

export default function FileDropZone({
  accept,
  multiple = false,
  onFiles,
  label,
  hint,
  disabled = false,
}: FileDropZoneProps) {
  const [isDragOver, setIsDragOver] = useState(false)
  const inputRef = useRef<HTMLInputElement>(null)

  const handleDragOver = useCallback(
    (e: DragEvent) => {
      e.preventDefault()
      if (!disabled) setIsDragOver(true)
    },
    [disabled]
  )

  const handleDragLeave = useCallback((e: DragEvent) => {
    e.preventDefault()
    setIsDragOver(false)
  }, [])

  const handleDrop = useCallback(
    (e: DragEvent) => {
      e.preventDefault()
      setIsDragOver(false)
      if (disabled) return
      const files = Array.from(e.dataTransfer.files)
      if (files.length > 0) onFiles(files)
    },
    [disabled, onFiles]
  )

  const handleChange = useCallback(
    (e: ChangeEvent<HTMLInputElement>) => {
      const files = Array.from(e.target.files || [])
      if (files.length > 0) onFiles(files)
      if (inputRef.current) inputRef.current.value = ''
    },
    [onFiles]
  )

  return (
    <div
      role="button"
      tabIndex={disabled ? -1 : 0}
      aria-label={label}
      aria-disabled={disabled}
      onDragOver={handleDragOver}
      onDragLeave={handleDragLeave}
      onDrop={handleDrop}
      onClick={() => !disabled && inputRef.current?.click()}
      onKeyDown={(e) => {
        if (!disabled && (e.key === 'Enter' || e.key === ' ')) {
          e.preventDefault()
          inputRef.current?.click()
        }
      }}
      className={`relative flex flex-col items-center justify-center gap-2 rounded-xl border-2 border-dashed p-8 text-center transition-colors ${
        disabled
          ? 'cursor-not-allowed border-gx-border bg-gx-bg-alt opacity-50'
          : isDragOver
            ? 'border-gx-accent bg-gx-accent/10 cursor-copy'
            : 'border-gx-border hover:border-gx-accent hover:bg-gx-accent/5 cursor-pointer'
      }`}
    >
      <svg
        width="32"
        height="32"
        viewBox="0 0 24 24"
        fill="none"
        stroke="currentColor"
        strokeWidth="1.5"
        className="text-gx-text-muted"
        aria-hidden="true"
      >
        <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4" />
        <polyline points="17,8 12,3 7,8" />
        <line x1="12" y1="3" x2="12" y2="15" />
      </svg>

      <p className="text-sm font-medium text-gx-text">{label}</p>
      {hint && <p className="text-xs text-gx-text-muted">{hint}</p>}

      <input
        ref={inputRef}
        type="file"
        accept={accept}
        multiple={multiple}
        onChange={handleChange}
        className="hidden"
        aria-hidden="true"
        tabIndex={-1}
      />
    </div>
  )
}
```

---

## components/ProcessingProgress.tsx

```typescript
'use client'

import type { ProcessingProgress as ProgressType } from '@/lib/types'

interface ProcessingProgressProps {
  progress: ProgressType
}

export default function ProcessingProgress({ progress }: ProcessingProgressProps) {
  if (!progress.active) return null

  return (
    <div className="space-y-2" aria-live="polite" aria-atomic="true">
      <div className="flex items-center gap-3">
        <svg
          className="h-5 w-5 animate-spin text-gx-accent"
          viewBox="0 0 24 24"
          fill="none"
          aria-hidden="true"
        >
          <circle
            className="opacity-25"
            cx="12"
            cy="12"
            r="10"
            stroke="currentColor"
            strokeWidth="4"
          />
          <path
            className="opacity-75"
            fill="currentColor"
            d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z"
          />
        </svg>
        <span className="text-sm font-medium text-gx-text">{progress.step}</span>
      </div>
      <div className="progress-bar">
        <div
          className="progress-bar-fill"
          style={{ width: `${progress.percent}%` }}
          role="progressbar"
          aria-valuenow={progress.percent}
          aria-valuemin={0}
          aria-valuemax={100}
          aria-label={`Processing: ${progress.percent}%`}
        />
      </div>
    </div>
  )
}
```

---

## components/Alert.tsx

Multi-variant alert with icons. Uses GX semantic colors.

```typescript
type AlertVariant = 'info' | 'warning' | 'error'

interface AlertProps {
  variant: AlertVariant
  children: React.ReactNode
}

const variantConfig: Record<AlertVariant, { bg: string; border: string; icon: string }> = {
  info: {
    bg: 'bg-gx-info/10',
    border: 'border-gx-info/30',
    icon: 'text-gx-info',
  },
  warning: {
    bg: 'bg-gx-warning/10',
    border: 'border-gx-warning/30',
    icon: 'text-gx-warning',
  },
  error: {
    bg: 'bg-gx-error/10',
    border: 'border-gx-error/30',
    icon: 'text-gx-error',
  },
}

const icons: Record<AlertVariant, React.ReactNode> = {
  info: (
    <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" aria-hidden="true">
      <circle cx="12" cy="12" r="10" />
      <line x1="12" y1="16" x2="12" y2="12" />
      <line x1="12" y1="8" x2="12.01" y2="8" />
    </svg>
  ),
  warning: (
    <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" aria-hidden="true">
      <path d="M10.29 3.86L1.82 18a2 2 0 0 0 1.71 3h16.94a2 2 0 0 0 1.71-3L13.71 3.86a2 2 0 0 0-3.42 0z" />
      <line x1="12" y1="9" x2="12" y2="13" />
      <line x1="12" y1="17" x2="12.01" y2="17" />
    </svg>
  ),
  error: (
    <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" aria-hidden="true">
      <circle cx="12" cy="12" r="10" />
      <line x1="15" y1="9" x2="9" y2="15" />
      <line x1="9" y1="9" x2="15" y2="15" />
    </svg>
  ),
}

export default function Alert({ variant, children }: AlertProps) {
  const config = variantConfig[variant]

  return (
    <div
      className={`flex gap-3 rounded-lg border p-4 ${config.bg} ${config.border}`}
      role="alert"
    >
      <span className={`flex-shrink-0 ${config.icon}`}>{icons[variant]}</span>
      <div className="text-sm text-gx-text">{children}</div>
    </div>
  )
}
```
