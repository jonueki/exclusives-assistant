# Round 1 Synthesis — Conflicts, Alignments, Decisions

Inputs:
- `docs/seat-overlay-architecture.md` (v0, SWE pre-research)
- `docs/prd.md` (PM round 1)
- `docs/feasibility.md` (SWE round 1)
- `docs/ux-mockup.md` + Figma (Designer round 1)

This doc identifies where the three agents converged, where they diverged, and what each needs to address in round 2.

---

## Alignments (no further discussion needed)

| Topic | Status |
|---|---|
| Strategy A (DOM observation) as primary | All three agree. |
| Strategy B (fetch interception) — drop entirely | SWE escalated from "flagged off" to "do not implement." PM and Designer agree (PM has it in Won't-Have). |
| "Candidates" framing, no buy button, no automation | Designer chose copy; PM non-goals enforce it; SWE confirms no auto-anything. |
| Closed Shadow DOM with scoped class prefix (`tmx-*`) | All three. SWE adds: random `tmx-{buildHash}` prefix per build to make selector-based detection harder. |
| Dark mode default | Designer chose; PM/SWE no objection. |
| Single user, no accounts, no backend, no CWS distribution in v1 | All three explicit. |
| Score shown as 0–1 + bar (not stars, not %) | PM asked; Designer answered with rationale; resolved. |
| Map is clickable end-to-end by the user; we never intercept clicks | All three explicit. |
| Bundled venue intel JSON for ~5–10 venues | All three want; venue list still pending from user. |

---

## Conflicts to resolve in round 2

### C1. Strategy C exists — PM didn't know about it
**SWE** added Strategy C (read TM's SPA state via React fibers from a MAIN-world content script) as a fallback when DOM is too thin. **PM**'s PRD was written before this existed and lists only Strategy A; Strategy B is in Won't-Have but Strategy C is unmentioned. **Designer**'s mockups assume exact prices are readable, which Strategy A alone may not deliver.

**Decision needed:** Strategy C is in v1 (fallback for missing fields). PM PRD must add it. Designer must accept that prices may be tier-bucketed if Strategy C also fails.

### C2. Kill-switch priority mismatch
**SWE** says kill-switch is **P0 v1** ("must-have, not v3 polish") because PerimeterX may silently challenge mid-drop. **PM** has it as **Should Have**. **Designer** has the surface ready (banner/toast).

**Decision needed:** Promote kill-switch to Must Have. PRD must move it. Designer's `auto-disable banner` becomes a v1 deliverable.

### C3. Popup as "arm" surface vs "status" surface
**PM** says popup is where you "arm" a profile on drop day. **Designer** demoted popup to status display and moved arming into a dedicated screen (frame 5 "Pre-drop arming"). **PM** thinks the popup is the muscle-memory drop-day surface; **Designer** thinks the popup is too small and emotional for that.

**Decision needed:** Arm where? My recommendation: **both**. Arming-as-flow lives in frame 5 for the pre-drop ritual (the deliberate, 60-second setup). Popup shows "armed: <profile name>" + a 1-click "switch profile" dropdown for last-second swaps on drop day. Round 2: both reconcile to this hybrid.

### C4. Drop-day hotfix path → does v1 need a backend?
**SWE** says "if TM changes DOM mid-drop, Chrome's auto-update cycle is too slow. Either ship remote selector overrides (small extension-owned server) OR ship `options.json`-driven selector overrides + diagnostic mode the user can hot-patch in <5min." **PM** non-goals explicitly forbid backend in v1.

**Decision needed:** Side with PM's non-goal — **no backend in v1**. Ship local-only diagnostic mode + a selector override file the user can edit. If during the first real drop this proves insufficient, revisit. SWE's Mitigation B is the v1 path; Mitigation A defers to v2.

### C5. Dim-by-default → confirmed, with nuance
**PM** wants dim ON by default. **Designer** kept it ON but tuned the dim level so the bowl shape still reads (spatial context preserved). **SWE** has no opinion on visual treatment.

**Decision needed:** No conflict, just nuance. PRD should be updated to "dim ON by default at a level that preserves spatial context."

### C6. Side panel layout — pinned right vs floating vs vertical strip
**Designer** chose right-pinned full-height panel. **SWE** worries about z-index fights with TM's checkout modals and asked for a fallback. **PM** didn't specify.

**Decision needed:** Default pinned-right; collapse to a 48px vertical strip when TM opens a modal (detected by observing a `.modal-open` body class or equivalent). Round 2: SWE confirms feasibility; Designer mocks the collapsed state.

### C7. Pre-flight checklist — Designer added, PM didn't have it
**Designer** added a 6-item pre-flight checklist to frame 5 (login, card, captcha, DND, etc.). **PM** has nothing equivalent. **SWE** has the "drop-day lock profile to local snapshot" mitigation that should ride alongside it.

**Decision needed:** Yes, ship the checklist. Add to PRD as Should Have. Designer adds "lock profile" as a checklist item per SWE's sync-race concern.

### C8. Scroll-to-seat from panel — designer hedged honestly
**Designer** showed a `›` indicator but said "until we know whether we can read TM's map state, the mock UI shows a `›` rather than promising scroll-to-seat." **SWE** also flagged this as unknown. **PM** asks the same question.

**Decision needed:** v1 ships **pulse-on-map** as the visual fallback (no scroll required). Scroll-to-seat is a v2 enhancement if SWE confirms it's reachable safely.

### C9. Obstructed-view inline tooltips
**Designer** asked whether obstructed warnings appear inline on the seat map. **PM** didn't address. **SWE** would say "passive only, no hover detection that triggers TM event handlers."

**Decision needed:** v1: obstructed-view info lives in the side panel candidate detail only. No inline tooltips (would require either hover detection or our own overlay layer with z-order on top of TM seats — both add complexity and detection surface for marginal value).

---

## Gaps (raised by 2+ but answered by 0)

| Gap | Raised by | Plan |
|---|---|---|
| Venue list for v1 intel | PM, SWE, Designer | Resolved by user: Climate Pledge Arena, T-Mobile Park, Tacoma Dome, Lumen Field (Seattle-area). |
| Risk appetite (account suspension tolerance) | SWE explicitly, implicit in PM | Default: tolerate CAPTCHA challenge, do NOT tolerate suspension on a target drop. Kill-switch + soft-launch on a low-stakes event before any target drop. |
| Drop-day hotfix SLA | SWE explicitly | Default: no SLA (single user, no on-call). Diagnostic mode + local override is the hotfix path. |
| Post-drop logging in v1 | PM (Won't), Designer (asked) | Confirmed v2. No surface in v1. |

---

## What round 2 needs to produce

Each agent gets the other two's docs to read, plus the conflicts and decisions above. Updates:

### PM v2 — `docs/prd.md` update
- Add Strategy C to architecture references (C1).
- Move kill-switch from Should → Must (C2).
- Update popup spec to hybrid arm/status (C3).
- Clarify no-backend means no remote selector overrides in v1; ship local diagnostic + override (C4).
- Update dim default to "ON at spatial-context-preserving level" (C5).
- Add pre-flight checklist as Should Have (C7), include "lock profile" item.
- Update "scroll-to-seat" to "pulse-on-map" in v1 (C8).
- Explicitly exclude inline obstructed-view tooltips from v1 (C9).
- Use default venue list as placeholder.

### SWE v2 — `docs/feasibility.md` update + new `docs/v1-build-plan.md`
- Confirm `position: fixed` panel feasibility + propose collapse-to-strip on modal (C6).
- Spec the local diagnostic-mode + selector override file format (C4) — drop the remote-overrides idea.
- Confirm message flow for popup "47 matches" stat (Designer Q7).
- Confirm Shadow DOM theming via CSS custom props doesn't leak (Designer Q6).
- Address obstructed-view inline tooltip technical viability (C9) and confirm "passive only" rules it out.
- Write a tight v1 build plan: file structure, manifest, content-script split (isolated + MAIN), replay harness setup.

### Designer v2 — `docs/ux-mockup.md` update + Figma additions
- Add: collapsed side-panel state (48px strip) for modal-open mode (C6).
- Add: kill-switch error state UI (banner + toast spec) — now P0 (C2).
- Update: popup mock to show "armed: <name>" + 1-click switch (C3).
- Add: "lock profile" item to pre-flight checklist (C7).
- Update: panel scroll-to-seat → pulse-on-seat in v1 (C8).
- Spec: color-blind-safe match treatment (color + pulse or border), already noted in accessibility section but ensure mock reflects it.

---

## What I'm asking the user to confirm (non-blocking — round 2 will proceed with defaults)

1. **Venue list** for initial intel sheets. ✅ Confirmed by user: Climate Pledge Arena, T-Mobile Park, Tacoma Dome, Lumen Field.
2. **Risk floor:** is "tolerate CAPTCHA, not suspension" right? Translates to: kill-switch P0, soft-launch on a low-stakes event first.
3. **Distribution:** confirm unpacked extension for personal use only — no Chrome Web Store path planned for v1+.
4. **Drop-day hotfix:** confirm no remote-update backend in v1; local diagnostic + selector override file is the only path.
