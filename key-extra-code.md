# Research Hub Logic Guide

## Pagination

- Sessions are paginated only within the **Sessions** tab.

### State variables

- `sessionsPerPage: 10`
- `sessionsCurrentPage`
- `sessionsForActiveArea`

### Pagination rendering

- `hsrtx26np20260316RenderSessionsPage(areaConfig)`

### Output includes

- Top pagination controls  
- Session list  
- Bottom pagination controls  

### Controls

- Refresh → reloads the page  
- Back → previous page  
- Next → next page  

### Event binding

- `hsrtx26np20260316BindSessionsPagination()`

### On page change

- Re-render sessions  
- Re-sync join buttons:
  - `hsrtx26np20260316SyncRenderedJoinLinksWithOriginalButtons()`  
- Re-run countdown logic:
  - `hsrtx26np20260316TickCountdowns()`  

---

## Join button logic on sessions

### Core behaviour

- Join buttons are rendered as:

  - `.hsrtx26np20260316-join-link`
  - `.track-btn-click`
  - `.primary-btn`
  - `.btn-webinar-play`
  - `.play-webinar`

- The system mirrors vFairs’ native webinar player behaviour.

### How it works

- The code locates original vFairs play buttons outside the component:

  - `.play-webinar[data-webinar-id]`
  - `.btn-webinar-play[data-webinar-id]`
  - `a[data-webinar-id][data-url]`

- It copies all relevant `data-*` attributes onto the custom Join button:

  - `hsrtx26np20260316ApplyOriginalPlayerDataToJoinLink(joinLinkNode)`

### Key data attributes used

- `data-webinar-id`
- `data-url`
- `data-title`
- `data-webinar-type`
- `data-duration`
- `data-tr_duration`
- `data-chatroom`
- `data-chatlink-text`
- `data-checkpoint_duration`
- `data-checkpoint_message`
- `data-checkpoint_frequency`
- `data-linked_webinar`

### When Join button is shown

The Join button is only visible when:

- The session is within its join window  
- The original vFairs play button is available  
- A valid `data-webinar-id` or `data-url` exists  
- The session has not ended  

### Join window logic

Uses:

- Session start time  
- `join_before_time`  
- `joinBeforeTime`  
- `join_before_seconds`  
- `joinBeforeSeconds`  

Evaluated via:

- `hsrtx26np20260316IsJoinWindowOpen(seconds, joinBeforeSeconds)`

---

## Secret Cinema tab logic

### Identification

Secret Cinema is identified via:

- Root config:
  - `data-hsrtx26np20260316-secret-cinema-group-id="17151"`
- Session group IDs  
- Fallback keyword match: “secret cinema”  

### Behaviour

Secret Cinema sessions:

- Are removed from the main Sessions list  
- Only appear in the Secret Cinema tab  

### UI differences

- No countdown displayed  
- No session date/time displayed  
- No “ended” logic applied  

Button label is:

- “Play” instead of “Join”  

### Enforcement

Applied via:

- `hsrtx26np20260316PolishSecretCinemaPanel()`

Uses a `MutationObserver` to:

- Hide countdown elements  
- Force button text to “Play”  
- Maintain behaviour after DOM updates  

---

## Has ended / session end logic

### How end time is calculated

Start time:

- `hsrtx26np20260316ParseStartDate(detail)`

Duration:

- `hsrtx26np20260316BuildDurationSeconds(detail)`

End time:

- `startTimestamp + durationSeconds * 1000`

---

### Explicit ended fields

If provided, these override calculated logic:

- `is_ended`
- `isEnded`
- `session_ended`
- `sessionEnded`
- `webinar_ended`
- `webinarEnded`
- `ended`

Handled via:

- `hsrtx26np20260316BuildExplicitEndedState(detail)`

---

### Automatic end detection

If no explicit value:

- Session is considered ended when:

  - `Date.now() >= endTimestamp`

Handled via:

- `hsrtx26np20260316IsSessionEnded(sessionCardNode)`

---

### What happens when a session ends

- Join button is hidden  
- Countdown is hidden  

Session card gets:

- `.is-ended`
- `data-hsrtx26np20260316-session-ended="true"`

UI message shown:

- “On demand content coming soon”

---

### Reordering logic

- Ended sessions are moved to the bottom of the list  

Handled by:

- `hsrtx26np20260316MoveEndedSessionsToBottom()`

---

### Live updates

End state is continuously recalculated via:

- `hsrtx26np20260316TickCountdowns()`

This runs every second using:

- `setInterval`

---

### Special case

Secret Cinema sessions:

- Ignore all “ended” logic  
- Always remain playable if a valid video source exists  
