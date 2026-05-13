---
status: CUSTOM
description: SEO audit methodology for React/Vite/Netlify projects. Scans codebase + live site, scores against a prioritised checklist, and produces actionable findings ready to feed into the autofeature pipeline.
---

# Feature SEO Audit

Produces a prioritised SEO findings report for React/Vite/Tailwind sites deployed on Netlify.

**Findings severity:**
- 🔴 CRITICAL — prevents crawling or indexing right now
- 🟡 HIGH — significant ranking impact, low effort to fix
- 🟢 MEDIUM — good practice, moderate effort
- ⚪ LOW — nice-to-have, low urgency

---

## Step 1: Codebase Scan

Run each check. Record present / missing / partial for each item.

### 1a. Rendering strategy
```bash
# Check for prerendering / SSR
cat package.json | grep -E 'prerender|vite-ssg|react-snap|react-static|next|gatsby|remix|astro' 2>/dev/null
ls vite.config.* 2>/dev/null | xargs grep -l 'prerender\|ssr' 2>/dev/null
ls netlify/functions netlify/edge-functions 2>/dev/null
```

| Finding | Severity |
|---------|----------|
| No prerendering, no SSR, no edge functions | 🔴 CRITICAL |
| vite-plugin-prerender or vite-ssg present | ✓ |
| Netlify edge function rendering bots | ✓ |
| Next.js / Astro / Remix (SSR by default) | ✓ |

> **Why it matters:** React SPAs render entirely in the browser. Googlebot does execute JS, but delays indexing. All other crawlers (social previews, Bing, screaming frog) get an empty `<div id="root">`. No content = no ranking.

### 1b. Meta tag management
```bash
# Check for head management library
cat package.json | grep -E 'react-helmet|@vueuse|next/head|@tanstack/head' 2>/dev/null
grep -r 'react-helmet\|Helmet\|useHead' src/ --include='*.tsx' --include='*.ts' --include='*.jsx' -l 2>/dev/null | head -10
grep -r '<title>' src/ public/ --include='*.tsx' --include='*.html' -l 2>/dev/null | head -5
```

Check for per-route title and meta description variation:
```bash
grep -r 'og:title\|og:description\|og:image\|twitter:card' src/ --include='*.tsx' --include='*.jsx' --include='*.html' -l 2>/dev/null | head -10
```

| Finding | Severity |
|---------|----------|
| No library — all pages share one static title | 🔴 CRITICAL |
| Library present but only on some routes | 🟡 HIGH |
| Library present, og:* missing | 🟡 HIGH |
| og:image missing | 🟡 HIGH |
| All present and per-route | ✓ |

### 1c. Sitemap
```bash
ls public/sitemap.xml public/sitemap*.xml 2>/dev/null
cat package.json | grep -E 'sitemap|vite-plugin-sitemap' 2>/dev/null
grep -r 'sitemap' vite.config.* 2>/dev/null
```

| Finding | Severity |
|---------|----------|
| No sitemap.xml | 🟡 HIGH |
| Static sitemap (not auto-generated) | 🟢 MEDIUM |
| Auto-generated via vite-plugin-sitemap | ✓ |

### 1d. robots.txt
```bash
cat public/robots.txt 2>/dev/null
```

| Finding | Severity |
|---------|----------|
| Missing entirely | 🟡 HIGH |
| Present but no Sitemap: directive | 🟢 MEDIUM |
| Present with correct Sitemap: URL | ✓ |

### 1e. Structured data (JSON-LD)
```bash
grep -r 'application/ld\+json\|json-ld\|schema.org' src/ --include='*.tsx' --include='*.jsx' --include='*.html' -l 2>/dev/null | head -5
```

| Finding | Severity |
|---------|----------|
| No structured data on any page | 🟢 MEDIUM |
| Present on key pages (home, product, org) | ✓ |

### 1f. Netlify configuration
```bash
cat netlify.toml 2>/dev/null
cat public/_redirects 2>/dev/null
cat public/_headers 2>/dev/null
```

Check for:
- HTTPS enforcement (`https://yourdomain.com/* 301` in _redirects or `force = true` in netlify.toml)
- WWW canonical redirect (www → apex or apex → www, consistently one way)
- Security/SEO headers (`X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`)

| Finding | Severity |
|---------|----------|
| No HTTPS enforcement (Netlify enforces by default but redirect should be explicit) | 🟢 MEDIUM |
| No www/apex canonical redirect | 🟡 HIGH |
| Missing security headers | 🟢 MEDIUM |

### 1g. Semantic HTML patterns
```bash
# Check for h1 usage per route/page
grep -r '<h1' src/ --include='*.tsx' --include='*.jsx' -l 2>/dev/null | head -10
# Check for alt attributes on images
grep -r '<img' src/ --include='*.tsx' --include='*.jsx' | grep -v 'alt=' | head -10
# Check for descriptive link text (flag empty links)
grep -r '<a ' src/ --include='*.tsx' --include='*.jsx' | grep -v 'aria-label\|>[^<]' | head -5
```

| Finding | Severity |
|---------|----------|
| Images with missing alt text | 🟡 HIGH |
| Multiple h1s on one page | 🟢 MEDIUM |
| Links with no text content | 🟢 MEDIUM |

### 1h. Canonical tags
```bash
grep -r 'canonical' src/ --include='*.tsx' --include='*.jsx' --include='*.html' 2>/dev/null | head -5
```

| Finding | Severity |
|---------|----------|
| No canonical tags on any page | 🟡 HIGH |
| Canonical present and per-route | ✓ |

