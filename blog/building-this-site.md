---
slug: building-this-site
title: "How I built this site from scratch."
date: 2026-04-20
excerpt: Most developer portfolios are clones of the same template. I built mine from scratch with a functional terminal emulator, a browser-native PDF engine, a documented design system, and real security hardening. Here's the full technical breakdown.
tags: [meta, frontend, react, security, design-systems, terminal]
---

Most developer portfolios are just digital business cards designed to pass an ATS scan. A Next.js template, some placeholder copy, deploy to Vercel, done. I wanted something different. I wanted a site that *is* an engineering project, one that demonstrates how I think about architecture, security, and craft, not just one that *says* I can code.

If you're a developer thinking about building your own portfolio from scratch, a recruiter trying to figure out if I actually know what I'm doing, or just someone who likes reading about how things are built, this post is the full technical breakdown. No hand-waving. No "it just works." Every design decision, every bug, every hard lesson.

---

## Why skip the templates?

When I started my first year as a CSE student, I noticed that every portfolio in my class was the same three templates rotated. They look polished. They deploy in 20 minutes. But they don't say anything about the person who "built" them.

As someone focused on **backend engineering and security research**, my natural habitat isn't a landing page. It's a shell. The first version of this site was a modified Hugo template. It felt like wearing someone else's clothes.

I decided to rebuild from scratch with three hard constraints:

1. **Zero UI libraries.** No Radix, no Shadcn, no Bootstrap, no component library of any kind. Every button, card, tag, and modal is built from raw HTML and CSS.
2. **Terminal-first.** The entire site should be navigable via a keyboard-driven terminal overlay.
3. **Proof of work.** Every feature must solve a real engineering problem: state management, performance, type safety, or security. No decorative features that only exist to look good on a portfolio.

---

## The stack

**React 19 + TypeScript + Vite.** That's it for the framework layer.

No CSS-in-JS, no Tailwind, no Sass. I built a design system centered around **CSS Custom Properties** (design tokens). Everything from spacing (`--sp-4`, `--sp-8`) to typography (`--font-display`, `--font-mono`) to color (`--mint`, `--lavender`, `--peach`, `--sky`) is driven by tokens defined in a single authoritative stylesheet: `index.css`.

This was actually a hard-won lesson. Early in development, I had tokens split across three files: `tokens.css`, `styles.css`, and `index.css`. Only `index.css` is imported by the React app through `index.tsx`. The other two files are never imported. I spent hours debugging why my logo SVG appeared as solid black in both themes before realizing the token `var(--logo-glyph)` was only defined in `tokens.css`, which the app never loads. An undefined CSS variable in an SVG `fill` attribute falls back to `currentColor`, which inherits from the body, which is black. A subtle, maddening bug.

**Lesson learned:** In this project, `index.css` is the single source of truth. `tokens.css` and `styles.css` exist only as documentation artifacts. This is now documented in the codebase and will never bite me again.

### The token system

Dark/light theming is dead simple with this architecture. I don't swap CSS classes on elements. I toggle a `data-theme` attribute on the `<html>` tag, and the tokens cascade:

```css
:root {
  --bg: #111113;
  --text: #f0eee8;
  --mint: #a8d8c8;
  --lavender: #c4b5e8;
  --peach: #f4b8a4;
  --sky: #9ec8e8;
}

[data-theme="light"] {
  --bg: #f0eee8;
  --text: #1a1a1f;
  /* ... overrides cascade automatically */
}
```

Every component references these tokens. When the theme flips, everything updates. No conditional rendering, no React state for colors, no class toggling on 40 different elements.

> The theme toggle itself uses a radial `clip-path` wipe that originates from the toggle button's position. It's about 30 lines of CSS and JS, but it makes the transition feel deliberate instead of a jarring flash.

### Typography: three families, zero system fallbacks

- **Cormorant Garamond** for display headings. A high-contrast serif that looks sharp at 80px+ and reads as editorial, not corporate.
- **Plus Jakarta Sans** for body text. Clean geometric sans, weights 400/500/600. Line-height 1.7 for readability.
- **JetBrains Mono** for code, terminal UI, labels, and metadata. The monospace workhorse.

