# UX Mockup — Seat Overlay Chrome Extension

**Status:** Round 2 — synthesis decisions applied
**Last updated:** 2026-05-14
**Synthesis ref:** `docs/round-1-synthesis.md`
**PM ref:** `docs/prd.md`
**SWE ref:** `docs/feasibility.md`

## Figma file

https://www.figma.com/design/U21KFRTw7tkmnbx6UnUefy

Round 1 frames 1–6 live on Page 1, left-to-right in drop-day order. Round 2 appends frames 7–8 to the right of frame 6 and modifies frames 1, 4, and 5 in place.

> **Figma sync note (2026-05-14):** Figma MCP write calls are blocked by the Starter-plan tool-call limit on this account. The frame specs below (frames 7 and 8, plus diffs for 1, 4, 5) are written tightly enough to apply by hand or via a single `use_figma` batch once the limit resets. No frame on Page 1 has been touched yet in this round.

## Round 2 changes

What changed from round 1 and why. Each row cites the source of the change.

| # | Change | Source | Where it lands |
|---|---|---|---|
| C2 | Kill-switch promoted to Must-Have; added three explicit states (active pill, auto-disabled banner, diagnostic panel). | `prd.md:94` moves it from Should → Must; `feasibility.md:131` calls it "hard requirement, not nice-to-have"; `round-1-synthesis.md:36` | New frame `8. Kill-switch states` |
| C3 | Popup rebuilt as a status + last-second-swap surface. Arming dropped from primary surface; toggles demoted to a Quick Toggles row. | `prd.md:69`, `round-1-synthesis.md:41` | Modify frame `1. Toolbar Popup` |
| C5 | Dim default tuned so the bowl shape reads and candidate clusters jump out at a glance. | `prd.md:67` ("level that preserves spatial context"), `round-1-synthesis.md:52` | Modify frame `4. Live seat map · overlay on` |
| C6 | Side-panel collapsed strip (48px) for when TM opens a checkout modal. | `feasibility.md` z-index concern + `round-1-synthesis.md:56` | New frame `7. Side panel — collapsed strip` |
| C7 | Added 7th pre-flight item: "Profile locked to local snapshot". Hover-explainer references chrome.storage.sync eventual consistency. | `feasibility.md:140` (§5.4), `round-1-synthesis.md:61` | Modify frame `5. Pre-drop arming` |
| C8 | Candidate `›` action now triggers a pulse-on-seat glow (not scroll). Panel footer microcopy updated. | `prd.md:67`, `round-1-synthesis.md:66` | Modify frame `4. Live seat map · overlay on` |
| A11y | Match seats get a subtle outer ring in addition to mint color (color-blind-safe). | `round-1-synthesis.md:116`, deuteranopia note | Modify frame `4` + Accessibility section below |

## Frames

| # | Frame name | Size | What it shows |
|---|---|---|---|
| 1 | `1. Toolbar Popup` *(modified R2)* | 360x480 | Status surface. "Armed: <profile name>" in big type at top with a 1-click profile-switch dropdown. Live page stats (matches / total / median price). Quick Toggles row at bottom: Overlay on/off, Dim on/off. Link to options. |
| 2 | `2. Options · Profile Editor` | 1280x800 | Full-page profile CRUD. Two-column: profile list left, structured editor right. |
| 3 | `3. Options · Venue Intel` | 1280x800 | Venue list left, per-venue editor right with stage-config tabs + section quality table. |
| 4 | `4. Live seat map · overlay on` *(modified R2)* | 1440x900 | The hero. Tuned dim level (bowl shape readable). Matching seats: neon mint fill + soft glow + 1px lighter outer ring (color-blind-safe). Side panel pinned right with top-10 candidates + score bars. Clicking a row pulses the seat on the map (no scroll). |
| 5 | `5. Pre-drop arming` *(modified R2)* | 560x680 | Pre-drop ritual screen. 7-item pre-flight checklist now includes "Profile locked to local snapshot". |
| 6 | `6. First-run / empty` | 1280x800 | Install welcome. Three-step explainer, primary "Create first profile" CTA, trust footer. |
| 7 | `7. Side panel — collapsed strip` *(new R2)* | 1440x900 | Collapsed 48px vertical strip pinned right, shown when a TM checkout modal is open. |
| 8 | `8. Kill-switch states` *(new R2)* | 1440x900 | Three states side-by-side: active pill, auto-disabled neutral-amber banner, diagnostic mode panel. |

