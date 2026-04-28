# vFairs Handover Notes (from `main.html`)

## Scope
This handover focuses on session rendering and runtime behavior in `main.html`, with emphasis on:

- Stability and race-condition protections.
- Load strategy and perceived speed.
- Refresh/update behavior.
- Join button visibility logic and guardrails.
- Session pagination.
- Session ended handling.
- Potential integration conflicts and how the code avoids (or may still hit) them.

---

## 1) Stability Safeguards

### 1.1 Duplicate boot prevention
- The widget exits early if the root is missing.
- It also exits if `data-hsrtx26np20260316-initialised="true"` is already set, preventing duplicate event binding and duplicate intervals in repeat injections/re-renders.

**Why this matters for vFairs:** prevents “double init” bugs (e.g., duplicate click handlers, duplicate countdown timers, repeated fetches).

### 1.2 Stale async response protection (load token pattern)
- The state keeps `latestLoadToken`.
- Area activation increments the token and passes it through async loaders.
- Multiple async paths check token equality before mutating DOM/state.

**Why this matters:** if users switch research areas quickly, older responses are dropped instead of overwriting current view.

### 1.3 Defensive fallbacks for missing platform APIs
- Network layer supports:
  - `fetch` first.
  - jQuery fallback for GET/POST when available.
  - soft-fail (`''` / `null`) + warnings if unavailable.
- `requestIdleCallback` has a `setTimeout` fallback.
- `closest`/`matches` polyfill-style wrappers are used.

**Why this matters:** reduces breakage across varied vFairs runtime shells.

### 1.4 Safe parsing/normalization
- Broad key aliases are accepted for webinar objects (id, title, time, ended flags, etc.).
- JSON parsing is wrapped with safe parse helper.
- HTML/text sanitization helper paths are used before rendering text content/attributes.

**Why this matters:** resilient to inconsistent API payload shapes.

---

## 2) Platform Load Speed / Performance Characteristics

### 2.1 Data source strategy
- Primary session data path is bulk agenda endpoint: `POST /en/getWebinarAgendaWithDates/` with `app_id`.
- Fallback path calls `GET /service-api/webinar-detail/?id=...` in batches (`batchSize = 8`) if bulk agenda is unavailable/unusable.

**Performance implication:** bulk path is fast and minimizes request fan-out; fallback is slower but controlled via batching.

### 2.2 Caching behavior
- Agenda index is memoized (`agendaPromise`, `agendaDetailsMap`, `agendaDetailsList`) to avoid refetching on each area switch.
- Keyword HTML lookups are cached (`keywordHtmlCache`) and de-duplicated with in-flight promise cache (`keywordHtmlPromises`).

**Performance implication:** repeated area/tab switches should feel faster after first load.

### 2.3 Deferred lower-priority work
- Documents/videos (keyword tables) are queued with `requestIdleCallback` or delayed timeout fallback.
- Sessions/posters render first; secondary content hydrates later.

**Performance implication:** faster time-to-interactive for primary sessions panel.

### 2.4 Render cost profile
- Sessions are rendered as HTML strings and injected once for page-level output.
- Pagination defaults to `10` sessions/page, reducing DOM size per render.
- Countdown tick scans visible session cards every second.

**Performance implication:** good default for medium lists; very large lists may still incur 1s tick overhead.

---

## 3) Refresh Logic

### 3.1 Manual refresh control
- Pagination UI includes a refresh button.
- Clicking refresh performs `window.location.reload()` (full page reload).

**Behavioral note:** this is intentionally hard refresh logic, not in-widget re-fetch only.

### 3.2 Live countdown refresh loop
- A single interval (`setInterval(..., 1000)`) updates countdowns/join visibility.
- Existing interval is cleared before starting a new one.

**Stability note:** prevents timer stacking on reactivation.

### 3.3 Join-link sync refresh
- After page changes (pagination), it explicitly:
  - Re-renders current page.
  - Re-syncs join links from original vFairs play buttons.
  - Re-ticks countdowns immediately.

**Result:** page navigation does not wait for next timer cycle to correct join visibility.

---

## 4) Pagination Logic (Sessions)

### 4.1 Core rules
- State fields:
  - `sessionsPerPage` (default `10`)
  - `sessionsCurrentPage`
  - `sessionsForActiveArea`
- Total pages = `ceil(totalSessions / perPage)` with minimum 1.
- Current page is clamped into valid bounds.
- `Back`/`Next` are disabled at bounds.

### 4.2 UI behavior
- Pagination controls render both above and below the list when `totalPages > 1`.
- Status text shows range: `Showing X–Y of Z`.

### 4.3 Event behavior
- Click delegation on root handles next/back/refresh.
- On next/back: updates page index, re-renders, syncs join links, ticks countdowns.

### 4.4 Edge handling
- If no sessions in area: empty state is rendered (no pagination controls).

---

## 5) Join Button Logic

### 5.1 Data source and inheritance from native vFairs buttons
- The custom join link tries to inherit `data-*` attributes from original external play buttons (`.play-webinar`, `.btn-webinar-play`, matching `data-webinar-id`).
- If original button state is disabled/hidden (via attributes, classes, inline style, computed style), availability is treated as blocked.

**Why this matters:** custom UI honors native vFairs gating/availability signals instead of bypassing them.