No system fallback stacks as primary fonts. If the fonts fail to load, the site degrades, but it's a progressive enhancement tradeoff I'm comfortable with for a portfolio.

### Color: pastel and precise

Four accent colors, all tuned to pass **WCAG AA** contrast on `#111113`:

| Color | Hex | Ratio vs. `#111113` | Role |
|---|---|---|---|
| Mint | `#a8d8c8` | 12.4:1 | Primary CTA, links, terminal prompt |
| Lavender | `#c4b5e8` | 10.3:1 | Section labels, display emphasis |
| Peach | `#f4b8a4` | 10.8:1 | Tags, badges, inline code |
| Sky | `#9ec8e8` | 11.2:1 | Info states, skill badges |

The text color is `#f0eee8` (warm cream), not pure white. This keeps the dark theme from feeling clinical. The muted text is `#8a8a8f`, which still passes AA at 5.2:1.

Every pair is documented and verified in the design system's contrast audit section. No guessing. No "it looks fine to me."

---

## The terminal

The most-used feature on this site isn't the hero section. It's the terminal overlay.

This isn't a styled `<textarea>` with some regex. It's a structured command interpreter with a typed pipeline, history management, tab completion, and SPA navigation support.

### Architecture

The terminal is split across several modules:

- **`TerminalMode.tsx`** — The full-screen overlay container. Handles the open/close animation with Framer Motion's `AnimatePresence`.
- **`TerminalInput.tsx`** — The input line. Manages the blinking cursor, ghost suggestions, and keyboard event handling.
- **`TerminalOutput.tsx`** — The scrollable output buffer. Auto-scrolls to the latest line.
- **`TerminalLine.tsx`** — Individual line renderer. Maps line types (`command`, `error`, `success`, `info`, `sys`) to color tokens.
- **`useTerminal.ts`** — The hook that owns all terminal state: command history (persisted to `localStorage`), history navigation, and the submission pipeline.
- **`commands/index.ts`** — The command registry. Every command is a typed handler function.

### The command pipeline

Every command handler returns a `ProcessResult`:

```typescript
type ProcessResult =
  | TerminalLine[]
  | 'EXIT'
  | 'CLEAR'
  | `NAVIGATE:${string}`
  | Promise<TerminalLine[]>;
```

That template literal type `NAVIGATE:${string}` is one of my favorite decisions. I needed the terminal to trigger SPA navigation (e.g., typing `design` navigates to `/design`). The obvious solution is a discriminated union:

```typescript
{ type: 'navigate', to: string }
```

But that would have required refactoring every command handler and every `if/switch` on the result type. Using a tagged string kept `ProcessResult` as a flat primitive union (strings, arrays, Promises) and avoided touching 20+ files. The template literal provides enough type safety without object overhead.

In `useTerminal.ts`, the NAVIGATE handler is two lines:

```typescript
if (typeof result === 'string' && result.startsWith('NAVIGATE:')) {
  onExit();
  onNavigate(result.slice('NAVIGATE:'.length));
  return;
}
```

`onNavigate` is wired to React Router's `navigate()` function in `TerminalMode.tsx`. The terminal can now navigate to any route in the SPA.

### Tab completion

Tab completion isn't a simple prefix match on command names. The system understands argument context:

```typescript
function getArgCompletions(cmd: string, priorArgs: string[]): string[] {
  switch (cmd) {
    case 'open':
      return SOCIALS.map((s) => s.platform.toLowerCase());
    case 'projects':
      if (priorArgs.length === 0) return ['--open', '--details'];
      if (priorArgs[0] === '--open' || priorArgs[0] === '--details')
        return PROJECTS.map((p) => p.id);
      return [];
    case 'cat':
      return CAT_FILES;
    // ...
  }
}
```

Type `projects --details ` and hit Tab; it cycles through project IDs. Type `open ` and Tab; it cycles through `github`, `linkedin`, `email`. The input component shows all candidates in a hint row below the prompt, highlighting the current selection in mint.

### The commands

