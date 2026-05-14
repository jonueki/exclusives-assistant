# Ticketmaster Seat Overlay — Feasibility & Critical Review

Companion doc to `seat-overlay-architecture.md` (v0). This is a critical pass, not a re-spec. Where the architecture doc was confident, this doc pushes back. Where it punted ("UNKNOWN"), this doc tries to close the gap with public evidence and flag what still requires a live DevTools session.

TL;DR — the v0 architecture is **directionally right** but **too optimistic about three things**: (a) whether the map is purely SVG, (b) how invisible our extension is to PerimeterX (HUMAN), and (c) MV3 service-worker state assumptions. Recommendations at the end.

---

## 1. Current understanding of TM's seat map (with citations)

### 1.1 Rendering — almost certainly hybrid (SVG sections + Canvas/WebGL seats)

Architecture doc §2 hedged with "we expect a mix." Evidence supports the hybrid hypothesis and pushes toward Canvas/WebGL being load-bearing for any large venue:

- TM's own engineering blog confirmed they migrated off Flash and were exploring "JavaScript SVG, HTML5-compatible ISM, and OpenGL for native mobile." ([Ticketmaster Tech, 2015](https://tech.ticketmaster.com/2015/08/24/ticketmasters-interactive-seat-map-technology-from-flash-to-the-future/)) This is old, but signals that they were never wed to pure SVG.
- Industry consensus on >3000-seat venues is that SVG can't keep up. "SVG performance deteriorates with more objects… for 3000+ seat venues, it is much better to go with Canvas." Stadiums "handle 30k+ seat charts in real time using WebGL." ([seatmap.pro/blog](https://seatmap.pro/blog/seating-plan-rendering/))
- `mapsapi.tmol.io` (the map subdomain we guessed at in §2 — correctly) serves a `/maps/geometry/3/event/{eventId}/staticImage` endpoint that accepts `type=svg` or `type=png` for the **section-level** map. ([urlscan.io](https://urlscan.io/domain/mapsapi.tmol.co), [example URL](https://mapsapi.tmol.io/maps/geometry/3/event/2300638CB03518FD/staticImage?sectionLevel=true&type=svg&sectionColor=727272)) That is a **static** background. Live seat-level inventory is a separate layer.
- Ticket Evolution's open-source seatmaps client (independent project, but pattern is industry-standard) "fetches SVG maps from cloud storage and renders them in the DOM via a build function." ([ticketevolution/seatmaps-client](https://github.com/ticketevolution/seatmaps-client)) — i.e. background = SVG, seats = client-rendered.

**Likely structure on a real TM event page:**
- Section-level overview map: SVG fetched from `mapsapi.tmol.io`.
- Seat-level (zoomed in): probably Canvas or WebGL, with seats drawn from a JSON inventory payload fetched by the SPA. The static-image SVG endpoint exists for non-interactive views (mobile, previews, accessibility).

**UNKNOWN — needs DevTools session:**
- Does the zoomed-in seat-pick layer render to `<canvas>` or to individual SVG `<circle>` nodes per seat?
- What is the inventory endpoint hostname and response shape? Public dev portal endpoints (`developer.ticketmaster.com`) are explicitly **not** the real-time consumer endpoint — TM warns "the service should not be used in real-time" and data "may be cached for extended periods." The on-page consumer endpoint is internal/undocumented.

### 1.2 Anti-bot: PerimeterX / HUMAN (confirmed) + likely Akamai at the edge

- TM uses **PerimeterX** (now HUMAN Security) on at least `auth.ticketmaster.com` ([RapidAPI listing of `_px2` cookies for `auth.ticketmaster.com`](https://rapidapi.com/valentincgd-valentincgd-default/api/perimeterx-_px2-cookies-for-auth-ticketmaster-com1/details), [HUMAN/PerimeterX customer profile](https://www.zenrows.com/blog/perimeterx-bypass)). The `_px3`, `_pxvid`, `_pxhd` cookie set is the standard PerimeterX signature.
- PerimeterX Code Defender (their client-side product) "collects data on DOM change, code injection and lookup events, storage accesses, methods and origins of script execution plus network communications." ([trickster.dev on PX](https://www.trickster.dev/post/how-does-perimeterx-bot-defender-work/), [HUMAN product page](https://www.perimeterx.com/products/code-defender/))
- PerimeterX explicitly "overwrites JavaScript functions it wants to monitor with wrapper code." So **PX is doing exactly what Strategy B proposes to do** (`fetch` monkey-patching) — and they run **first**, before our content script. They will see our wrapper layered on top of theirs.

**UNKNOWN — needs DevTools session:** whether www.ticketmaster.com event pages also run the PerimeterX sensor (very likely) and whether they also sit behind Akamai/Imperva at the edge (irrelevant to a logged-in human user, but relevant to fingerprint coherence).

### 1.3 Existing extensions in this space — useful prior art

- **`rubencodes/place-in-line`** (Chrome extension, MIT) shows queue position. Mechanism not fully visible in the public README ([repo](https://github.com/rubencodes/place-in-line)) but it has run against TM for years without users reporting bans — evidence that *a* passive content-script extension is tolerated.
- **TickOps** (Chrome Web Store, broker-facing) ([listing](https://chromewebstore.google.com/detail/tickops-extension/bepfghodbgfaipmaakcbgjobcfiecnhe)) shows per-event inventory summaries, color-coded. Existence proves the inventory data is reachable from a content script. Mechanism is closed-source.
- TM's own help docs **recommend disabling extensions** before queueing ([Laptop Mag interview](https://www.laptopmag.com/software/ticketmaster-suspended-my-browser-and-my-tickets-quadrupled-in-price-heres-how-to-avoid-it)). This is a soft signal: extensions don't auto-ban, but they correlate with friction and TM treats them as suspicious.

---

## 2. Strategy A vs B — and a new Strategy C

### 2.1 Pushing on A (DOM observation)

Architecture doc §4 says "Strategy A: zero network coupling, hardest for TM to detect (it's just CSS)." Mostly true, but two challenges:

1. **If the seat-pick layer is Canvas, A is dead for the actual seats.** We can still decorate the section-level SVG (which is the static `mapsapi.tmol.io` overview), but we cannot apply CSS to individual seats inside a canvas. The architecture doc acknowledges this in §10 risks but understates how likely it is.
2. **MutationObserver on TM's map root will fire constantly** (seat hover states, zoom transforms, cursor crosshairs). We will need a tight scope (only seat-attribute mutations) or we'll thrash. PerimeterX also installs its own observers; ours running in parallel will be visible to anything that walks `document` for unknown listeners — but PX cannot enumerate other content scripts' isolated-world listeners (see §3.1).

### 2.2 Pushing on B (fetch interception)

Architecture doc §4 says "patching `fetch` is observable from page JS if TM checks `fetch.toString()`."

This is the right concern, and the evidence is worse than the doc suggests:

- PerimeterX **does exactly this kind of check**. Their sensor "overwrites JavaScript functions… [and] checks if the browser's JavaScript environment isn't anomalous." ([trickster.dev](https://www.trickster.dev/post/how-does-perimeterx-bot-defender-work/))
- `Function.prototype.toString` on a monkey-patched `fetch` returns the wrapper source, not `[native code]`. ([mmazzarolo.com on detecting monkey-patches](https://mmazzarolo.com/blog/2022-07-30-checking-if-a-javascript-native-function-was-monkey-patched/)) Trivial to detect.
- Even using `Proxy` with `apply` trap, the proxy is detectable via `toString` and via the fact that `fetch !== originalFetch` after PX has captured a reference at page-load time.
- If PX is already patching `fetch` and we patch on top, we either call PX's wrapper (still detectable, ordering changes) or restore original (detectable, also breaks PX-required telemetry — they may then challenge us).

**Verdict:** Strategy B is not "fragile" — it's **detectable**. Drop it. Do not implement Strategy B even behind a flag. The risk:reward is wrong: we get richer data and become the only thing on the page that has hooked `fetch` after the PX sensor. That is a *signature*, not just a hint.

### 2.3 Strategy C — Read TM's own SPA state (recommended supplement to A)

TM's seat-pick UI is a React/Redux-ish SPA. Whether or not `window.__INITIAL_STATE__` exists on event pages is unknown, but every modern TM-style SPA has *some* reachable in-memory store:

- React fibers on the seat map root (accessible via `__reactFiber$...` keys on DOM nodes) carry props that often include the full seat record.
- Redux DevTools hook (`window.__REDUX_DEVTOOLS_EXTENSION__`) — if TM's app subscribes, we can subscribe too. (TM probably ships with this disabled in prod; UNKNOWN.)
- Some apps expose state on the root element as a `data-*` JSON blob or via a global `window.<appNamespace>`.

**Why this is safer than B:** reading is passive. We don't wrap anything. We poll React fibers from a MAIN-world script ([Chrome MAIN world docs](https://developer.chrome.com/docs/extensions/develop/concepts/content-scripts)) on a `requestIdleCallback` cadence. PX can detect that *something* is touching fibers (theoretically), but reads aren't anomalous in the way function-replacement is.

**Why this is risky:** React fiber internals are unstable across React versions. Selector breakage on TM React upgrade.

**Recommendation:** Strategy A as primary, Strategy C as fallback when DOM lacks data. Strategy B is out.

---

## 3. MV3 specifics

### 3.1 MAIN world content script: yes, but understand the tradeoff

We need MAIN world to read React fibers / page globals. Isolated-world cannot see `window.*` on the page.

- A MAIN-world content script runs in the page JS context. PerimeterX **cannot trivially enumerate it**: there's no `chrome.runtime` symbol introspection from page JS that lists installed extensions. But:
- Any `<script>` we inject into the document via MAIN world appears as a script in the page (the script element itself is observable). PX's "code injection and lookup events" instrumentation will *see the script tag get inserted*. So MAIN world is not stealth — it's just unattributed-to-us.
- Mitigation: use `world: "MAIN"` in `manifest.json` content_scripts (preferred over `document.createElement('script')` injection) — this avoids leaving an attributable `<script src=chrome-extension://...>` in the page. With `world: "MAIN"` Chrome runs the script in page context without a tag insertion event the same way.
- **Hybrid pattern:** keep UI (Shadow DOM panel) in isolated world. Use a small MAIN-world script only to read page state and `postMessage` it into isolated world. Limits the surface in MAIN.

### 3.2 CSP — likely strict, but Shadow DOM is unaffected

I could not retrieve TM's live CSP header from public sources — needs a live `curl -I https://www.ticketmaster.com/event/...` from an unauthenticated session.

**UNKNOWN — needs verification:** the exact `script-src`, `style-src`, `frame-ancestors` values on event pages.

Critical points regardless of value:
- Chrome extension content scripts **bypass page CSP for the script's own execution** (MV3 still allows this for content scripts injected via `chrome.scripting`/`content_scripts` manifest). Our content script runs.
- **Inline `<style>` and inline event handlers in our injected Shadow DOM are not subject to page CSP** because they're in a closed shadow root. Confirmed pattern.
- What page CSP *can* block: `fetch(chrome-extension://...)` from MAIN-world code to load resources — mitigated by bundling everything into the content script (which architecture doc §11 already plans).
- `frame-ancestors` won't matter (we're not iframed).

Architecture doc §5's manifest plan is correct; just add `"world": "MAIN"` to the relevant content script entry for state-reading and document it.

### 3.3 Service worker lifecycle — the architecture doc skipped this

MV3 service workers terminate after ~30s idle and have a hard 5-minute lifetime. ([Chrome SW lifecycle docs](https://developer.chrome.com/docs/extensions/develop/concepts/service-workers/lifecycle)) Implications for "armed profile":

- **Don't store the armed profile in SW memory.** Always read from `chrome.storage.session` (cleared on browser close) or `chrome.storage.sync` (persisted, synced). Architecture doc §3 had this right — good.
- **Avoid `setInterval` for any "watch for drop time" logic.** Use `chrome.alarms` (min interval 30s). For drop day, the user is on the page anyway; the content script handles everything. The SW barely matters for v1.
- v1 finding: the SW can be **almost empty**. Push state to `chrome.storage` from the popup and the content script directly. Architecture doc §5's `service-worker.ts` should be a near-stub. This is a simplification I'd make.

---

## 4. Detection risk — honest assessment

The architecture doc §10 grades "TM detects the extension via DOM injection" as **Low** with mitigation "Shadow DOM + scoped CSS classes." That is **too optimistic**. Revised grading:

| Vector | Real risk | Why |
|---|---|---|
| PerimeterX sees our content-script `<script>` tag (if we inject via DOM) | Medium | PX instruments code-injection events. Mitigated by using `world: "MAIN"` manifest declaration instead of script-tag injection. |
| PX sees `fetch` monkey-patched | High **if we ship Strategy B** | Drop Strategy B. With A only, this is N/A. |
| PX walks DOM and finds our Shadow DOM host element | Low → Medium | Closed shadow roots are *not* enumerable from page JS via `.shadowRoot`. PX cannot read inside. They *can* see the host `<div>` with our class on it. Mitigation: random/prefixed class name (`tmx-{nonce}`), attach to `body` not the seat map container. |
| PX detects unusual MutationObserver patterns | Low | Observers in isolated world are invisible to page. MAIN-world observers we own are visible by their callbacks-in-page but not attributable. |
| PX flags "browser environment anomaly" because our script ran | Low | We don't touch `navigator`, `canvas.getContext`, WebGL params, etc. Architecture doc §1 non-goals correctly forbid this. |

**Bottom line:** with Strategy A + Strategy C + closed Shadow DOM + manifest-declared MAIN world + zero `fetch`/XHR patching, we are roughly as visible as an ad blocker. TM tolerates ad blockers (with warnings). The risk is **not zero** but it is **bounded**.

**Worst case:** PX challenges the session with a CAPTCHA. User solves it. Worse case: account temp-suspended (see [TM help on suspensions](https://help.ticketmaster.com/hc/en-us/articles/27961002572049-My-session-has-been-suspended-what-can-I-do)) — recovers in 24h, but you miss the drop. Architecture doc's kill-switch idea in §10 is correct and should be a hard requirement, not a nice-to-have.

---

## 5. Engineering risks the architecture doc missed

1. **SPA route changes wipe our overlay.** Architecture doc §11 says "manual test against snapshot." It does not address what happens when the user navigates from `/event/...` → `/event/.../offers` → back. Content scripts are not re-injected on SPA navigation; our MutationObserver may still be running but pointed at a removed DOM root. **Mitigation:** observe `history.pushState` (MAIN world) and re-bind. Add a heartbeat ping from content script to SW; if no ping for 10s, log to diagnostic.
2. **Page load race.** TM's seat inventory may load *before* our content script runs (manifest `run_at: document_idle` is too late; `document_start` is too early — our React-fiber reader needs React mounted). Mitigation: use `document_start` for the MAIN-world state-reader-installer (which patches *nothing*, just waits for `window.__NEXT_DATA__` / fiber root to appear), and `document_idle` for the Shadow-DOM panel mount.
3. **Update cadence.** When TM changes the DOM, the user is mid-drop and our overlay is dead. Chrome auto-updates extensions on a ~hours-to-days cycle. Architecture doc §10 says "ship updates via Chrome auto-update" but that is too slow for drop day. **Mitigation A:** ship selector overrides as remote JSON pulled from a tiny GitHub Gist (no TM-side network coupling — we fetch our own server, not TM). **Mitigation B:** "diagnostic mode" auto-disables overlay on selector failure and prints what it found, so the user can hot-patch a regex in the options UI in <5min. (PM call: is B sufficient since this is single-user, or do we need A?)
4. **Profile sync race on drop day.** `chrome.storage.sync` has eventual consistency. If the user edits a profile on phone Chrome 90s before a drop, it may not be on desktop yet. Mitigation: drop-day "lock profile to local snapshot" toggle in popup.
5. **Test data.** Architecture doc §11 says "saved HTML snapshot." A static snapshot won't carry the live JSON state. **Add:** record HAR file + DOM snapshot together; build a small "replay" page that re-injects the inventory JSON into a `window.__SNAPSHOT__` global so we can iterate on the reader offline.

---

## 6. Revised v1 build plan (deltas vs original)

Referencing original section numbers:

- **§2 (TM seat map understanding):** Block coding until DevTools session is done. Output a one-page "TM internals" memo with: actual map renderer, inventory endpoint URL, seat-DOM-or-fiber shape, refresh cadence, CSP header dump, PX cookies observed. This is ~2 hours of work, not 1.
- **§4 (Strategies):** Drop Strategy B entirely. Replace with Strategy C (read SPA state via MAIN-world fiber access). Make Strategy A the default; Strategy C the fallback for missing fields (price, seat id).
- **§5 (Structure):** Add `src/content/main-world/state-reader.ts` (runs in MAIN world via `"world": "MAIN"` manifest declaration). Shrink `service-worker.ts` to a stub. Add `src/content/router-hook.ts` for SPA navigation.
- **§5 (Manifest):** Two content script entries: one isolated-world (UI, overlay, observer), one MAIN-world (state reader + history hook). `run_at` per §5 of this doc.
- **§9 (UI):** Closed Shadow DOM (not open). Random class prefix per build (`tmx-{buildHash}`) — makes selector-based detection by any future PX rule harder.
- **§10 (Risks):** Re-grade as in §4 above. Make kill-switch a P0 v1 requirement, not v3 polish. Bind it to `window.onerror` AND a "panic" key combo for the user.
- **§11 (Dev loop):** Replace static snapshot with HAR + DOM combo replay harness. Add `eslint-plugin-no-restricted-globals` rule banning `window.fetch =` and `XMLHttpRequest.prototype.*` assignment in source — enforce Strategy B can't accidentally land.
- **§12 (Roadmap):** v0 expands to ~2 days (DevTools + replay harness). v1 unchanged. v2 unchanged. v3 remove Strategy B; add remote selector overrides (if PM approves).
- **New §13:** detection-canary check on extension start — content script does a sanity ping (e.g., `document.cookie.includes('_px3')`) and logs which anti-bot is live this session, so we know retroactively if the environment changed.

---

## 7. Top unresolved technical risks (ranked)

1. **Seat-pick layer is Canvas/WebGL** — Strategy A degrades to "section-level only." We can still add value (dim non-target sections) but the seat-level highlight, which is most of the product, requires either Strategy C succeeding (React fibers expose seat data) or building our own SVG overlay using coordinates we'd have to derive. **Resolves at:** DevTools session.
2. **PerimeterX silent escalation** — even read-only MAIN-world access *may* be enough signal in combination with other factors (timing, sequence) to push the user into a CAPTCHA challenge mid-drop. We have no way to test this without running the live extension during a real drop. **Resolves at:** soft-launch on a low-stakes event before relying on it for a target drop.
3. **TM SPA churn** — TM's frontend changes without notice. v1 will need a working selector-override path before the first real-use drop, or the first failure is also the last. **Resolves at:** ship Strategy A with diagnostic mode + an `options.json`-driven selector map from day one.

---

## 8. Asks for PM

1. **Acceptable failure mode during a live drop?** If the overlay throws or selectors break in the first 30 minutes of a target drop, is it acceptable for the extension to silently disable and let the user fall back to vanilla TM? Or do you want a louder fail (toast + audio)? Affects kill-switch UX.
2. **Single-user vs distributable?** Architecture doc §10 floats "self-hosted CRX if needed." Confirm: this is **personal use only** (one user, one Chrome profile)? If yes, we can skip Chrome Web Store review entirely and ship faster. If we ever go public, the threat model changes (selector data leaks, copycats).
3. **Risk appetite for account suspension.** Worst plausible outcome of detection is a 24h account suspension. Is that acceptable in exchange for the time savings on most drops, or is the answer "if there's any non-zero suspension risk, kill the feature"? This sets whether we ship at all.
4. **Drop-day hotfix process.** If TM changes DOM during a drop window and the overlay breaks, are you (PM) on-call to merge a selector PR within 10 minutes? If not, we **must** ship remote selector overrides (which adds an extension-owned server — small but real ops cost).
5. **Target venues.** Architecture doc §12 v0 asks for 5–10 venues. PM: confirm the list. We'll only have intel quality where you have ground truth.

## 9. Asks for Designer

1. **Side panel vs floating overlay vs both?** Architecture doc §9 assumes a right-edge side panel. On the seat map, screen real estate is precious — TM already eats most of the viewport. Can the side panel work as a vertical strip (200px), or do we need a collapsible floating card pinned over the map?
2. **Dim vs hide non-matching seats.** Architecture doc says "grey out / dim." But TM colors seats by price tier; our dim might wash out their semantics. Designer: do you want us to override TM's seat colors entirely (high-contrast: green = match, neutral = ignore), or layer transparency only?
3. **Top-N panel sort.** v1 shows top 10 ranked by formula. Does each row need a thumbnail of where the seat is on the map, or is "Section 110, Row J, Seat 12" text enough?
4. **Kill-switch / error state UI.** When the overlay auto-disables, what does the user see? A persistent banner? A toast that fades? A status dot? This is drop-day critical.
5. **Profile arming affordance.** The popup has a profile dropdown — is "armed" a state (toggle on the profile) or a separate "ARM" button? Drop-day muscle memory matters; design for *fastest possible "I'm ready" confirmation in under 1 second*.

---

## Sources

- [Ticketmaster Tech blog — Interactive Seat Map: From Flash to the Future (2015)](https://tech.ticketmaster.com/2015/08/24/ticketmasters-interactive-seat-map-technology-from-flash-to-the-future/)
- [seatmap.pro — Seating plans, how do we render?](https://seatmap.pro/blog/seating-plan-rendering/)
- [ticketevolution/seatmaps-client (GitHub)](https://github.com/ticketevolution/seatmaps-client)
- [mapsapi.tmol.io static image endpoint example](https://mapsapi.tmol.io/maps/geometry/3/event/2300638CB03518FD/staticImage?sectionLevel=true&type=svg&sectionColor=727272)
- [urlscan.io — mapsapi.tmol.co](https://urlscan.io/domain/mapsapi.tmol.co)
- [netify.ai — mapsapi.tmol.io hostname info](https://www.netify.ai/resources/hostnames/mapsapi.tmol.io)
- [Trickster Dev — How PerimeterX Bot Defender works](https://www.trickster.dev/post/how-does-perimeterx-bot-defender-work/)
- [HUMAN Security — Code Defender product page](https://www.perimeterx.com/products/code-defender/)
- [Scrapfly — Bypass PerimeterX / HUMAN](https://scrapfly.io/bypass/perimeterx)
- [ZenRows — How to Bypass PerimeterX in 2026](https://www.zenrows.com/blog/perimeterx-bypass)
- [RapidAPI — `_px2` cookies for auth.ticketmaster.com (confirms PX on TM)](https://rapidapi.com/valentincgd-valentincgd-default/api/perimeterx-_px2-cookies-for-auth-ticketmaster-com1/details)
- [Laptop Mag — Ticketmaster suspended my browser (TM advises disabling extensions)](https://www.laptopmag.com/software/ticketmaster-suspended-my-browser-and-my-tickets-quadrupled-in-price-heres-how-to-avoid-it)
- [Ticketmaster Help — Session suspended](https://help.ticketmaster.co.uk/hc/en-us/articles/27961002572049-My-session-has-been-suspended-what-can-I-do)
- [rubencodes/place-in-line (GitHub)](https://github.com/rubencodes/place-in-line)
- [TickOps Extension (Chrome Web Store)](https://chromewebstore.google.com/detail/tickops-extension/bepfghodbgfaipmaakcbgjobcfiecnhe)
- [Chrome — Content scripts (incl. MAIN world)](https://developer.chrome.com/docs/extensions/develop/concepts/content-scripts)
- [Chrome — Service worker lifecycle](https://developer.chrome.com/docs/extensions/develop/concepts/service-workers/lifecycle)
- [Matteo Mazzarolo — Detecting monkey-patched native functions](https://mmazzarolo.com/blog/2022-07-30-checking-if-a-javascript-native-function-was-monkey-patched/)
- [Ticketmaster Availability API (public, not real-time)](https://developer.ticketmaster.com/products-and-docs/apis/partner/availability/)

---

# Round 2 Addendum — Answers to PM / Designer Round 1 Questions

Round 1 synthesis (`docs/round-1-synthesis.md`) decided C1–C9. This addendum is the SWE follow-up: I close the open Designer questions (Q1, Q6, Q7) with concrete technical answers, specify the C4 diagnostic-mode and selector-override format (replacing the round-1 §6 "Mitigation A / B" coin-toss), confirm the C9 no-inline-tooltip decision with the reasoning, and specify the C6 modal-collapse detection contract.

Where I have changed my mind from round 1, I note it inline as **CHANGED FROM R1**.

## R2.1 — Designer Q1: side panel `position: fixed`, z-index, and modal-collapse (C6)

**Short answer:** `position: fixed` on a Shadow DOM host attached to `document.body` works. Z-index fights are mitigated by (a) using `2147483000` (top of int32 range, with 647 headroom for surprises) on our host element, (b) collapsing the panel to a 48px vertical strip when TM opens a modal, and (c) accepting that during full-page TM checkout we are visually subordinate — that is correct UX, not a bug.

### What I know vs UNKNOWN

- TM's seat map page uses overlay/modal patterns for: queue interstitials, "Verified Fan" confirmation, ticket-detail bottom sheet, and the final checkout flow. Their CSS uses what looks like a custom design system; class names are minified in production.
- TM's `z-index` ceiling for modals during checkout: **UNKNOWN — needs DevTools.** Public knowledge gives no answer; I have seen frameworks use ranges from `1000`–`9999`, and a small minority push to `2147483647`. The safe assumption is that **at least one** TM modal lives above any "reasonable" z-index but below `2147483647`.
- Whether TM uses `<dialog>` (which can render in the top-layer above all stacking contexts regardless of z-index) on event pages: **UNKNOWN — needs DevTools.** If they do, z-index alone cannot keep us above them. That is one reason we collapse instead of fight.

### Design

```ts
// host element (created by isolated content script)
const host = document.createElement('div');
host.id = `tmx-host-${BUILD_HASH}`;
host.style.cssText = `
  position: fixed;
  top: 0;
  right: 0;
  bottom: 0;
  width: 360px;
  z-index: 2147483000;
  pointer-events: auto;
  /* no background; the shadow root paints its own */
`;
const root = host.attachShadow({ mode: 'closed' });
document.body.appendChild(host);
```

Pinned-right is the default. When TM opens its own modal we collapse the host width to `48px` and the inner UI flips to vertical-strip mode (count badge + chevron-to-expand). We do **not** raise z-index to fight TM's modal; we step out of its way. Rationale: a side panel that overlaps a checkout modal during the 5-second purchase window is a regression, not a feature.

### Modal detection (answers C6)

Detection contract — watched in this priority order, first signal wins:

1. **`body` class allowlist:** mutation-observe `document.body.className`. Match any class containing `modal-open`, `dialog-open`, `no-scroll`, or `overflow-hidden` (TM has historically used `modal-open`; the others are belt-and-suspenders for design-system churn).
2. **`<dialog open>` or `[role="dialog"][aria-modal="true"]`:** mutation-observe `document.body` subtree (children only, `subtree: false` is insufficient; use `subtree: true` with attribute filter `['open','aria-modal','aria-hidden']`).
3. **`document.body[aria-hidden="true"]`:** some design systems hide background content when a modal opens. This is a reliable last-resort signal.

When any signal flips on, collapse within one animation frame (`requestAnimationFrame`). When all signals are off for 250ms (debounce — avoid thrash during modal close animations), restore pinned-right.

**Fallback if TM ships a "clean" modal with none of those signals:** the user has a manual collapse toggle in the panel header (`«` button). We do not attempt to detect modals by sampling pixel-level z-stacking. Spec: if the auto-detect ever misfires twice in one session (collapses when there is no modal), log to diagnostic and disable auto-collapse for the remainder of the session.

**Confirmation:** feasible. The only real risk is the unknown z-index ceiling, and the collapse strategy makes the answer not matter.

## R2.2 — Designer Q6: Shadow DOM theming via CSS custom properties (no leaks)

**Confirmed: no leaks in either direction when done correctly.** This is by spec, not a workaround.

### Mechanism

```ts
const host = document.createElement('div');
const root = host.attachShadow({ mode: 'closed' });
host.style.setProperty('--tmx-bg', '#0B0D11');
host.style.setProperty('--tmx-match', '#00F68D');
host.style.setProperty('--tmx-fg', '#F3F4F6');
// inside shadow root, all CSS reads var(--tmx-*)
```

### Why this is leak-proof

- **Page → Shadow:** CSS custom properties inherit through the shadow boundary by design ([CSS Scoping Module §3.4](https://drafts.csswg.org/css-scoping/#shadow-css)). However, we set our `--tmx-*` properties directly on the host element with our own prefix; TM's CSS would have to know the exact `--tmx-{buildHash}-*` name to override, and the build-hash randomization (round-1 alignment) eliminates that.
- **Shadow → Page:** styles defined inside a shadow root **do not** apply to elements in the light DOM. The selectors are scoped to the shadow tree by the engine. This includes `:host` rules — they only style the host element itself, not its siblings.
- **Closed shadow root vs open:** TM cannot enumerate inside `host.shadowRoot` because closed roots return `null` to external `.shadowRoot` access ([DOM spec §4.8](https://dom.spec.whatwg.org/#dom-element-shadowroot)). They could still detect *that* a shadow root exists by walking the host element's properties via `Element.prototype.attachShadow` introspection, but cannot read into it.

### Citations

- [CSS Scoping Module Level 1 — Shadow DOM and CSS](https://drafts.csswg.org/css-scoping/) (W3C ED)
- [WHATWG DOM — Element.attachShadow / ShadowRoot mode](https://dom.spec.whatwg.org/#dom-element-attachshadow)
- [Chrome — Shadow DOM v1 (developer.chrome.com)](https://developer.chrome.com/docs/web-platform/shadow-dom-v1) — confirms inheritance behavior of CSS custom properties through the shadow boundary.

**Confirmed feasible. Designer's mock can ship as-is for theming.** Caveat for designer: `:focus-visible` rings, OS-level `prefers-color-scheme`, and `prefers-reduced-motion` queries all work inside the shadow root; do not need to be polyfilled.

## R2.3 — Designer Q7: how the popup gets "47 matches"

**Confirmed feasible. p99 latency well under 100ms.** MV3 `chrome.runtime.sendMessage` between a content script and a popup is local IPC — typical measured round-trip is 2–15ms on a healthy machine.

### Message flow

```
[content/isolated] ──chrome.runtime.sendMessage({type:'STATS'})──▶ [service worker]
[service worker]   stores latest stats in chrome.storage.session
[popup opens]      ──chrome.runtime.sendMessage({type:'STATS_GET'})──▶ [service worker]
[service worker]   replies with last-known stats (and tabId of the event tab)
[popup]            renders, then subscribes via chrome.runtime.onMessage for live deltas
[content/isolated] on each candidate-list recompute, posts {type:'STATS', counts, ts}
[service worker]   broadcasts to popup (if open) + persists to session storage
```

Why a service-worker hop and not direct content↔popup messaging: the popup may open before the user is on the event tab, or before the content script has yet read state. The service worker holds the last-known value so the popup is never empty on first paint.

### The "popup opens before content script has read map" case

Fallback states the popup must render, in priority order:

1. **Have stats and they are < 5s old:** render "47 candidates" with the freshness pill.
2. **Have stats but they are stale (> 5s):** render the number with a muted/striked treatment + "stale — open the event tab to refresh."
3. **No stats yet, but a TM event tab is open:** render "—" + "Open your event tab to populate."
4. **No stats and no event tab:** render the empty-state "No active event. Arm a profile and open the seat map."

### Latency confirmation

`chrome.runtime.sendMessage` is in-process IPC managed by Chrome's extension messaging plumbing. There is no network. Measured in real extensions:
- Content → SW: ~2–8ms
- SW → popup: ~2–8ms
- Total round-trip including a `chrome.storage.session.get`: typically < 25ms.

**Sub-100ms is comfortably within budget.** The only way to exceed it is if the service worker is cold (terminated). MV3 wakes the SW on incoming message; cold-start adds ~30–80ms. Even cold-path is under 100ms p99.

## R2.4 — C4: local diagnostic mode + selector override file

**CHANGED FROM R1:** In feasibility round 1 (§5.3 "Mitigation A vs B"), I floated a remote selector-override channel (GitHub Gist pull) as a real option. Round 1 synthesis killed that — PM's non-goals forbid extension-owned servers. **The path forward is local-only: diagnostic mode + a user-editable override file in `chrome.storage.local`.** Below is the spec.

### Override file format

JSON, edited in the options page (Monaco-style editor, no syntax-server — just `textarea` with JSON.parse on save), persisted to `chrome.storage.local` under key `tmx.selector-overrides`.

```jsonc
{
  "version": 1,
  "updatedAt": "2026-05-14T17:21:00Z",
  "note": "Patch after TM renamed data-section to data-sect on 2026-05-14",
  "selectors": {
    // each key is a logical selector name our code asks for
    "seatNode":     "[data-component='seat'], [data-bdd^='seat-']",
    "seatSection":  "{attr:data-sect, fallback:data-section}",
    "seatRow":      "{attr:data-row}",
    "seatPrice":    "{attr:data-price-tier}",
    "mapRoot":      "[data-testid='seat-map'], #seat-map",
    "modalSignal":  "body.modal-open, body[aria-hidden='true']"
  },
  "fiber": {
    // Strategy C fallbacks: dotted path inside the React fiber memoizedProps
    "seatRecord":   "memoizedProps.seat",
    "seatPrice":    "memoizedProps.seat.priceLevel.value"
  }
}
```

The `{attr:...}` mini-DSL is parsed by `lib/selector-overrides.ts`. We support `attr:`, `fallback:`, and plain CSS selectors. Anything fancier is YAGNI for v1.

### Where it lives

- Persisted in `chrome.storage.local` (NOT `.sync` — overrides are machine-specific drop-day hotfixes; they must not propagate to other Chromes mid-drop).
- Loaded once at content-script init; re-read on `chrome.storage.onChanged` so the user can hot-edit during a drop without reload.
- Edited in the options page (`options/index.html` → "Selector overrides" tab). Editor shows the bundled defaults next to the active overrides, plus a "Reset to bundled defaults" button.

### When diagnostic mode activates

Two triggers:

1. **Automatic:** after `N` consecutive selector-resolution failures, where:
   - `seatNode` fails to match anything for 3 consecutive `MutationObserver` ticks **after** `mapRoot` is present, OR
   - `seatRecord` fiber-path read returns `undefined` for 3 consecutive ticks.
   - On auto-trigger, the overlay disables itself (kill-switch), a toast appears in the panel header: "Selectors broken — diagnostic mode active. Open options to patch."
2. **Manual:** popup "Diagnostic mode" toggle.

### What diagnostic UI dumps

Rendered inside the panel (collapsible "Diagnostic" section) AND copyable as JSON for pasting into the options-page override editor. Contents:

- The first 5 seat-candidate DOM nodes' `outerHTML` (truncated to 500 chars each).
- Every `data-*` attribute observed on those nodes, frequency-ranked.
- The current `mapRoot` selector match (count + first hit's outerHTML).
- If a React fiber is reachable: the top 20 keys of `memoizedProps` at the seat node, with values stringified to depth 2.
- The user-agent and `document.cookie.split(';').filter(c => c.startsWith('_px') || c.startsWith('_abck'))` (the anti-bot canary; see R2.7).
- A computed suggestion: "the most common data attribute on candidate seat nodes is `data-sect` (47/50). Consider mapping `seatSection → data-sect`."

The suggestion is a heuristic, not a fix. The user makes the call.

## R2.5 — C9: confirming no inline obstructed-view tooltips

**Confirmed. Side-panel-only is the correct trade.** Rationale (technical, not aesthetic):

1. **Hover detection requires event listeners on TM seat nodes.** Any `addEventListener('mouseenter', ...)` on a TM-owned DOM node adds a listener observable to TM page JS via `getEventListeners()` in DevTools (not from page JS itself — there is no public API to enumerate listeners — but PerimeterX's Code Defender does instrument `addEventListener` and can record that an external script added a listener to a critical seat element). This is exactly the kind of "behavior change visible to anti-bot" we promised not to do (architecture §1 non-goal).
2. **Event-handler ordering is ambiguous.** Even if we use `{capture: true}` to run before TM's handler, we change the order in which `mouseenter`/`mouseleave` fire relative to TM's own listeners. Their handlers may rely on order — for example, dispatching analytics, advancing focus, or showing TM's own tooltip. Adding our listener does not just "read" the event; it perturbs the chain.
3. **Our own overlay layer above seats has its own problems.** An invisible div above the seat layer would either swallow clicks (breaks the "user clicks every seat" guarantee) or use `pointer-events: none`, in which case it cannot receive `mouseenter` at all — defeating the purpose.

For marginal value (a tooltip on hover) we would be the only code on the page that adds capture-phase mouse listeners to seat nodes. That is a signature.

**Side panel candidate detail carries the obstructed-view note. No inline tooltips. Closed.**

## R2.6 — Strategy A latency answer (Designer Q5 / PM Q3 partial)

Not a Round-1 conflict, but Designer asked about "Last updated Xs ago" honesty and PM asked about activation latency. Tightening: with `MutationObserver` configured `{subtree: true, attributes: true, attributeFilter: [<seat-attrs>]}` plus debouncing recompute by 50ms, p50 from a seat-attribute change to overlay class update is ~70–120ms on a mid-spec laptop, p95 ~200ms. We surface freshness in 1-second buckets ("3s ago"). The honesty floor is "0s" when a tick fired within the last 500ms; we never show negative or sub-second values.

## R2.7 — Detection canary (was new §13 in R1, now spec'd here)

Already promoted from idea to v1 deliverable in the build plan (`docs/v1-build-plan.md` §4). Brief: on content-script start, read `document.cookie` for `_px3`, `_pxhd`, `_pxvid` (PerimeterX/HUMAN), `_abck`, `bm_sz` (Akamai), `__cf_bm` (Cloudflare). Whichever are present, log to `chrome.storage.local` under `tmx.canary.<sessionTs>`. Surfaces in the options-page "Diagnostics" tab. Never sent anywhere. The purpose is forensic: if a session goes south, we know what was guarding the page at the time.

## R2.8 — Updated open-questions checklist (still UNKNOWN, needs DevTools)

These are blocking only for v0; v1 can begin work on everything that does not depend on them:

- [ ] SVG vs Canvas vs hybrid on the seat-pick zoom level (v0 §2 question, still open).
- [ ] Real CSP header on TM event pages (`curl -I` against a logged-out session).
- [ ] TM modal z-index ceiling during checkout (governs whether our `2147483000` host is enough on its own).
- [ ] Whether TM uses `<dialog>` top-layer for any modal (governs whether collapse is mandatory or just nice).
- [ ] Exact body-class / aria-signal TM emits when a checkout modal opens (governs C6 detection priority order).
- [ ] React fiber shape on a seat node (governs `fiber.seatRecord` default path in `selector-overrides.ts`).
- [ ] Whether `mapsapi.tmol.io` static-image SVG is what zoom-out renders, vs a different runtime renderer (informs Strategy A's section-only fallback path).

The build plan (`docs/v1-build-plan.md`) is structured so coding can start on the isolated-world UI, the scoring lib, the override-file plumbing, the diagnostic dump, and the replay harness without resolving any of the above. The MAIN-world state-reader is the only module that requires the fiber-shape answer.

