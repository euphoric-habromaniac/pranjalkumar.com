# Design System Specification

> Complete visual language reference: color, typography, spacing, radii, motion, components, and accessibility.

---

## Principles

1. **Token-driven.** Every visual property — color, spacing, font, radius, timing — is a CSS custom property. Components never use magic numbers.
2. **Theme-agnostic components.** Components reference tokens, not hex values. Toggle `data-theme` on `<html>` and everything updates.
3. **Zero dependencies.** No Tailwind, no Sass, no CSS-in-JS. Standard CSS with custom properties, `clamp()`, `color-mix()`, and native nesting.
4. **Accessibility-first.** Every color pair passes WCAG AA. `prefers-reduced-motion` kills all animations. Focus rings on all interactive elements.

---

## Color

### Surfaces

| Token | Dark | Light | Role |
|---|---|---|---|
| `--bg` | `#111113` | `#faf7f0` | Page background |
| `--bg-deep` | `#0a0a0c` | `#f1ede2` | Recessed areas |
| `--surface` | `#1a1a1f` | `#ffffff` | Cards, containers |
| `--surface-2` | `#202027` | `#f4f0e6` | Elevated surfaces |
| `--border` | `rgba(255,255,255,0.07)` | `rgba(24,22,30,0.10)` | Default borders |
| `--border-strong` | `rgba(255,255,255,0.14)` | `rgba(24,22,30,0.22)` | Emphasized borders |

### Accents

| Token | Dark | Light | Usage |
|---|---|---|---|
| `--mint` | `#a8d8c8` | `#1f7a5e` | Primary CTA, links, terminal prompt, success |
| `--lavender` | `#c4b5e8` | `#5a44a8` | Section labels, headings, logo dot |
| `--peach` | `#f4b8a4` | `#b3553a` | Tags, badges, errors, warnings |
| `--sky` | `#9ec8e8` | `#2a6a98` | Info states, skill tags |

### Text

| Token | Dark | Light |
|---|---|---|
| `--text` | `#f0eee8` (warm cream, not white) | `#18161e` (near-black) |
| `--text-muted` | `#8a8a8f` | `#5e5b66` |

### Semantic

| Token | Value | Purpose |
|---|---|---|
| `--link` | `var(--mint)` | All hyperlinks |
| `--focus-ring` | `var(--mint)` | Focus outlines |
| `--pulse` | `#7ec9a8` / `#1f7a5e` | Status pulse animation |

### Tag fills

Pre-mixed for easy theme overriding:

```css
--tag-peach-bg:  rgba(244, 184, 164, 0.08);
--tag-peach-bd:  rgba(244, 184, 164, 0.22);
--tag-sky-bg:    rgba(158, 200, 232, 0.08);
--tag-sky-bd:    rgba(158, 200, 232, 0.22);
--tag-mint-bg:   rgba(168, 216, 200, 0.06);
```

---

## Typography

### Families

| Token | Font | Role |
|---|---|---|
| `--font-display` | Cormorant Garamond | Display headings, hero text |
| `--font-body` | Plus Jakarta Sans | Body text, UI labels |
| `--font-mono` | JetBrains Mono | Code, terminal, metadata, tags |

### Scale

All sizes use `clamp()` for fluid responsiveness:

| Token | Value | Usage |
|---|---|---|
| `--fs-hero` | `clamp(56px, 9vw, 132px)` | Hero heading |
| `--fs-h1` | `clamp(40px, 5vw, 72px)` | Page titles |
| `--fs-h2` | `clamp(32px, 3.5vw, 48px)` | Section titles |
| `--fs-h3` | `28px` | Subsection titles |
| `--fs-lg` | `20px` | Large body text |
| `--fs-md` | `16px` | Default body |
| `--fs-sm` | `14px` | Small text |
| `--fs-xs` | `12px` | Captions, metadata |

### Line heights