The registry includes 20+ commands: `help`, `about`, `projects`, `skills`, `contact`, `whoami`, `scan`, `ping`, `ls`, `cat`, `echo`, `uname`, `neofetch`, `cert`, `history`, `date`, `sudo`, `clear`, `design`, and more.

`whoami` hits the `ipapi.co` API and displays the visitor's IP, location, ISP, and timezone. `scan` simulates an Nmap-style port scan (no packets are actually sent). `sudo` always fails with a sarcastic message. `cat resume.pdf` navigates to the resume page.

### History persistence

Command history is stored in `localStorage` under the key `terminal_history`. Arrow up/down navigates the history stack. History survives page reloads and browser restarts:

```typescript
function loadHistory(): string[] {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    return raw ? JSON.parse(raw) : [];
  } catch {
    return [];
  }
}
```

The `try/catch` handles private browsing mode where `localStorage` may throw.

---

## Client-side resume generation

Most portfolios host a static PDF on S3 or Google Drive. The problem: your resume and your website drift apart. You update a job title on the site but forget to re-export the PDF.

I eliminated the drift by generating the resume PDF entirely in the browser using `@react-pdf/renderer`.

When you click "Download PDF," the site:

1. **Lazy-loads** the PDF engine. The `@react-pdf/renderer` package is ~400KB. It's split into a separate chunk via Vite's `manualChunks` config and only loaded on demand:

```typescript
const handleDownload = async () => {
  const [{ pdf }, { ResumePDFDoc }] = await Promise.all([
    import('@react-pdf/renderer'),
    import('./ResumePDF'),
  ]);
  const blob = await pdf(React.createElement(ResumePDFDoc)).toBlob();
  // ... trigger download
};
```

2. **Pulls data** from the same TypeScript files (`data/profile.ts`, `data/experience.ts`, `data/projects.ts`, `data/certs.ts`) that drive the website UI.
3. **Renders a styled PDF** in memory and triggers a browser download. No server. No S3. No manual export.

The resume and the website are **always in sync**. Update a job title in one file, and both the UI and the downloadable PDF reflect it immediately.

---

## The brand identity system

The site has a proper brand mark: a calligraphic **P** with a base serif and a lavender accent dot. It exists as two SVGs:

- **`public/logo-mark.svg`** — 78x93 primary mark. The P letterform uses an `evenodd` fill rule to create the counter (the hole in the P). Two serif lines (base and cap) are rendered as `<line>` elements. The dot is a `<circle>`.
- **`public/favicon.svg`** — 32x32 simplified version. Dark rounded-square background with the P glyph and dot scaled down.

The mark adapts to dark and light themes via five CSS tokens:

```css
--logo-dot:            #c4b5e8;  /* lavender dot, dark mode */
--logo-glyph:          #f0eee8;  /* cream glyph, dark mode */
--logo-divider:        rgba(255,255,255,0.10);

[data-theme="light"] {
  --logo-dot:    #534ab7;        /* indigo dot, light mode */
  --logo-glyph:  #111113;       /* near-black glyph, light mode */
  --logo-divider: rgba(0,0,0,0.10);
}
```

The About section displays a horizontal lockup: mark + 1px divider + italic wordmark ("pranjal."). The period character in the wordmark uses `--logo-dot`, so it's lavender in dark mode and indigo in light mode.

A bug during development: the design system page's brand mark preview boxes initially used `var(--bg)` as their background. This made the dark variant invisible on a dark theme and the light variant invisible on a light theme. The fix was to hardcode the backgrounds (`#111113` for dark previews, `#f0eee8` for light previews) so both variants are always simultaneously visible regardless of the active theme.

---

## The design system page

Most portfolios don't expose their internal design language. I decided to make mine public at `/design`, partly because it demonstrates engineering rigor, partly because it's useful to me as a development reference.

The page has 10 sections:

| # | Section | What it documents |
|---|---|---|
| 00 | Brand Mark | Primary mark, favicon, lockup, and all logo tokens |
| 01 | Tokens | Every CSS custom property with its role |
| 02 | Typography | Three font families with specimens at every scale |
| 03 | Color | Surface palette and accent swatches |
| 04 | Spacing & Radii | 4px base unit scale, border radii, and breakpoints |
| 05 | Components | Live previews of every atom: nav, buttons, tags, cards, terminal |
| 06 | Markdown | Rendered preview of the GitHub README stylesheet |
| 07 | Motion | Live animation demos + a full spec table (19 entries) |
| 08 | Contrast Audit | Every text/bg pair with its WCAG ratio |
| 09 | Accessibility | Six non-negotiable rules enforced across the site |

### Live motion demos

Section 07 includes three interactive demos rendered directly in the page:

1. **Status Pulse** — The actual `.pulse` CSS animation from the hero, running live: `scale 1 to 2.6, opacity 0.6 to 0, 2400ms, infinite`.
2. **Terminal Caret** — The block cursor blinking at `1100ms, steps(2), infinite`.
3. **Card Hover** — A project card that lifts on hover. The hover effect uses React `onMouseEnter`/`onMouseLeave` with direct style mutation instead of a CSS class. This keeps the demo self-contained without polluting the global stylesheet with demo-only styles.

### Contextual navigation

The navbar adapts to the current route:

- **Home:** Section scroll links (work, experience, skills, about, contact) + theme toggle + terminal button.
- **Blog / Resume:** A back-arrow link to the portfolio + theme toggle + terminal button.
- **Design System:** 10 stacked section anchors (number above label, both in mono) + theme toggle. No terminal button here because 10 anchors + toggle already fill the nav.

Each design system nav item uses a two-line stacked format (e.g., "01" above "tokens") to reduce horizontal width. This was necessary to fit all 10 sections + the theme toggle in a single row without overflow.

---

## Security hardening

This is a static site deployed on Cloudflare Pages. "What is there to secure?" More than you'd think.

### The README XSS vulnerability

The project detail page fetches `README.md` files from GitHub repositories and renders them with `react-markdown`. The original implementation included `rehype-raw`, a plugin that parses embedded HTML in Markdown and passes it through to the DOM.

The problem: GitHub READMEs are not trusted content. Even if I own the repos, compromise can happen through careless PR merges, a hijacked GitHub account, a malicious collaborator, or CI poisoning. With `rehype-raw` enabled, an attacker could inject:

```markdown
<img src=x onerror=alert(document.cookie)>
```

into a README, and it would execute as JavaScript on my domain. That gives them access to `localStorage`, session state, the ability to create phishing overlays, everything.

**The fix:** I removed `rehype-raw` entirely. The site doesn't need raw HTML in README rendering. Standard Markdown formatting is sufficient. This is cleaner and safer than adding a sanitization layer on top.

### DOMPurify for static content

Several components use `dangerouslySetInnerHTML` to render HTML strings from static data files (experience bullet points, about section copy). While the data is controlled by me, I wrapped every instance in `DOMPurify.sanitize()` as defense-in-depth. If the data files are ever compromised, at least the blast radius is limited.

### Content Security Policy

The production CSP is served via `vercel.json` headers:

```
default-src 'self';
script-src 'self';
style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
font-src 'self' https://fonts.gstatic.com;
img-src 'self' data: https://raw.githubusercontent.com;
connect-src 'self' https://raw.githubusercontent.com https://api.github.com;
```

Note: `script-src` does not include `'unsafe-inline'`. Even if an XSS payload slips through, the browser blocks inline script execution.

### Other headers

- `X-Frame-Options: DENY` — prevents clickjacking.
- `X-Content-Type-Options: nosniff` — prevents MIME-type sniffing.
- `Referrer-Policy: strict-origin-when-cross-origin` — limits referrer leakage.
- `Permissions-Policy: camera=(), microphone=(), geolocation=()` — disables device APIs.
- All `target="_blank"` links use `rel="noopener noreferrer"` to prevent reverse tabnabbing.

### API hardening

The `whoami` and `scan` commands fetch from `ipapi.co`. Both now use `AbortController` with a 5-second timeout:

