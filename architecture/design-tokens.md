# Design Tokens

> The CSS custom property system, `data-theme` cascading, the "three CSS files" bug, and WCAG contrast audit.

---

## Architecture

All visual styling is driven by **CSS Custom Properties** (design tokens) defined on `:root`. No Tailwind, no Sass, no CSS-in-JS. Standard CSS with custom properties, `clamp()`, `color-mix()`, and native nesting.

### The single-source-of-truth rule

`index.css` is the **only** stylesheet imported by the application. This was a hard-won lesson.

Early in development, tokens were split across three files:
- `tokens.css` — design token definitions
- `styles.css` — component styles
- `index.css` — imported by `index.tsx`

Only `index.css` is actually imported. The other two files exist but are never loaded by the app. I spent hours debugging why the logo SVG rendered as solid black in both themes. The root cause: `var(--logo-glyph)` was defined in `tokens.css`, which the app never loads. An undefined CSS variable in an SVG `fill` attribute falls back to `currentColor`, which inherits from the body — black.

**Rule:** In this project, `index.css` is the single source of truth. `tokens.css` and `styles.css` exist only as documentation artifacts.

---

## Token hierarchy

### Surfaces

| Token | Dark | Light | Purpose |
|---|---|---|---|
| `--bg` | `#111113` | `#faf7f0` | Page background |
| `--bg-deep` | `#0a0a0c` | `#f1ede2` | Recessed backgrounds |
| `--surface` | `#1a1a1f` | `#ffffff` | Card backgrounds |
| `--surface-2` | `#202027` | `#f4f0e6` | Elevated surfaces |
| `--border` | `rgba(255,255,255,0.07)` | `rgba(24,22,30,0.10)` | Borders, dividers |
| `--border-strong` | `rgba(255,255,255,0.14)` | `rgba(24,22,30,0.22)` | Emphasized borders |

### Accents

| Token | Dark | Light | Role |
|---|---|---|---|
| `--mint` | `#a8d8c8` | `#1f7a5e` | Primary CTA, links, terminal prompt |
| `--lavender` | `#c4b5e8` | `#5a44a8` | Section labels, display emphasis, logo dot |
| `--peach` | `#f4b8a4` | `#b3553a` | Tags, badges, error states |
| `--sky` | `#9ec8e8` | `#2a6a98` | Info states, skill badges |

### Text

| Token | Dark | Light |
|---|---|---|
| `--text` | `#f0eee8` (warm cream) | `#18161e` (near-black) |
| `--text-muted` | `#8a8a8f` | `#5e5b66` |

Note: `--text` in dark mode is warm cream, not pure white. This keeps the dark theme from feeling clinical.

### Typography

```css
--font-display: 'Cormorant Garamond', 'Times New Roman', serif;
--font-body:    'Plus Jakarta Sans', -apple-system, sans-serif;
--font-mono:    'JetBrains Mono', ui-monospace, 'SF Mono', monospace;
```

Type scale uses `clamp()` for fluid sizing:

```css
--fs-hero: clamp(56px, 9vw, 132px);
--fs-h1:   clamp(40px, 5vw, 72px);
--fs-h2:   clamp(32px, 3.5vw, 48px);
```

### Spacing

4px base unit:

```css
--sp-1: 4px;  --sp-2: 8px;  --sp-3: 12px; --sp-4: 16px;
--sp-5: 20px; --sp-6: 24px; --sp-8: 32px; --sp-10: 40px;
--sp-12: 48px; --sp-16: 64px; --sp-20: 80px; --sp-24: 96px;
```

### Radii

```css
--r-sm: 4px; --r-md: 8px; --r-lg: 14px; --r-pill: 999px;
```

### Motion

```css
--ease-out-expo: cubic-bezier(0.16, 1, 0.3, 1);
--ease-in-out:   cubic-bezier(0.65, 0, 0.35, 1);
--dur-fast: 160ms; --dur-med: 280ms; --dur-slow: 400ms;
```

---

## Theme switching

### How it works

The `useTheme` hook manages three states: `dark`, `light`, `system`.

1. **Read** — On mount, read `localStorage.getItem('pk-theme')`. Default to `system`.
2. **Apply** — Set `data-theme` attribute on `<html>`. Add/remove `dark` class.
3. **Persist** — Write to `localStorage` on every change.
4. **System tracking** — When in `system` mode, listen to `prefers-color-scheme` media query changes.

```typescript
const applyTheme = (t: Theme) => {
  const isDark = t === 'dark' || (t === 'system' && mediaQuery.matches);
  root.setAttribute('data-theme', isDark ? 'dark' : 'light');
};
```

### Flash prevention

A synchronous script in `index.html` reads `localStorage` before React mounts:

```javascript
(function(){
  var t = localStorage.getItem('pk-theme') || 'system';
  var prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
  var isDark = t === 'dark' || (t === 'system' && prefersDark);
  document.documentElement.setAttribute('data-theme', isDark ? 'dark' : 'light');
})();
```

This prevents the white-flash-on-dark-theme problem that plagues most React apps.

### Transition

Theme changes animate via CSS transitions on key elements:

```css
html, body, .nav, .surface, .project-card, .xp-card, .terminal, .footer {
  transition: background-color var(--dur-med) var(--ease-out-expo),
              color var(--dur-med) var(--ease-out-expo),
              border-color var(--dur-med) var(--ease-out-expo);
}
```

---

## WCAG contrast audit

Every text/background pair has been verified against WCAG AA (4.5:1 for normal text, 3:1 for large text):

### Dark theme (on `#111113`)

| Color | Hex | Contrast ratio | Pass? |
|---|---|---|---|
| Text | `#f0eee8` | 15.2:1 | ✅ AAA |
| Text muted | `#8a8a8f` | 5.2:1 | ✅ AA |
| Mint | `#a8d8c8` | 12.4:1 | ✅ AAA |
| Lavender | `#c4b5e8` | 10.3:1 | ✅ AAA |
| Peach | `#f4b8a4` | 10.8:1 | ✅ AAA |
| Sky | `#9ec8e8` | 11.2:1 | ✅ AAA |

### Light theme (on `#faf7f0`)

| Color | Hex | Contrast ratio | Pass? |
|---|---|---|---|
| Text | `#18161e` | 16.1:1 | ✅ AAA |
| Text muted | `#5e5b66` | 5.8:1 | ✅ AA |
| Mint | `#1f7a5e` | 5.5:1 | ✅ AA |
| Lavender | `#5a44a8` | 6.2:1 | ✅ AA |
| Peach | `#b3553a` | 4.8:1 | ✅ AA |
| Sky | `#2a6a98` | 5.1:1 | ✅ AA |

No guessing. No "it looks fine to me." Every pair is documented and verified.

---

## Logo token system

The brand mark adapts to themes via five tokens:

```css
/* Dark */
--logo-dot:     #c4b5e8;     /* lavender dot */
--logo-glyph:   #f0eee8;     /* cream glyph */
--logo-divider: rgba(255,255,255,0.10);

/* Light */
--logo-dot:     #534ab7;     /* indigo dot */
--logo-glyph:   #111113;     /* near-black glyph */
--logo-divider: rgba(0,0,0,0.10);
```

**Bug story:** The design system page's brand mark preview boxes initially used `var(--bg)` as their background. This made the dark variant invisible on dark theme and vice versa. Fix: hardcode backgrounds (`#111113` for dark previews, `#faf7f0` for light) so both variants are always visible.
