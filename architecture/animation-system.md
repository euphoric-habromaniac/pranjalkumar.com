# Animation System

> The minimal motion layer, easing curve philosophy, IntersectionObserver scroll-reveal, and the performance budget.

---

## Philosophy

Two easing curves set how everything on the site moves:

```typescript
// "Editorial" — sharp entrance, slow settle. Used for content reveals.
const EASE_EDITORIAL: [number, number, number, number] = [0.22, 1, 0.36, 1];

// "Snappy" — fast, responsive. Used for interactive feedback.
const EASE_SNAPPY: [number, number, number, number] = [0.16, 1, 0.3, 1];
```

**Editorial** feels deliberate, like a magazine page turn. It's used for scroll reveals, section entrances and line draws.

**Snappy** feels quick, like tapping a physical button. It's used for scale effects, pop-ins and anything the user triggers.

Motion uses these curves rather than ad-hoc `ease` or `ease-in-out`, which keeps the feel consistent.

The overall rule: quiet from a distance, detailed up close. Nothing decorative moves while the page is at rest. Details appear when something happens: a scroll, a section coming into view, a hover.

---

## The detail layer

An earlier build had a much louder motion set: a text-scramble effect on headings, an animated astrolabe in the hero, and equation dividers between sections. They were removed, along with a large Framer Motion variant library. What remains lives in `components/motion/`:

| Detail | Trigger | Cost |
|---|---|---|
| Scroll hairline | A 1px line across the top of the page, scaled by scroll position | One passive scroll listener, at most one update per frame |
| Reveal fades | Small fade-ups as elements enter the viewport | `IntersectionObserver`, once per element |
| Section labels | A faint numbered label fades in once when a section appears | `IntersectionObserver` plus a CSS transition |
| Cursor ring | Follows a real mouse pointer | Fine pointers only, never on touch |
| Project card details | Crop marks and a faint grid spotlight on hover | Only while the pointer is over a card |

Framer Motion is still used for a few interface transitions: the mobile drawer, route changes, the back-to-top button and the terminal overlay.

### Performance budget

The site has to run on slow machines, so the layer sticks to three rules:

- no canvases and no always-on JavaScript loops
- continuous motion animates only `transform` and `opacity`, and pauses off screen
- everything else is static CSS, or fires once on an event

All of it is scoped under one class on `<html>`. A switch in the footer turns the whole layer off and remembers the choice.

---

## Scroll-reveal system

The `useReveal` hook uses `IntersectionObserver` for scroll-triggered animations:

```typescript
export function useReveal() {
  useEffect(() => {
    const els = document.querySelectorAll('.reveal');
    const io = new IntersectionObserver(
      (entries) => {
        entries.forEach((e) => {
          if (e.isIntersecting) e.target.classList.add('in');
        });
      },
      { threshold: 0.12, rootMargin: '0px 0px -10% 0px' }
    );
    els.forEach((el) => io.observe(el));
    return () => io.disconnect();
  }, []);
}
```

**Configuration:**
- `threshold: 0.12` — Element triggers when 12% is visible
- `rootMargin: '0px 0px -10% 0px'` — Triggers 10% before the element reaches the bottom of the viewport
- One-directional: elements animate in but don't animate out when scrolled past

Any element with class `reveal` automatically gets observed. Adding class `in` triggers CSS transitions defined in `index.css`.

---

## CSS animations

Some animations bypass Framer Motion and use pure CSS for performance:

| Animation | Duration | Easing | Usage |
|---|---|---|---|
| Status pulse | 2400ms, infinite | ease-out | Hero status indicator |
| Terminal caret | 1100ms, steps(2), infinite | step | Blinking cursor |
| Reveal transitions | 700ms | `cubic-bezier(0.22, 1, 0.36, 1)` | Scroll-reveal elements |

### Reduced motion

All animations respect `prefers-reduced-motion`:

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

---

## Motion spec summary

The design system page at `/design` documents every animation with live interactive demos:

1. **Status Pulse** — The actual `.pulse` animation running live
2. **Terminal Caret** — Block cursor blinking at production timing
3. **Card Hover** — Project card lifting on hover with React event handlers

Each demo is self-contained (inline styles, no global CSS pollution).