### 5.2 Join eligibility for normal sessions
A normal session join link is shown only when all are true:
1. Join target exists (`data-webinar-id` or `data-url`).
2. Source availability is not false.
3. Join window is open (`secondsLeft <= joinBeforeSeconds`).
4. Session is not ended.

### 5.3 Secret Cinema exceptions
- Secret Cinema sessions ignore countdown window and can show Play whenever source is available and target exists.
- Additional polish forces label/ARIA to “Play / Play Secret Cinema” and hides countdown visuals.

### 5.4 Force join override (`forceJoin` URL param)
- Query/hash/parent URL can set `forceJoin`.
- Truthy values force join mode, making join controls immediately eligible when target exists.
- In force mode countdown/status are suppressed and cards are marked join-ready.

### 5.5 Click-time guard (anti-race safety)
- Capturing click handler re-validates if join can open at click time.
- If invalid, it prevents default + immediate propagation and hides the link.

**Why this matters:** protects against stale DOM state or timing drifts between render and click.

---

## 6) “Has Ended” Logic

### 6.1 Ended determination hierarchy
For non–Secret Cinema sessions:
- If explicit ended field exists (`is_ended`, `webinarEnded`, etc.) and parses to boolean, it wins.
- Otherwise, ended state falls back to computed `Date.now() >= endTimestamp` (start + duration).

### 6.2 Runtime updates
- Countdown tick recalculates ended state each second.
- Card attributes/classes are updated (`data-...-session-ended`, `.is-ended`).
- “On demand content coming soon” note is toggled for ended sessions.
- Join link is forcibly hidden when ended.

### 6.3 Ordering impact
- If ended state changes, sessions are re-ordered so ended sessions move to bottom.

---

## 7) Sorting / Ordering Rules

- Initial normal-session sort:
  1. Non-ended first.
  2. Earlier `sortTime` first.
  3. Title lexical fallback.
- Secret Cinema sorted separately by time then title.
- During live ticks, rendered lists can be re-ordered when ended status flips.

---

## 8) Possible Conflicts (and Existing Mitigations)

### 8.1 Conflict: double initialization in CMS/widget reinjection
- **Mitigation:** root `data-...-initialised` gate prevents second init.

### 8.2 Conflict: stale async writes after fast area switching
- **Mitigation:** `latestLoadToken` checks abort stale DOM updates.

### 8.3 Conflict: custom join link drifting from native button attributes
- **Mitigation:** repeated sync from original play button data attributes.

### 8.4 Conflict: showing join button while native source is disabled
- **Mitigation:** own-state + computed-style disabled/hidden detection.

### 8.5 Conflict: forceJoin misuse
- **Reality:** `forceJoin` can intentionally bypass time-window gating.
- **Operational note:** this is powerful for QA/emergency visibility, but should be treated as controlled behavior.

### 8.6 Conflict: hash/search mutation interactions around poster hall
- Poster-group opener manipulates hash and calls clicks via polling.
- Could conflict with other hash-based routers/scripts if present.

### 8.7 Conflict: timer overhead on very large DOMs
- 1s tick walks visible cards; still efficient for paginated sessions, but secret cinema plus other panels may add cost.

---

## 9) Operational Notes for vFairs Team

### 9.1 Key data dependencies to keep stable
- `app_id` availability for bulk agenda endpoint.
- Valid webinar IDs discoverable via selector or explicit root attribute.
- Presence/shape of original vFairs play button `data-*` attributes.
- Accurate duration/start fields when relying on computed ended state.

### 9.2 Quick troubleshooting checklist
1. No sessions showing:
   - verify webinar IDs, `app_id`, and bulk endpoint response.
2. Join not appearing:
   - check source button visibility/disabled state and join-before window.
3. Session marked ended too early/late:
   - verify timezone/date parsing and duration metadata.
4. Secret Cinema behavior odd:
   - confirm correct group ID and keyword detection.
5. Area switching glitch:
   - inspect stale token flow and ensure single init occurred.

---

## 10) Suggested Low-Risk Hardening (optional future)

1. Add lightweight telemetry hooks (counts + timings) around:
   - bulk agenda fetch latency,
   - fallback request counts,
   - render duration,
   - countdown tick duration.
2. Make refresh button optionally “soft refresh” (in-widget data reload) before full page reload.
3. Add max card count guard for countdown tick (or `requestAnimationFrame` chunking) for extreme scale cases.
4. Consider explicit timezone normalization strategy for date parsing edge cases.

---

## 11) Pointers for Maintainers

- Pagination: `hsrtx26np20260316RenderSessionsPage`, `hsrtx26np20260316BindSessionsPagination`
- Join logic: `hsrtx26np20260316RenderSessionCard`, `hsrtx26np20260316IsRenderedJoinLinkOpen`, `hsrtx26np20260316BindJoinLinkGuard`, sync helpers.
- Ended logic: `hsrtx26np20260316BuildSessionModel`, `hsrtx26np20260316IsSessionEnded`, `hsrtx26np20260316TickCountdowns`, `hsrtx26np20260316MoveEndedSessionsToBottom`.
- Load/race protection: `hsrtx26np20260316ActivateArea`, `hsrtx26np20260316LoadSessionsAndPostersForArea`, token checks.