| Token | Value | Usage |
|---|---|---|
| `--lh-display` | `1.1` | Headings |
| `--lh-body` | `1.7` | Body text (generous for readability) |

### Utility classes

| Class | Effect |
|---|---|
| `.display` | `font-family: var(--font-display); font-weight: 400; line-height: 1.1; letter-spacing: -0.01em` |
| `.mono` | `font-family: var(--font-mono)` |
| `.label-sc` | Small caps label: 11px, uppercase, 0.22em tracking, lavender |

---

## Spacing

4px base unit. Consistent scale throughout:

| Token | Value |
|---|---|
| `--sp-1` | 4px |
| `--sp-2` | 8px |
| `--sp-3` | 12px |
| `--sp-4` | 16px |
| `--sp-5` | 20px |
| `--sp-6` | 24px |
| `--sp-8` | 32px |
| `--sp-10` | 40px |
| `--sp-12` | 48px |
| `--sp-16` | 64px |
| `--sp-20` | 80px |
| `--sp-24` | 96px |

---

## Radii

| Token | Value | Usage |
|---|---|---|
| `--r-sm` | 4px | Small elements, tags |
| `--r-md` | 8px | Cards, containers |
| `--r-lg` | 14px | Large cards |
| `--r-pill` | 999px | Pills, badges |

---

## Motion

### Easing curves

| Token / Constant | Value | Feel |
|---|---|---|
| `--ease-out-expo` | `cubic-bezier(0.16, 1, 0.3, 1)` | Fast start, long settle |
| `--ease-in-out` | `cubic-bezier(0.65, 0, 0.35, 1)` | Symmetric |
| `EASE_EDITORIAL` | `[0.22, 1, 0.36, 1]` | Sharp entrance, slow settle |
| `EASE_SNAPPY` | `[0.16, 1, 0.3, 1]` | Fast, responsive |

### Durations

| Token | Value | Usage |
|---|---|---|
| `--dur-fast` | 160ms | Hovers, micro-interactions |
| `--dur-med` | 280ms | Theme transitions, state changes |
| `--dur-slow` | 400ms | Page transitions |

---

## Layout

| Token | Value |
|---|---|
| `--container` | 1200px |
| `--gutter` | 32px (20px below 700px) |

---

## Components

### Navigation

The navbar adapts per route:
- **Home:** Scroll links + theme toggle + terminal button
- **Sub-pages:** Back arrow + theme toggle + terminal button
- **Design System:** 10 section anchors (stacked `01\ntokens` format) + theme toggle

### Cards

- Project cards: `--surface` background, `--border`, `--r-md` radius
- Experience cards: Same base, with `--peach` period label
- Hover: translateY(-4px), shadow, border-color lightening

### Tags

- Peach tags: `--tag-peach-bg` + `--tag-peach-bd` + `--peach` text
- Sky tags: `--tag-sky-bg` + `--tag-sky-bd` + `--sky` text
- Pill shape: `--r-pill`, mono font, 10-11px

### Terminal

- Background: `--bg-deep`
- Input prompt: `~$` in `--mint`
- Output colors mapped from `TerminalLineType` to accent tokens
- Caret: block cursor, 1100ms blink

---

## Accessibility

### Non-negotiable rules

1. **WCAG AA contrast on all text.** Every text/background pair verified. No exceptions.
2. **Focus rings everywhere.** `2px solid var(--focus-ring)` with `2px` offset on all focusable elements.
3. **Reduced motion support.** `prefers-reduced-motion` kills all animations and transitions.
4. **Semantic HTML.** Proper heading hierarchy (`h1` → `h2` → `h3`). One `h1` per page.
5. **Keyboard navigation.** Tab order follows visual order. Terminal is fully keyboard-driven.
6. **No device API access.** `Permissions-Policy` disables camera, microphone, and geolocation.

### Contrast audit

See [Design Tokens — WCAG Contrast Audit](../architecture/design-tokens.md#wcag-contrast-audit) for the full table.
