# OC MENA Festival — React Port Handoff

**From:** Wahhab (static build)
**To:** Moe & Zubaid (React port)
**Date:** 2026-04-17
**Source of truth:** `DESIGN_SYSTEM.md` + `music-entertainment.html`
**Codename:** Heritage Hearth / Heritage Cinema

This document hands off the static OC MENA Festival site to a React implementation. It describes what was built, what conventions were baked in, and what the dev team (Moe & Zubaid) should know before starting the port. Read this end-to-end before cloning a component into JSX.

---

## 1. Architecture overview

The current site is a **plain static HTML site**. There is no build step, no bundler, no package manager. Each page is a self-contained `.html` file with an inline `<style>` block at the top and an inline `<script>` block at the bottom. Styling is hand-written CSS using custom properties; there is no CSS framework and no Tailwind.

- **Fonts:** Newsreader and Inter are loaded from Google Fonts via `<link>` on every page. Aldhabi is loaded locally from `./assets/fonts/aldhabi.ttf` via `@font-face`. The local TTF is intentional — do NOT replace Aldhabi with a Google Fonts Aldhabi in the React port. The local file is the tested render and matches what we ship on mobile Safari without CORS edge cases.
- **Responsive:** All display typography uses `clamp()`; no fixed pixel font-sizes on headings. Breakpoints are applied at `960px` (desktop → tablet) and `640px` (tablet → mobile).
- **No build output:** Every page serves its CSS from its own `<style>` block. When porting to React, lift the tokens and shared component CSS to a single stylesheet or Tailwind theme — do not copy per-page inline styles verbatim; many duplicate across pages.
- **Coming-soon stubs:** `tickets.html`, `schedule.html`, `map.html` ship as Heritage Hearth "coming soon" pages with full nav + footer + eyebrow + CTAs (Notify me / Explore lineup / See map). They are not backed by real data yet. Replace bodies with real pages when the backend track lands; the shell (nav/footer/type) matches every other page.
- **Assets:** Everything lives under `./assets/` — fonts (`aldhabi.ttf`), 30 images (`./assets/images/`), and the hero video (`./assets/hero-final.mp4`, 8.7MB H.264, looping muted). No CDN asset pipeline.

---

## 2. Design tokens

These are the CSS custom properties defined in `:root` on every page. They are identical across all 31 HTML files — port them once to a Tailwind theme (`tailwind.config.js`) or CSS-vars entry in the React app.

### Color tokens

| Token | Value | Role |
|---|---|---|
| `--cream` | `#fff8f4` | Page base — every page starts here |
| `--sand` | `#fff1e4` | Nested section background (one step deeper than cream) |
| `--sand-deep` | `#ffe4c2` | Card / "well" background on sand |
| `--ink` | `#281801` | Primary text + deepest shadow tint — replaces pure black |
| `--ink-soft` | `#5c403c` | Secondary text, captions |
| `--red` | `#dc2626` | Festival red — primary energy color |
| `--red-deep` | `#b70011` | Gradient start for primary CTA |
| `--maroon` | `#930046` | Heritage maroon — depth + tradition, accent text, scrims |
| `--maroon-ink` | `#66002f` | Hover / pressed state for maroon |
| `--gold` | `#f5c842` | Zaffre gold — accent only, never body text |
| `--gold-deep` | `#d0a61e` | Gold on warm backgrounds where contrast is needed |

### Layout tokens

| Token | Value | Role |
|---|---|---|
| `--maxw` | `1200px` | Container max-width |
| `--pad` | `clamp(1.25rem, 5vw, 3rem)` | Horizontal page padding |
| `--nav-h` | `72px` | Fixed nav height |

### Type scale

| Token | Value | Role |
|---|---|---|
| `--fs-display` | `clamp(2.75rem, 6vw, 5.5rem)` | Newsreader, hero headlines |
| `--fs-h1` | `clamp(2rem, 4vw, 3.5rem)` | Newsreader, page titles |
| `--fs-h2` | `clamp(1.5rem, 2.5vw, 2.25rem)` | Newsreader, section titles |
| `--fs-h3` | `clamp(1.125rem, 1.5vw, 1.375rem)` | Inter 600, subsection |
| `--fs-body` | `1rem` | Inter 400, body |
| `--fs-meta` | `0.8125rem` | Inter 500 +0.04em tracking, pills/meta |

### Radii

| Token | Value | Role |
|---|---|---|
| `--r-sm` | `0.5rem` | Pills, small chips |
| `--r-md` | `0.75rem` | Buttons, inputs, nav items |
| `--r-lg` | `1rem` | Cards |
| `--r-xl` | `1.5rem` | Feature tiles, hero media |
| (heritage-tile) | `1.5rem 0.25rem 1.5rem 0.25rem` | Alternating-corner radii, applied to `:nth-child(even)` on card grids (not a named token — computed per component) |

