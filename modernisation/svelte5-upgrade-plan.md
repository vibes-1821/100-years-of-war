# Migration Plan: "100 Years of War" Guardian Interactive — jQuery/TimelineJS to Svelte 5

## Overview

Migrate the Guardian Interactive "100 Years of War" timeline (a chronicle of British military conflicts from 1914–2014) from its current jQuery/TimelineJS v2 implementation to a modern Svelte 5 application. The original was built in 2014 using a monolithic ~6000-line JavaScript library, jQuery 1.7.2, JSONP data loading, and a Grunt build system. The target is a lightweight, accessible, maintainable Svelte 5 codebase that faithfully reproduces the original timeline experience.

---

## Current Architecture

### Technology Stack

| Layer | Current Technology | Notes |
|---|---|---|
| Core Framework | TimelineJS v2.29.0 | Monolithic ~6000-line JS file |
| DOM / Animation | jQuery 1.7.2 | Used for all DOM manipulation, animation, AJAX |
| Data Format | JSONP | `storyjs_jsonp_data()` callback wrapping JSON |
| Data Source | Google Spreadsheets via Tabletop.js | Cached as static JSONP files |
| Build System | Grunt | Local dev server, S3 deployment, data fetching |
| Embedding | guardian/iframe-messenger | For embedding in Guardian article iframes |
| Fonts | Guardian custom web fonts | Loaded from pasteup.guim.co.uk CDN |
| Deployment | AWS S3 / Guardian CDN | Static file hosting |

### Key Source Files

