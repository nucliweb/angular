# Audit Findings Output Format

Use this template to produce a structured findings report after completing the diagnostic workflow. Always include measurements — findings without data are opinions.

---

## Report Template

```markdown
## Performance Audit — [Project Name]

**Date**: YYYY-MM-DD  
**Angular version**: X.Y.Z  
**Routes audited**: [list of URLs]

---

### Measurements

| Metric | Lab (Lighthouse) | Field (CrUX) | Target | Status |
|--------|-----------------|--------------|--------|--------|
| LCP    | 3.2 s           | 3.8 s        | < 2.5 s | ❌ |
| INP    | 180 ms          | 220 ms       | < 200 ms | ⚠️ |
| CLS    | 0.18            | 0.22         | < 0.1  | ❌ |
| FCP    | 1.8 s           | —            | < 1.8 s | ✅ |
| TTFB   | 320 ms          | —            | < 800 ms | ✅ |

Initial bundle: 1.2 MB (budget: 500 KB) ❌

---

### Critical Findings

Issues causing CWV failures in field data, or Lighthouse score < 50.

1. **[LCP] Hero image not using NgOptimizedImage**  
   File: `src/app/home/hero.component.html:12`  
   Impact: LCP +1.8 s (image loads at low priority, no preload)  
   Fix: Replace `<img src="hero.webp">` with `<img ngSrc="hero.webp" width="1200" height="600" priority />`

2. **[CLS] Product images missing dimensions**  
   Files: `src/app/product/product-card.component.html:8`, `src/app/search/result-item.component.html:15`  
   Impact: CLS 0.18 — images shift content below on load  
   Fix: Add `NgOptimizedImage` with `width` and `height` on all product images

---

### High Priority

Issues where CWV is at risk (score 50–89), or bundle significantly over budget.

3. **[INP] Default change detection on 47 components**  
   Run: `grep -rL "OnPush" src/app --include="*.component.ts"`  
   Impact: INP 220 ms in field — full tree check on every interaction  
   Fix: Add `ChangeDetectionStrategy.OnPush` to each component; prioritize those in the critical interaction path (identified via Angular DevTools Profiler)

4. **[Bundle] Initial bundle 1.2 MB — dashboard route is eagerly loaded**  
   File: `src/app/app.routes.ts:18`  
   Impact: +480 KB in initial bundle; dashboard is accessed by < 30% of users on first load  
   Fix: Convert to `loadComponent: () => import('./dashboard/dashboard.component')`

5. **[Bundle] moment.js in initial bundle (298 KB)**  
   Fix: Replace with `date-fns` and import only used functions, or use `Intl.DateTimeFormat` for formatting

---

### Medium Priority

Anti-patterns with no immediate user-visible impact, but risk regressions.

6. **[Reactivity] 3 effects used for state propagation**  
   Files: `src/app/cart/cart.service.ts:45`, `src/app/user/user.store.ts:92`, `src/app/filters/filter.component.ts:31`  
   Risk: Potential infinite update loops; violates effect contract  
   Fix: Replace with `computed()` or `linkedSignal()`

7. **[Memory] 12 RxJS subscriptions without takeUntilDestroyed**  
   Run: `grep -rn "\.subscribe(" src/app --include="*.ts"` (filter for missing cleanup)  
   Risk: Memory leaks in long-running sessions  
   Fix: Add `takeUntilDestroyed(this.destroyRef)` to each subscription

8. **[Network] No preconnect to image CDN**  
   File: `src/index.html`  
   Risk: First image request adds ~200 ms DNS + TLS setup cost  
   Fix: Add `<link rel="preconnect" href="https://images.cdn.example.com" />`

9. **[Network] Web fonts missing font-display**  
   File: `src/styles.css:12`  
   Risk: FOIT on slow connections; contributes to CLS  
   Fix: Add `font-display: swap` to each `@font-face` declaration

---

### Not an Issue

Document ruled-out concerns to avoid re-investigating them.

- SSR: Already enabled with `withHttpTransferCache()` — no duplicate fetches on hydration
- Lazy routing: All non-landing routes are lazy except dashboard (tracked above)
```

---

## Severity Definitions

| Severity | Criteria |
|---|---|
| **Critical** | CWV metric failing in field data, OR Lighthouse performance score < 50, OR bundle > 2× budget |
| **High** | CWV metric at risk (score 50–89), OR bundle 1–2× budget, OR clear user-visible regression in specific interactions |
| **Medium** | Anti-patterns with no current user-visible impact; risk of future regression |

## Per-Finding Format

Each finding must include:

- **[Metric]** affected: LCP, INP, CLS, Bundle, Memory, or Network
- **Location**: file path and line number
- **Impact**: quantified where possible (ms, KB, CLS score contribution)
- **Fix**: specific and actionable — not "improve performance" but the exact code change

Avoid vague findings like "the app is slow" or "consider lazy loading." Every finding must be actionable by a developer who was not part of the audit.
