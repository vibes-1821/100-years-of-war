# AGENTS.md — 100 Years of War

This repository contains the Guardian Interactive "100 Years of War" timeline,
originally built in 2014 (jQuery + TimelineJS v2 + Grunt), currently being migrated
to a modern **Svelte 5 + Vite + TypeScript** application.

The migration plan lives in `modernisation/svelte5-upgrade-plan.md`.  
User stories with acceptance criteria live in `modernisation/prd.json`.  
The target git branch is `migration/svelte5-timeline`.

---

## Project Status

The Svelte 5 migration is **in progress**. The legacy source files under `src/`
are the reference implementation. All new code goes into the Svelte 5 project
scaffolded at the repository root (or a dedicated sub-directory per the scaffold
step in the migration plan).

---

## Build, Dev, and Test Commands

### Legacy (Grunt — original 2014 code)

```bash
npm install              # Install Grunt + Tabletop + S3 deps
grunt                    # Start local dev server on http://localhost:9001 (serves src/)
grunt pushdata           # Fetch Google Sheets data and upload JSONP to S3
grunt deploy             # Full deployment to Guardian CDN (S3)
```

### Svelte 5 / Vite (target architecture)

Once the project is scaffolded these are the expected commands:

```bash
npm install              # Install all dependencies
npm run dev              # Vite dev server with HMR
npm run build            # Production build (static output)
npm run preview          # Preview the production build locally
npm run check            # svelte-check — TypeScript + Svelte type checking
npm run lint             # ESLint across src/
npm run format           # Prettier — format all files
```

### Testing (Vitest + Playwright)

```bash
npm run test             # Run all unit tests (Vitest)
npm run test:watch       # Vitest in watch mode
npm run test -- --reporter=verbose   # Verbose output per test

# Run a single test file
npx vitest run src/lib/utils/dates.test.ts

# Run tests matching a name pattern
npx vitest run -t "parses comma-separated date"

# End-to-end tests (Playwright)
npx playwright test
npx playwright test tests/navigation.spec.ts   # Single e2e spec
npx playwright test --headed                   # Headed mode for debugging
```

---

## Directory Structure (Svelte 5 target)

```
src/
  app.css                        # Global CSS: variables, fonts, reset
  App.svelte                     # Root component; composes Slider + TimeNav
  lib/
    components/
      Slide.svelte               # Single event: headline, date, body, media
      SlideContent.svelte        # Text portion of a slide
      Slider.svelte              # Scrollable slide track (scroll-snap)
      MediaElement.svelte        # Dispatches to Image/Video sub-components
      ImageMedia.svelte
      VideoPlayer.svelte
      TimeNav.svelte             # Bottom chronological navigation bar
      TimeAxis.svelte            # Tick marks and era labels
      TimeMarker.svelte          # Clickable event marker
      NavigationControls.svelte  # Prev/next buttons
    state/
      timelineState.svelte.ts    # Module-level Svelte 5 $state / $derived
    types.ts                     # TimelineEvent, TimelineEra, TimelineAsset
    utils/
      dates.ts                   # "YYYY,MM,DD" ↔ Date conversions
      sanitize.ts                # DOMPurify wrapper for HTML fields
    data/
      timeline.json              # Converted from src/datahttpsfix.jsonp
modernisation/
  svelte5-upgrade-plan.md        # Full migration plan
  prd.json                       # User stories with acceptance criteria
src/ (legacy)
  index.html                     # Original TimelineJS entry point
  datahttpsfix.jsonp             # Raw timeline data (JSONP wrapper)
  timelinejs/                    # Vendored TimelineJS v2 — DO NOT EDIT
```

---

## Code Style Guidelines

### Language and Framework

- **TypeScript** everywhere in the Svelte 5 project — no plain `.js` files.
- **Svelte 5 with Runes** — use `$state`, `$derived`, `$effect`, `$props`.  
  Never use Svelte 4 stores (`writable`, `readable`, `derived` from `svelte/store`).
- `lang="ts"` on all `<script>` blocks in `.svelte` files.

### TypeScript

- Strict mode is enabled (`"strict": true` in `tsconfig.json`).
- Always type function parameters and return values explicitly.
- Prefer `interface` over `type` for object shapes.
- Use the interfaces in `src/lib/types.ts`; do not inline ad-hoc object types.
- Avoid `any`; use `unknown` and narrow with guards where the type is truly dynamic.
- Use `satisfies` instead of `as` casts where possible.

