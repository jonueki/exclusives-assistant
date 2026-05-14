# Ticketmaster Seat Overlay — Architecture (v0 draft)

> **Status: v0 draft, superseded by round-2 specs.** Read alongside `feasibility.md`, `prd.md`, and `v1-build-plan.md` — those are the authoritative current specs.
>
> Specific deltas:
> - **Strategy B** (§4.2, §10 risk-table row, §12 v3 roadmap row): permanently excluded per `feasibility.md` §R2.4 and `round-1-synthesis.md` C-alignment. Do not implement.
> - **Venue list** (§5 file tree): superseded by `prd.md` §3 — confirmed Seattle-area list is Climate Pledge Arena, T-Mobile Park, Tacoma Dome, Lumen Field.
> - **Scoring formula** (§7) and **value-profile schema** (§6) remain authoritative; `v1-build-plan.md` §3.6/§3.7 inherits them.

A Chrome extension that helps **you** (a human, not a bot) pick a good-value seat faster on Ticketmaster's interactive seat map. It is a passive visual layer over the page you are already looking at — no automation, no extra network calls to TM, no synthetic events.

> Guiding principle: **Don't be faster at clicking. Be faster at deciding.**
> By the time TM finishes rendering the map, your eyes should already be locked on 5–10 candidate seats, not scanning 3000.

---

## 1. Goals & non-goals

### Goals
- Highlight seats matching a user-defined **value profile** (price range, sections, rows, min adjacent seats).
- Grey out / dim everything that doesn't match so the eye locks on candidates.
- Side panel with the **top-N ranked seats** by a configurable scoring formula.
- Per-venue **intel layer** (good sections, obstructed views, stage-config notes) loaded as static data.
- Persist profiles per artist / tour so drop day is one click to "arm."

