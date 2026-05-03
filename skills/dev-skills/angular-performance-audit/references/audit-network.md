# Network and Resource Loading Audit

Network loading problems affect LCP (slow resource fetches), CLS (late-loading fonts/images), and overall page load time. This reference covers the resource loading patterns to audit in Angular applications.

## Resource Hints

Resource hints in `index.html` tell the browser to set up connections or fetch resources before they are discovered in HTML or CSS.

### `preconnect`

Establishes TCP + TLS connections to critical origins ahead of time. Add for every origin that serves LCP-critical resources (image CDNs, font hosts, API hosts used for SSR data).

```html
<!-- index.html -->
<link rel="preconnect" href="https://cdn.example.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
```

Check: open Chrome DevTools → Network → filter by the CDN domain → look at the "Waiting (TTFB)" time on the first request. If TTFB is > 200 ms and there is no preconnect, adding one will improve LCP.

### `preload`

Fetches a resource at high priority before the browser discovers it. Use for:
- Critical font files
- LCP hero images (when `NgOptimizedImage priority` is not sufficient, e.g., CSS backgrounds)

```html
<link rel="preload" href="/fonts/inter-var.woff2" as="font" type="font/woff2" crossorigin />
```

Do not preload more than 1–2 resources. Every preload competes for bandwidth.

### `dns-prefetch`

Lighter than `preconnect` — only resolves DNS. Use for origins that are not critical path but load content shortly after:

```html
<link rel="dns-prefetch" href="https://analytics.example.com" />
```

## Font Loading

Web fonts are a common CLS and LCP source. Audit each `@font-face` declaration:

```css
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter-var.woff2') format('woff2');
  font-display: swap;   /* show fallback immediately, swap when loaded */
  font-weight: 100 900;
}
```

`font-display: swap` prevents invisible text (FOIT) but may cause text reflow (FOUT) which contributes to CLS. Mitigate CLS from font swap by:
1. Matching fallback font metrics to the web font (use `size-adjust`, `ascent-override` in `@font-face`)
2. Preloading the critical font file

Check: in Chrome DevTools → Performance → look for "Recalculate Style" events that follow a font load. These indicate a font swap causing reflow.

## HTTP Caching

Angular's build output includes content-hashed filenames for JS and CSS chunks. Verify caching headers:

```
Cache-Control: public, max-age=31536000, immutable  ← for hashed assets (JS/CSS/images)
Cache-Control: no-cache                              ← for index.html
```

Check: Chrome DevTools → Network → reload → look at the **Cache-Control** response header for `main-<hash>.js`. If it is missing or set to `no-cache`, assets are re-downloaded on every visit.

The `index.html` must use `no-cache` so browsers always fetch the latest version, which then references the hashed assets.

## Sequential vs Parallel Fetches

In SSR or SSG, data fetched during route resolution should be parallelized. Sequential fetches add up:

```ts
// WRONG: sequential — total latency = A + B + C
const user = await firstValueFrom(this.userService.getUser(id));
const orders = await firstValueFrom(this.orderService.getOrders(user.id));
const recommendations = await firstValueFrom(this.recService.get(user.id));

// CORRECT: parallel — total latency = max(A, B, C)
const [user, orders, recommendations] = await Promise.all([
  firstValueFrom(this.userService.getUser(id)),
  firstValueFrom(this.orderService.getOrders(id)),
  firstValueFrom(this.recService.get(id))
]);
```

Check for sequential `await` chains in route resolvers and component `ngOnInit` methods.

## Transfer State (SSR)

In SSR applications, data fetched on the server should be transferred to the browser to avoid duplicate requests during hydration. Verify `withHttpTransferCache()` is configured:

```ts
// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideClientHydration(withHttpTransferCache())
  ]
};
```

Check: open the rendered HTML source of an SSR page and search for `<script id="ng-state">`. If absent, HTTP requests are being repeated on the client during hydration, doubling API load time.

## Third-Party Scripts

Third-party scripts (analytics, tag managers, chat widgets) can block the main thread and delay LCP. Audit with:

1. Chrome DevTools → Performance → record load → look for long tasks (red) from third-party origins
2. Lighthouse → **Reduce the impact of third-party code** opportunity

For non-critical scripts, defer loading:

```html
<!-- WRONG: blocks parser -->
<script src="https://analytics.example.com/script.js"></script>

<!-- CORRECT: deferred -->
<script src="https://analytics.example.com/script.js" defer></script>
```

For Angular-managed scripts, use the `DOCUMENT` token and inject scripts only after the LCP event:

```ts
import { afterNextRender } from '@angular/core';

constructor() {
  afterNextRender(() => {
    const script = document.createElement('script');
    script.src = 'https://analytics.example.com/script.js';
    document.head.appendChild(script);
  });
}
```

## Network Audit Checklist

- [ ] `<link rel="preconnect">` present for image CDN, font host, and critical API origins
- [ ] `<link rel="preload">` for critical fonts (not more than 1–2)
- [ ] `font-display: swap` on all `@font-face` declarations
- [ ] Hashed JS/CSS assets have `Cache-Control: immutable` headers
- [ ] `index.html` has `Cache-Control: no-cache`
- [ ] Route resolvers use `Promise.all` for parallel data fetches
- [ ] SSR app uses `withHttpTransferCache()` to avoid duplicate requests
- [ ] Non-critical third-party scripts loaded with `defer` or after LCP
