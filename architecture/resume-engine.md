# Resume Engine

> Client-side PDF generation with @react-pdf/renderer, lazy-loading strategy, and the single-source-of-truth pattern.

---

## The problem

Most portfolios host a static PDF on S3 or Google Drive. The problem: your resume and your website drift apart. You update a job title on the site but forget to re-export the PDF. Now your website says one thing and your downloadable resume says another.

---

## The solution

The resume PDF is generated **entirely in the browser** using `@react-pdf/renderer`. No server. No S3. No manual export.

When you click "Download PDF," the site:

1. **Lazy-loads** the PDF engine
2. **Pulls data** from the same TypeScript files that drive the website
3. **Renders a styled PDF** in memory
4. **Triggers a browser download**

The resume and the website are **always in sync**. Update a job title in `data/experience.ts`, and both the web view and the downloadable PDF reflect it immediately.

---

## Architecture

```mermaid
graph TD
    DATA[data/*.ts] -->|import| WEB[Resume.tsx — web view]
    DATA -->|import| PDF[ResumePDF.tsx — PDF definition]
    
    USER[User clicks Download] --> LAZY[Dynamic import]
    LAZY --> ENGINE[@react-pdf/renderer ~400KB]
    LAZY --> PDF
    
    ENGINE -->|pdf| BLOB[PDF blob in memory]
    BLOB -->|URL.createObjectURL| DOWNLOAD[Browser download]
```

### Two views, one data source

| Component | Purpose | Size |
|---|---|---|
| `Resume.tsx` | Web view — styled resume page with all sections | 13KB |
| `ResumePDF.tsx` | PDF document definition — `@react-pdf` `<Document>` | 9KB |

Both import from the same data files:
- `data/profile.ts` — name, role, location, contact
- `data/experience.ts` — work history
- `data/projects.ts` — project list
- `data/certs.ts` — certifications

No data duplication. Change once, update everywhere.

---

## Lazy-loading strategy

The `@react-pdf/renderer` package is ~400KB. Loading it eagerly would bloat the initial bundle for a feature most visitors never use.

The solution: dynamic `import()` triggered only when the user clicks "Download PDF":

```typescript
const handleDownload = async () => {
  setPdfLoading(true);
  try {
    const [{ pdf }, { ResumePDFDoc }] = await Promise.all([
      import('@react-pdf/renderer'),
      import('./ResumePDF'),
    ]);
    const blob = await pdf(React.createElement(ResumePDFDoc)).toBlob();
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'pranjal-kumar-resume.pdf';
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
  } catch (err) {
    console.error('[ResumePDF] generation failed:', err);
  } finally {
    setPdfLoading(false);
  }
};
```

Key details:

- **`Promise.all`** — The renderer and the PDF document definition load in parallel
- **`React.createElement`** — Used instead of JSX because the import is dynamic
- **`URL.createObjectURL` → `URL.revokeObjectURL`** — Creates a temporary URL for the blob, triggers download, then cleans up to avoid memory leaks
- **Loading state** — Button shows "generating…" with reduced opacity and `cursor: wait` while the PDF is being built

### Vite chunk splitting

The PDF engine is additionally split into a separate chunk via Vite's `manualChunks`, ensuring it's never bundled with the main application code:

```typescript
// vite.config.ts
build: {
  rollupOptions: {
    output: {
      manualChunks: {
        'react-vendor': ['react', 'react-dom', 'react-router-dom'],
        'framer-motion': ['framer-motion'],
        // @react-pdf ends up in its own chunk via dynamic import
      }
    }
  }
}
```

---

## The web resume

`Resume.tsx` renders a styled resume page with numbered sections:

| # | Section | Data source |
|---|---|---|
| 01 | Objective | Hardcoded (specific to resume context) |
| 02 | Education | Hardcoded |
| 03+ | Experience | `data/experience.ts` |
| — | Achievement | Hardcoded (SIH 2025) |
| — | Projects | `data/projects.ts` |
| — | Skills | Local constant (resume-specific grouping) |
| — | Certifications | `data/certs.ts` |
| — | Extracurriculars | Hardcoded |

The skills grouping in the resume differs slightly from the portfolio's `data/skills.ts` because the resume uses a more traditional categorization (Languages, Frontend, Backend/DB, DevOps, Security) suited for ATS scanning, while the portfolio uses a more visual grouping.

### Security note

Experience bullet points contain HTML (`<b>`, `&amp;`). Every `dangerouslySetInnerHTML` is wrapped in `DOMPurify.sanitize()`:

```typescript
<span dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(b) }} />
```

This is defense-in-depth. The data is controlled by me, but if the files are ever compromised, at least the blast radius is limited.

---

## PDF styling

`ResumePDF.tsx` uses `@react-pdf/renderer`'s `StyleSheet.create()` for PDF-specific styling. The PDF replicates the web resume's visual language but adapted for print:

- Cormorant Garamond for headings (registered as a custom font)
- Clean single-column layout optimized for A4
- Section numbering matching the web view
- Tech stack tags rendered as bordered pills
- Consistent spacing via a 4px-based scale