```typescript
// Good
function parseDate(raw: string): Date { ... }

// Bad
const parseDate = (raw) => { ... }
```

### Svelte 5 Runes Patterns

```typescript
// State store pattern (src/lib/state/timelineState.svelte.ts)
let activeIndex = $state(0);
const currentEvent = $derived(events[activeIndex]);

// Component props
const { event }: { event: TimelineEvent } = $props();

// Effects
$effect(() => {
  iframeMessenger.resize();
});
```

- Keep `$effect` bodies free of side-effects that belong in event handlers.
- Never reassign `$derived` values — they are read-only.
- Use module-level `$state` in `.svelte.ts` files for shared cross-component state.

### Naming Conventions

| Entity | Convention | Example |
|---|---|---|
| Components | PascalCase `.svelte` | `TimeMarker.svelte` |
| Variables / functions | camelCase | `activeIndex`, `parseDate()` |
| TypeScript interfaces | PascalCase | `TimelineEvent` |
| CSS custom properties | `--kebab-case` | `--color-accent` |
| CSS class names | `kebab-case` | `slide-content` |
| Test files | `*.test.ts` | `dates.test.ts` |
| E2E specs | `*.spec.ts` | `navigation.spec.ts` |

### Imports

- Use the `$lib` alias for imports within `src/lib/`.
- Group imports: (1) Svelte/framework, (2) third-party, (3) internal `$lib`, (4) relative.
- No default re-exports from barrel files unless explicitly needed.

```typescript
import { onMount } from 'svelte';
import DOMPurify from 'dompurify';
import type { TimelineEvent } from '$lib/types';
import { parseDate } from '$lib/utils/dates';
```

### Formatting (Prettier)

- 2-space indentation.
- Single quotes for strings in TypeScript/JavaScript.
- No trailing semicolons (Prettier default for Svelte projects).
- 100-character print width.
- Trailing commas in multi-line structures (`"trailingComma": "all"`).

### Error Handling

- Never swallow errors silently. At minimum, `console.error` with context.
- In data utilities, throw `Error` with descriptive messages rather than returning
  `null` or `undefined` on unexpected input.
- Media components must implement a visible fallback UI for broken/missing URLs
  (stale 2014-era CDN links are a known issue).
- Sanitise all HTML from the data layer with DOMPurify before using `{@html}`.

```svelte
<!-- Good -->
{@html sanitized}

<!-- Bad — never do this with untrusted data -->
{@html event.text}
```

### CSS and Styling

- All design tokens (colours, fonts, spacing) are defined as CSS custom properties
  in `src/app.css`. Reference tokens in component styles; do not hard-code values.

```css
/* Canonical tokens */
--color-accent: #FFAD00;
--color-background-dark: #222;
--font-headline: 'Guardian Egyptian Headline', Georgia, serif;
--space-2: 16px;
```

- Use scoped `<style>` blocks in Svelte components. Avoid global selectors except
  in `app.css`.
- Layout must use CSS Grid or Flexbox — no absolute positioning.
- All interactive elements must have a minimum touch target of 44×44 px.
- Respect `prefers-reduced-motion` for all transitions and animations.

### Accessibility

- Use semantic HTML (`<article>`, `<nav>`, `<time>`, `<figure>`, `<figcaption>`).
- Every `<img>` requires a meaningful `alt` attribute.
- Navigation controls need `aria-label` attributes.
- ARIA live regions must announce slide changes to screen readers.
- The app must be fully keyboard-navigable (arrow keys, Tab, Enter, Space,
  Home, End).

---

## Guardian-Specific Conventions

- External links inside the timeline must be intercepted and opened via
  `iframeMessenger.navigate(href)` rather than native browser navigation,
  to work correctly when embedded in a Guardian article iframe.
- `iframeMessenger.enableAutoResize()` must be called on app mount.
- Detect iframe context before calling messenger APIs to allow standalone
  development without errors.
- Guardian fonts are loaded from `pasteup.guim.co.uk` CDN via `fonts.css`.
  Do not inline or bundle them.

---

## Performance Targets

| Metric | Target |
|---|---|
| JS bundle (gzipped) | < 50 KB |
| First Contentful Paint | < 1.5 s |
| Time to Interactive | < 2.5 s |

Use `npm run build` and inspect `dist/` with `npx vite-bundle-visualizer` to
check bundle size.