### Non-token conventions

- **Ambient warm shadow** (the only shadow pattern allowed): `0 8px 24px rgba(40, 24, 1, 0.06)`. Never `rgba(0,0,0, ...)`.
- **Default transition:** `all 0.25s ease`.
- **Hover lift:** `transform: translateY(-2px)` on cards and primary CTAs.

---

## 3. Component inventory

Repeated components that should be factored in React. For each: file(s) of origin, variants, data/props needed, and any JS it owns. `music-entertainment.html` is the pattern reference for the top-level shell (nav/footer/closing CTA).

### `Nav` (top + mobile drawer)

- **Origin:** defined inline on every page (31 files); canonical in `music-entertainment.html` lines ~128–252 (CSS) and ~892–939 (markup).
- **Structure:** fixed cream-glassmorphism bar, left logo, middle links (with hover dropdowns on "Experience" and "For Partners"), right `Get Tickets` primary CTA with heartbeat pulse, mobile burger button.
- **Variants:** active link state (gold-deep underline); dropdown parent active state (on `<li class="nav-dropdown is-active">`); scrolled state (adds `.scrolled` on the `<header>` bumping bg opacity from 0.72 → 0.92).
- **Props:** `activePath` (for highlighting current link). Dropdowns are static markup — the items don't change between pages.
- **JS owned:** scroll listener that toggles `.scrolled` at `scrollY > 40`.
- **Recommended factoring:** `<Nav activePath="…" />` consuming a static nav-config object. On mobile, the hamburger opens a full-screen `--cream` panel (NOT a bottom tab bar per DESIGN_SYSTEM §5).

### `MobileMenu`

- **Origin:** inline on every page; markup at `<nav class="mobile-menu" id="mobileMenu">` block directly after `<header class="nav">`.
- **Structure:** full-screen `--cream` panel below the nav. Flat list of every top-level link (dropdown items flattened in). Ends with a primary Get Tickets CTA.
- **Variants:** open / closed (`.is-open`).
- **Props:** `isOpen`, `onClose`. Same nav-config object as `Nav`.
- **JS owned:** burger click handler toggles `.is-open`, swaps icon `☰` ↔ `✕`, and sets `document.body.style.overflow = 'hidden'` when open. Close should mirror.

### `Footer`

- **Origin:** inline on every page; canonical `music-entertainment.html` lines ~791–854 (CSS) and ~1354–1405 (markup).
- **Structure:** deep `--ink` base (not black), 5-column grid (`1.4fr repeat(4, 1fr)`) with brand block + four link columns (Event / Experience / About / Business). Top tonal-shift divider via a background gradient (no 1px border). Bottom row: © line + 4 social links.
- **Brand block:** logo image + Aldhabi tagline ("مهرجان أورنج كاونتي") at `1.85rem` (= 1.7× the `~1.1rem` Latin tagline line) + plain-text tagline.
- **Props:** none really — footer is identical across all 26 pages. Factor as a single `<Footer />` with static link data.

### `PageHead` (hero with scrim + eyebrow + title + meta pills)

- **Origin:** every content page except `index.html` and the 13 artist/comedian pages. Canonical: `music-entertainment.html` lines ~257–361 (CSS) and ~945–967 (markup).
- **Structure:** `position: relative; min-height: 62vh;` centered column. Layers: `.page-head__bg` (absolute inset-0 with a `background-image: url(...)`), `.page-head__scrim` (absolute inset-0 with a maroon → maroon-ink → ink gradient), `.page-head-inner` (z-index 2 content column).
- **Content slots:** eyebrow (Aldhabi gold word over uppercase English label) · H1 title (Newsreader with optional `<em>` italic and `<span class="ar-word">` gold Aldhabi accent) · subcopy · meta pills row.
- **Variants:** hero image path varies per page (see §7 for the full list of pending images). Background-color fallback is always `--maroon`, so pages render correctly even if the JPG is missing.
- **Props:** `heroImage` (url), `eyebrowAr`, `eyebrowEn`, `titleHtml` (accept rich markup — Newsreader + em + ar-word + optional `<br>`), `sub`, `pills: string[]`.
- **JS owned:** none.

### `ArtistHero` (hero variant used only on artist/comedian detail pages)

- **Origin:** `artist-template.html` lines ~250–366 (CSS) and ~858–873 (markup).
- **Structure:** similar to `PageHead` but uses a `--hero-bg` CSS custom property (passed inline as `style="--hero-bg: url(…)"`) and adds an `artist-back` link, origin text line, and an `artist-pills` row with day/stage/variant modifiers (e.g. `.day-fri`, `.day-sat`, `.day-sun`, `.stage`, `.headliner`, `.comedy`).
- **Props:** `artistImage`, `artistNameAr`, `artistName`, `artistOrigin`, `day: 'fri' | 'sat' | 'sun'`, `dayLabel`, `stageLabel`, optional `variantPill` (`headliner` | `comedy`).
- **JS owned:** none.