---

## Step 2: Live Site Check (if SITE_URL available)

Use WebFetch to check the actual deployed site. Fetch `<SITE_URL>` and inspect the raw HTML response.

```
WebFetch({ url: SITE_URL, prompt: "Return the raw <head> section HTML. Do not truncate." })
```

Check the returned HTML for:
- `<title>` — is it descriptive or a placeholder like "Vite + React"?
- `<meta name="description">` — present and non-empty?
- `og:title`, `og:description`, `og:image` — all three present?
- `twitter:card` — present?
- `<link rel="canonical">` — present and correct URL?
- Any content in `<body>` beyond `<div id="root"></div>` — if body is empty, prerendering is missing

Also fetch:
```
WebFetch({ url: SITE_URL + '/robots.txt', prompt: "Return the full content." })
WebFetch({ url: SITE_URL + '/sitemap.xml', prompt: "Return the first 500 chars." })
```

---

## Step 3: Netlify MCP Check (if HAS_NETLIFY=true)

Use Netlify MCP to read the site's configuration:
- List environment variables — check if any SEO-relevant vars are set (e.g., `VITE_SITE_URL`, `VITE_API_URL`)
- Check if the site has a custom domain configured (vs netlify.app subdomain)
- Check recent deploy status (failed deploys may mean broken sitemap generation)

---

## Step 4: Compile Findings

Group findings by severity. For each finding, include:
- What the issue is
- Why it matters (one sentence)
- Recommended fix (specific library or pattern for this stack)
- Estimated effort: XS / S / M / L

**Output format:**

```
=== SEO Audit: [project name] ===
Live URL: [SITE_URL or "not checked"]
Scanned: [timestamp]

🔴 CRITICAL ([N] issues)
  1. [Issue] — [Why] — Fix: [what to do] — Effort: [XS/S/M/L]

🟡 HIGH ([N] issues)
  1. [Issue] — [Why] — Fix: [what to do] — Effort: [XS/S/M/L]

🟢 MEDIUM ([N] issues)
  ...

⚪ LOW ([N] issues)
  ...

SEO Score: [N/20 checks passing]
```

---

## Step 5: Fix Proposal

After showing the audit report, offer:

```
Found [N] SEO issues ([C] critical, [H] high priority).

What would you like to do?

A) Fix all critical + high issues in one autofeature run
B) Fix a specific issue — [show numbered list]
C) Generate a Trello card for the SEO work
D) Audit only — no changes
```

**If A or B selected:**
Construct a feature request string summarising the chosen fixes and hand off to the autofeature pipeline. Set:
- `FEATURE_REQUEST` = "SEO improvements: [list of chosen fixes]"
- `SEO_AUDIT_PATH` = path to the saved audit report (`.autofeature/seo-audit-[date].md`)
- The SEO architect agent will be spawned as part of the implementation step

**If C selected:**
Format a Trello card description with the audit findings and post via `mcp__trello__add_card_to_list` (ask which list first).

---

## React/Vite/Netlify SEO Fix Reference

Canonical fix patterns for this stack. Used by the SEO architect when implementing.

### Prerendering (CRITICAL fix)
```bash
npm install vite-plugin-prerender
```
```ts
// vite.config.ts
import prerender from 'vite-plugin-prerender'
export default { plugins: [prerender({ staticDir: 'dist', routes: ['/', '/about', '/pricing'] })] }
```
For dynamic routes: use `vite-ssg` or render to Netlify Edge Functions.

### Meta tags (react-helmet-async)
```bash
npm install react-helmet-async
```
```tsx
// main.tsx — wrap app
import { HelmetProvider } from 'react-helmet-async'
<HelmetProvider><App /></HelmetProvider>

// Per page
import { Helmet } from 'react-helmet-async'
<Helmet>
  <title>Page Title — Site Name</title>
  <meta name="description" content="150-160 char description." />
  <meta property="og:title" content="Page Title" />
  <meta property="og:description" content="Same as description or variation." />
  <meta property="og:image" content="https://yourdomain.com/og-image.jpg" />
  <meta property="og:url" content="https://yourdomain.com/current-path" />
  <meta name="twitter:card" content="summary_large_image" />
  <link rel="canonical" href="https://yourdomain.com/current-path" />
</Helmet>
```

### Sitemap (vite-plugin-sitemap)
```bash
npm install vite-plugin-sitemap
```
```ts
// vite.config.ts
import sitemap from 'vite-plugin-sitemap'
export default { plugins: [sitemap({ hostname: 'https://yourdomain.com' })] }
```

### robots.txt
```
# public/robots.txt
User-agent: *
Allow: /
Sitemap: https://yourdomain.com/sitemap.xml
```

### Netlify canonical redirect
```
# public/_redirects
https://www.yourdomain.com/* https://yourdomain.com/:splat 301!
http://yourdomain.com/* https://yourdomain.com/:splat 301!
```

### JSON-LD (Organization + WebSite)
```tsx
// src/components/SchemaOrg.tsx
export function SchemaOrg() {
  return (
    <script type="application/ld+json" dangerouslySetInnerHTML={{ __html: JSON.stringify({
      "@context": "https://schema.org",
      "@graph": [
        { "@type": "Organization", name: "...", url: "https://...", logo: "https://.../logo.png" },
        { "@type": "WebSite", url: "https://...", potentialAction: {
          "@type": "SearchAction", target: "https://.../?q={query}", "query-input": "required name=query"
        }}
      ]
    })}} />
  )
}
```
