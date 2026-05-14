# v1 Build Plan — Ticketmaster Seat Overlay Extension

**Status:** Round 2, post-synthesis  
**Author:** SWE  
**Last updated:** 2026-05-14  
**Companion docs:** `docs/seat-overlay-architecture.md` (v0), `docs/feasibility.md` (round-1 + R2 addendum), `docs/round-1-synthesis.md` (decisions C1–C9), `docs/prd.md` (round 2), `docs/ux-mockup.md` (round 1).

This document is the concrete v1 build spec. It turns the agreed architecture (Strategy A primary, Strategy C fallback, no Strategy B ever; MV3; closed Shadow DOM with `tmx-{buildHash}` prefix; kill-switch P0; local-only diagnostic + override; no backend) into files, signatures, invariants, and ship checks. Where v0 was aspirational, this is what you actually build.

---

## 1. Repository layout

```
exclusives-assistant/
├── docs/                              # design / decision docs (this file lives here)
├── extension/
│   ├── manifest.json                  # MV3 manifest (full source in §2)
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts                 # Vite + @crxjs/vite-plugin; emits buildHash into define
│   ├── .eslintrc.cjs                  # enforces the "do not implement" list (§7)
│   ├── public/
│   │   └── icons/                     # 16/32/48/128 PNG; greyscale variant for kill-switch
│   └── src/
│       ├── content/
│       │   ├── isolated/
│       │   │   ├── index.ts           # Isolated-world entry. Boots observer, overlay, panel.
│       │   │   ├── observer.ts        # MutationObserver wrapper, debounce, scope guards.
│       │   │   ├── overlay.ts         # Adds/removes tmx-match / tmx-dim classes on seat nodes.
│       │   │   ├── modal-watch.ts     # C6 modal-collapse signal detection.
│       │   │   ├── bridge.ts          # postMessage IPC with MAIN-world script (typed envelope).
│       │   │   ├── kill-switch.ts     # window.onerror + panic key combo; disables overlay.
│       │   │   └── panel/
│       │   │       ├── mount.tsx      # Creates closed shadow root, mounts Preact app.
│       │   │       ├── Panel.tsx      # Top-level UI: header + SeatList + footer.
│       │   │       ├── SeatList.tsx   # Top-10 candidates list, click → pulse on map.
│       │   │       ├── KillSwitchBanner.tsx  # Visible when overlay disabled.
│       │   │       ├── DiagnosticDump.tsx    # Collapsible diagnostic panel.
│       │   │       └── styles.css     # Scoped via shadow root, reads --tmx-* vars on :host.
│       │   └── main-world/
│       │       ├── state-reader.ts    # Reads React fibers, posts seat data to isolated world.
│       │       └── router-hook.ts     # Hooks history.pushState/replaceState; emits navigation events.
│       ├── background/
│       │   └── service-worker.ts      # Near-stub: relays popup ↔ content messages, holds last-known stats.
│       ├── popup/
│       │   ├── index.html
│       │   ├── Popup.tsx              # Armed-profile dropdown, toggles, live stats.
│       │   └── styles.css
│       ├── options/
│       │   ├── index.html
│       │   ├── Options.tsx            # Tabs: Profiles, Venues, Selector Overrides, Diagnostics, Canary.
│       │   ├── ProfileEditor.tsx
│       │   ├── VenueEditor.tsx
│       │   ├── OverrideEditor.tsx     # JSON textarea editor for tmx.selector-overrides.
│       │   ├── DiagnosticsViewer.tsx
│       │   └── CanaryViewer.tsx
│       ├── lib/
│       │   ├── scoring.ts             # Pure scoring formula.
│       │   ├── profile.ts             # ValueProfile types + matcher.
│       │   ├── storage.ts             # chrome.storage wrappers, lock-to-snapshot mode.
│       │   ├── selector-overrides.ts  # Loads + merges user overrides with bundled defaults.
│       │   ├── selectors-default.ts   # Bundled selector defaults (the only place TM-specific CSS lives).
│       │   ├── diagnostic.ts          # Build the diagnostic dump payload.
│       │   ├── canary.ts              # Anti-bot cookie sniffer.
│       │   ├── types.ts               # Shared types (Seat, Profile, Stats, Envelope).
│       │   └── build-hash.ts          # Re-export of BUILD_HASH from Vite define.
│       ├── data/
│       │   └── venues/                # Bundled venue intel.
│       │       ├── index.ts                   # Barrel: exports the venue map keyed by venueId.
│       │       ├── climate-pledge-arena.json
│       │       ├── t-mobile-park.json
│       │       ├── tacoma-dome.json
│       │       └── lumen-field.json
│       └── styles/
│           └── overlay.css            # The light-DOM CSS we inject for tmx-match / tmx-dim on TM seats.
└── tools/
    ├── capture-replay.mjs             # Records HAR + DOM snapshot from a Chromium session.
    ├── replay-serve.mjs               # Static-serves a captured snapshot for offline dev.
    └── lint-no-restricted.cjs         # ESLint rule preset for the §7 ban list.
```