### Non-goals (hard constraints — do not violate)
- **No synthetic clicks** on TM UI. The user clicks every seat themselves.
- **No auto-cart, auto-checkout, auto-anything.** Decoration only.
- **No outbound network calls to ticketmaster.com or any TM subdomain** beyond what the page itself initiates. We read the page; we never call TM ourselves.
- **No DOM event spoofing**, no mouse/scroll synthesis, no key dispatch.
- **No queue automation**, no re-joining queues, no refresh loops on protected pages.
- **No fingerprint or anti-detect modifications** (don't touch `navigator`, canvas, WebGL, etc.).
- **No multi-account, no proxy, no captcha solving.**

Anything that would change TM's view of the session is out of scope, full stop. The extension must be invisible from TM's perspective — it's just CSS and a side panel rendered into your own page.

---

## 2. How TM's seat map works (current understanding — needs verification)

This is the part with the most uncertainty. Before writing code we need to confirm the following by inspecting a live event page in DevTools (Network + Sources + a heap snapshot of the seat map state):

- TM's interactive seat map is a heavily-protected SPA, served from `ticketmaster.com` with map data hosted on `mapsapi.tmol.io` (or similar TM map subdomain).
- The map renders to SVG or Canvas (we expect a mix — SVG for sections/seats, canvas for some venue chrome). **This matters a lot**: SVG = each seat is a DOM node we can decorate with CSS. Canvas = we'd need to overlay our own SVG/DOM on top using seat coordinates, which is more work and more fragile.
- Seat inventory comes from a JSON payload via `fetch` / `XHR` to a TM endpoint. Each seat record likely has: `section`, `row`, `seatNumber`, `priceLevel` / `price`, `availability`, plus an `id` that maps to a DOM node or canvas coordinate.
- The payload is polled / re-fetched as seats are released/grabbed (this is what makes seats "disappear" while you're deciding).

**Action item before coding:** spend ~1 hour in DevTools on a real event page documenting:
1. Is the map SVG, Canvas, or both?
2. What's the exact endpoint and response shape for seat inventory?
3. How often does it re-fetch / how does it diff updates?
4. What identifier ties a seat record to its visual element?

Until verified, treat sections 3–5 as a target architecture, not a spec.

---

## 3. Data flow

```
┌─────────────────────────────────────────────────────────────┐
│  Ticketmaster event page (user's real browser session)      │
│                                                             │
│   TM JS  ──fetch──▶  TM seat inventory API                  │
│      │                                                      │
│      ▼                                                      │
│   Page DOM / Canvas (rendered seat map)                     │
│      ▲                                                      │
│      │ read-only                                            │
│   ┌──┴────────────────────────────────────────┐             │
│   │  Content script (our extension)           │             │
│   │  • Observe seat inventory                 │             │
│   │  • Apply value profile filter             │             │
│   │  • Inject CSS classes / overlay layer     │             │
│   │  • Render side panel (Shadow DOM)         │             │
│   └──┬────────────────────────────────────────┘             │
│      │                                                      │
└──────┼──────────────────────────────────────────────────────┘
       │ chrome.storage (sync) — profiles, venue intel
       ▼
   ┌────────────────────────┐
   │  Extension storage     │
   │  • Value profiles      │
   │  • Venue intel sheets  │
   │  • Per-tour overrides  │
   └────────────────────────┘
```

There is **no extension backend** in v1. Everything lives in `chrome.storage.sync` so it follows the user across Chrome installs. Venue intel ships as bundled JSON in the extension package; users can override.

---

## 4. Reading TM's seat data — two strategies

We need to pick one. They have different tradeoffs.

### Strategy A: DOM observation (preferred for v1)
- Use a `MutationObserver` on the seat map container.
- Each seat is (presumably) an SVG `<circle>` or `<rect>` with data attributes like `data-section`, `data-row`, `data-price-level`, or class names encoding the price tier.
- Apply our overlay by adding CSS classes (`tm-overlay-match`, `tm-overlay-dim`) to matching/non-matching seats.
- Read seat metadata from data attributes; if metadata is sparse, cross-reference with a global page state object (`window.__INITIAL_STATE__` or similar) — read-only.

**Pros:** zero network coupling, survives endpoint changes, hardest for TM to detect (it's just CSS).
**Cons:** breaks if TM renames data attributes or swaps SVG for Canvas. Maintenance burden.

### Strategy B: `fetch` response interception
- Monkey-patch `window.fetch` (and `XMLHttpRequest`) in a page-context script to read seat inventory responses as they come back.
- Build our own in-memory model of available seats.
- Apply overlay either to DOM (if SVG) or to our own overlay canvas (if TM map is canvas-rendered).

**Pros:** rich data — full price, exact seat identity, faster updates.
**Cons:** patching `fetch` is observable from page JS if TM checks `fetch.toString()`. Higher risk of being noticed by anti-bot heuristics even though it's read-only. More fragile to TM SPA refactors.

### Recommendation
Start with **A**. Add **B** behind a feature flag only if A's data is too thin to make good decisions (e.g. price tier isn't on the DOM node).

---

## 5. Extension structure (Manifest V3)

```
extension/
├── manifest.json                # MV3 manifest
├── src/
│   ├── content/
│   │   ├── index.ts             # Entry, sets up MutationObserver, mounts panel
│   │   ├── seat-reader.ts       # Strategy A: DOM-based seat extraction
│   │   ├── seat-reader-fetch.ts # Strategy B (flagged off in v1)
│   │   ├── overlay.ts           # Applies CSS classes to matching seats
│   │   └── panel/               # Side panel UI (React or Preact, Shadow DOM)
│   │       ├── Panel.tsx
│   │       ├── SeatList.tsx
│   │       └── ProfileEditor.tsx
│   ├── background/
│   │   └── service-worker.ts    # Storage helpers, profile sync
│   ├── popup/
│   │   ├── Popup.tsx            # Toolbar popup: arm profile, toggle overlay
│   │   └── index.html
│   ├── options/
│   │   ├── Options.tsx          # Full profile + venue intel editor
│   │   └── index.html
│   ├── data/
│   │   └── venues/              # Bundled venue intel JSON
│   │       ├── msg.json
│   │       ├── kia-forum.json
│   │       └── ...
│   ├── lib/
│   │   ├── scoring.ts           # Seat value formula
│   │   ├── profile.ts           # Profile types + matching
│   │   └── storage.ts           # chrome.storage wrapper
│   └── styles/
│       └── overlay.css          # The actual visual layer
├── package.json
├── tsconfig.json
└── vite.config.ts               # Vite + crxjs plugin for MV3 builds
```

### `manifest.json` essentials
- `manifest_version: 3`
- `host_permissions`: only `https://www.ticketmaster.com/*` (and any TM map subdomain we observe responses on, if we go with Strategy B).
- `permissions`: `storage`, `activeTab`. No `tabs`, no `webRequest`, no `<all_urls>`.
- `content_scripts`: injected on event pages only, matched by URL pattern.
- No `background.persistent` (MV3 service worker only).
- CSP: default; no remote code, all bundled.

---

## 6. The value profile

A profile is what the user defines once and arms before each drop.

```ts
type ValueProfile = {
  id: string;
  name: string;                 // "Lower bowl, mid-row, under $250"
  maxPrice: number;             // e.g. 250 — kills Platinum
  minPrice?: number;            // optional — kills nosebleeds you don't want
  sectionWhitelist?: string[];  // e.g. ["110","111","112","113","114"]
  sectionBlacklist?: string[];  // e.g. obstructed sections
  rowRange?: { min?: string; max?: string };  // alphanumeric, venue-specific
  minAdjacentSeats: number;     // 1, 2, 4...
  preferAisle?: boolean;
  notes?: string;
};
```

Profiles live in `chrome.storage.sync`. The popup lets you pick which one is "armed" — that's the one the content script uses on the next event page load.

---

## 7. Scoring formula (top-N panel)

Given a candidate seat that passes the filter, rank it by:

```
score = sectionQuality * rowQuality * priceValue * adjacencyBonus
```

- `sectionQuality`: 0–1, from the venue intel sheet for this venue + stage config.
- `rowQuality`: 0–1, decays with distance from the front (per-section curve).
- `priceValue`: `(maxPrice - price) / maxPrice`, clamped 0–1. Cheaper-within-budget wins ties.
- `adjacencyBonus`: +N% if it satisfies the `minAdjacentSeats` requirement comfortably.

Tunable weights live in the profile. Top 10 surface in the side panel with a "scroll to seat on map" link.

This is intentionally simple in v1 — we want a heuristic, not a model. Once the user has logged ~10 drops we'll know what features actually predict "the seat they wished they got."

---

## 8. Venue intel sheet

Bundled JSON per venue. Schema:

```ts
type VenueIntel = {
  venueId: string;              // TM's venue id if we can scrape it, else slug
  name: string;
  stageConfigs: {
    [configName: string]: {     // "end-stage", "in-the-round", "b-stage-floor"
      sectionQuality: { [section: string]: number };   // 0–1
      obstructed: string[];                            // section ids
      notes?: { [section: string]: string };
    };
  };
  // Round-2 addition (v1-build-plan §3.6/§3.7). Used when Strategy A returns
  // a price tier on the seat node but no exact price. Optional — scoring falls
  // back to a neutral 0.5 priceValue if absent.
  tierToPriceRange?: { [tier: string]: { min: number; max: number } };
};
```

v1 ships with intel for 5–10 venues the user actually goes to (we'll list them together). The options page has a JSON editor so the user can refine numbers after each show.

Sources for initial intel: aviewfrommyseat.com, r/<venue>, SeatGeek deal-score history, user's own memory.

---

## 9. UI

### Toolbar popup (quick)
- Dropdown: armed profile.
- Toggle: overlay on/off.
- Toggle: dim non-matching seats vs only highlight matches.
- Link to options page.

### Side panel (injected on event page, Shadow DOM)
- Pinned to right edge, collapsible.
- Top of panel: armed profile summary ("Lower bowl, $150–250, ≥2 together").
- Scrollable list: top 10 candidate seats, ranked. Each row shows section, row, seat, price, score.
- Clicking a row scrolls/zooms the TM map to that seat (we do this by reading and setting TM's own map state if reachable; if not, we just visually indicate which seat on the map via a pulse).
- **No "buy" button. No automation.** Just navigation aid.

### Options page
- Profile editor (CRUD).
- Venue intel editor (JSON-backed form).
- Export / import as JSON for backup.

---

## 10. Risks & mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| TM renames data attributes on seat nodes, breaking the overlay | High | Keep selectors in one file. Ship updates via Chrome auto-update. Have a "diagnostic mode" that dumps detected attributes so we can patch quickly. |
| TM swaps SVG for Canvas | Medium | Strategy B fallback (fetch interception) reconstructs map from inventory data. Render our own overlay canvas. |
| TM detects the extension via DOM injection | Low (if we use Shadow DOM + scoped CSS classes) | Use unique prefixed classes (`tmx-`). All UI lives in Shadow DOM. No global CSS leaks. |
| User assumes overlay is real-time and clicks a seat that just got taken | Medium | Overlay refreshes on every MutationObserver tick. Show a "last updated Xs ago" indicator. |
| Extension breaks during a drop because we shipped a bad version | High impact, low likelihood | Bundle a kill-switch: if the page is on an event we tagged as "drop day" and we detect any thrown exception, auto-disable and toast "overlay off — fall back to manual." |
| Chrome Web Store rejects (TM TOS concern) | Medium | We don't bypass anything, don't automate, don't scrape. We are purely a personal accessibility layer on a page the user already paid to visit. Distribute as unpacked / self-hosted CRX if needed. |

---

## 11. Build & dev loop

- **Toolchain**: TypeScript + Vite + `@crxjs/vite-plugin` for MV3 hot reload.
- **UI**: Preact (smaller than React, same API) inside Shadow DOM.
- **State**: zustand or just useState — there's not much.
- **Lint/format**: ESLint `--max-warnings 0`, Prettier.
- **Tests**: Vitest for `lib/` (scoring, profile matching). No E2E in v1 — we don't have a non-prod TM to test against and we won't automate prod.
- **Manual test**: a saved HTML snapshot of a real seat map page, served locally, that we develop against. Re-snapshot when TM changes.

---

## 12. Roadmap

### v0 (research, ~1 day, no code)
- [ ] DevTools session on a live event page. Document: SVG vs Canvas, seat node selectors, inventory endpoint shape, refresh cadence.
- [ ] Pick 5–10 venues for initial intel sheets.

### v1 (MVP, ~1 week)
- [ ] Extension skeleton (manifest, content script, popup).
- [ ] Strategy A seat reader.
- [ ] CSS overlay: highlight + dim.
- [ ] Single hard-coded profile.
- [ ] Manual test against snapshot.

### v2 (~1 week)
- [ ] Side panel with top-N ranking.
- [ ] Profile CRUD in options page.
- [ ] Venue intel JSON loading + 5 venues bundled.
- [ ] chrome.storage.sync persistence.

### v3 (polish, ongoing)
- [ ] Diagnostic mode (dump selectors when broken).
- [ ] Strategy B fetch interception behind flag.
- [ ] Post-drop logger (what you targeted vs what you got).
- [ ] Per-tour profile overrides.

---

## 13. Open questions

1. Is the seat map SVG, Canvas, or hybrid? **Need DevTools confirmation.**
2. Does the seat DOM expose price on each node, or only price tier? If only tier, do we need Strategy B for v1?
3. How aggressive is TM's MutationObserver / runtime integrity checking? Worth a quick probe with a no-op extension before we build anything real.
4. Which 5–10 venues should v1 ship intel for? List your top targets and I'll seed the JSON.
5. Do we want the side panel scoring to be **deterministic** (pure formula) or **learned** (adjust weights based on what you actually picked)? v1 deterministic; v2 maybe both.
