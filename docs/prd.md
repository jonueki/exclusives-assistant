# Ticketmaster Seat Overlay — Product Requirements (v1)

**Status:** Round 2 — conflicts C1–C9 resolved  
**Author:** PM  
**Last updated:** 2026-05-14  
**Architecture ref:** `docs/seat-overlay-architecture.md`  
**Synthesis ref:** `docs/round-1-synthesis.md`

---

## 1. Problem Statement

Jonathan attends 10–20 concert drops per year and consistently loses good seats — not to bots, but to other humans who decide faster. Ticketmaster's seat map dumps 3,000+ seats at once with no signal about which 10 actually match his criteria. By the time he finishes scanning sections, cross-referencing prices, and checking row depth, a human in better position has already clicked. The problem is decision latency, not click latency. He needs the map to pre-answer "which seats are worth looking at" so his eyes lock on candidates within seconds of render, not minutes.

---

## 2. Target User

Single user: Jonathan. No other users in v1, no multi-tenant, no sharing.

**Context:**
- ~10–20 drops/year, mostly arena and mid-tier venue concerts.
- Budget: lower bowl, mid-row, under ~$250. Will not buy Platinum. Will not buy nosebleeds.
- Comfortable enough with a Chrome extension to install it unpacked if needed.
- Uses the standard Ticketmaster queue flow — no presale workarounds or automation.

---

## 3. Goals & Non-Goals

### Goals
- Reduce time-to-decision on drop day by surfacing only the seats that match a pre-configured value profile.
- Dim everything that doesn't match so the eye immediately goes to candidates.
- Rank candidates in a side panel so Jonathan can evaluate top-10 without scanning the map.
- Support multiple profiles (per artist/tour) configured before drop day.
- Bundle venue intel (section quality, obstructed views) for the ~10 venues Jonathan actually attends.
- Be invisible from Ticketmaster's perspective.

### Non-Goals (v1)
- **No automation of any kind.** No synthetic clicks, no auto-cart, no auto-checkout. See arch doc lines 20–28.
- **No additional network calls to Ticketmaster.** Read-only on the live page.
- **No DOM event spoofing, no queue automation, no fingerprint or anti-detect modifications.**
- **Not a resale tool.** No resale value data, no flip-profit scoring.
- **Not multi-tenant.** No accounts, no login, no cloud sync beyond `chrome.storage.sync`.
- **Not a general ticket-sniping or price-alert service.**
- **Not built for bots to ride on top of.** No public API, no remote scripting interface.
- **No extension-owned server, no remote selector overrides, no remote update channel beyond Chrome's own extension auto-update.** Drop-day hotfix path is local diagnostic mode + a user-editable selector override file. (C4)
- **Strategy B fetch interception is permanently excluded.** Not "deferred" — do not implement even behind a flag. (feasibility.md:64)
- **Unpacked extension, personal use only. No Chrome Web Store path planned for v1.** (C4, synthesis gap 3)
- **No inline obstructed-view tooltips.** Obstructed warnings appear in the side panel candidate detail only. (C9)
- **Confirmed venue list (Seattle-area):** Climate Pledge Arena, T-Mobile Park, Tacoma Dome, Lumen Field. No other venues bundled in v1.

---

## 4. Core User Journeys

### 4a. Pre-Drop Preparation (days before the sale)

Jonathan hears a tour is announced for a venue he knows — say, Climate Pledge Arena. He opens the extension options page and either selects an existing profile or creates one: "Lower bowl, Sections 110–114, rows D–P, max $230, 2+ adjacent." He loads or reviews the Climate Pledge Arena venue intel sheet (bundled in the extension per arch doc line 138), checks that the section quality numbers look right from his last show there, and adjusts if the stage config is different this time (e.g., end-stage vs in-the-round). He saves the profile and tags it to this drop. On drop day, he opens the TM event page in Chrome, sees the extension toolbar icon, confirms the right profile is armed in the popup, and clicks into the queue.

### 4b. Drop Day in Queue (waiting, pre-map)

Jonathan is in the Ticketmaster queue. The extension is dormant — it does nothing on the queue page and makes no network calls. The popup is accessible so he can swap profiles or toggle the overlay on/off if he realizes he armed the wrong one. No other activity. When the queue resolves and TM loads the event/seat-selection page, the content script activates.

### 4c. Live Seat Selection (the moment that matters)