## Frame 7 spec — `7. Side panel — collapsed strip` (new)

**User goal:** When TM opens a checkout modal, the side panel must not fight for z-index with TM's UI, but the user must still see that we're alive and be able to expand back to the full panel.

**Entry point:** Whenever the content script detects `body.modal-open` (or equivalent — SWE to confirm in `feasibility.md`). Auto-collapses on detection; user can also collapse manually.

**Layout (1440x900 canvas):**
- Background: dimmed mock of a TM checkout modal (gray rectangle centered, ~720x560, with placeholder "Reviewing your tickets" header and a list of seats). Behind the modal: dimmed seat map from frame 4.
- Pinned right edge: 48px wide strip, full viewport height, surface `#11141A` (same as full panel), 1px left border `#1F2430`.
- Strip contents, top to bottom, centered horizontally:
  1. 8px top padding
  2. Match-count number, e.g. `47`, in 18px Inter Semi Bold, color `#00F68D` (mint match color)
  3. Tiny rotated label `CANDIDATES`, 10px tracking +0.08em, color `#9CA3AF`, rotated -90deg, vertically centered below the count
  4. Expand caret `‹` (24px hit target, 16px glyph), color `#F3F4F6`, bottom-anchored at 16px from bottom
  5. Optional: small `tmx` wordmark dot at very bottom for identity

**UI states:**
- Default: as above.
- Hover/focus on strip: background lightens to `#161A22`, caret pulses subtly.
- No matches (count = 0): show `0` in dim `#6B7280`, no glow.
- Diagnostic mode active (see frame 8): strip background gets a 2px left edge in `#E0A458` (neutral amber), count is replaced with `!` glyph.

**Edge cases:**
- Modal closes → strip auto-expands to full panel (re-uses frame 4 layout).
- Viewport < 1024px wide → strip remains 48px (already minimal).
- Count > 99 → display `99+`.

**Accessibility:**
- Strip is a `<button aria-expanded="false" aria-label="Expand candidates panel, 47 matches">`. On expand, focus moves to first candidate row.
- Tab order: strip is between TM's last focusable element and TM's modal close button — but since it lives in a Shadow DOM, it'll be at the end of the document tab order; this is acceptable for v1.
- Match count change announces via the same `aria-live="polite"` region as the full panel.

**Microcopy:**
- Strip label: `CANDIDATES`
- Strip aria-label: `Expand candidates panel, {n} matches`
- Expand caret tooltip: `Expand panel`

## Frame 8 spec — `8. Kill-switch states` (new)

**User goal:** When the overlay breaks mid-drop, the user must (a) know in <1 second that vanilla TM is back, (b) not waste time troubleshooting, and (c) be able to grab the diagnostic info in one click if they want to hot-patch.

**Layout (1440x900 canvas):** Three columns, each 460px wide with 30px gutters, all on a unified mock TM seat-map background.

### State A — Overlay active (column 1, leftmost)
The baseline. Frame 4's overlay running normally, plus one new affordance:
- **Status pill** anchored top-right of the side panel header: `● Overlay active`. 8px circle in mint `#00F68D` + label in `#F3F4F6`, 12px. Background `#11141A`, 8px radius.
- Hover: pill expands inline to show last MutationObserver tick: `● Overlay active · updated 2s ago`.

This pill lives in the full panel header in production (not a separate frame in v1 if space is tight — but show it here for the state comparison).