### `ButtonPrimary` / `ButtonSecondary` / `HeartbeatWrapper`

- **Origin:** inline on every page; canonical `music-entertainment.html` lines ~79–123.
- **Primary:** `.btn-primary` — red-deep → red 135° gradient, white text, Inter 600, `--r-md`, `0.85rem 2rem` padding. Lifts `-2px` on hover. Ambient warm shadow.
- **Secondary:** `.btn-secondary` — `--sand-deep` bg, `--ink` text. Same radius/padding. Lifts `-2px` on hover. On dark closing-CTA backgrounds it's re-styled to `rgba(255,248,244,0.15)` glass + white text (see closing CTA section in `music-entertainment.html` line ~779–786).
- **HeartbeatWrapper:** adding the `.heartbeat` class runs the ring-pulse animation. The animation is defined in two flavors:
  - `heartbeat-maroon` (default `.heartbeat`) — maroon shadow pulse, for buttons on light/cream/glass backgrounds.
  - `heartbeat-gold` — gold shadow pulse, applied via the nested selector `.closing-ctas .heartbeat` inside any dark-maroon closing CTA section. Baked in because gold on cream fails contrast but reads correctly on maroon.
- **Props / API in React:**
  - `<ButtonPrimary variant="default | heartbeat" onDarkBg?>` — when inside a dark maroon closing CTA, pass `onDarkBg` so the component emits the gold-pulse variant.
  - `<ButtonSecondary variant="default | glass">` — glass variant is for dark closing CTAs.
- **JS owned:** none (CSS animation).

### `ArtistCard` (with day-badge, stage-pill variants)

- **Origin:** `music-entertainment.html` lines ~495–679 (CSS) and ~1011–1318 (12 instances) — also used on the index fan carousel as a different `.fan-card` component (see below).
- **Structure:** `<a>` wrapper → `.ac-media` (4:5 photo with maroon scrim + day-badge + caption overlay: Aldhabi name, Newsreader Latin name, country + stage pill) → `.ac-body` (bio + "Artist page" CTA, expand-on-hover).
- **Variants:**
  - Day badge modifier: `.ac-day-badge.fri` (red), `.sat` (maroon), `.sun` (olive-gold `#755b00`), `.comedy` (gold with ink text).
  - Stage pill modifier: `.ac-stage-pill.pac`, `.festival`, `.indoor`.
  - Every even card gets heritage-tile corners via `:nth-child(even)` — this is a style concern, not a prop.
  - `.is-hidden` state (filter hides), `.is-open` state (touch devices show body by default via `@media (hover: none)`).
- **Props:** `href`, `nameAr`, `name`, `country`, `day: 'fri'|'sat'|'sun'`, `dayLabel`, `stage: 'pac'|'festival'|'indoor'`, `stageLabel`, `variantBadge?: 'comedy'`, `image`, `bio`, `ctaLabel` (default "Artist page"; "Read more" for comedians).
- **JS owned:** none on the card itself; the lineup page's filter logic toggles `.is-hidden` externally.

### `FanCard` (home-page carousel card)

- **Origin:** `index.html` lines ~453–596 (CSS) and ~1293–1412 (12 markup instances).
- **Structure:** smaller photo-forward card laid out inside a 3D fan carousel. Has `.fc-img`, `.fc-label` (Aldhabi `.fc-ar` + Latin `.fc-en` + `.fc-meta`), and a small `.fc-cta` that appears when the card is active.
- **Variants:** `:nth-child(even)` heritage-tile corners; `.is-active` / `.is-dragging` states driven by the carousel JS.
- **Props:** `href`, `image`, `nameAr`, `nameEn`, `meta` (origin · stage), `dataIndex` (numeric, used by the carousel).
- **JS owned:** none on the card — see `HomeCarousel` below.

### `HeritageTile` (alternating-corner image/content card)

- **Not a standalone component in the source** — it's a style pattern applied via CSS on many surfaces: `.exp-card`, `.wte-card`, `.eat-tile`, `.ride-card`, `.info-card:nth-child(4n)`, `.faq-item:nth-child(3n+1)`, etc.
- **Pattern:** alternating corner radii `border-radius: var(--r-xl) var(--r-sm) var(--r-xl) var(--r-sm)` (or its mirror `var(--r-sm) var(--r-xl) var(--r-sm) var(--r-xl)` on the opposite phase).
- **Recommended factoring:** a utility prop (`heritage-tile` boolean) or a shared CSS class available to any card component. Don't build a dedicated `<HeritageTile>` — apply the shape via a mixin so it composes with whatever card the page needs.

### `StatCounter` (Aldhabi numeral + Inter label)