The seat map renders. Within ~1 second of map render, the overlay activates: seats matching his armed profile are highlighted (vivid color or ring); every other seat is dimmed at a level that preserves spatial context — the bowl shape remains readable (ux-mockup.md:26). The side panel (Shadow DOM, right edge, collapsible) shows his top-10 candidates ranked by the scoring formula (`architecture.md:188`). Each row in the list shows: section, row, seat, price, score. He scans the list top-to-bottom. Clicking a row **pulses the seat on the map** (visual highlight) so he can locate it spatially. He then clicks the seat himself to begin checkout. Scroll-to-seat is v2 pending SWE confirmation that TM map state is safely reachable (C8). The entire process from map render to click target is under 15 seconds.

The toolbar popup primarily shows **"Armed: \<profile name\>"** status and a 1-click profile-switch dropdown for last-second swaps (C3). Deliberate pre-drop arming lives in the dedicated pre-drop arming screen (ux-mockup.md frame 5).

If seats go fast, the overlay refreshes on every `MutationObserver` tick — seats that disappear drop off the list automatically. A "last updated Xs ago" indicator (`architecture.md:257`) tells him when the data is fresh.

---

## 5. Functional Requirements (MoSCoW)

### Must Have (v1 ship-blocker)
- Activate content script on Ticketmaster event/seat-selection pages only (scoped URL match).
- Read seat section, row, and price (or price tier) from the live DOM via Strategy A (`MutationObserver`). When DOM data is too thin, fall back to Strategy C: read TM's SPA state via React fibers from a MAIN-world content script (`postMessage` into isolated world). (C1; feasibility.md:66–78)
- Apply visual overlay: highlight matching seats, dim non-matching seats. **Dim ON by default** at a level that preserves spatial context (bowl shape still visible). Toggle available for fully-off mode. (C5; ux-mockup.md:26)
- Toolbar popup: displays **"Armed: \<profile name\>"** status + 1-click profile-switch dropdown. Overlay toggle. (C3)
- At least one hard-coded or manually-entered profile (section whitelist, max price, min adjacent seats).
- "Last updated" freshness indicator.
- Zero network calls to Ticketmaster from extension code.
- All UI in Shadow DOM with scoped class prefix (`tmx-{buildHash}`) — no CSS leaks, no global DOM pollution. Random hash per build to harden against selector-based detection. (feasibility.md:153)
- **Kill-switch: Must Have.** Auto-disable the overlay and surface a visible-but-non-alarming banner if the extension throws or detects selector failure on a live event page. Auto-re-enable on next page load. Jonathan does not need to manually re-arm. Rationale: PerimeterX silent CAPTCHA escalation makes fail-safe a P0, not polish. (C2; feasibility.md:131)

### Should Have (v1 if feasible)
- Side panel with top-10 ranked candidates (scoring formula per `architecture.md:188`).
- Clicking a panel row **pulses the seat on the map** (visual highlight). Scroll-to-seat deferred to v2. (C8)
- Profile persistence in `chrome.storage.sync`.
- Multiple profiles, selectable via popup dropdown.
- Bundled venue intel for the confirmed Seattle-area venues (section quality, obstructed views, stage config): Climate Pledge Arena, T-Mobile Park, Tacoma Dome, Lumen Field.
- Dim toggle: "dim non-matching" vs "highlight only" — user-selectable in popup. Default is dim-on.
- **Pre-flight checklist** on the pre-drop arming screen (ux-mockup.md frame 5). Items: (1) logged in to TM, (2) card on file unexpired, (3) billing zip matches card, (4) captcha not currently triggered, (5) DND/notifications silenced, (6) **profile locked to local snapshot** (guards against `chrome.storage.sync` eventual-consistency race on drop day). (C7; feasibility.md:140)

### Could Have (v1 stretch)
- Options page with full profile CRUD editor.
- Options page with venue intel JSON editor and override capability.
- Per-tour profile tagging (associate profile to a specific upcoming event).
- Diagnostic mode: dump detected DOM attributes when overlay breaks, to speed up selector patching.
- Export/import profiles as JSON backup.
- Aisle-preference scoring bonus.

### Won't Have (v1 — explicit exclusions)
- Strategy B fetch interception — permanently excluded. Do not implement even behind a flag. (feasibility.md:64)
- Post-drop logger (what you targeted vs what you got) — v2.
- Adaptive/learned scoring weights — v2+.
- Any automation: no auto-click, no auto-cart, no queue re-join, no refresh loops.
- Any data sent to an extension backend. No backend in v1.
- Remote selector overrides or any extension-owned server. Local diagnostic mode + user-editable selector override file is the only drop-day hotfix path. (C4)
- Inline obstructed-view tooltips on the seat map. (C9)
- Scroll-to-seat from the side panel — v2. (C8)

