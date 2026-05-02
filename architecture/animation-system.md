# Animation System

> Framer Motion variant library, easing curve philosophy, IntersectionObserver scroll-reveal, and the full motion spec.

---

## Philosophy

Two easing curves define the site's motion personality:

```typescript
// "Editorial" — sharp entrance, slow settle. Used for content reveals.
const EASE_EDITORIAL: [number, number, number, number] = [0.22, 1, 0.36, 1];

// "Snappy" — fast, responsive. Used for interactive feedback.
const EASE_SNAPPY: [number, number, number, number] = [0.16, 1, 0.3, 1];
```

**Editorial** feels deliberate and considered — like a magazine page turn. It's used for scroll-reveal animations, section entrances, and line draws.

**Snappy** feels responsive and alive — like tapping a physical button. It's used for scale effects, pop-ins, and user-triggered interactions.

Every animation in the library uses one of these two curves. No ad-hoc `ease` or `ease-in-out` anywhere. This creates unconscious visual consistency.

---

## Variant library

The animation system is a library of **25+ Framer Motion variant presets**, all defined in `animations/micro.ts`. Every variant follows the `hidden → visible` pattern for use with Framer Motion's `initial`, `animate`, and `whileInView` props.

### Entrance animations

| Variant | Effect | Duration | Easing |
|---|---|---|---|
| `FADE_UP_BOUNCE` | Fade + translateY(40px) + overshoot | 800ms | `[0.34, 1.56, 0.64, 1]` (bounce) |
| `SLIDE_LEFT` | Fade + translateX(-60px) | 700ms | Editorial |
| `SLIDE_RIGHT` | Fade + translateX(60px) | 700ms | Editorial |
| `SLIDE_UP_FADE` | Fade + translateY(60px) | 800ms | Editorial |
| `FADE_DOWN` | Fade + translateY(-30px) | 600ms | Editorial |
| `SCALE_UP` | Fade + scale(0.8) | 600ms | Snappy |
| `SCALE_ROTATE` | Fade + scale(0.6) + rotate(-10°) | 700ms | Bounce |
| `BLUR_IN` | Fade + blur(10px) | 800ms | Editorial |
| `ZOOM_BLUR` | Fade + scale(1.2) + blur(8px) | 800ms | Editorial |
| `FLIP_IN` | Fade + rotateX(90°) | 800ms | Editorial |
| `TILT_IN` | Fade + rotateY(-20°) + translateX(-30px) | 700ms | Editorial |
| `SKEW_IN` | Fade + skewX(-10°) + translateX(-30px) | 600ms | Editorial |

### Spring animations

| Variant | Stiffness | Damping | Effect |
|---|---|---|---|
| `POP_IN` | 400 | 15 | Scale from 0 with spring |
| `ELASTIC_SCALE` | 300 | 10 | Scale from 0.5 with bounce |
| `BOUNCE_UP` | 150 | 12 | translateY(100px) with spring |
| `SWING_IN` | 200 | 15 | Rotate(15°) with spring |
| `WAVE_ITEM` | 200 | 12 | translateY(30px) + rotate(-5°) with spring |

### Stagger containers

| Container | Item Variant | Stagger delay | Child delay |
|---|---|---|---|
| `STAGGER_SCALE` | `STAGGER_SCALE_ITEM` | 80ms | 100ms |
| `WAVE_STAGGER` | `WAVE_ITEM` | 60ms | 200ms |
| `TYPEWRITER` | `TYPEWRITER_CHAR` | 50ms | — |

### Continuous animations

| Variant | Effect | Duration | Loop |
|---|---|---|---|
| `MICRO_PULSE` | Opacity 0.5 → 1 → 0.5 | 2s | Infinite |
| `GLOW_PULSE` | Same as MICRO_PULSE | 2s | Infinite |
| `FLOAT` | translateY(-5px → 5px → -5px) | 3s | Infinite |

### Structural animations

| Variant | Effect |
|---|---|
| `LINE_DRAW` | scaleX(0 → 1), origin left. For horizontal dividers. |
| `BORDER_EXPAND` | scaleX(0 → 1). For border reveals. |
| `MORPH` | borderRadius(50% → 8px) + scale(0.5 → 1). Shape morphing. |
| `CHAR_REVEAL` | Per-character fade+translateY with staggered delay (30ms/char). |
| `COUNT_UP` | Fade + translateY for number counters. |

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
| Status pulse | 2400ms, infinite | ease-out | Hero availability indicator |
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