- **Origin:** `index.html` `.mena-visual` / `.mena-stat` blocks at lines ~986–1032. Also appears on `what-is-mena.html` (22 · 400M+ · 30+ · 1M+ row) and `about.html` (pillar numbers).
- **Structure:** big Aldhabi numeral (e.g. `٣`, `٢٢`) at `--fs-display` range, in `--gold` or `--maroon`, above an Inter label.
- **Props:** `numeralAr`, `numeralLatin?` (some surfaces ship both), `label`, `tone: 'gold' | 'maroon'`.
- **JS owned:** none (the counters are static, not animated). If an animation is ever wanted, see §5.

### `FilterBar`

- **Origin:** `music-entertainment.html` lines ~366–425 (CSS) and ~971–993 (markup) — only used on that one page.
- **Structure:** sticky bar directly under the fixed nav (`top: var(--nav-h); z-index: 40`). Glass sand background. Two filter groups (day, stage), each a label + row of `<button class="filter-btn">`. Plus a right-side `.filter-count` span.
- **Props:** `groups: { key, label, options: { value, label }[] }[]`, `onChange(state)`, `count`.
- **JS owned:** maintains `state = { day, stage }`, toggles `.is-active` on buttons, hides cards whose `data-day` / `data-stage` don't match, hides any `.day-block` with zero visible cards, updates the count text and toggles an empty-state panel. In React this becomes parent-level state: `useState({ day: 'all', stage: 'all' })` drives the filtered list.

### `ClosingCTA`

- **Origin:** inline on most content pages; canonical `music-entertainment.html` lines ~705–786 (CSS) and ~1334–1348 (markup).
- **Structure:** full-width `padding: 5.5rem var(--pad)` section. Background: maroon → maroon-ink 135° gradient + a radial gold-glow `::before`. Centered column: eyebrow · Newsreader title (with `<em>` + `.ar-word`) · sub · CTA row.
- **Variants:**
  - **Dark (maroon) — default.** Primary CTA gets gold heartbeat, secondary CTA becomes glass.
  - **Light (sand-deep) — one instance: `experience-rides.html`.** Primary CTA keeps the maroon heartbeat.
- **Props:** `eyebrow`, `titleHtml`, `sub`, `primaryHref`, `primaryLabel`, `secondaryHref`, `secondaryLabel`, `tone: 'dark' | 'light'`.
- **JS owned:** none.

### `DayBlock` (lineup day section)

- **Origin:** `music-entertainment.html` lines ~434–492 (CSS) and ~1001–1320 (markup, 3 instances).
- **Structure:** anchor-target wrapper (`id="day-fri"` etc.), header row (Newsreader day name with Aldhabi accent + all-caps date + italic hint), then the artist grid beneath.
- **Props:** `dayKey: 'fri'|'sat'|'sun'`, `dayName`, `dayAr`, `date`, `hint`, `artists: ArtistCardProps[]`.
- **JS owned:** none directly — visibility is driven externally by the filter bar.

### `InfoCard` (tonal sand card for info pages)

- **Origin:** `info.html` lines ~203–230 (CSS) and ~452–620 (markup, 12 instances).
- **Structure:** `<article class="info-card span-{N}">` inside a 12-col grid. Every 3rd card shifts tonal (sand / sand-deep / cream-with-inset-ring), every 4th gets heritage-tile radii. Inside: eyebrow → h3 title → body (paragraph or list-with-gold-dots).
- **Variants:** `span-4 | span-6 | span-8 | span-12` controls grid width. The nth-child tonal shifts are purely visual — don't parameterize them.
- **Props:** `span: 4|6|8|12`, `id` (for TOC anchors), `eyebrow?`, `title`, `children` (body).
- **JS owned:** none.

### `FAQDisclosure` (styled `<details>`)

- **Origin:** `faq.html` lines ~229–310 (CSS) and category blocks in `<section class="faq-body">`. Also used inside `vendors.html` FAQ.
- **Structure:** native `<details>` + `<summary>`. Custom `+` marker pill rotates 45° on open (`[open] .faq-marker` → ×). Open state shifts background to sand-deep. Every 3rd item gets heritage-tile radii.
- **Props:** `question`, `answer` (supports rich content — `<strong>`, inline links, multi-paragraph).
- **JS owned:** none — uses the browser's native `<details>` toggle. If you want controlled open/close in React, wrap it, but keep the semantics of a `<details>` element for accessibility.

### Other one-off components worth noting (not reused enough to factor but flagging)