---

## 6. Success Metrics

This is a single-user tool. Engagement and retention metrics don't apply. Measure outcomes:

| Metric | Baseline | Target |
|---|---|---|
| **Seat win rate** — % of drops where Jonathan gets a seat matching his profile criteria | Estimated ~30% today (gut check) | 60%+ after 10 drops with extension active |
| **Time-to-click** — elapsed seconds from seat map render to Jonathan's seat click | No baseline yet; establish in first 3 drops | Median under 15 seconds |
| **False-positive rate** — % of highlighted seats that don't actually match the armed profile | — | 0% (correctness requirement, not a stretch goal) |
| **False-negative rate** — % of seats that match the profile but weren't highlighted | — | < 5% (some tolerance for edge cases if DOM data is sparse) |
| **Extension-caused miss rate** — % of drops where the extension broke and Jonathan missed the window because he was troubleshooting it | — | 0% (kill-switch must catch errors and fall back gracefully) |

Instrument by: after each drop, Jonathan logs into a lightweight post-drop note (manually, in the options page or a simple text field): did he get a seat? Did it match the profile? How long did it feel? This is a 30-second self-report, not telemetry.

---

## 7. Open Questions & Risks

### Risk Floor (resolved — synthesis gaps)

- **Account suspension tolerance:** Tolerate CAPTCHA challenge mid-drop. Do NOT tolerate account suspension. Kill-switch is P0 precisely because a missed drop is acceptable; a suspended account is not. Soft-launch the extension on a low-stakes event before relying on it for a target drop. (synthesis gap 2; feasibility.md:131)
- **Drop-day hotfix SLA:** None. Single user, no on-call. Diagnostic mode + local selector override file is the only hotfix path. If this proves insufficient after the first real drop, revisit remote overrides in v2. (C4; synthesis gap 3)
- **Post-drop logging:** Confirmed v2. No surface in v1. (synthesis gap 4)

### Resolved (no longer open)

| # | Question | Decision |
|---|---|---|
| Q1 | SVG vs Canvas? | Still needs DevTools session — highest-risk unknown. Strategy C added as fallback precisely because Canvas/WebGL would kill Strategy A for seat-level data. (feasibility.md:163) |
| Q2 | Price data on DOM nodes: exact or tier? | Strategy C (React fiber read) is the fallback for missing price. If both A and C fail, display tier. Side panel design must accommodate tier-badge fallback (ux-mockup.md:69). |
| Q4 | Scroll-to-seat feasibility? | Resolved: v1 ships pulse-on-map. Scroll-to-seat is v2. (C8) |
| Q5 | Kill-switch: auto-re-enable on next load? | Yes. Auto-re-enable on next page load. No manual re-arm required on drop day. (C2) |
| Q6 | Dim on by default? | Yes. Dim ON by default at a level that preserves bowl-shape spatial context. Toggle available for fully-off. (C5; ux-mockup.md:26) |
| Q9 | Score display: raw 0–1 or stars? | 0–1 bar + decimal. Stars feel like product reviews; percentages imply a probability we can't promise. (synthesis alignment; ux-mockup.md:31) |
| Q10 | Overlay color for matches? | Neon mint (`#00F68D`). High contrast on dark bg; not in TM's palette. (ux-mockup.md:23) |

### Still Open (for SWE before coding starts)

1. **SVG vs Canvas** (`architecture.md:37`): Needs DevTools session. Determines whether Strategy A delivers seat-level highlights or degrades to section-level only.
2. **`MutationObserver` timing**: p50/p95 activation latency from map render to overlay render. If > 500ms, move trigger earlier (feasibility.md:138).
3. **Side panel z-index vs TM modals** (C6): SWE to confirm `position: fixed` panel feasibility and propose the collapse-to-48px-strip behavior when TM opens a checkout modal. Designer to mock the collapsed state. Not yet resolved — needs round-3 confirmation.
4. **`window.__REDUX_DEVTOOLS_EXTENSION__`**: Does TM's prod build subscribe? Determines whether Strategy C has a simpler Redux path or must rely on raw fiber walking. (feasibility.md:70)
5. **CSP header on event pages**: Needs live `curl -I` from an unauthenticated session. (feasibility.md:97)

### Still Open (for Designer)

7. **Override: can Jonathan pick any seat even if dimmed?** No interception — he can always click any TM seat. But should the side panel support manually adding a dimmed seat to candidates? Lean no in v1 (YAGNI; toggle dim off if needed).
8. **Panel auto-open signal**: Auto-open side panel when a profile is armed and the seat map renders (option b from round 1). Designer to validate this is the right default trigger.

