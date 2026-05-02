# Security Hardening

> CSP headers, the README XSS vulnerability, DOMPurify defense-in-depth, API timeout hardening, and the full header audit.

---

## Threat model

This is a static site deployed on Cloudflare Pages. "What is there to secure?" More than you'd think.

The site fetches and renders external content (GitHub READMEs), uses `dangerouslySetInnerHTML` in several components, makes API calls to third-party services, and opens links in new tabs. Each of these is an attack surface.

---

## The README XSS vulnerability

### The bug

The project detail page fetches `README.md` files from GitHub repositories and renders them with `react-markdown`. The original implementation included `rehype-raw`, a plugin that parses embedded HTML in Markdown and passes it through to the DOM.

GitHub READMEs are **not trusted content**. Even if I own the repos, compromise can happen through:
- Careless PR merges
- A hijacked GitHub account
- A malicious collaborator
- CI pipeline poisoning

With `rehype-raw` enabled, an attacker could inject:

```markdown
<img src=x onerror=alert(document.cookie)>
```

into a README, and it would execute as JavaScript on my domain. That gives them access to `localStorage` (theme, terminal history), the ability to create phishing overlays, session state manipulation — everything.

### The fix

Removed `rehype-raw` entirely. The site doesn't need raw HTML in README rendering. Standard Markdown formatting (headings, lists, code blocks, links, images, tables) is sufficient.

This is cleaner and safer than adding a sanitization layer on top. No `rehype-raw` means no HTML parsing, which means no injection vector.

### Why not `rehype-raw` + `rehype-sanitize`?

If you must use both, the order matters: `[rehypeRaw, rehypeSanitize]`, not the reverse. `rehype-raw` converts HTML strings to AST nodes; `rehype-sanitize` cleans those nodes afterward. Wrong order = ineffective sanitization because `rehype-sanitize` can't clean what hasn't been parsed yet.

I removed `rehype-raw` entirely because the safer option is to not parse HTML at all.

---

## DOMPurify for static content

Several components use `dangerouslySetInnerHTML` to render HTML strings from static data files:

- **Experience bullets** — `data/experience.ts` contains `<b>` tags and `&amp;` entities
- **About section copy** — may contain inline HTML formatting

While the data is controlled by me, every instance is wrapped in `DOMPurify.sanitize()`:

```typescript
<span dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(b) }} />
```

This is **defense-in-depth**. If the data files are ever compromised — through a supply chain attack, an accidental merge, or a compromised development machine — the sanitization layer limits the blast radius.

---

## Content Security Policy

The production CSP is served via `vercel.json` response headers and additionally enforced during development via a custom Vite plugin:

```
default-src 'self';
script-src 'self';
style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
font-src 'self' https://fonts.gstatic.com;
img-src 'self' data: https://raw.githubusercontent.com https://avatars.githubusercontent.com;
connect-src 'self' data: https://raw.githubusercontent.com https://api.github.com https://ipapi.co;
```

### Key decisions

| Directive | Value | Rationale |
|---|---|---|
| `script-src` | `'self'` only | **No `unsafe-inline`**. Even if an XSS payload slips through, the browser blocks inline script execution. |
| `style-src` | `'self' 'unsafe-inline'` | Required — React and Framer Motion inject inline styles for animations. |
| `img-src` | Includes `raw.githubusercontent.com` | GitHub README images are loaded from this domain. |
| `connect-src` | Includes `api.github.com`, `ipapi.co` | The two external APIs the site calls. |

### Dev-time enforcement

Most sites only add CSP in production. This site enforces it during development too, via a custom Vite server middleware:

```typescript
{
  name: 'security-headers',
  configureServer(server) {
    server.middlewares.use((_req, res, next) => {
      res.setHeader('Content-Security-Policy', "...");
      res.setHeader('X-Frame-Options', 'DENY');
      res.setHeader('X-Content-Type-Options', 'nosniff');
      res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');
      next();
    });
  },
}
```

This catches CSP violations during development, not after deployment. If a new feature violates the policy, I know immediately.

---

## Security headers

| Header | Value | Purpose |
|---|---|---|
| `X-Frame-Options` | `DENY` | Prevents clickjacking — the site cannot be embedded in an iframe |
| `X-Content-Type-Options` | `nosniff` | Prevents MIME-type sniffing — browser must respect declared Content-Type |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Limits referrer leakage to origin only for cross-origin requests |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` | Disables device APIs — no JavaScript can request camera, mic, or location |

---

## API hardening

The `whoami` and `scan` terminal commands fetch from `ipapi.co`. Both use `AbortController` with a 5-second timeout:

```typescript
const controller = new AbortController();
const timeoutId = setTimeout(() => controller.abort(), 5000);

try {
  const res = await fetch('https://ipapi.co/json/', {
    signal: controller.signal
  });
  clearTimeout(timeoutId);
  // ... process response
} catch (err) {
  clearTimeout(timeoutId);
  // Show graceful error in terminal
}
```

If the API is down, slow, or MITMed, the terminal shows a graceful error message instead of hanging forever.

### Session caching

The `whoami` command caches its API response in `sessionStorage`. Repeated calls within the same session don't hit the network:

```typescript
const cached = sessionStorage.getItem('whoami_cache');
if (cached) return JSON.parse(cached);
// ... fetch, then sessionStorage.setItem('whoami_cache', JSON.stringify(data))
```

This reduces unnecessary API calls and provides instant responses on repeat invocations.

---

## Link safety

All `target="_blank"` links use `rel="noopener noreferrer"`:

- **`noopener`** — Prevents the opened page from accessing `window.opener`, which could be used to navigate the parent page to a phishing site (reverse tabnabbing).
- **`noreferrer`** — Prevents the `Referer` header from being sent to the linked site.

This is enforced across all components: Navbar, Footer, Contact, Resume, ProjectDetail, terminal `open` command.

---

## Summary

| Layer | Protection | Implementation |
|---|---|---|
| Markdown rendering | No HTML parsing | Removed `rehype-raw` |
| Static HTML content | Sanitization | `DOMPurify.sanitize()` on every `dangerouslySetInnerHTML` |
| Script injection | CSP | `script-src 'self'` — no `unsafe-inline` |
| Clickjacking | Frame blocking | `X-Frame-Options: DENY` |
| API calls | Timeouts | `AbortController` + 5s timeout |
| API calls | Caching | `sessionStorage` for repeated calls |
| External links | Tabnabbing prevention | `rel="noopener noreferrer"` |
| Device APIs | Permissions | `Permissions-Policy` disabling camera/mic/geo |
| Dev environment | Early detection | CSP enforced in Vite dev server |