- **`HomeCarousel`** (`index.html`) — the fan-card 3D carousel is a ~165-line custom JS component with pointer drag, keyboard nav, auto-advance, responsive `VISIBLE_SIDES`, and click-to-focus. Port faithfully; this is the homepage centerpiece. See §5 for details.
- **`Countdown`** (`index.html`) — four `.cd-num` slots driven by a 1s `setInterval` against the June 19 2026 17:00 PDT target.
- **`IntroBlock` / `PullQuote` / `InlinePull`** — editorial typographic surfaces that appear on `about.html`, `what-is-mena.html`, and the artist pages. Small enough to be inline JSX; shared styling on `.pull-quote` already.
- **`StagesSection`** on `experience-stages.html` — three tone-shifted stage sections (sand / cream-reversed / maroon) with `.motif-strip` Arabic calligraphic accents and `.stage-spec` grids. Page-local, not reused.
- **`TeamSection`** / **`PillarGrid`** on `about.html` — three-pillar grid with tonal backgrounds; page-local.
- **`Toc`** sticky pill row — a reusable helper pattern appears on `info.html` (`.info-toc`) and `faq.html` (`.faq-toc`). Same shape, different IDs. Worth a shared `<StickyToc links={...} />`.

---

## 4. Aldhabi rendering notes

This is the single most important thing to get right in the React port.

- **The 1.7× rule.** Aldhabi glyphs render optically smaller than their nominal em-size. Every Aldhabi instance ships at **1.7× the size of the neighboring Latin text.** The `.ar` utility class bakes this in (`font-size: 1.7em`). When you set an explicit size on an Arabic element (e.g. an artist-card caption), keep the 1.7× ratio against the Latin it pairs with. Example: artist card name is `1.3rem` Newsreader → Arabic accent is `2.2rem` Aldhabi (≈ 1.7×).
- **Don't rely on Tailwind's prose/typography plugins.** They won't handle this. Put Aldhabi sizing in a first-class component/utility that enforces 1.7×.
- **Font is LOCAL.** Load Aldhabi via `@font-face` from `./assets/fonts/aldhabi.ttf` (or its equivalent path once in `/public/`). Do NOT swap to Google Fonts Aldhabi — the local TTF is the tested render and matches on mobile Safari without CORS/edge-case issues.
- **Recommended component:** `<Arabic>{text}</Arabic>` that emits `<span class="ar">` by default; plus an optional prop for when you need to override the `1.7em` (e.g. when it sits standalone with no Latin neighbor). The DESIGN_SYSTEM §3 "full-width standalone calligraphic display" exception is rare.
- **The `.ar-word` selector** is used inside headline elements where the Aldhabi word is inline with a Newsreader sentence. It's essentially `.ar` plus `color: var(--gold)` (or `--maroon` depending on background) and `letter-spacing: 0`. Surface both as `<Arabic accent>` / `<Arabic tone="gold|maroon|ink">` variants.
- **`font-display: swap`** is set on the `@font-face` so there is no invisible flash. Keep this in React.
- **RTL direction:** set `direction: rtl` on any Arabic span that sits standalone (like `.ac-ar` on the artist cards). Inline accents (`.ar-word`) inside a Latin sentence keep `direction: ltr` / default — Aldhabi renders correctly either way when it's a single word.

---

## 5. JS behaviors to port

Every behavior below currently lives in a `<script>` block at the bottom of the relevant page. Port each to a React hook or component-owned effect.

### Nav scroll behavior (every page)

```js
window.addEventListener('scroll', () => {
  nav.classList.toggle('scrolled', window.scrollY > 40);
}, { passive: true });
```

Simple `useEffect` with a scroll listener. Store `isScrolled` in state and add the class conditionally.

### Mobile menu toggle (every page)

Burger click → toggle `.is-open` on the mobile menu, swap `☰` ↔ `✕`, set `document.body.style.overflow = 'hidden' | ''`. Should also close on route change and on `Escape`.

### Lineup filter logic (`music-entertainment.html`)

State: `{ day: 'fri'|'sat'|'sun'|'all', stage: 'pac'|'festival'|'indoor'|'all' }`. Toggle `.is-active` on the clicked filter button. Hide any `.artist-card` whose `data-day`/`data-stage` doesn't match. Collapse any `.day-block` with zero visible children. Update the count text and the empty-state panel. In React: lift the state to the page, render the filtered list, show an `<EmptyState>` component when nothing matches.

### Homepage fan carousel (`index.html`)

This is the heaviest JS on the site. Features to preserve:
- **3D fan layout.** Each card gets a `translateX/Y/Z + rotateZ + scale` transform computed from its signed offset from the active index. Constants: `SPREAD_DEG=8`, `DEPTH_PX=20`, `ACTIVE_LIFT=22`, `ACTIVE_SCALE=1.06`, `INACTIVE_SCALE=0.88`.
- **Responsive visible-sides:** 4 at `>=1200px`, 3 at `>=900px`, 2 otherwise (cards past that range fade to `opacity: 0`).
- **Auto-advance** every 3000 ms; pauses on `mouseenter`; resumes on `mouseleave` and after any user input.
- **Pointer drag** with direction-lock at 8 px, horizontal-only gating, wrap-around, snap-at-threshold, and a fling heuristic (if drag < 300 ms and |dx| > 15 px, advance one in the drag direction).
- **Click-to-focus** on any non-active card (detected via `didMove=false` on pointer up).
- **Keyboard** arrows on the `.fan-stage` element when focused.
- **Dots** are auto-generated from the cards and reflect the active state.

