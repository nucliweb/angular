# Measuring and Diagnosing Core Web Vitals in Angular

## Measurement Procedures

### Lab measurement — Lighthouse

Run Lighthouse against each key route. In Chrome DevTools: **Audits** tab → select Performance → Analyze page load.

If Chrome DevTools MCP is available:

```
lighthouse_audit({ url: 'https://example.com', categories: ['performance'] })
```

Record the score and raw metric values for LCP, INP, and CLS. Repeat for:
- Landing page (always critical — affects bounce rate and SEO)
- Primary conversion page (product page, checkout, signup)
- Any route with heavy interactivity

### Field measurement — CrUX

Check real-user data at [PageSpeed Insights](https://pagespeed.web.dev/) for the domain. CrUX data (28-day rolling) reflects actual conditions, not Lighthouse's simulated throttling. If lab and field data diverge significantly, field data takes precedence.

### In-app field measurement — `web-vitals`

```bash
npm install web-vitals
```

```ts
// main.ts
import { onLCP, onINP, onCLS, onFCP, onTTFB } from 'web-vitals';

function sendToAnalytics(metric: Metric) {
  // Replace with your analytics endpoint
  navigator.sendBeacon('/analytics', JSON.stringify(metric));
}

onLCP(sendToAnalytics);
onINP(sendToAnalytics);
onCLS(sendToAnalytics);
```

---

## LCP Diagnosis

### Identify the LCP element

In Chrome DevTools → Performance → record a page load → click the LCP marker in the timeline. The LCP element is highlighted in the viewport.

Common LCP elements in Angular apps:
- Hero `<img>` or CSS background image
- Large `<h1>` or heading rendered by SSR/SSG
- First product image in a grid

### LCP decision tree

```
LCP > 2.5 s
    │
    ├── Is the page CSR-only?
    │       YES → Evaluate SSG (static content) or SSR (dynamic content)
    │             See rendering-strategies.md in angular-developer
    │
    ├── Is the LCP element an image?
    │       YES → Is NgOptimizedImage used with priority?
    │               NO → Add ngSrc, width, height, priority
    │
    ├── Is the LCP element inside @defer?
    │       YES → Remove @defer from above-fold content immediately
    │
    ├── Is there a <link rel="preconnect"> to the image CDN?
    │       NO → Add to index.html:
    │            <link rel="preconnect" href="https://cdn.example.com" />
    │
    └── Check TTFB — if > 800 ms, server response is the bottleneck
            → CDN caching, server-side caching, or SSG
```

### LCP checklist

- [ ] Rendering strategy appropriate for content type (SSG/SSR for content pages)
- [ ] LCP image uses `NgOptimizedImage` with `priority`
- [ ] No `@defer` wrapping the LCP element
- [ ] `<link rel="preconnect">` present for image CDN origin
- [ ] TTFB < 800 ms (check Network tab → time to first byte)

---

## INP Diagnosis

### Identify slow interactions

In Chrome DevTools → Performance → check **Interactions** track. Interactions with INP > 200 ms appear highlighted. Click an interaction to see the event handler breakdown.

Look for:
- Long tasks (red blocks) in the main thread during the interaction
- Change detection cycles triggered by the event (visible as "Angular" in the call stack with Angular DevTools)
- Synchronous XHR or heavy computation in event handlers

### INP decision tree

```
INP > 200 ms
    │
    ├── Long tasks (> 50 ms) during event handlers?
    │       YES → Yield with scheduler.yield() between chunks
    │
    ├── Change detection taking > 16 ms?
    │       YES → Open Angular DevTools → Profiler
    │             → Identify components with Default strategy
    │             → Add OnPush + signals
    │
    ├── Third-party libraries (maps, charts) firing many events?
    │       YES → Wrap in NgZone.runOutsideAngular()
    │
    └── New project or full refactor possible?
            YES → provideExperimentalZonelessChangeDetection()
```

### INP checklist

- [ ] Angular DevTools Profiler run during a representative interaction
- [ ] Components with Default CD strategy identified and converted to OnPush
- [ ] Third-party event-heavy code moved outside Angular's zone
- [ ] Event handlers broken up with `scheduler.yield()` for tasks > 50 ms

---

## CLS Diagnosis

### Identify layout shifts

In Chrome DevTools → Performance → **Layout Shifts** track shows shift events with their score contribution. Click a shift to see which elements moved.

Common sources in Angular apps:
- Images without dimensions (load asynchronously, push content down)
- `@defer` blocks without `@placeholder` (space collapses then reappears)
- Web fonts loading and replacing fallback fonts
- Dynamic banners or alerts inserted above the fold after initial render

### CLS decision tree

```
CLS > 0.1
    │
    ├── Images causing layout shifts?
    │       YES → Add NgOptimizedImage with explicit width and height
    │
    ├── @defer blocks without @placeholder?
    │       YES → Add @placeholder with matching height
    │
    ├── Web fonts causing text reflow?
    │       YES → Add font-display: swap to @font-face
    │             Add <link rel="preload"> for critical fonts
    │
    └── Dynamic content inserted above the fold?
            YES → Reserve space with min-height, or insert below fold
```

### CLS checklist

- [ ] All images use `NgOptimizedImage` with `width` and `height`
- [ ] All `@defer` blocks have `@placeholder` with matching height
- [ ] `font-display: swap` on all `@font-face` declarations
- [ ] Critical fonts have `<link rel="preload" as="font">`
- [ ] No content inserted above the fold after initial render
