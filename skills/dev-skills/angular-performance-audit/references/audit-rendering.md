# Change Detection and Rendering Profiling

Poor change detection efficiency is the primary cause of Angular INP regressions. This reference covers how to profile, identify, and fix change detection problems.

## Angular DevTools Profiler

The Angular DevTools browser extension provides a change detection profiler:

1. Install **Angular DevTools** from the Chrome Web Store
2. Open DevTools → **Angular** tab → **Profiler**
3. Click **Start recording**
4. Perform the interaction that feels slow (click, keypress, form input)
5. Click **Stop recording**
6. Inspect the flame chart

The profiler shows:
- Which components ran change detection
- How many milliseconds each check took
- Which change detection cycle was triggered by which event

A healthy interaction: 1–3 components checked, each < 2 ms.  
A problematic interaction: 50+ components checked, total > 16 ms.

## Identifying Components with Suboptimal Change Detection

### v22+ — OnPush is the default

From v22, `OnPush` is the default CD strategy. Components re-render only when inputs change, signals update, or events fire within them. There is no need to audit for missing `OnPush` declarations.

Focus instead on signals anti-patterns that undermine reactivity: effects used for state propagation, and signals read after async boundaries (see sections below).

### v20 and earlier — Audit for OnPush

In zone-based Angular apps (v20 and earlier), the default CD strategy checks the entire component tree on every event. Search for components that have not opted in to `OnPush`:

```bash
# Find components that do NOT have OnPush
grep -rL "OnPush" src/app --include="*.ts" | grep "\.component\.ts$"
```

This lists component files that never reference `OnPush`. Cross-reference with the DevTools profiler to prioritize which ones cause the most CD work.

```ts
import { ChangeDetectionStrategy, Component } from '@angular/core';

@Component({
  selector: 'app-user-card',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `...`
})
export class UserCard {}
```

After adding `OnPush`, run tests to verify the component still updates correctly. The most common breakage is a component that relied on default CD to detect mutations to object properties without signal or observable notification.

## Diagnosing Effects Used for State Propagation

Search for `effect()` calls that write to signals:

```bash
grep -rn "effect(" src/app --include="*.ts" -A 5 | grep -B 3 "\.set\|\.update"
```

An `effect()` that calls `.set()` or `.update()` on another signal is propagating state — this should be a `computed()` or `linkedSignal()` instead.

```ts
// WRONG: effect for state propagation
effect(() => {
  this.fullName.set(`${this.firstName()} ${this.lastName()}`);
});

// CORRECT: computed for derived state
fullName = computed(() => `${this.firstName()} ${this.lastName()}`);
```

## Finding Missing `takeUntilDestroyed`

RxJS subscriptions without cleanup leak after component destruction and may trigger updates in destroyed components.

```bash
grep -rn "\.subscribe(" src/app --include="*.ts" -B 2 | grep -v "takeUntil\|takeUntilDestroyed\|async pipe\|firstValueFrom\|lastValueFrom"
```

Each raw `.subscribe()` call should use `takeUntilDestroyed(this.destroyRef)`:

```ts
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { DestroyRef, inject } from '@angular/core';

private destroyRef = inject(DestroyRef);

ngOnInit() {
  this.data$.pipe(
    takeUntilDestroyed(this.destroyRef)
  ).subscribe(value => this.value.set(value));
}
```

## Third-Party Code and Zone.js

> **Zone-based apps only (v20 and earlier).** From v21, new Angular projects are zoneless by default and Zone.js is absent. This section does not apply to v21+ applications.

Libraries that register their own event listeners (maps, charts, WebSocket clients, analytics) run inside Angular's zone by default, triggering change detection on every event.

Identify the source of unexpected CD cycles in the DevTools profiler by looking at the call stack. If the top frame is a third-party library rather than an Angular component, it should run outside the zone.

```ts
@Component({ ... })
export class MapComponent implements AfterViewInit {
  private ngZone = inject(NgZone);
  private map!: SomeMapLibrary;

  ngAfterViewInit() {
    this.ngZone.runOutsideAngular(() => {
      this.map = new SomeMapLibrary(this.mapEl.nativeElement);
      this.map.on('move', this.onMapMove.bind(this));
    });
  }

  private onMapMove(e: MapEvent) {
    // Update raw properties here — no CD triggered
    // If you need to update Angular state, re-enter the zone:
    // this.ngZone.run(() => this.position.set(e.center));
  }
}
```

## Migrating to Zoneless Angular

> **Applies to v20 and earlier.** From v21, new Angular projects are zoneless by default. This section covers migrating an existing zone-based app.

Zoneless Angular eliminates Zone.js overhead entirely. The stable API is available from v19:

```ts
// app.config.ts
import { provideZonelessChangeDetection } from '@angular/core';

export const appConfig: ApplicationConfig = {
  providers: [
    provideZonelessChangeDetection()  // stable from v19; default in new projects from v21
  ]
};
```

Remove `zone.js` from `polyfills` in `angular.json`.

Zoneless requires:
- All async state flows through signals or `AsyncPipe`
- No `detectChanges()` or `markForCheck()` calls (replace with signal updates)
- All component tests updated to use `fixture.whenStable()` instead of `fixture.detectChanges()`

Run the full test suite with zoneless enabled to identify incompatible patterns before committing to the migration.