### Venue list — pending user confirmation

Confirmed: **Climate Pledge Arena, T-Mobile Park, Tacoma Dome, Lumen Field** (Seattle-area). Venue intel JSON to be authored for these four. (synthesis gap 1 — resolved)

---

## 8. Out of Scope (v1) — Candidates for v2+

| Feature | Why deferred / excluded |
|---|---|
| Strategy B fetch interception | **Permanently excluded.** Detectable by PerimeterX; risk:reward is wrong. (feasibility.md:64) |
| Inline obstructed-view tooltips | **Permanently excluded from v1.** Would require hover detection or our own z-layer over TM seats — detection surface + complexity for marginal value. Obstructed info lives in side panel detail only. (C9) |
| Scroll-to-seat from panel | v2. v1 ships pulse-on-map. SWE must confirm TM map state is safely reachable first. (C8) |
| Remote selector overrides / extension-owned server | v2 at earliest, and only if local diagnostic + override file proves insufficient on first real drop. (C4) |
| Post-drop outcome logger | Valuable for improving the scoring formula; not needed to get value from v1. |
| Adaptive scoring (learn from choices) | Requires a corpus of drops. Build it after 10+ logged drops. |
| Safari / Firefox support | Chrome-only in v1 per hard constraint. |
| Additional venues beyond initial placeholder list | Add as Jonathan attends new shows. |
| Sharing profiles or venue intel with others | Out of scope permanently unless the project expands beyond single-user. |
| Presale code management or presale flow | Different page flow; out of scope unless Jonathan specifically targets presales. |
| Floor/GA section handling | GA is usually first-come-first-served with no seat selection; overlay doesn't apply. Flag if this assumption is wrong. |
| Chrome Web Store distribution | Personal use only; unpacked extension. No CWS path planned for v1. |

---

## Asks for Designer (updated round 2)

- Color system resolved: neon mint (`#00F68D`). No further ask. (ux-mockup.md:23)
- Score display resolved: 0–1 bar + decimal. (ux-mockup.md:31)
- **Still open:** Mock the collapsed 48px side-panel strip state for when TM opens a checkout modal (C6 — not yet resolved, needs round-3 SWE confirmation first).
- **Still open:** Kill-switch auto-disable banner is now a v1 deliverable (C2). Need spec for the banner: visible but non-alarming; does not require user interaction to dismiss (auto-clears on next page load).
- **Still open:** Add "lock profile to local snapshot" as item 6 to the pre-flight checklist in frame 5 (C7).
- **Still open:** Update frame 4 panel footer microcopy from "click to scroll" to "click to pulse" (C8). The `›` indicator can stay.

## Asks for SWE (updated round 2)

- **DevTools audit (blocker):** Still required before coding. SVG vs Canvas, seat node selectors, price data, refresh cadence, CSP header, PX cookies. (feasibility.md:149)
- **Strategy A + C viability after audit:** Confirm whether Strategy C (React fiber read) successfully fills the price field when Strategy A produces only tier. If both fail, confirm the tier-badge fallback UI path with Designer.
- **C6 — Side panel z-index:** Confirm `position: fixed` feasibility and specify the collapse-to-48px-strip trigger (e.g., `.modal-open` body class or equivalent). Designer blocks on this for the collapsed-state mock.

---

## New Conflicts Surfaced in Round 2

None discovered. C6 (side panel collapse on TM modal) is still unresolved from round 1 — intentionally deferred to round 3 pending SWE confirmation of the `position: fixed` + z-index approach. Not a new conflict; synthesis already flagged it.

---

## Changelog

| Round | Changes |
|---|---|
| Round 1 | Initial PRD. |
| Round 2 | Applied decisions C1–C9 from `round-1-synthesis.md`: added Strategy C as in-v1 fallback (C1); promoted kill-switch to Must Have with PerimeterX rationale (C2); updated popup to hybrid status/arm + pre-drop arming screen reference (C3); locked no-backend non-goal to include no remote selector overrides (C4); specified dim default at spatial-context-preserving level (C5); added pre-flight checklist as Should Have with lock-profile item (C7); changed scroll-to-seat to pulse-on-map in v1 (C8); added explicit inline tooltip exclusion to §8 (C9). Resolved answered open questions into §7 resolved table. Added venue placeholder list, risk floor, and distribution confirmation. Stale asks for Designer/SWE pruned and updated. |
