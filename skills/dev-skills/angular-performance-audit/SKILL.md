---
name: angular-performance-audit
description: Audits existing Angular applications for Core Web Vitals regressions, bundle size issues, change detection inefficiencies, and network loading problems. Trigger when asked to assess, diagnose, improve, or report on performance in an Angular project already in production or active development.
license: MIT
metadata:
  author: Copyright 2026 Google LLC
  version: '1.0'
---

# Angular Performance Audit

This skill diagnoses performance problems in existing Angular applications and produces a prioritized set of findings with actionable fixes. Always measure first — never prescribe fixes based on code inspection alone.

## Diagnostic Workflow

Follow these phases in order. Do not skip measurement.

### Phase 1 — Measure

Establish baseline CWV values before looking at code:

```
1. Run Lighthouse on 2–3 key routes (landing page, primary feature, checkout/conversion)
2. Note which metrics fail: LCP / INP / CLS
3. If web-vitals field data is available (analytics, CrUX), compare lab vs field
4. Record initial bundle sizes: ng build --stats-json
```

Read [audit-cwv.md](references/audit-cwv.md) for measurement procedures.

### Phase 2 — Diagnose by Symptom

Once you know which metric fails, use the targeted diagnostic path:

**LCP > 2.5 s**
- Is it CSR with no SSR/SSG? → Evaluate rendering strategy change
- Hero image without `NgOptimizedImage priority`? → Fix immediately
- LCP element inside `@defer`? → Remove defer from above-fold content
- No `<link rel="preconnect">` to image CDN? → Add to `index.html`

Read [audit-cwv.md](references/audit-cwv.md) → LCP section.

**INP > 200 ms**
- Default change detection strategy on most components? → Audit for `OnPush`
- Heavy synchronous event handlers? → Profile with DevTools, yield to scheduler
- Third-party event-heavy libraries inside Angular's zone? → `runOutsideAngular`
- New project? → Evaluate zoneless Angular

Read [audit-rendering.md](references/audit-rendering.md).

**CLS > 0.1**
- Images without explicit dimensions? → `NgOptimizedImage` with `width`/`height`
- `@defer` blocks without `@placeholder`? → Add sized placeholders
- Web fonts without `font-display`? → Add `swap` + preload for critical fonts
- Dynamic content inserted above existing content after load?

Read [audit-cwv.md](references/audit-cwv.md) → CLS section.

### Phase 3 — Audit Bundle

Read [audit-bundle.md](references/audit-bundle.md) for the full workflow.

```bash
ng build --stats-json
npx source-map-explorer dist/my-app/browser/*.js
```

Look for:
- Routes that should be lazy but are eagerly bundled
- Large third-party deps (moment.js, lodash, heavy chart libraries)
- Duplicate dependencies from mismatched versions
- Primary route that is lazy (should be eager)

### Phase 4 — Audit Change Detection

Read [audit-rendering.md](references/audit-rendering.md).

- Open Angular DevTools → Profiler → record an interaction
- List components with Default change detection strategy
- Identify `effect()` calls that should be `computed()`
- Find subscriptions missing `takeUntilDestroyed()`

### Phase 5 — Audit Network Loading

Read [audit-network.md](references/audit-network.md).

- Check for missing `preconnect` hints to critical origins
- Verify font loading strategy (`font-display`, preload)
- Check HTTP caching headers on static assets
- Look for sequential resource fetches that could be parallelized

### Phase 6 — Produce Findings

Read [audit-checklist.md](references/audit-checklist.md) for the output format.

Produce a structured findings report organized by severity:
- **Critical**: CWV failing in field data or Lighthouse score < 50
- **High**: CWV at risk (score 50–89), bundle significantly over budget
- **Medium**: Anti-patterns with no immediate user-visible impact

Each finding must include: metric affected, file and line reference, and the specific fix.

---

## Tools to Use

| Tool | Purpose |
|---|---|
| Chrome DevTools → Lighthouse | Lab CWV measurement |
| Chrome DevTools → Performance panel | Long task detection, INP profiling |
| Chrome DevTools MCP `lighthouse_audit` | Automated Lighthouse (if MCP available) |
| Angular DevTools → Profiler | Change detection cycle analysis |
| `ng build --stats-json` | Bundle output for analysis |
| `source-map-explorer` | Visual bundle breakdown |
| `web-vitals` npm package | Field measurement instrumentation |
| PageSpeed Insights / CrUX | Real-user (field) data |

---

## References

- [audit-cwv.md](references/audit-cwv.md) — Measuring and diagnosing CWV in Angular
- [audit-bundle.md](references/audit-bundle.md) — Bundle analysis workflow
- [audit-rendering.md](references/audit-rendering.md) — Change detection and rendering profiling
- [audit-network.md](references/audit-network.md) — Resource loading, caching, preloading
- [audit-checklist.md](references/audit-checklist.md) — Prioritized findings output format