Notes vs `seat-overlay-architecture.md` §5:
- Renamed `seat-reader.ts` → `state-reader.ts` and moved to `content/main-world/` (Strategy A reads from DOM in isolated world via `observer.ts`; Strategy C reads from fibers in MAIN world).
- Deleted `seat-reader-fetch.ts` (Strategy B is permanently out per round-1 alignment).
- Added `content/main-world/router-hook.ts` for SPA navigation (feasibility §5.1 missed risk).
- Added `content/isolated/kill-switch.ts` and `KillSwitchBanner.tsx` (kill-switch is now P0).
- Added `lib/selector-overrides.ts`, `lib/selectors-default.ts`, `lib/diagnostic.ts`, `lib/canary.ts`.
- `service-worker.ts` is a near-stub (feasibility §3.3): it holds last-known stats and relays messages. No alarms in v1.
- Added `tools/` for the replay harness.

---

## 2. `manifest.json` (full)

```json
{
  "manifest_version": 3,
  "name": "Exclusives Assistant",
  "short_name": "tmx",
  "version": "0.1.0",
  "description": "Decision-support overlay for Ticketmaster seat maps. Read-only. No automation.",
  "minimum_chrome_version": "116",
  "action": {
    "default_title": "Exclusives Assistant",
    "default_popup": "src/popup/index.html",
    "default_icon": {
      "16": "public/icons/icon-16.png",
      "32": "public/icons/icon-32.png",
      "48": "public/icons/icon-48.png",
      "128": "public/icons/icon-128.png"
    }
  },
  "options_ui": {
    "page": "src/options/index.html",
    "open_in_tab": true
  },
  "background": {
    "service_worker": "src/background/service-worker.ts",
    "type": "module"
  },
  "permissions": ["storage", "activeTab"],
  "_comment": "Paths below are source-relative (.ts/.tsx/.html). @crxjs/vite-plugin rewrites them to built .js paths during `vite build`. Do not load the raw src/ tree via chrome://extensions — load the dist/ output.",
  "host_permissions": [
    "https://www.ticketmaster.com/event/*"
  ],
  "content_scripts": [
    {
      "matches": [
        "https://www.ticketmaster.com/event/*"
      ],
      "js": ["src/content/isolated/index.ts"],
      "css": ["src/styles/overlay.css"],
      "run_at": "document_idle",
      "world": "ISOLATED",
      "all_frames": false
    },
    {
      "matches": [
        "https://www.ticketmaster.com/event/*"
      ],
      "js": [
        "src/content/main-world/router-hook.ts",
        "src/content/main-world/state-reader.ts"
      ],
      "run_at": "document_start",
      "world": "MAIN",
      "all_frames": false
    }
  ],
  "web_accessible_resources": [],
  "content_security_policy": {
    "extension_pages": "script-src 'self'; object-src 'self'; style-src 'self' 'unsafe-inline';"
  }
}
```

