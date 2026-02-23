# GenomicX CSS Design Token System

The complete `--gx-*` CSS custom property system used across all GenomicX apps.

- **Vite apps** use the full token set below in `src/index.css` (includes reset, spacing, radius, shadows, fonts, gradients).
- **Next.js apps** use a trimmed token set in `app/globals.css` (Tailwind handles spacing/radius/shadows/fonts). See `references/nextjs-scaffold.md` for the exact Next.js globals.css.

---

## Full CSS (for Vite apps — copy into src/index.css)

```css
/* GenomicX Design Tokens */

*,
*::before,
*::after {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

/* --- LIGHT THEME (default) + shared tokens --- */
:root {
  /* Shared (theme-independent) */
  --gx-indigo:       #6366F1;
  --gx-success:      #16a34a;
  --gx-warning:      #F59E0B;
  --gx-error:        #EF4444;
  --gx-info:         #2563eb;

  --gx-radius:       8px;
  --gx-radius-lg:    12px;
  --gx-radius-sm:    4px;
  --gx-radius-pill:  999px;
  --gx-transition:   0.2s ease;
  --gx-max-width:    1080px;
  --gx-nav-height:   60px;

  --gx-font-sans: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', system-ui, sans-serif;
  --gx-font-mono: 'JetBrains Mono', 'Fira Code', monospace;

  --gx-space-1:  0.25rem;
  --gx-space-2:  0.5rem;
  --gx-space-3:  0.75rem;
  --gx-space-4:  1rem;
  --gx-space-6:  1.5rem;
  --gx-space-8:  2rem;

  /* Light theme tokens */
  --gx-bg:            #f8fafc;
  --gx-bg-alt:        #f1f5f9;
  --gx-surface:       #ffffff;
  --gx-surface-hover: #f1f5f9;
  --gx-border:        #e2e8f0;
  --gx-text:          #0f172a;
  --gx-text-muted:    #64748b;
  --gx-text-inverted: #ffffff;
  --gx-accent:        #0d9488;
  --gx-accent-hover:  #0f766e;

  --gx-bg-elevated:   var(--gx-surface);
  --gx-text-bright:   var(--gx-text);

  --gx-accent-dim:    rgba(13, 148, 136, 0.08);
  --gx-accent-15:     rgba(13, 148, 136, 0.12);
  --gx-indigo-dim:    rgba(99, 102, 241, 0.08);
  --gx-nav-bg:        rgba(255, 255, 255, 0.9);
  --gx-code-bg:       #f1f5f9;
  --gx-shadow:        0 1px 3px rgba(0, 0, 0, 0.08);
  --gx-shadow-md:     0 4px 12px rgba(0, 0, 0, 0.1);
  color-scheme: light;

  --gx-tag-bg:        #f0fdfa;
  --gx-tag-text:      #115e59;
  --gx-tag-border:    #99f6e4;
  --gx-gradient:      linear-gradient(135deg, #0d9488 0%, #06b6d4 100%);
}

/* --- DARK THEME --- */
[data-theme="dark"] {
  --gx-bg:            #0f172a;
  --gx-bg-alt:        #1e293b;
  --gx-surface:       #1e293b;
  --gx-surface-hover: #334155;
  --gx-border:        #334155;
  --gx-text:          #f1f5f9;
  --gx-text-muted:    #94a3b8;
  --gx-text-inverted: #0f172a;
  --gx-accent:        #2dd4bf;
  --gx-accent-hover:  #14b8a6;

  --gx-success:      #22c55e;
  --gx-warning:      #fbbf24;
  --gx-error:        #f87171;
  --gx-info:         #60a5fa;

  --gx-bg-elevated:   var(--gx-surface);
  --gx-text-bright:   var(--gx-text);

  --gx-accent-dim:    rgba(45, 212, 191, 0.1);
  --gx-accent-15:     rgba(45, 212, 191, 0.15);
  --gx-indigo-dim:    rgba(99, 102, 241, 0.1);
  --gx-nav-bg:        rgba(15, 23, 42, 0.9);
  --gx-code-bg:       #0f172a;
  --gx-shadow:        0 1px 3px rgba(0, 0, 0, 0.3);
  --gx-shadow-md:     0 4px 12px rgba(0, 0, 0, 0.4);
  color-scheme: dark;

  --gx-tag-bg:        #0d2d2a;
  --gx-tag-text:      #5eead4;
  --gx-tag-border:    #134e4a;
  --gx-gradient:      linear-gradient(135deg, #2dd4bf 0%, #22d3ee 100%);
}

/* --- System preference auto-detection --- */
@media (prefers-color-scheme: dark) {
  :root:not([data-theme]) {
    --gx-bg:            #0f172a;
    --gx-bg-alt:        #1e293b;
    --gx-surface:       #1e293b;
    --gx-surface-hover: #334155;
    --gx-border:        #334155;
    --gx-text:          #f1f5f9;
    --gx-text-muted:    #94a3b8;
    --gx-text-inverted: #0f172a;
    --gx-accent:        #2dd4bf;
    --gx-accent-hover:  #14b8a6;
    --gx-success:      #22c55e;
    --gx-warning:      #fbbf24;
    --gx-error:        #f87171;
    --gx-info:         #60a5fa;
    --gx-bg-elevated:   var(--gx-surface);
    --gx-text-bright:   var(--gx-text);
    --gx-accent-dim:    rgba(45, 212, 191, 0.1);
    --gx-accent-15:     rgba(45, 212, 191, 0.15);
    --gx-indigo-dim:    rgba(99, 102, 241, 0.1);
    --gx-nav-bg:        rgba(15, 23, 42, 0.9);
    --gx-code-bg:       #0f172a;
    --gx-shadow:        0 1px 3px rgba(0, 0, 0, 0.3);
    --gx-shadow-md:     0 4px 12px rgba(0, 0, 0, 0.4);
    color-scheme: dark;
    --gx-tag-bg:        #0d2d2a;
    --gx-tag-text:      #5eead4;
    --gx-tag-border:    #134e4a;
    --gx-gradient:      linear-gradient(135deg, #2dd4bf 0%, #22d3ee 100%);
  }
}

html {
  scroll-behavior: smooth;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

body {
  font-family: var(--gx-font-sans);
  background: var(--gx-bg);
  color: var(--gx-text);
  line-height: 1.7;
  font-size: 1rem;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  transition: background-color 0.3s ease, color 0.3s ease;
}

a {
  color: var(--gx-accent);
  text-decoration: none;
  transition: color var(--gx-transition);
}

a:hover {
  color: var(--gx-accent-hover);
}
```