Port as a single `<HomeCarousel artists={…} />`. The source is 165 lines starting at `index.html:1726` — read it fully before rewriting. The physics constants are tuned to feel right; don't re-tune.

### Countdown (`index.html`)

`setInterval(tick, 1000)` against `new Date('2026-06-19T17:00:00-07:00').getTime()`. Zero-pad to 2 digits. When `diff <= 0`, flat 00s. Trivial `useEffect` in React; remember to clear the interval on unmount.

### Stat counters (homepage)

**No animation is currently shipped.** Phase 2 log notes them as "static, not animated" — they're just Aldhabi numerals above Inter labels. If a count-up animation is ever wanted, add it at port-time; it was deliberately omitted from the static build (no-confetti, no-spinning rule in DESIGN_SYSTEM §7).

### FAQ accordion

Pure `<details>/<summary>` — no JS needed. Preserve the native element in React (`<details>` works inside React out of the box).

### Contact form (`contact.html`)

Form is styled only. The submit handler currently runs `e.preventDefault()` and swaps the button text to "Thanks — we'll be in touch." There is **no backend wired up**. Route through Netlify Forms, Formspree, or a custom endpoint at port time (see §8).

---

## 6. Content sources

- **Artist data** (12 entries) is extractable directly from `music-entertainment.html`. Every `<a class="artist-card">` carries `data-day`, `data-stage`, `href`, and structured inner markup (`.ac-ar`, `.ac-name`, `.ac-country`, `.ac-stage-pill`, `.ac-bio`). The same data is re-used on the homepage fan carousel (`index.html` `.fan-card` blocks) and on each `artist-*.html` / `comedian-*.html` detail page. When porting, lift to a single `artists.ts` / `artists.json` and consume it everywhere.
- **Artist images:** `./assets/images/artist-*.webp` and `./assets/images/comedian-*.webp` — these are placeholders and will be swapped with real portraits pre-launch (see §7).
- **Festival info** (location, parking, ADA, weather, re-entry, prayer, cashless, etc.) is canonical on `info.html`. Lift to a structured `info.ts`.
- **FAQ** (7 categories, 30 questions) is on `faq.html`. Lift to `faq.ts` as `{ category, items: { q, a }[] }[]`.
- **Day & schedule metadata** (dates, opening times, headliner hints) is duplicated on `index.html` `.day-card` blocks and on `music-entertainment.html` `.day-header` blocks. Normalize.
- **Copy source of truth** for anything not already in the static files: the pitch wireframe (`../Pitch_wireframe/`) and the hosted mirror at melodies.cloud. Osama is the content owner and will push updates.

---

## 7. Deliberately open — for the dev team to architect

These are things I intentionally did not wire up so Moe & Zubaid can pick the architecture.

- **`tickets.html` / `schedule.html` / `map.html`** ship as Heritage Hearth "coming soon" stubs with full nav/footer/type, because the backend + data model for ticketing, day-of schedule, and map/venue data is the dev team's call. Each stub has a clear CTA slot ready for the real implementation (ticket purchase link, schedule grid, interactive map). Decide together how those connect — Shopify / Eventbrite / self-hosted ticketing, static schedule JSON vs. CMS, custom map vs. Google Maps embed — and replace the stub body. Shell (nav/footer/eyebrow/type) stays identical; don't rebuild those.
- **`contact.html` form submit** is intentionally stubbed to a client-side `e.preventDefault()` + thank-you swap. No backend is wired. Route it through whatever the React app's form layer will be (Netlify Forms, Formspree, a custom API route, Resend, etc.) — pick this with the rest of the backend architecture, don't rush a point solution now.
- **Placeholder content ready for swap when Osama delivers:**
  - **Real artist portraits.** 10 of 12 `artist-*.webp` / `comedian-*.webp` files are day-themed AI placeholders. **Already swapped:** MC Abdul + Norhan (real shoots). Remaining: Anees, Bayou, Dana Salah, Dystinct, Ehab Tawfik, Gaidaa, Issam Alnajjar, Lana Lubany, Mohammed Assaf, Tul8te + 2 comedians (Nasser Al-Rayess, Saad Alessa).
  - **Sponsor / partner logos.** `sponsors.html` has 12 placeholder Logo tiles; `partners.html` has 24 placeholder tiles (Community/Media/Nonprofit). Slot to ~180×90, SVG preferred with PNG fallback.
  - **Experience page tile imagery.** Food bazaar cuisine tiles, ride cards, stage photos — all currently CSS-gradient placeholders per the no-stock-Arabia rule. Real photography pending.
  - **Day-card composite images** (`friday-artists.webp`, `saturday-artists.webp`, `sunday-artists.webp`) are AI composites. Phase 2 log flags they may fail the "feels like a photograph" DESIGN_SYSTEM §6 checklist — reshoot is optional if real composites look better.