| File | Purpose |
|---|---|
| `src/index.html` | Entry point; configures TimelineJS options (100% width, 1200px height, JSONP source, watercolor map tiles) |
| `src/datahttpsfix.jsonp` | Timeline data: ~40 war events with dates, headlines, body text, and media assets |
| `src/timelinejs/js/timeline.js` | Entire TimelineJS framework: VMM core, browser detection, date parsing, external API integrations, slider system, drag handling, time navigation, data parsing |
| `src/timelinejs/js/storyjs-embed.js` | Bootstrap/loader that lazy-loads dependencies then initialises VMM.Timeline |
| `src/timelinejs/css/timeline.css` | Heavily customised styles: Guardian fonts, custom colours (#FFAD00 accent, #222 slider background), marker/flag styles |
| `src/timelinejs/css/fonts.css` | @font-face declarations for Guardian Agate Sans, Guardian Egyptian Text, Guardian Egyptian Headline |
| `Gruntfile.js` | Build tasks: local server (port 9001), Google Sheets fetch + S3 upload, full deployment |
| `fetchData.js` | Node script using Tabletop.js to pull data from a Google Spreadsheet and output as JSONP |
| `package.json` | Project config with Grunt, Tabletop, and S3 deployment dependencies |

### Data Structure

Each timeline event contains:

- `startDate` / `endDate` — comma-separated date strings (`"YYYY,MM,DD"`)
- `headline` — event title
- `text` — HTML body content (contains `<br>`, links, and in some cases full `<video>` elements)
- `asset` — object with `media` (URL or HTML), `credit`, and `caption`

Media types include static images (Guardian CDN URLs), HTML5 `<video>` elements (with ogv, webm, m3u8, mp4 sources), and potentially external embeds.

---

## Target Architecture

### Technology Stack

| Layer | Target Technology |
|---|---|
| Framework | Svelte 5 (runes-based reactivity) |
| Build System | Vite |
| Language | TypeScript |
| Data Format | Static typed JSON |
| Styling | Scoped Svelte styles + CSS custom properties |
| Embedding | @guardian/iframe-messenger (current npm version) |
| Fonts | Guardian custom web fonts (preserved) |
| Testing | Vitest + Svelte Testing Library + Playwright |

---

## Phase 1: Project Setup and Data Layer

### 1.1 Scaffold the Project

- Initialise a Svelte 5 + Vite project with TypeScript
- Configure for static site output (pre-rendered / SPA suitable for iframe embedding)
- Set up ESLint, Prettier, and project structure
- Add `.env` or config module for Guardian CDN paths and deployment targets

### 1.2 Convert Data from JSONP to Typed JSON

- Strip the `storyjs_jsonp_data()` wrapper from `datahttpsfix.jsonp` to produce clean JSON
- Create TypeScript interfaces:

```typescript
interface TimelineAsset {
  media: string;    // URL or HTML string (may contain <video> elements)
  credit: string;
  caption: string;
}

interface TimelineEvent {
  startDate: string;      // "YYYY,MM,DD" format
  endDate: string;
  headline: string;
  text: string;           // HTML content
  asset: TimelineAsset;
  tag?: string;
}

interface TimelineEra {
  startDate: string;
  endDate: string;
  headline: string;
  tag?: string;
}

interface TimelineData {
  timeline: {
    type: string;
    date: TimelineEvent[];
    era?: TimelineEra[];
  };
}
```

- Parse and validate all ~40 events
- Build a utility to convert comma-separated date strings into proper `Date` objects
- Audit embedded HTML in `text` fields for sanitisation needs
- Identify and catalogue all video entries (events with `<video>` tags in `asset.media`)

### 1.3 Optional: Data Fetch Script

- If Google Sheets integration is still desired, create a modern Node fetch script replacing `fetchData.js` + Tabletop.js
- Otherwise, maintain the static JSON file directly as the source of truth

---

## Phase 2: Component Architecture

### 2.1 Component Tree

```
App.svelte
├── Timeline.svelte
│   ├── SlideViewer.svelte
│   │   └── Slide.svelte (one per event, transitions in/out)
│   │       ├── SlideContent.svelte (headline, date, body text)
│   │       └── MediaElement.svelte
│   │           ├── ImageMedia.svelte
│   │           └── VideoPlayer.svelte
│   ├── TimeNav.svelte
│   │   ├── TimeAxis.svelte (date labels, era groupings)
│   │   └── TimeMarker.svelte (per-event marker/flag)
│   └── NavigationControls.svelte (prev/next buttons)
```

### 2.2 Component Responsibilities

| Component | Responsibility |
|---|---|
| **App.svelte** | Top-level layout; loads data; provides global dimensions |
| **Timeline.svelte** | Orchestrates slide viewer and time nav; handles keyboard navigation |
| **SlideViewer.svelte** | Displays one event at a time; manages slide transitions |
| **Slide.svelte** | Renders a single event: headline, date range, body text, media |
| **SlideContent.svelte** | Headline, formatted dates, and HTML body text |
| **MediaElement.svelte** | Determines media type and delegates to appropriate sub-component |
| **ImageMedia.svelte** | Renders images with lazy loading, alt text, proper sizing |
| **VideoPlayer.svelte** | Native HTML5 video with multiple sources and accessible controls |
| **TimeNav.svelte** | Bottom navigation bar; scrollable timeline with markers |
| **TimeAxis.svelte** | Horizontal axis with date labels and era indicators |
| **TimeMarker.svelte** | Individual clickable marker/flag on the navigation bar |
| **NavigationControls.svelte** | Previous/next buttons |

---

## Phase 3: State Management with Svelte 5 Runes

### 3.1 Core Reactive State (`$state`)

| State Variable | Type | Purpose |
|---|---|---|
| `currentIndex` | `number` | Index of the active event |
| `viewportWidth` | `number` | Current viewport width for responsive layout |
| `viewportHeight` | `number` | Current viewport height |
| `isNavigating` | `boolean` | Animation lock during transitions |
| `timeNavScrollPosition` | `number` | Scroll offset of the bottom nav bar |

### 3.2 Derived Values (`$derived`)

| Derived Value | Source | Purpose |
|---|---|---|
| `currentEvent` | `currentIndex` + events array | The active event object |
| `previousEvent` | `currentIndex - 1` | For pre-loading and "previous" display |
| `nextEvent` | `currentIndex + 1` | For pre-loading and "next" display |
| `timelineSpan` | First and last event dates | Total date range for axis calculation |
| `markerPositions` | Events array + nav width | Computed pixel positions for each marker |
| `hasPrevious` / `hasNext` | `currentIndex` bounds | Enable/disable nav buttons |

### 3.3 Effects (`$effect`)

- Update iframe height via iframeMessenger when slide content changes
- Track viewport resize via `ResizeObserver` or `svelte:window`
- Optional: analytics tracking on event navigation
- Optional: browser history / deep-linking to specific events

### 3.4 Shared State Module

Create `lib/timeline-state.svelte.ts` using Svelte 5 module-level `$state` for cross-component state, avoiding the need for a separate store library.

---

## Phase 4: Interaction and Animation

### 4.1 Replace jQuery Animation

| Original (jQuery) | Replacement |
|---|---|
| `$.animate()` for slide transitions | Svelte `fly`, `fade`, `slide` transitions |
| `$.animate()` for nav scrolling | CSS `scroll-behavior: smooth` or `requestAnimationFrame` |
| jQuery `.css()` manipulation | Svelte reactive style bindings |
| `VMM.Util.animate()` custom wrapper | CSS transitions + Svelte transition directives |

### 4.2 Replace DragSlider with Pointer Events

- Implement `pointerdown`, `pointermove`, `pointerup` handlers for drag-to-navigate on both slide viewer and time nav
- Add momentum/inertia scrolling using `requestAnimationFrame` (replacing jQuery's animation queue)
- Support mouse, touch, and pen input through a single Pointer Events code path
- Implement snap-to-event behaviour after drag release

### 4.3 Keyboard Navigation

- Arrow keys (left/right) to navigate between events
- Home/End to jump to first/last event
- Enter/Space to activate focused markers in the time nav
- Tab navigation through interactive elements

### 4.4 Event Handling

- Replace jQuery `.on('click', selector)` delegation with Svelte `onclick` handlers
- Replace `$(window).resize()` with `<svelte:window bind:innerWidth bind:innerHeight />`
- Replace `$(document).keydown()` with `<svelte:window onkeydown={handler} />`

---

## Phase 5: CSS Migration

### 5.1 Design Tokens and Variables

Extract from the original `timeline.css` into a central `variables.css`:

```css
:root {
  /* Colours */
  --color-accent: #FFAD00;
  --color-background-dark: #222;
  --color-background-light: #fff;
  --color-text-primary: #333;
  --color-text-secondary: #666;
  --color-border: #e5e5e5;

  /* Typography */
  --font-headline: 'Guardian Egyptian Headline', Georgia, serif;
  --font-body: 'Guardian Egyptian Text', Georgia, serif;
  --font-sans: 'Guardian Agate Sans', Arial, sans-serif;

  /* Spacing (8px system) */
  --space-1: 8px;
  --space-2: 16px;
  --space-3: 24px;
  --space-4: 32px;
  --space-6: 48px;
  --space-8: 64px;

  /* Layout */
  --slide-max-width: 1200px;
  --timenav-height: 200px;
}
```

### 5.2 Component-Scoped Styles

- Migrate relevant portions of `timeline.css` into each component's `<style>` block
- Preserve the distinctive visual identity: gold accent markers, dark navigation bar, white content areas
- Convert `.vco-*` class naming to semantic, component-scoped names

### 5.3 Font Loading

- Preserve `fonts.css` as a global stylesheet loaded at the app level
- Keep @font-face declarations pointing to Guardian font CDN (`pasteup.guim.co.uk`)
- Use `font-display: swap` for better loading performance

### 5.4 Responsive Design

- Replace `VMM.Browser` detection with CSS media queries and container queries
- Define breakpoints for mobile, tablet, and desktop
- Adjust time nav layout, slide content sizing, and font sizes per breakpoint
- Ensure touch-friendly hit targets on mobile (minimum 44px)

---

## Phase 6: Media Handling

### 6.1 Static Images

- Render with native `<img>` using `loading="lazy"` for off-screen events
- Add `srcset` / `sizes` attributes if multiple resolutions are available
- Derive `alt` text from `asset.caption` or `headline`
- Ensure proper aspect ratio handling to prevent layout shift

### 6.2 HTML5 Video

Several events embed `<video>` tags directly in `asset.media` with multiple sources:

- Parse the video HTML from the data to extract source URLs and types
- Build `VideoPlayer.svelte` with native `<video>` element
- Support source formats: MP4, WebM, OGV, HLS (m3u8)
- Evaluate whether HLS and OGV sources are still necessary for modern browser support (MP4 + WebM likely sufficient)
- Add accessible playback controls
- Implement poster images where available

### 6.3 Media URL Audit

- All media URLs reference 2014-era Guardian CDN paths (`static.guim.co.uk`)
- Verify each URL is still accessible
- Document any broken links and identify replacement assets or fallbacks

---

## Phase 7: Guardian Embedding and iframe Integration

### 7.1 iframe-messenger Compatibility

- Install `@guardian/iframe-messenger` from npm (current version)
- Call auto-resize on app initialisation
- Update iframe height dynamically when slide content changes (variable height events)

### 7.2 External Link Handling

- Replicate the original behaviour: intercept all `<a>` clicks within the timeline
- Use `iframeMessenger.navigate()` to open links in the parent Guardian page (not inside the iframe)
- Ensure links work normally when running standalone (outside an iframe)

### 7.3 Dual-Mode Operation

- The interactive must work both:
  - **Embedded**: inside a Guardian article iframe with iframeMessenger
  - **Standalone**: directly accessible for development and preview
- Detect iframe context and conditionally enable messenger features

---

## Phase 8: Accessibility

### 8.1 Semantic Structure

- Use appropriate ARIA roles: `role="tablist"` for time nav markers, `role="tabpanel"` for slide content (or equivalent landmark roles)
- Add `aria-label` attributes to navigation controls
- Use semantic HTML elements (`<article>`, `<nav>`, `<time>`, `<figure>`, `<figcaption>`)

### 8.2 Keyboard Support

- Full keyboard navigation (arrow keys, Tab, Enter, Space, Home, End)
- Visible focus indicators on all interactive elements
- Focus management: move focus appropriately when navigating between slides

### 8.3 Screen Reader Support

- ARIA live regions to announce slide changes
- Meaningful alt text on all images
- Accessible video controls with proper labels
- Date formatting that reads naturally (not raw comma-separated strings)

### 8.4 Visual Accessibility

- Verify colour contrast ratios meet WCAG AA:
  - Gold `#FFAD00` on white and dark backgrounds
  - Body text on all background colours
- Ensure text remains readable at 200% zoom
- Respect `prefers-reduced-motion` for transitions and animations

---

## Phase 9: Testing

### 9.1 Unit / Component Tests (Vitest + Svelte Testing Library)

- Data parsing: date string conversion, media type detection, HTML sanitisation
- Component rendering: each component renders correctly with sample data
- Navigation logic: index management, bounds checking, keyboard handling
- State management: derived values compute correctly

### 9.2 End-to-End Tests (Playwright)

- Timeline loads and displays the first event
- Forward/backward navigation works (click, keyboard, drag)
- Time nav markers are clickable and navigate to correct events
- Media renders correctly (images load, video players appear)
- Responsive behaviour at mobile, tablet, and desktop widths
- iframe embedding works correctly with iframeMessenger

### 9.3 Visual Regression

- Screenshot comparison of key slides against the original implementation
- Verify marker/flag styles on the time navigation bar match the original design

---

## Phase 10: Build, Performance, and Deployment

### 10.1 Build Configuration

- Vite build with static output
- Tree-shaking and minification
- Asset hashing for cache busting
- Source maps for debugging

### 10.2 Performance Targets

| Metric | Target | Rationale |
|---|---|---|
| Bundle size (JS) | < 50KB gzipped | Original loaded jQuery (94KB) + TimelineJS (~200KB) |
| First Contentful Paint | < 1.5s | Fast initial render of first slide |
| Time to Interactive | < 2.5s | All navigation functional |
| Largest Contentful Paint | < 3s | Including first slide's media |

### 10.3 Performance Techniques

- Lazy-load media for off-screen events
- Pre-render the initial slide for fast first paint
- Use Svelte's minimal runtime overhead advantage
- Optimise font loading with `font-display: swap` and preload hints
- Consider `<link rel="preload">` for the first slide's image

### 10.4 Deployment

- Configure static asset output compatible with Guardian's hosting infrastructure
- Set appropriate cache headers for assets
- Verify the built output works when served from S3/CDN and embedded via iframe

---

## Implementation Order

| Step | Phase | Description |
|---|---|---|
| 1 | Phase 1 | Scaffold Svelte 5 + Vite + TypeScript project |
| 2 | Phase 1 | Convert JSONP data to typed JSON; build date parsing utilities |
| 3 | Phase 5 | Set up global styles: CSS variables, fonts, reset |
| 4 | Phase 2 | Build `App.svelte` and `Timeline.svelte` shell with layout |
| 5 | Phase 3 | Implement core state management (current index, navigation) |
| 6 | Phase 2 | Build `Slide.svelte` and `SlideContent.svelte` |
| 7 | Phase 2 | Build `SlideViewer.svelte` with transitions |
| 8 | Phase 6 | Build `MediaElement.svelte`, `ImageMedia.svelte`, `VideoPlayer.svelte` |
| 9 | Phase 2 | Build `TimeNav.svelte`, `TimeAxis.svelte`, `TimeMarker.svelte` |
| 10 | Phase 2 | Build `NavigationControls.svelte` |
| 11 | Phase 4 | Implement drag/swipe navigation with Pointer Events |
| 12 | Phase 4 | Implement keyboard navigation |
| 13 | Phase 7 | Integrate iframe-messenger for Guardian embedding |
| 14 | Phase 8 | Accessibility audit and fixes |
| 15 | Phase 5 | Responsive design refinement |
| 16 | Phase 9 | Write tests |
| 17 | Phase 10 | Performance optimisation and deployment configuration |

---

## Risks and Considerations

| Risk | Mitigation |
|---|---|
| Stale media URLs (2014-era CDN links) | Audit all URLs early; identify replacements before building media components |
| Guardian font licensing / CDN availability | Verify fonts still load from pasteup.guim.co.uk; prepare fallback font stack |
| Video format compatibility | Test m3u8/ogv/webm/mp4 sources; likely can simplify to MP4 + WebM only |
| Faithful visual reproduction | Use screenshot comparison against the original throughout development |
| iframe-messenger API changes | Check current @guardian/iframe-messenger npm package API against original usage |
| Embedded HTML in event text fields | Sanitise HTML content; use `{@html}` carefully with known-safe content |

---

## Success Criteria

- [ ] All ~40 timeline events render correctly with their original content and media
- [ ] Navigation works via click, keyboard, and drag/swipe
- [ ] Time navigation bar displays correctly with markers at appropriate positions
- [ ] Visual design matches the original (Guardian fonts, gold accent, dark nav bar)
- [ ] Works when embedded in a Guardian article iframe
- [ ] All media (images and videos) loads and displays correctly
- [ ] Keyboard accessible and screen reader compatible
- [ ] Bundle size under 50KB gzipped (JS)
- [ ] All tests pass
- [ ] Builds and deploys as a static site