Notes:
- Two content-script entries: the isolated-world UI/overlay runs at `document_idle` (DOM is ready, observer can attach), and the MAIN-world reader runs at `document_start` so its router-hook installs before TM's app code calls `history.pushState`.
- Host permission is **only** `https://www.ticketmaster.com/event/*`. Not `<all_urls>`. Not the auth subdomain. Not the maps subdomain (we never call it; we read its already-rendered output).
- No `webRequest`, no `tabs`, no `scripting` (we don't dynamically inject; manifest declares everything).
- `"world": "MAIN"` is the correct way to enter page context per Chrome docs ([Content scripts — MAIN world](https://developer.chrome.com/docs/extensions/develop/concepts/content-scripts)). No `<script>` tag injection.
- `web_accessible_resources` is empty: there is no extension-served URL TM page code can reach, by design.
- `content_security_policy.extension_pages` allows `'unsafe-inline'` for **our own** popup/options styles only — page CSP is unaffected.

---

## 3. Module-by-module spec

For each module: purpose, public API, key invariants, what it must NEVER do.

### 3.1 `content/isolated/index.ts`

**Purpose:** Isolated-world entry. Orchestrates observer, overlay, panel mount, modal-watch, kill-switch, bridge.

**Public API:** none (entry point). Side effect on import: `start()`.

```ts
export async function start(): Promise<void>;
```

**Invariants:**
- Runs in isolated world. Cannot read `window.*` page globals; must request them via `bridge.requestPageState()`.
- All DOM writes use the shadow root or apply CSS classes from `styles/overlay.css` to TM seat nodes — never inline styles on TM nodes (round-1 §5 — keeps removal clean on kill-switch).
- Wraps the boot in `try { ... } catch (e) { killSwitch.trip(e) }`. A throw inside `start()` must never propagate to TM page code.

**Never:**
- Never adds any event listener to TM seat nodes (R2.5).
- Never calls `.click()`, `.dispatchEvent()`, `.focus()` on TM-owned elements.
- Never reads or sets `navigator.*`, `canvas.getContext`, WebGL params (architecture §1 non-goal).

### 3.2 `content/main-world/state-reader.ts`

**Purpose:** Read seat inventory from React fibers; post to isolated world.

**Public API:** none (entry). Posts on `window.postMessage` with envelope:
```ts
type Envelope =
  | { kind: 'tmx:state'; ts: number; seats: SeatRecord[] }
  | { kind: 'tmx:route'; ts: number; pathname: string }
  | { kind: 'tmx:ping'; ts: number };
```

Reader loop:
```ts
function tick(): void;          // called via requestIdleCallback at ~250ms cadence
function findFiberRoot(): Fiber | null;
function extractSeats(root: Fiber, path: string): SeatRecord[];
```

**Invariants:**
- Reads only. Never writes to fibers, never replaces React internals, never calls into the dispatcher.
- The fiber path is supplied via `tmx.selector-overrides` (`fiber.seatRecord`) or defaults from `lib/selectors-default.ts`.
- Posts use `window.postMessage(envelope, location.origin)`. Origin-locked.
- If three consecutive reads return no seats while the isolated-world observer reports `mapRoot` present, post `{kind:'tmx:state', seats:[]}` only once, then suspend the loop until a route change.

**Never:**
- Never patches `window.fetch`, `XMLHttpRequest`, `Response.prototype.*`, `Function.prototype.toString` (Strategy B is out).
- Never adds listeners on `window` for `'mousemove'`, `'click'`, `'keydown'` — anything that PerimeterX timing telemetry could correlate with user input.
- Never accesses `__REDUX_DEVTOOLS_EXTENSION__` even if present (subscribing would emit telemetry visible to PX).

### 3.3 `content/main-world/router-hook.ts`

**Purpose:** Detect SPA navigation and emit `tmx:route` so the isolated world can re-bind the observer.

**Public API:** installs side effects on `history.pushState` and `history.replaceState`. Also listens for `popstate`.

```ts
function install(): void;
```

Implementation sketch:
```ts
const origPush = history.pushState;
const origReplace = history.replaceState;
history.pushState = function (...args) {
  const r = origPush.apply(this, args);
  postMessage({ kind: 'tmx:route', ts: Date.now(), pathname: location.pathname }, location.origin);
  return r;
};
// mirror for replaceState; addEventListener('popstate', ...).
```

**Invariants:**
- Wraps `history.*` exactly once. Re-entry-guarded with a module-level flag.
- The wrapper calls the original first, then posts. Order matters: the user's navigation must not depend on our work.

**Never:**
- Never blocks/cancels a navigation. Never throws from the wrapper — `try/catch` around the postMessage.
- Never patches anything besides `history.pushState` / `history.replaceState`.

### 3.4 `content/isolated/overlay.ts`

**Purpose:** Add/remove `tmx-match-{hash}` and `tmx-dim-{hash}` classes on TM seat nodes based on the current candidate set.

**Public API:**
```ts
type SeatId = string;
export function apply(matches: Set<SeatId>, allSeats: Set<SeatId>): void;
export function clear(): void;
export function pulse(seatId: SeatId, opts?: { reducedMotion?: boolean }): void;
```

**Invariants:**
- Only mutates `classList`. No inline style. No attribute writes other than `classList`.
- Uses a build-hash-suffixed class name (e.g., `tmx-match-7f3a9b`) loaded from `lib/build-hash.ts`. The CSS rules in `styles/overlay.css` are templated at build time to match.
- Removes our classes before applying new ones — never leaves stale classes on a re-render.
- On `clear()` (kill-switch), iterates every node we ever tagged and removes our classes. Tracks a `WeakSet` of tagged nodes to make this O(tagged).

**`pulse(seatId, opts?)` invariants:**
- Idempotent. No-op if the seat node is not currently in the DOM (e.g., the seat was taken between read and click).
- No inline styles. Adds a `tmx-pulse-{buildHash}` class to the seat node; the animation is CSS-driven and lives in `styles/overlay.css`.
- Latest call wins: a subsequent `pulse(seatId)` (same or different seat) removes the pulse class from any in-flight target before retagging, so two rapid clicks don't double-animate or queue.
- Under `prefers-reduced-motion` (detected via `window.matchMedia` and passed in via `opts.reducedMotion`, or read directly when `opts` is omitted), the pulse degrades to an instant hold — full-opacity outer ring, no expansion — then fades. This matches the accessibility spec in `ux-mockup.md` Accessibility notes.
- The `SeatId → DOM node` lookup uses the `WeakSet`/`Map` of tagged nodes maintained by `apply()` (the same per-seat map keyed by `SeatId` that `apply()` populates from the observer tick). `overlay.ts` is the only module that owns this lookup; `pulse()` does not re-query the DOM.

**Never:**
- Never sets `data-*` attributes on TM nodes (would be visible as a behavior change).
- Never modifies TM-owned classes — only adds and removes our own prefixed classes.
- `pulse()` never sets inline `style` and never uses `Element.animate()` (we don't want a Web Animations API surface inspectable from the page).

### 3.5 `content/isolated/panel/`

**Purpose:** The right-pinned side-panel UI (Preact in a closed shadow root).

**Public API** (`mount.tsx`):
```ts
export function mountPanel(): { setStats: (s: Stats) => void; teardown: () => void };
```

**Data path for `setStats`:** the isolated-world entry (`content/isolated/index.ts`) is the sole caller. It computes stats from the seat reader on each observer tick and invokes `setStats({ matches, total, medianPrice, lastUpdatedAt })`. The popup's `47 matches` reads through the SW relay (per `feasibility.md` §R2.3), not through this function — `setStats` only feeds the in-page panel.

Components:
- `Panel.tsx` — layout shell, theme-token application on `:host`, modal-collapsed state binding.
- `SeatList.tsx` — top-10 list. `onSeatClick(seatId)` calls `overlay.pulse(seatId)`.
- `KillSwitchBanner.tsx` — visible when `useKillSwitch().tripped`.

**Invariants:**
- Renders inside `host.attachShadow({ mode: 'closed' })`.
- Host element is appended to `document.body`, never to a TM-owned subtree.
- All colors come from CSS custom properties set on `:host` via inline `style.setProperty('--tmx-*', ...)`.
- `aria-live="polite"` region announces candidate count changes (per Designer accessibility notes).

**Never:**
- Never queries TM DOM inside React components — components are pure renderers fed by store state.
- Never opens the shadow root in `open` mode.
- Never renders a "Buy" button or anything that performs a purchase action.

### 3.6 `lib/scoring.ts`

**Purpose:** Pure scoring formula. No I/O. Trivially unit-testable.

**Public API:**
```ts
export function score(seat: Seat, profile: ValueProfile, venue: VenueIntel, stageConfig: string): number;
export function rank(seats: Seat[], profile: ValueProfile, venue: VenueIntel, stageConfig: string): Seat[];
```

Formula (from architecture §7, unchanged):
```
score = sectionQuality * rowQuality * priceValue * adjacencyBonus
```

**Invariants:**
- Pure: same inputs → same outputs. No `Date.now()`, no `Math.random()`.
- Returns a number in `[0, 1]`. Asserts (in dev builds) the result is finite and in range.
- **Missing `sectionQuality` fallback:** if `venue.stageConfigs[stageConfig].sectionQuality[seat.section]` is `undefined` (we haven't authored intel for this section yet), `sectionQuality` defaults to a neutral `0.5`. Never throws on missing data; never silently drops the seat from ranking.
- **Tier-only price fallback:** when `seat.price` is `undefined` but `seat.priceTier` is set (Strategy A returned tier-only), `priceValue` is computed from `venue.tierToPriceRange[seat.priceTier]` midpoint. If the venue has no tier map either, `priceValue` defaults to `0.5` and the seat is flagged in the diagnostic log.

**Never:**
- Never reads `chrome.storage`, never calls `console.log` in production builds.

### 3.7 `lib/profile.ts`

**Purpose:** `ValueProfile` type and the predicate that decides whether a seat is a candidate.

**Public API:**
```ts
export type ValueProfile = {
  id: string;
  name: string;
  maxPrice: number;
  minPrice?: number;
  sectionWhitelist?: string[];
  sectionBlacklist?: string[];
  rowRange?: { min?: string; max?: string };
  minAdjacentSeats: number;
  preferAisle?: boolean;
  notes?: string;
};

export function matches(seat: Seat, profile: ValueProfile, neighbors: Seat[]): boolean;
```

**Invariants:**
- Pure.
- A seat with `price === undefined` (Strategy A returned tier only, no exact price) is treated as **passing** the price filter only if the corresponding tier maps under `maxPrice`. The mapping comes from `VenueIntel.tierToPriceRange` (added in §3.13 schema delta); `lib/profile.ts` reads it and uses the tier's `max` for the compare. If no tier map exists for the venue, the seat is treated as **passing** with a diagnostic-log warning (we'd rather show a candidate that turns out to be over budget than hide it silently).

**Never:**
- Never mutates `profile` or `seat`.

### 3.8 `lib/storage.ts`

**Purpose:** Wrap `chrome.storage` with typed get/set and a "lock to local snapshot" mode for drop-day (C7 / round-1 §5.4).

**Public API:**
```ts
export const StorageKeys = {
  profiles:    'tmx.profiles',        // sync
  armed:       'tmx.armed',           // sync
  venueOverrides: 'tmx.venues',       // sync
  selectorOverrides: 'tmx.selector-overrides',  // local
  canary:      'tmx.canary',          // local
  diagnosticLog: 'tmx.diagnostic-log',// local
  snapshotLock: 'tmx.snapshot-lock',  // local — { lockedAt, profileSnapshot }
} as const;

export async function get<K extends keyof Schema>(k: K): Promise<Schema[K] | undefined>;
export async function set<K extends keyof Schema>(k: K, v: Schema[K]): Promise<void>;
export async function lockToSnapshot(): Promise<void>;
export async function unlock(): Promise<void>;
export function onChanged<K extends keyof Schema>(k: K, fn: (v: Schema[K]) => void): () => void;
```

**Invariants:**
- Each key is bound to exactly one area (`sync` or `local`). The function picks the area; callers never specify.
- `lockToSnapshot()` reads the current armed profile from `sync`, writes a frozen copy to `local.snapshotLock`, and from that point all reads of `armed` return the local snapshot until `unlock()`.

**Never:**
- Never uses `chrome.storage.session` for the armed profile (session-scoped is wiped on browser close, which is the wrong default for drop day).
- Never writes user PII to storage. There is none to write.

### 3.9 `lib/selector-overrides.ts`

**Purpose:** Load user selector overrides from `chrome.storage.local`, merge with bundled defaults, expose a typed lookup.

**Public API:**
```ts
type SelectorMap = {
  seatNode: string;
  seatSection: string;
  seatRow: string;
  seatPrice: string;
  mapRoot: string;
  modalSignal: string;
};
type FiberMap = { seatRecord: string; seatPrice: string };

export async function loadSelectors(): Promise<SelectorMap & { fiber: FiberMap }>;
export function onSelectorsChanged(fn: (s: SelectorMap & { fiber: FiberMap }) => void): () => void;
```

**Invariants:**
- Bundled defaults from `selectors-default.ts` are the source of truth when no override exists.
- Override file is validated against a Zod schema on load; invalid override is logged and ignored (defaults win).
- Returns frozen objects.

**Never:**
- Never persists computed values from MAIN-world fiber reads back into the override file — that path is user-edit-only.

### 3.10 `lib/diagnostic.ts`

**Purpose:** Build the diagnostic dump (feasibility R2.4).

**Public API:**
```ts
export type DiagnosticDump = {
  ts: number;
  buildHash: string;
  url: string;
  seatNodesSample: { outerHTML: string; attrs: Record<string,string> }[];
  attrFrequency: Record<string, number>;
  mapRootHit: { selector: string; matchCount: number; first?: string };
  fiberSample?: Record<string, unknown>;
  canary: ReturnType<typeof import('./canary').readCanaryCookies>;
  suggestion: string;
};

export function buildDump(): DiagnosticDump;
export async function persist(dump: DiagnosticDump): Promise<void>;
```

**Invariants:**
- Truncates `outerHTML` samples to 500 chars each, max 5 samples.
- Stringifies fiber values to depth 2; never serializes functions, never serializes React internal pointers.

**Never:**
- Never sends the dump anywhere outside `chrome.storage.local`. There is no telemetry.

### 3.11 `lib/canary.ts`

**Purpose:** Detect which anti-bot is in front of TM this session (feasibility §6 new §13 / R2.7).

**Public API:**
```ts
export type CanaryReading = {
  ts: number;
  px3: boolean;       // PerimeterX/HUMAN
  pxhd: boolean;
  pxvid: boolean;
  abck: boolean;      // Akamai Bot Manager
  bmsz: boolean;      // Akamai
  cfbm: boolean;      // Cloudflare Bot Management
};

export function readCanaryCookies(): CanaryReading;
export async function recordOnBoot(): Promise<void>;
```

**Invariants:**
- Reads `document.cookie` only. Does not parse values; only presence of the cookie name.
- Persists at most 50 readings in `chrome.storage.local` (FIFO drop).

**Never:**
- Never sends a reading off-device.
- Never inspects cookie *values* (no risk of accidentally exfiltrating a session token in a log).

### 3.12 `popup/` and `options/`

**`popup/Popup.tsx`** — Pure Preact. Renders:
- Armed-profile dropdown (live from `chrome.storage.sync`).
- Toggles: overlay on/off, dim-non-matching on/off, side-panel on/off.
- Live stats from service worker (R2.3 fallback states).
- "Lock profile to local snapshot" toggle for drop day.
- Link to options page.

**`options/Options.tsx`** — Tabs:
1. Profiles (CRUD).
2. Venues (JSON-backed editor + import/export).
3. Selector Overrides (R2.4 editor).
4. Diagnostics (last N diagnostic dumps).
5. Canary (last N canary readings).

Both render in their own document context (not in TM page), so no Shadow DOM needed. They share styles via plain CSS modules.

### 3.13 `data/venues/`

Bundled venue intel JSONs. Initial keys (confirmed Seattle-area venue list): `climate-pledge-arena.json`, `t-mobile-park.json`, `tacoma-dome.json`, `lumen-field.json`. Data fills incrementally; each file may ship empty `sectionQuality` maps in v1. Schema as in architecture §8, plus the `tierToPriceRange` addition described in §3.6.

**Barrel:** `data/venues/index.ts` exports `const VENUES: Record<string, VenueIntel>` keyed by `venueId`. All venue lookups go through this map; per-file JSON imports stay internal to the barrel.

**Empty-state behavior:**
- `sectionQuality` may be `{}` in v1 (we author it as we attend shows). Scoring falls back to a neutral `0.5` when a section is missing from the map (see §3.6).
- `stageConfigs` MUST contain at least one entry per venue file (e.g., `"end-stage"`) so lookups never return `undefined` and the scoring path does not NPE. A venue with no real intel still ships a single empty stage config rather than an empty object.

---

## 4. Detection-canary spec

**Trigger:** content script start (`content/isolated/index.ts` → `start()` → first line: `await canary.recordOnBoot()`).

**Code path:**

```ts
// lib/canary.ts
export function readCanaryCookies(): CanaryReading {
  const cookies = document.cookie.split(';').map(s => s.trim().split('=')[0]);
  const has = (name: string) => cookies.includes(name);
  return {
    ts: Date.now(),
    px3:   has('_px3'),
    pxhd:  has('_pxhd'),
    pxvid: has('_pxvid'),
    abck:  has('_abck'),
    bmsz:  has('bm_sz'),
    cfbm:  has('__cf_bm'),
  };
}

export async function recordOnBoot(): Promise<void> {
  const reading = readCanaryCookies();
  const log = (await storage.get(StorageKeys.canary)) ?? [];
  log.push(reading);
  while (log.length > 50) log.shift();
  await storage.set(StorageKeys.canary, log);
}
```

**Surface:** options page → "Canary" tab. Read-only table of `ts | px3 | pxhd | pxvid | abck | bm_sz | __cf_bm`.

**Why this matters:** if a drop goes badly, the canary table tells us in retrospect what was in front of the page that day. If suddenly `__cf_bm` appears alongside the PX cookies, TM may have layered Cloudflare on top. That changes the threat model.

---

## 5. Replay harness

**Goal:** iterate the reader / overlay offline against a frozen capture of a real TM event page, without ever automating against prod.

### Capture

Manual, run from a hand-driven Chromium session (we are the user; this is not automation):

```bash
# In a Chromium window logged into TM (the user's own browser):
# 1. Open DevTools → Network → "Preserve log", "Disable cache" off.
# 2. Navigate to an event page; wait for the seat map to render.
# 3. Right-click the Network tab → "Save all as HAR with content".
# 4. Save the HAR as: tools/captures/<event-id>-<timestamp>.har
# 5. From the Console:
#      copy(document.documentElement.outerHTML)
#    and save to: tools/captures/<event-id>-<timestamp>.html
# 6. From the Console, optionally:
#      const fiberRoot = /* manual: walk to React fiber root via $0 */;
#      copy(JSON.stringify(serializeFiber(fiberRoot), null, 2));
#    save to: tools/captures/<event-id>-<timestamp>.fiber.json
```

A captured event is the triple `(HAR, HTML, fiber.json)`. The HAR provides response bodies; the HTML provides post-hydration DOM; the fiber.json provides Strategy C's expected payload shape.

### Replay

```bash
# Serve the captured snapshot locally:
node tools/replay-serve.mjs tools/captures/<event-id>-<timestamp>
# → http://localhost:5174/ serves a static page that:
#   - replays the HTML at /
#   - intercepts any matched fetch URL from HAR and replays the response body
#   - injects window.__TMX_SNAPSHOT_FIBER__ from fiber.json
#
# In a separate terminal, run the extension dev build:
cd extension && pnpm dev
# → loads at chrome://extensions (Load unpacked); content script will match
#   http://localhost:5174/event/* (a dev-only manifest entry in vite.config.ts).
```

`replay-serve.mjs` is small (~100 lines): `http.createServer` + a HAR matcher (URL, method) → returns the recorded `response.content.text` with the recorded headers. No live network.

### Why HAR + DOM (not just DOM)

Static HTML alone misses the post-load fetch responses Strategy C reads. HAR alone misses the hydrated DOM Strategy A reads. Both are needed.

### What this enables offline

- Selector regression tests (the canary in `lib/selectors-default.ts` against the latest captured event).
- Strategy C fiber-shape unit tests.
- Modal-collapse detection — capture an event after triggering a checkout modal, replay, assert `modal-watch.ts` flips correctly.

---

## 6. Build & ship checklist (before the first real drop)

A signed-off checklist. Every item is verifiable; none are vibes-based.

- [ ] **Manifest:** scoped to `https://www.ticketmaster.com/event/*` only; no `<all_urls>`.
- [ ] **Lint:** `eslint --max-warnings 0` passes with the §7 rules enabled.
- [ ] **Type-check:** `tsc --noEmit` clean.
- [ ] **Tests:** `vitest run` green for `lib/scoring.ts`, `lib/profile.ts`, `lib/selector-overrides.ts`, `lib/canary.ts`.
- [ ] **Replay harness:** an event capture from within the last 14 days replays cleanly; overlay applies; panel renders.
- [ ] **Kill-switch tested with a forced exception:** in dev build, throw inside the observer callback; assert overlay disables, panel shows the banner, no TM nodes are left with `tmx-*` classes, panic key combo (`Ctrl+Shift+0`) also trips it.
- [ ] **Diagnostic mode tested with intentionally-broken selectors:** override `seatNode` to `[data-nonexistent]`; assert diagnostic auto-trips within 3 ticks, dump persists to storage, options page renders the dump and the heuristic suggestion.
- [ ] **Override hot-reload tested:** edit `tmx.selector-overrides` in the options page while the event tab is open; selector swap takes effect on the next observer tick without page reload.
- [ ] **C6 modal collapse tested:** replay a capture with `body.modal-open`; assert panel collapses to 48px within one frame, restores on signal-off after 250ms debounce.
- [ ] **C9 confirmed in code:** grep for `addEventListener` against any selector resolved from `seatNode` returns zero hits.
- [ ] **Strategy B static check:** §7 ESLint rules pass; grep for `window.fetch =`, `XMLHttpRequest.prototype`, `Response.prototype` returns zero hits.
- [ ] **Canary recorded on boot:** load any TM event page; assert `tmx.canary` has at least one reading from the current session.
- [ ] **Service-worker stub:** SW handles `STATS_GET` and `STATS` messages; cold-start latency measured < 100ms with `performance.now()`.
- [ ] **`chrome.storage.sync` quota:** profile JSON size under 8KB per profile (Chrome's documented quotas: 8,192 bytes per item, 102,400 bytes total).
- [ ] **Shadow DOM mode:** every `attachShadow` call in source uses `{ mode: 'closed' }`. Grep confirms.
- [ ] **Build hash threaded:** `BUILD_HASH` is non-empty at runtime; `tmx-*` class names include it; rebuilding produces a different hash.
- [ ] **Soft-launch event chosen:** a low-stakes event (general-on-sale, plenty of inventory, no resale value if the account is suspended — e.g., a comedy show or local theater event) is bookmarked. We use this drop, not a target drop, as the first real run.
- [ ] **Rollback rehearsed:** the user has tested disabling the extension via `chrome://extensions` and confirmed TM works normally with it off. (Drop-day fallback: turn it off, refresh, continue manually.)

---

## 7. Do not implement (lint-enforced)

These patterns must never appear in source. Enforced via `eslint-plugin-no-restricted-globals`, `no-restricted-properties`, `no-restricted-syntax`, and `no-restricted-imports`. Sample config in `.eslintrc.cjs`:

```js
module.exports = {
  rules: {
    // 1. No fetch / XHR monkey-patching (Strategy B is permanently out).
    'no-restricted-syntax': ['error',
      {
        selector: "AssignmentExpression[left.object.name='window'][left.property.name='fetch']",
        message: "Strategy B is permanently out. Do not patch window.fetch."
      },
      {
        selector: "MemberExpression[object.object.name='XMLHttpRequest'][object.property.name='prototype']",
        message: "Do not touch XMLHttpRequest.prototype. Strategy B is out."
      },
      {
        selector: "MemberExpression[object.object.name='Response'][object.property.name='prototype']",
        message: "Do not patch Response.prototype. Strategy B is out."
      },
      {
        selector: "MemberExpression[object.object.name='Function'][object.property.name='prototype'][property.name='toString']",
        message: "Do not override Function.prototype.toString. PerimeterX inspects it."
      },
      // 2. No synthetic events on TM nodes.
      {
        selector: "CallExpression[callee.property.name='dispatchEvent']",
        message: "No synthetic events. The user clicks; we don't."
      },
      {
        selector: "CallExpression[callee.property.name='click']",
        message: "No programmatic .click(). The user clicks every seat themselves."
      },
      {
        selector: "NewExpression[callee.name='MouseEvent']",
        message: "No MouseEvent synthesis."
      },
      {
        selector: "NewExpression[callee.name='PointerEvent']",
        message: "No PointerEvent synthesis."
      },
      {
        selector: "NewExpression[callee.name='KeyboardEvent']",
        message: "No KeyboardEvent synthesis."
      },
      // 3. No fingerprinting surface access.
      {
        selector: "MemberExpression[object.name='navigator'][property.name='userAgent']",
        message: "Do not read navigator.userAgent. Fingerprint-adjacent."
      },
      {
        selector: "MemberExpression[object.name='navigator'][property.name='webdriver']",
        message: "Do not read navigator.webdriver."
      },
      {
        selector: "MemberExpression[object.name='navigator'][property.name='plugins']",
        message: "Do not read navigator.plugins."
      },
      {
        selector: "CallExpression[callee.property.name='getContext'][arguments.0.value=/^(webgl|webgl2|2d)$/]",
        message: "Do not call canvas.getContext. Fingerprinting surface."
      },
      // 4. No open shadow roots.
      {
        selector: "CallExpression[callee.property.name='attachShadow'][arguments.0.properties.0.value.value='open']",
        message: "Shadow roots must be closed."
      },
    ],
    // 5. No third-party telemetry.
    'no-restricted-imports': ['error', {
      paths: [
        { name: 'posthog-js',          message: 'No telemetry.' },
        { name: '@sentry/browser',     message: 'No remote error reporting.' },
        { name: 'mixpanel-browser',    message: 'No telemetry.' },
        { name: 'amplitude-js',        message: 'No telemetry.' },
      ],
      patterns: ['**/analytics/*', '**/telemetry/*'],
    }],
    // 6. No setting attributes on TM nodes (only classList).
    // Enforced by a custom rule in tools/lint-no-restricted.cjs:
    //   forbid setAttribute on nodes obtained via document.querySelector with a
    //   selector that resolves from the seatNode override. Approximate via
    //   a comment-tag convention: callsites annotated with /* @tm-node */
    //   must only invoke .classList.add / .classList.remove.
  }
};
```

**Plain-English summary of the ban list** (also lives at the top of `.eslintrc.cjs` as a comment):

- `window.fetch = ...` — banned. Strategy B is out.
- `XMLHttpRequest.prototype.*` — banned.
- `Response.prototype.*` — banned.
- `Function.prototype.toString` overrides — banned. PerimeterX checks this.
- `.dispatchEvent(...)` on any element — banned.
- `.click()` programmatic calls — banned.
- `new MouseEvent / new PointerEvent / new KeyboardEvent` — banned.
- `navigator.userAgent`, `navigator.webdriver`, `navigator.plugins` reads — banned.
- `canvas.getContext('webgl' | 'webgl2' | '2d')` — banned (we never draw on canvas; we never need to fingerprint).
- `attachShadow({ mode: 'open' })` — banned. Closed only.
- Third-party telemetry imports (`@sentry/browser`, `posthog-js`, `mixpanel-browser`, `amplitude-js`) — banned.
- Setting any attribute on a TM-owned node other than via `classList.add` / `classList.remove` of a `tmx-*` class — banned by convention with a custom rule.

---

## 8. UNKNOWNS to resolve before first real drop

Carried forward from feasibility R2.8. Coding can start; the build plan above is not blocked on these, but the soft-launch event cannot happen until each item is checked.

- [ ] SVG vs Canvas vs hybrid at the zoom-in seat-pick level.
- [ ] TM event-page CSP header.
- [ ] TM modal z-index ceiling during checkout.
- [ ] Whether TM uses `<dialog>` top-layer for any modal.
- [ ] Exact body-class or aria signal TM emits on modal open.
- [ ] React fiber shape on a seat node (for `selectors-default.ts → fiber.seatRecord`).
- [ ] Whether `mapsapi.tmol.io` static SVG is the zoom-out renderer.

Each unknown maps to a single ~15-minute DevTools task. Aggregate budget: one focused session of ~2 hours.