- **All 13 hero images are in place** as `.webp` files (`hero-home`, `hero-music-entertainment`, `hero-food-bazaar`, `hero-rides`, `hero-stages`, `hero-about`, `hero-faq`, `hero-info`, `hero-sponsors`, `hero-vendors`, `hero-partners`, `hero-crowd` used on what-is-mena, `hero-partners` re-used on contact). No pending hero swaps.
- **Homepage hero** is a looping muted MP4 (`./assets/hero-final.mp4`, 8.7 MB, H.264, 1920×1080, 30fps) layered over `hero-home.webp` as the `<video>` poster for the first frame. Preserve this dual setup in the React port so the page has something to render before the video loads.

---

## 8. Known issues + TODOs

Synthesized from the Phase 2–7 logs.

### Already shipped during this build
- All 13 hero images in place as `.webp` files (no more missing-JPG fallbacks).
- Homepage hero video swapped from the old `ocmena-30fps-v2.mp4` AI reference to a Kling-generated final cut at `./assets/hero-final.mp4` (8.7 MB, H.264, muted, looping).
- Tickets / Schedule / Map pages ship as Heritage Hearth coming-soon stubs with full nav + footer + eyebrow.
- MC Abdul and Norhan artist pages added (real portrait shoots).

### Content to swap when ready (not blocking — site looks finished without these)
- Replace the remaining 10 artist + 2 comedian AI placeholder portraits with real shoots when available.
- Replace sponsor and partner placeholder Logo tiles with real logos (SVG preferred, PNG fallback, slotted to ~180×90) once delivered.
- Replace experience-page tile placeholders (food bazaar cuisine tiles, ride cards, stage photos) with real photography when shot.
- Swap the three day-card composites (`friday-artists.webp` / `saturday-artists.webp` / `sunday-artists.webp`) for real composites if the AI versions don't pass the DESIGN_SYSTEM §6 feels-like-a-photograph check.

### Backend / integration decisions for Moe & Zubaid
- **`contact.html` form** — currently client-side `e.preventDefault()` + thank-you swap. Decide with the backend architecture: Netlify Forms, Formspree, a custom API route, Resend, etc.
- **`tickets.html` / `schedule.html` / `map.html`** — currently Heritage Hearth coming-soon stubs. Decide the backend per page (Shopify / Eventbrite / self-hosted ticketing, static schedule JSON vs. CMS, custom map vs. Google Maps embed), then replace the stub body while keeping the shell.

### Pricing / deadlines pending from Osama (Phase 6)
- Sponsor tier pricing (Presenting / Platinum / Gold / Supporting) — currently "Contact for pricing".
- Vendor pricing ranges (food & beverage, merchandise & crafts) — currently directional.
- Vendor Equity Scholarship program details (eligibility, slots) — confirm.
- Final application + contract deadlines (currently Feb 28 / April 15, 2026).
- Confirm the 6 inbox email addresses actually route (info / tickets / press / sponsors / vendors / artists @ ocmenafestival.com).
- Real social handles (currently all `@ocmenafestival` placeholders) + WhatsApp channel invite link + press kit URL.
- Audience percentages on `sponsors.html` are directional — replace with first-party numbers once ticketing + on-site surveys exist.
- Sponsor deliverables per tier — draft lifted from pitch wireframe; vet for accuracy.

### Copy decisions to confirm
- **Tul8te Arabic spelling.** Pitch used `تووليت`; site standardized on `تلاتة` (matching `music-entertainment.html`). Confirm canonical.
- **Pacific Amphitheatre capacity.** Pitch says 8,500; `experience-stages.html` ships 8,000+. Pick one.
- **Indoor stage capacity.** Pitch says 400; `experience-stages.html` ships 1,200. Pick one.
- **8th cuisine tile.** Pitch lists Sudanese / Horn; `experience-food-bazaar.html` ships "Gulf dates & coffee." Review whether Sudanese/Horn representation is a deliberate inclusion goal.
- **About page team section** ships deliberately coalition-style with only Osama Aresheh (sponsorships) and Razan Kesbeh (vendors) named. The Pitch_wireframe had Co-Founder placeholders; those were NOT ported. Decide whether to add named cards before launch.
- **Sponsor closing CTA** on `index.html` sponsor strip: primary-red button (loud but readable on sand-deep). Phase 2 offered a cream-secondary swap if it feels too loud.
- **Saturday three-days card** uses a `-12px` asymmetric lift on the homepage. Phase 2 offered a `-6px` or flat option if it reads as a CSS bug rather than intentional.
- **FAQ count** currently 30 questions vs the 15–20 target. Phase 5 notes three candidates for trimming (halal, stroller, DJs) but left them because every one is a real repeat-inbox question.