---

## Token Reference

### Colors (theme-aware)

| Token | Light | Dark | Usage |
|-------|-------|------|-------|
| `--gx-bg` | `#f8fafc` | `#0f172a` | Page background |
| `--gx-bg-alt` | `#f1f5f9` | `#1e293b` | Alternate/secondary background |
| `--gx-surface` | `#ffffff` | `#1e293b` | Card/panel background |
| `--gx-surface-hover` | `#f1f5f9` | `#334155` | Hovered surface |
| `--gx-border` | `#e2e8f0` | `#334155` | Borders and dividers |
| `--gx-text` | `#0f172a` | `#f1f5f9` | Primary text |
| `--gx-text-muted` | `#64748b` | `#94a3b8` | Secondary/muted text |
| `--gx-text-inverted` | `#ffffff` | `#0f172a` | Text on accent backgrounds |
| `--gx-accent` | `#0d9488` | `#2dd4bf` | Primary accent (teal) |
| `--gx-accent-hover` | `#0f766e` | `#14b8a6` | Accent hover state |

### Semantic Colors (theme-aware)

| Token | Light | Dark | Usage |
|-------|-------|------|-------|
| `--gx-success` | `#16a34a` | `#22c55e` | Success states (green) |
| `--gx-warning` | `#F59E0B` | `#fbbf24` | Warning states (amber) |
| `--gx-error` | `#EF4444` | `#f87171` | Error states (red) |
| `--gx-info` | `#2563eb` | `#60a5fa` | Informational states (blue) |

### Colors (theme-independent, Vite only)

| Token | Value | Usage |
|-------|-------|-------|
| `--gx-indigo` | `#6366F1` | Secondary accent |

### Typography

| Token | Value | Usage |
|-------|-------|-------|
| `--gx-font-sans` | Inter, system stack | Body text |
| `--gx-font-mono` | JetBrains Mono, Fira Code | Code, data tables |

### Spacing (Vite only — Next.js uses Tailwind spacing)

| Token | Value |
|-------|-------|
| `--gx-space-1` | `0.25rem` (4px) |
| `--gx-space-2` | `0.5rem` (8px) |
| `--gx-space-3` | `0.75rem` (12px) |
| `--gx-space-4` | `1rem` (16px) |
| `--gx-space-6` | `1.5rem` (24px) |
| `--gx-space-8` | `2rem` (32px) |

### Border Radius (Vite only — Next.js uses Tailwind radius)

| Token | Value | Usage |
|-------|-------|-------|
| `--gx-radius-sm` | `4px` | Small elements |
| `--gx-radius` | `8px` | Default (cards, inputs) |
| `--gx-radius-lg` | `12px` | Large panels |
| `--gx-radius-pill` | `999px` | Pill buttons, tags |

### Shadows (Vite only — Next.js uses Tailwind shadows)

| Token | Light | Dark |
|-------|-------|------|
| `--gx-shadow` | `0 1px 3px rgba(0,0,0,0.08)` | `0 1px 3px rgba(0,0,0,0.3)` |
| `--gx-shadow-md` | `0 4px 12px rgba(0,0,0,0.1)` | `0 4px 12px rgba(0,0,0,0.4)` |

### Theme Switching

Theme is controlled via the `data-theme` attribute on `<html>`:

```typescript
// Read
const theme = localStorage.getItem('gx-theme') || 'dark'

// Apply
document.documentElement.setAttribute('data-theme', theme)
localStorage.setItem('gx-theme', theme)
```

System preference is auto-detected when no `data-theme` attribute is set.

### Font Loading (Vite)

```html
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&family=JetBrains+Mono:wght@400;500&display=swap" rel="stylesheet" />
```

### Font Loading (Next.js)

```typescript
import { Inter, JetBrains_Mono } from 'next/font/google'

const inter = Inter({ subsets: ['latin'], variable: '--font-sans', display: 'swap' })
const jetbrains = JetBrains_Mono({ subsets: ['latin'], variable: '--font-mono', display: 'swap' })
```
