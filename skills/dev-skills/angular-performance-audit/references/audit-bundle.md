# Bundle Analysis Workflow

Large JavaScript bundles delay LCP by blocking the main thread during parse and execution. This reference covers how to measure and reduce bundle size in Angular applications.

## Generate Bundle Stats

```bash
ng build --stats-json
```

This creates `dist/my-app/browser/stats.json`. Then analyze it visually:

```bash
npx source-map-explorer dist/my-app/browser/*.js
# or
npx webpack-bundle-analyzer dist/my-app/browser/stats.json
```

`source-map-explorer` shows a treemap of which modules contribute to each chunk. `webpack-bundle-analyzer` shows a zoomable interactive treemap.

## What to Look For

### Routes that should be lazy but are eager

In the treemap, check whether large feature modules appear in the `main` chunk. If a route is not the primary landing page and its components appear in `main`, it should be lazy-loaded.

```ts
// Change from eager:
{ path: 'dashboard', component: DashboardComponent }

// To lazy:
{ path: 'dashboard', loadComponent: () => import('./dashboard/dashboard.component').then(m => m.DashboardComponent) }
```

### Landing page route that is lazy

The opposite problem: the primary route that every user hits is lazy-loaded, requiring an extra round-trip before rendering.

```ts
// WRONG: user always visits home — lazy adds unnecessary latency
{ path: '', loadComponent: () => import('./home/home.component') }

// CORRECT: eager
{ path: '', component: HomeComponent }
```

### Large third-party dependencies

Common offenders and their alternatives:

| Library | Size | Replacement |
|---|---|---|
| moment.js | ~300 KB | `date-fns` (~20 KB for used functions) or `Temporal` (native) |
| lodash (full) | ~70 KB | `lodash-es` with tree shaking, or native array methods |
| jQuery | ~90 KB | Native DOM APIs |
| chart.js (full) | ~200 KB | Import only used chart types; or use a lighter lib |

### Duplicate dependencies

Two versions of the same library in the bundle indicate version conflicts between packages. Check with:

```bash
npm ls <package-name>
```

Resolve by aligning versions in `package.json` or adding a `resolutions` entry (Yarn) or `overrides` (npm/pnpm).

## Budget Enforcement

Angular supports bundle budgets in `angular.json`:

```json
"budgets": [
  {
    "type": "initial",
    "maximumWarning": "500kb",
    "maximumError": "1mb"
  },
  {
    "type": "anyComponentStyle",
    "maximumWarning": "2kb",
    "maximumError": "4kb"
  }
]
```

Set `maximumError` to fail the build when the budget is exceeded. Set `maximumWarning` to catch regressions early.

## Code Splitting Verification

After adding lazy routes, verify that chunks are created:

```bash
ng build
ls -lh dist/my-app/browser/chunk-*.js
```

Each lazy route should produce a separate chunk. If a lazy route's component appears in `main.js`, check for circular imports or for the component being directly imported (not via `loadComponent`) somewhere in the eager module graph.

## Differential Loading

Angular builds modern (`es2020+`) and legacy bundles automatically. Verify that the build output includes both and that the modern bundle is the one being served to modern browsers:

```bash
ls dist/my-app/browser/*.js | head -20
```

Look for `-es2015` or `-es5` suffixes. If only one bundle is present, check the `target` setting in `tsconfig.json` and the browser support configuration in `angular.json`.