### State B — Auto-disabled banner (column 2, middle)
- Full-width banner pinned to the top of the TM page (above TM's own nav), 56px tall, background `#1F1A12` (deep amber-tinted neutral, not red), 1px bottom border `#3A2E1A`.
- Left icon: 20px warning glyph in `#E0A458` (amber, neutral). Not the red of "danger".
- Primary text (16px Inter Medium, `#F3F4F6`): `Overlay disabled — selector mismatch. Falling back to vanilla view.`
- Secondary text (13px Inter Regular, `#C9CDD4`): `Open extension for diagnostics.`
- Right side: two buttons in a row:
  - Ghost button: `Dismiss` (closes banner; overlay stays off until next page load)
  - Filled mint button: `Open diagnostics` (opens frame 8 column 3 in production; opens popup-anchored panel in real life)
- Below the banner, the TM page renders normally. The side panel is gone. Match-count strip is gone.

**Why neutral amber, not red:** the overlay failing is not an emergency — vanilla TM still works. Red would manufacture stress in the exact moment the user needs to stay calm. Amber says "heads up", not "crisis".

**Microcopy alternatives considered and rejected:**
- "Error: overlay crashed" — too negative; the user didn't do anything wrong.
- "Something went wrong" — vague; doesn't tell the user what to do.
- "Click here to fix" — false promise; we don't know if it's fixable in <5 min.

The accepted copy tells the user (a) what happened (selector mismatch), (b) the impact (vanilla view), (c) what to do next (open diagnostics). That's the error-message rule from our principles.

### State C — Diagnostic mode panel (column 3, rightmost)
Replaces the side panel content while diagnostic mode is on. Same 360px width as the full panel, same surface.

Sections, top to bottom:
1. **Header**: `Diagnostic mode` in 16px Inter Semi Bold, `#F3F4F6`. Below: `Last error: SeatNodeMissing — 14:32:11` in 12px `#9CA3AF`.
2. **Expected vs found** card (background `#161A22`, 8px radius, 16px padding):
   - Two-column compare: left col `Expected` / right col `Found`.
   - Row 1: `Selector` — `[data-bdd="seat-node"]` vs `[data-tm-seat]` (different)
   - Row 2: `Required attrs` — `section, row, seat, price` vs `section, row, seat` (price missing)
   - Row 3: `Sample node count` — `~2400` vs `2387`
   - Mismatches highlighted with a 1px left border in `#E0A458`.
3. **Detected node HTML** card: monospace 11px code block with the actual outerHTML of the first seat node, wrapped, max 4 lines visible with a vertical scroll. Copy icon top-right of card.
4. **Attribute dump** card: key/value list of every `data-*` attribute on the first seat node, 11px monospace. Copy icon top-right.
5. **Action row** (sticky to bottom):
   - Primary mint button: `Apply suggested override` (mocked, no-op label is `Apply override (preview)`).
   - Ghost button: `Open options to edit` — deep-links to `chrome-extension://.../options.html#overrides`.
   - Tertiary text link: `Copy all diagnostics` — copies a single JSON blob the user can paste into a Gist.

**UI states for State C:**
- Loading (running the diff): skeleton blocks in the Expected vs Found card.
- Empty (no error yet, user manually opened diagnostic mode): "No errors recorded this session" empty state with a `Run sanity check` button.
- Override applied: card shifts to a green-outline confirmation: `Override active — overlay will retry on next mutation`.

**Accessibility:**
- Banner has `role="status"` with `aria-live="polite"` — not "assertive". This is informational, not interruptive.
- Diagnostic mode is a `<section aria-labelledby="diag-title">`. All code blocks have `lang="html"` or `lang="json"` attributes so screen readers don't try to pronounce them.
- All copy-to-clipboard buttons announce "Copied" via a transient `aria-live` region.

**Edge cases:**
- Repeated errors (selector keeps failing every observer tick): debounce the banner — show once, don't re-fire. Diagnostic panel updates silently.
- Diagnostic mode + collapsed strip (TM modal open): strip shows `!` and amber edge per frame 7; clicking expands directly into diagnostic panel, not the candidate list.
- User dismisses banner, then opens popup: popup shows a small "Diagnostic mode active — open page panel" line under the armed-profile header.

## Frame 1 diff — `1. Toolbar Popup` (modified R2)

**Before (R1):** Armed profile dropdown was the top control; three toggles (Overlay, Dim, Side panel) were the dominant primary surface; live stats lower; options link at bottom.

**After (R2):**
- **Top zone (status):** `Armed:` label in 11px `#9CA3AF`, then the profile name in 22px Inter Semi Bold `#F3F4F6` (e.g., `Lower Bowl · Climate Pledge`). A small chevron to the right opens an inline dropdown for 1-click swap. No "ARM" button.
- **Dropdown behavior:** Click the chevron → list of saved profiles drops down inline, current one checked. Click another → instantly armed, status text updates, dropdown closes. No confirmation step. Switching profile mid-drop is the explicit drop-day use case.
- **Middle zone (stats):** 3-up grid — `47 matches` / `2,387 seats` / `$182 median` (12px label + 18px number each). "Last updated 3s ago" beneath in 11px dim.
- **Bottom zone (quick toggles):** A single row labeled `Quick toggles` in 11px tracking. Two pill toggles: `Overlay` (default on) and `Dim` (default on). Smaller than R1; not the focus. The third R1 toggle "Side panel" is removed — collapse is automatic per frame 7.
- **Footer:** `Open options` text link `#9CA3AF`. `Diagnostic mode` text link only appears when an error is active (links to State C of frame 8).

**Why:** Drop-day muscle memory says "open popup, glance at name, close popup". Anything more is friction. Deliberate arming lives in frame 5.

**Microcopy:**
- `Armed:` / `<profile name>`
- `47 matches · 2,387 seats · $182 median`
- `Last updated 3s ago`
- `Quick toggles` / `Overlay` / `Dim`
- `Open options` / `Diagnostic mode` (conditional)
- Empty state (no profile armed yet): `No profile armed.` + CTA `Choose a profile` (opens dropdown).

## Frame 4 diff — `4. Live seat map · overlay on` (modified R2)

Three changes to the existing frame:

1. **Pulse-on-seat instead of scroll:** The `›` glyph on candidate rows now triggers a pulse animation on the seat itself — a 24px diameter glow ring expanding outward from the seat over 600ms, then fading. Show in the mock as a single seat with an outward-expanding mint ring with a soft drop-shadow. Loop the animation on the static frame by showing two ghost rings at 50% and 80% expansion. The seat itself stays in place — no scroll, no zoom.
2. **Dim tuning:** Bring dimmed-seat opacity from R1's `0.18` up to approximately `0.32` so the bowl shape reads at a glance. Matching seats remain at full saturation + glow. Verify visually that candidate clusters still "pop" — they will, because the contrast ratio of match-vs-dim is now driven by the glow + ring, not just the alpha gap.
3. **Color-blind-safe match treatment:** Add a 1.5px outer ring in `#A8FFD8` (lighter mint tint) on every matching seat, offset 2px from the seat fill. This survives deuteranopia and protanopia because the *form difference* (ring vs solid) reads even when the hues compress.
4. **Footer microcopy:** Replace R1's "Click a row to scroll the map to that seat. The extension does not select for you." with: `Click a row to highlight the seat on the map. The extension does not select for you.` — "highlight" is accurate for pulse; "scroll" overpromised.

## Frame 5 diff — `5. Pre-drop arming` (modified R2)

Add a 7th checklist item between item 6 ("Notifications muted") and the bottom action row:

7. **Profile locked to local snapshot**
   - Lock icon (mint outline when checked, `#9CA3AF` when unchecked), 16px.
   - Label: `Profile locked to local snapshot`
   - Hover/focus inline explainer (tooltip or expanding subtext): `Prevents chrome.storage.sync from overwriting your armed profile mid-drop if you edit it on another device. Recommended on drop day.`
   - Default state: **checked** on the drop-day flow (unlike some other items which default unchecked).
   - Source: `feasibility.md:140` §5.4 mitigation.

**Why default checked:** the failure mode (a phone edit silently overwriting the desktop profile 90s before a drop) is silent and catastrophic; the cost of locking (can't sync edits during the session) is negligible during a 60-second arming window.

## Design decisions and why (carried from R1, retained)

### Dark mode by default, neon mint as the match color
Drops are usually late-night. The seat map is dense. Match color `#00F68D` is high-contrast on `#0B0D11` (> 13:1) and sits at a hue Ticketmaster's palette does not use. Matches read as "ours", not "theirs".

### Side panel is "Candidates", not "Best Deals" or "Recommended Seats"
Most important microcopy decision. "Candidates" is neutral; the user remains the decider.

### Score as 0–1 bar + decimal, not stars or %
Stars feel like reviews; % feels like a "chance" claim we can't make. Bar + decimal is honest about the heuristic nature of the score.

### "Last updated Xs ago" inline with match count
The number you're about to act on and the freshness of that number age together. Don't split them across the layout.

### Pre-drop arming is a flow, not buried in options
Drop-night nerves + 90 seconds = need a single-screen ritual. Options page is for non-drop-day setup.

### Empty / first-run frame leads with what we WILL NOT do
"No automation. No clicks on your behalf." in the welcome footer — sets the mental model from second one.

### Section whitelist/blacklist as chips
Scannable, removable, visually distinct (mint outline vs amber outline). Resists typos like `1110` vs `110`.

## Anti-patterns avoided (R1 + R2 additions)

- **No "Buy this seat" button anywhere.** Click-to-pulse only.
- **No countdown timer.** TM has its own queue clock.
- **No "X people are looking at this seat" / fake social proof.**
- **No celebration animation on a match found.** Drops are stressful; confetti is condescending.
- **No "best deal" or "steal" language.**
- **No red urgency colors on the live overlay.** Red/amber reserved for genuine warnings (obstructed sections, expired pre-flight items). **R2:** the auto-disabled banner is amber, not red — the overlay breaking is not a user emergency.
- **No keyboard shortcuts that collide with TM's own.**
- **No notifications, no sounds, no badge counts.**
- **No "Pro" / "Premium" tier hints.**
- **No telemetry, no analytics, no error reporting that calls home.**
- **R2 — No alarming microcopy on kill-switch.** The banner says "Falling back to vanilla view", not "Overlay crashed". TM still works; that's the lede.
- **R2 — No "scroll-to-seat" promise.** Pulse-on-seat is honest about what we can deliver in v1 without touching TM's map state.

## Accessibility notes

- Match color (`#00F68D`) on bg `#0B0D11`: contrast > 13:1 (WCAG AAA non-text and large text).
- Text colors: primary `#F3F4F6` (15.8:1), dim `#9CA3AF` (5.7:1, AA normal text), mute `#6B7280` — restrict to 14px+ or raise to `#8A92A0` (still flagged for review).
- **R2 — Color-blind-safe match affordance:** matching seats now use color + outer ring (1.5px `#A8FFD8`, offset 2px). The form difference survives deuteranopia/protanopia. Glow alone is insufficient on its own — the ring is the load-bearing redundancy.
- **R2 — Pulse-on-seat respects `prefers-reduced-motion`:** when the user has reduced motion enabled, the pulse becomes an instant 600ms hold of a static outer ring at full opacity, then fades — no expansion. The seat is still findable; we just don't animate.
- **R2 — Auto-disabled banner uses `role="status"` + `aria-live="polite"`**, not assertive. Vanilla TM is functional; the message is informational, not interruptive.
- **R2 — Diagnostic-mode code blocks** carry `lang="html"` / `lang="json"` so screen readers don't try to pronounce attribute strings.
- All interactive elements need keyboard focus rings (Shadow DOM must preserve `:focus-visible`).
- Popup tab order (R2): armed-profile dropdown → stats group (skipped, decorative) → Overlay toggle → Dim toggle → Open options → (Diagnostic mode, if visible).
- Profile editor tab order: name → price min → price max → section whitelist → blacklist → row range → min adjacent → aisle toggle → notes → save.
- Side panel announces `"47 candidates available, list updated 3 seconds ago"` via `aria-live="polite"` when the count changes.
- Collapsed strip (frame 7) is a `<button aria-expanded="false" aria-label="Expand candidates panel, {n} matches">`.
- "Last updated Xs ago" is a `<time datetime=...>` for correct SR announcement.

## Resolved questions (R1 → R2 decisions)

Moved from "Open questions" with the decision noted. Source column points to where the decision was recorded.

### From "Open questions for PM"

| R1 question | R2 decision | Source |
|---|---|---|
| Personas of one — designing for 6-months-from-now Jonathan or today Jonathan? | Today Jonathan, plus enough onboarding (frame 6) that 6-months-later Jonathan can re-derive his setup in <10 min. Not a full doc system. | Implicit in `prd.md:18` single-user framing. |
| Which venues seed v1? | Confirmed (Seattle-area): Climate Pledge Arena, T-Mobile Park, Tacoma Dome, Lumen Field. | `round-1-synthesis.md:82` (user confirmed) |
| Is post-drop logging a v1 feature? | **No, v2.** No surface in frame 4 for it. | `prd.md:106`, `round-1-synthesis.md:85` |
| What does "drop day" mean for the kill-switch? | The kill-switch is always on, not drop-day-tagged. Auto-disable + amber banner triggers on any uncaught exception during overlay execution on a TM event page. No tagging UI needed in v1. | `prd.md:94`, `feasibility.md:131` |
| Inline obstructed-view warnings on the seat map? | **No.** Obstructed info lives in the side panel candidate detail only. No hover detection on TM nodes. | `round-1-synthesis.md:72`, `prd.md:50` |

### From "Open questions for SWE"

| R1 question | R2 decision | Source |
|---|---|---|
| Can the side panel float over TM with `position: fixed`? Z-index fight on modals? | Yes for the default pinned-right panel. On TM modal-open, collapse to a 48px strip (frame 7). SWE to confirm `body.modal-open` (or equivalent) is the detection hook. | `round-1-synthesis.md:58` |
| Is the seat map SVG or Canvas? | **Unresolved — DevTools session required before coding.** Frame 4 assumes per-seat CSS targeting; if Canvas, this becomes an SVG overlay layer. Design holds; rendering path adapts. | `feasibility.md:163`, `prd.md:132` |
| Do seat DOM nodes expose price as a number or only a tier? | **Unresolved — DevTools session required.** If tier-only, the candidates list needs tier badges instead of dollar values. Designer to spec both densities once SWE confirms. | `feasibility.md:163`, `prd.md:133` |
| Can we read TM's map state to scroll-to-seat? | **No promise in v1.** Pulse-on-seat is the v1 behavior (frame 4 R2 diff). Scroll-to-seat is v2 if SWE confirms safe access. | `round-1-synthesis.md:66`, `prd.md:67` |
| MutationObserver tick rate vs. "last updated Xs ago" honesty? | SWE owns the actual cadence number; designer uses `Xs ago` with `<time datetime>` so the value can come from real timestamps. | `prd.md:134` |
| Shadow DOM theming via CSS custom properties? | Assume yes; SWE to confirm in `feasibility.md` round 2 follow-up. Mock uses CSS custom props pattern. | `round-1-synthesis.md:108` |
| Where does the popup get "47 matches"? | Content-script → service-worker → popup via `chrome.runtime.sendMessage` with last-known stats cached in `chrome.storage.session`. Fallback if no page open: popup shows "No active TM tab" empty state. | `round-1-synthesis.md:108` |

## Surfaced in round 2 (new conflicts / gaps)

1. **Diagnostic mode entry path on a collapsed strip.** Frame 7's strip shows `!` and an amber edge when diagnostic mode is active. But the auto-disabled banner (frame 8 State B) sits at the top of the page and the strip sits on the right. If TM's modal is open AND the overlay just failed, the banner could overlap the modal. **Need from SWE:** confirm we can suppress the banner while a TM modal is open and surface the error inside the strip instead, OR confirm the banner is z-indexed below the modal (probably the better default). Not blocking R2, but should land in R3.

2. **Pulse animation conflict with TM's own hover state.** If a user hovers a seat while our pulse is animating, TM may apply its own visual change to that seat. Two animations on the same node could look broken. **Need from SWE:** can our overlay layer sit above TM's seat node such that our pulse renders over TM's hover, or do we need to suppress pulse-on-seat while a seat is hovered? Designer-preferred behavior: let our pulse win on click, drop it on TM hover.

3. **"Profile locked" lock state is invisible after arming.** Once the user leaves frame 5 and is on the live seat map, there's no surface that shows "profile locked" status. If the user expects to be able to swap profiles via the popup mid-drop (the C3 hybrid resolution), but the profile is locked, the swap may behave inconsistently. **Recommendation:** the popup's profile-switch dropdown should show locked profiles with a small lock glyph, and unlocking requires an explicit click. Not yet mocked in frame 1 R2 diff — flag for R3.