### Soft-rule / minor refactor
- All 13 artist/comedian/template pages use a hex literal `#ffd560` for a gold-button hover state. It's a brightened `--gold`. Add `--gold-soft: #ffd560;` to `:root` in the React port and replace.
- On `about.html` and `what-is-mena.html`, the `About` nav link is marked active on both pages. Phase 5 flagged the ambiguity; Phase 7 confirmed the decision (What-is-MENA is a child of About in the IA). If the React IA adopts a dropdown or adds a top-level What-is-MENA link, revisit.
- Flag emojis in the `what-is-mena.html` country grid render correctly in real browsers but show as tofu in headless Chromium. Not a bug; worth a spot-check across real iOS/Android for MENA flags. If any platform renders a given flag poorly, replace emojis with tiny SVG roundels.
- The Rides hero places `العيلة` on its own line because of how the Newsreader headline wraps. Not a CSS bug; fix by adjusting the English phrasing so the Arabic stays inline with a Latin neighbor (per DESIGN_SYSTEM §3), not by changing the 1.7× rule.
- Primary nav active state on `index.html` has `.is-active` count = 0 (the homepage has no nav link; the logo is the home link). Acceptable edge case; preserve in React.

---

## 9. File tree

```
festival website/
├── DESIGN_SYSTEM.md                # Brand + design spec (source of truth)
├── STATE.md                        # Build state / checkpoint doc
├── HANDOFF.md                      # This file
│
├── index.html                      # Homepage (hero + fan carousel + three days + experience + MENA + sponsors + footer)
├── music-entertainment.html        # Full lineup — canonical pattern reference
│
├── artist-template.html            # Artist detail template ({{TOKEN}} placeholders preserved)
├── artist-anees.html
├── artist-bayou.html
├── artist-dana-salah.html
├── artist-dystinct.html
├── artist-ehab-tawfik.html
├── artist-gaidaa.html
├── artist-issam-alnajjar.html
├── artist-lana-lubany.html
├── artist-mohammed-assaf.html
├── artist-tul8te.html
├── comedian-nasser-al-rayess.html
├── comedian-saad-alessa.html
│
├── experience-food-bazaar.html     # The Souq
├── experience-rides.html           # Twilight Joy (rides + family)
├── experience-stages.html          # The Architectural Void (3 stages)
│
├── about.html                      # Our story
├── what-is-mena.html               # 22 countries, regions, diaspora
├── info.html                       # Location, parking, ADA, prayer, cashless, etc.
├── faq.html                        # 7 categories, 30 Q&A
│
├── sponsors.html                   # Become a sponsor (4 tiers)
├── vendors.html                    # Apply to vend (6 types, timeline, FAQ)
├── partners.html                   # Official / community / media / nonprofit partners
├── contact.html                    # 6 inbox cards, address, socials, contact form
│
├── assets/
│   ├── fonts/
│   │   └── aldhabi.ttf             # LOCAL — do not swap to Google Fonts
│   ├── images/
│   │   ├── Logo.webp
│   │   ├── favicon.png
│   │   ├── artist-*.webp           # 10 placeholder portraits (swap pre-launch)
│   │   ├── comedian-*.webp         # 2 placeholder portraits (swap pre-launch)
│   │   ├── friday-artists.{png,webp}
│   │   ├── saturday-artists.{png,webp}
│   │   ├── sunday-artists.{png,webp}
│   │   ├── hero-crowd.webp         # Legacy; not currently referenced
│   │   └── hero-*.jpg              # 13 missing files — see §7 full list
│   └── ocmena-30fps-v2.mp4         # Reference video for color grade (not used as bg anymore)
│
└── .phase-logs/
    ├── phase-2.md                  # Homepage retrofit
    ├── phase-3.md                  # Artist template + 12 artist pages
    ├── phase-4.md                  # Three experience pages
    ├── phase-4-screenshots/
    ├── phase-5.md                  # About / What-is-MENA / Info / FAQ
    ├── phase-6.md                  # Sponsors / Vendors / Partners / Contact
    └── phase-7.md                  # Global polish pass
```

Total: 26 HTML files + the template + 3 canonical docs (DESIGN_SYSTEM, STATE, HANDOFF).

---

## Closing note

Everything you need to port faithfully is either in `DESIGN_SYSTEM.md` (brand rules) or `music-entertainment.html` (shell pattern + artist card + filter + closing CTA + footer). If a decision isn't documented in one of those two files, the Phase 2–7 logs in `.phase-logs/` record the reasoning. When a Phase log and the pitch wireframe disagree, the Phase log is authoritative — those were the decisions I actually shipped.

The three rules that override everything: **no pure black, no 1px borders, Aldhabi always 1.7× the Latin neighbor.** Bake those into your primitives and the rest of the system falls out naturally.