```typescript
const controller = new AbortController();
const timeoutId = setTimeout(() => controller.abort(), 5000);
const res = await fetch('https://ipapi.co/json/', { signal: controller.signal });
clearTimeout(timeoutId);
```

If the API is down, slow, or MITMed, the terminal shows a graceful error instead of hanging forever.

---

## Performance

- **Code splitting.** Every route except the homepage is lazy-loaded with `React.lazy()` and wrapped in `Suspense`. The PDF renderer, the design system, the blog, the project detail page are all separate chunks that load on demand.
- **Manual chunks.** Vite's `rollupOptions.manualChunks` splits React, React DOM, React Router, and Framer Motion into a `react-vendor` chunk. This ensures the framework code is cached independently from application code.
- **Compression.** `vite-plugin-compression` generates pre-compressed assets for production.
- **IntersectionObserver for animations.** Scroll-reveal animations use `IntersectionObserver` with a threshold of 0.12 and a root margin of `0px 0px -10% 0px`. Elements only animate when they're actually entering the viewport.
- **Session caching.** The `whoami` command caches its API response in `sessionStorage` so repeated calls don't hit the network.

---

## What I learned the hard way

1. **The stylesheet problem.** Three CSS files, only one imported. Hours of debugging a black SVG before realizing the tokens were defined in the wrong file. Document your architecture.
2. **Preview boxes need hardcoded backgrounds.** When documenting a design system, preview boxes for "dark variant" and "light variant" can't use `var(--bg)`. They need fixed backgrounds so both variants are visible regardless of the active theme.
3. **Order matters in rehype plugins.** If you must use both `rehype-raw` and `rehype-sanitize`, the order must be `[rehypeRaw, rehypeSanitize]`, not the reverse. `rehype-raw` converts HTML to AST nodes; `rehype-sanitize` cleans those nodes afterward. Wrong order = ineffective sanitization. (I ended up removing `rehype-raw` entirely, but this is worth knowing.)
4. **Template literal types are underrated.** `NAVIGATE:${string}` gave me type-safe tagged strings without refactoring 20 files to use discriminated unions.
5. **Shipping beats perfection.** I spent three days on a glitch effect for the hero section before realizing it made the text unreadable. I deleted all of it.
6. **Standard CSS is enough.** With `color-mix()`, native nesting, custom properties, and `clamp()`, the argument for Sass or any CSS preprocessor has essentially vanished.

---

## Project structure

```
personal-website/
  index.html              — entry point, meta tags, CSP
  index.css               — single authoritative stylesheet (38KB)
  index.tsx               — React entry, imports index.css
  App.tsx                 — router, lazy routes, terminal state
  types.ts                — shared TypeScript interfaces
  config/sections.ts      — feature flags for homepage sections
  data/                   — all content (profile, projects, experience, certs, socials)
  components/
    layout/               — Navbar, Footer
    sections/             — Hero, Projects, Experience, Skills, About, Contact
    features/             — Blog, Resume, ResumePDF, DesignSystem, NotFound
    features/projects/    — ProjectDetail (GitHub README renderer)
    terminal/             — TerminalMode, TerminalInput, TerminalOutput, TerminalLine
    terminal/commands/    — 20+ command handlers
    terminal/types/       — TerminalLine, ProcessResult types
    ui/                   — CustomCursor, ProofStrip
  hooks/                  — useReveal, useTheme
  services/github.ts      — fetchProjectReadme, fetchProjectMeta
  public/                 — favicon.svg, logo-mark.svg, robots.txt, sitemap.xml, vercel.json
```

---

## What's next

- A proper **CTF writeups** section for security research.
- An automated **project archive** that pulls metadata directly from the GitHub API.
- **Blog RSS feed** for syndication.
- Exploring **View Transitions API** for route changes.

---

If you're reading this, you found the blog. Feel free to poke around the [architecture docs and design system](https://github.com/euphoric-habromaniac/pranjalkumar.com), open the terminal (`>_ terminal` in the nav, or hit the button), or drop me a line at `contact@pranjalkumar.com`.

---

*I'm a 1st year CSE student and Security Research Intern. I build systems and find the holes in them.*

