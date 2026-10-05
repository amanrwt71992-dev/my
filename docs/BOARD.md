> This file describes `live.html` in this repository (the AgentMesh board). It lives at the repo root, next to this `docs/` folder.
> The data contract and endpoints below are accurate. Ignore mentions of `server.py`, ports 5170/5199, `--e2e` and the tests folder:
> those are not part of this repository.

# HANDOFF — AgentMesh board (`live.html`)

Read this before touching anything. It says what exists, how it is wired, what is
deliberately unfinished, and how to verify a change. Everything here is verified
against the file that ships in this repo.

---

## 1. What this project is

`live.html` is a single-file mirror of the AgentMesh daemon: agents (IDEs) claim
tasks, work, report, block, go to review; the manager (a human) assigns, answers,
approves or sends back, and moves manual (chatbot) tasks by hand.

There is no build step and no dependency at runtime. The page is the whole app.

```
live.html          the deliverable (HTML + inline <style> + inline <script>)
server.py          mock daemon: serves live.html, WebSocket pushes, all REST endpoints
tests/             headless regression suites (jsdom) — see §7
package.json       test scripts + jsdom devDependency
README.md          short human intro
HANDOFF.md         this file
```

Run it:

```bash
python3 server.py                 # http://localhost:5170/  (board + daemon + live updates)
PORT=8080 python3 server.py
```

Open the file with `?demo=1` for offline sample data, or press **“Open the demo
board”** on the welcome card when no daemon is reachable. Demo mode never opens a
WebSocket and applies every action in memory.

---

## 2. Non-negotiable rules (do not regress these)

Agents and chatbots write most of the strings the UI shows, so all of it is hostile
input:

- Build DOM with `createElement` + `textContent` only. **No `innerHTML`,
  `outerHTML`, `insertAdjacentHTML`, `document.write`, `eval`, `new Function`.**
  The only helper that touches the DOM is `el(tag, props, ...children)` (§4).
- Only same-origin traffic: relative URLs and `ws(s)://location.host`. No CDN, no
  webfont, no external image. Avatars uploaded by the user become `data:` URIs and
  never leave the browser.
- `encodeURIComponent` every id that goes into a URL.
- Every `localStorage` access is inside `try/catch`; the page must work with
  storage disabled.
- Never mutate state optimistically. After a POST, wait for the server push
  (`MESH_UPDATE`). The one exception is demo mode, which emulates the push itself.
- Never rebuild the task modal or the board while the user is typing in it
  (`typingIn()` + `T.forceNext` guard, §5).
- Respect `prefers-reduced-motion` (a global media query already disables the
  animations).

---

## 3. Data contract

One WebSocket at `ws(s)://location.host` (no path). The daemon sends, on connect
and after every change:

```json
{ "type": "MESH_INIT" | "MESH_UPDATE",
  "data": { "tasks": [], "locks": [], "events": [], "agents": [], "launchers": [], "workspacePath": "" } }
```

`applyState(data)` replaces local state wholesale, then `render()`.
Reconnect uses exponential backoff 1s → 8s and drives the connection pill.

Shapes (unchanged from the original brief, plus three additions this revision uses):

- `Task` — `id, featureTitle, title, description, brief, assignee, assignedIDE,
  targetRole, status (backlog|claimed|in_progress|blocked|review|done|cancelled),
  allowedFiles[], forbiddenFiles[], acceptance[], workspacePath, tokensSpent,
  changesSummary?, diffPreview?, filesModified[], violations[], blockedQuestion?,
  managerAnswer?, reviewFeedback?, claimedAt?, deliverable?, progressLog[{at,
  status, summary}] (oldest first), updatedAt`.
- `Agent` — `id, displayName?, ide, role (MANAGER|WORKER), specialization,
  lastSeen, waitingUntil?, manual?, launch` where
  `launch = null | { label, mode: "open-app"|"command", running: boolean }`.
  `launch != null` means the board may start it.
- `launchers` — `[{ id, label, mode, running }]`, agents that can be started and
  may not have joined yet (they are not in `agents` until they do).
- `Lock` — `filePath, lockedByIDE, taskId, acquiredAt`.
- `Event` — `id, timestamp` (already formatted, display as-is),
  `ideName, action, message, badgeType (info|success|warning|purple)`.

Activity words: `lastSeen` within 90s = *active*, under 10min = *quiet*, else
*silent*. `waitingUntil` in the future = idle and will pick up work by itself.

### Endpoints the page calls (exact)

| Purpose | Request |
|---|---|
| Create task | `POST /api/mesh/task` `{title, brief, assignee?, allowedFiles, forbiddenFiles, acceptance, workspacePath}` (the three lists are strings, one item per line) |
| Edit task | `POST /api/mesh/task/:id/edit` same fields, all optional. `assignee` only while `backlog`; refused in `review`/`done`/`cancelled` |
| Approve | `POST /api/mesh/task/:id/review` `{approved:true}` |
| Send back | `POST /api/mesh/task/:id/review` `{approved:false, feedback}` |
| Cancel | `POST /api/mesh/task/:id/cancel` `{reason?}` |
| Answer blocked | `POST /api/mesh/answer` `{taskId, answer}` |
| Add agent | `POST /api/mesh/agent` `{id, label}` (id: letters, digits, `-`, `_`) |
| Rename agent | `POST /api/mesh/agent/:id/rename` `{name}` (empty resets to the id) |
| Run agent | `POST /api/mesh/agent/:id/launch` `{}` → `{success, message}`; show `message` as a toast |
| Manual: request text | `POST /api/mesh/task/:id/request` `{}` → `{success, text}`; copy `text` to the clipboard |
| Manual: paste response | `POST /api/mesh/task/:id/response` `{text}` → task goes to `review` |
| Manual: move by hand | `POST /api/mesh/task/:id/move` `{to:"backlog"\|"in_progress"\|"review"\|"done"\|"cancelled"}`; 400 if the task belongs to a real agent |
| Release lock | `POST /api/mesh/release-lock` `{filePath}` |

Every call goes through `api(path, body)`: `fetch` with an `XMLHttpRequest`
fallback, JSON body, errors surfaced as a toast from `readResponse()`. HTTP 400
`{error}` or `{message}` (e.g. the launch hint that the start prompt was copied)
are both handled.

Start-prompt text for a real worker with id `X` (exact, one line):

```
Join AgentMesh as WORKER with agent_id X. Call mesh_register_agent, then call mesh_wait_for_task repeatedly until a task arrives, and follow its brief exactly. Never commit, push or deploy unless the brief says so. Do not run any dispatch script.
```

---

## 4. How `live.html` is organised

Numbered sections in the `<script>`, in order; the CSS is grouped the same way
(`/* top bar */`, `/* layout */`, `/* board */`, `/* activity */`, `/* dialogs */`,
`/* menus, toasts */`, `/* responsive */`).

| Section | What lives there |
|---|---|
| 0 | `DEMO` flag from `?demo=1` |
| 1 | helpers: `el`, `add`, `replace`, `setText`, `$`, time (`durationText`, `agoText`, `agoNode`, `timerNode`, `refreshTimes`), identity (`hueOf`, `initialsOf`, `avatarNode`), `toast`, overlays (`openDialog`, `uiConfirm`, `uiPrompt`, focus trap) |
| 2 | state `S` (server) and `UI` (view), localStorage `agentmesh.ui.v1` + `agentmesh.prefs.v1`, lookup helpers (`agentById`, `workers`, `taskById`, `liveness`, `isWaiting`, `currentTaskOf`, `statusSince`, `taskWorkload`, `workloadBar`) |
| 3 | `api()` + WebSocket (`connect`, `applyState`, `push`) |
| 4 | filtering: `COLS`, `STATUS_LABEL`, `matchesFilter`, `visibleTasks`, `laneKey` |
| 5 | render: counts, “Needs you”, agent filter, rail (team cards, launchers, add-agent forms, locks), board (columns, swimlanes, tiles), activity |
| 6 | task modal (`openTask`, `updateTaskModal`, `headContent`, `tabContent`, panels, `barContent`, `doCancel`, `copyText`, `startPromptFor`) |
| 7 | new-task modal |
| 8 | agent popup |
| 9 | move menu (`movesFor`, `openMenu`, `openMoveMenu`) + `moveTask`, `sendMove`, `reassign`, `approveTask`, `runAgent` |
| 9b | **settings**: `openSettings` (Team & avatars / Appearance / About & storage), `teamSettings`, `agentRow` (avatar picker), `appearanceSettings`, `aboutSettings` |
| 10 | drag & drop: `tileDraggable`, `dropAction`, `startDrag`/`endDrag`, `markZones`, the cancel strip, `initStrip` |
| 11 | keyboard shortcuts (`N`, `G`, `/`, `?`, `Esc`) |
| 12 | demo mode: `demoState()` (+ `demoAvatars`), `demoApi()` (every endpoint emulated) |
| 13 | `enterDemo`, theme/density appliers, `wire()`, boot sequence |

Important helper contracts:

- `el(tag, props, ...children)` — `class`, `text`, `dataset`, `style` (object),
  `on*` handlers, `value` (textareas get `.value`, inputs get the attribute), any
  other key becomes an attribute; `false`/`null`/`undefined` are skipped so you can
  write `cond ? el(...) : null`.
- `openDialog({title, head, tabs, body, foot, cls, wide, label, onClose})` returns
  an overlay handle `{el, dialog, head, body, foot, close}`; overlays stack, Esc
  closes the top one, focus is trapped and restored. `uiConfirm`/`uiPrompt` return
  promises and must be used instead of `window.confirm`/`prompt`.
- `timerNode(task, {staleAt, class})` renders `⏱ 12m` with `data-since` so
  `refreshTimes()` (every 5s) keeps it live; it turns amber past `staleAt`.
  `statusSince(task)` is when the task entered its current status (walks
  `progressLog`, falls back to `claimedAt`/`updatedAt`).
- `taskWorkload(agentId)` → `{counts, total, open}`; `workloadBar(agentId)` draws
  the coloured mini-bar used on team cards, swimlane headers and settings rows.
- `avatarNode(agentId, cls)` — per-agent avatar: uploaded image, else emoji, else
  initials, on a colour from `AV[id].color` or derived from the id hash. Extra CSS
  sizes: `sm`, default, `lg`, `xl`.
- Animations are opt-in per element so re-renders stay calm: `tile()` adds
  `.tile-enter` only when the id was not on the previous paint and `.tile-flash`
  when its status changed (`seen` map at the end of `renderBoard`); the activity
  feed does the same with `.event-enter`.

---

## 5. Interaction rules (the parts that are easy to break)

**Drag & drop** — natives HTML5 events, one global `dragover`/`drop` pair.
`dropAction(task, zone, agentId)` decides everything, and every zone is a
`[data-zone]` element:

| Task owner | Zone (column/lane/card) | Result |
|---|---|---|
| real agent, status `review` | Done | approve (with confirm) |
| real agent, status `review` | Working | open the modal with the feedback box |
| real agent, `backlog` | agent card / swimlane | reassign via edit |
| real agent, anything live | cancel strip | cancel (confirm) |
| anything else | anything | refused, toast explains why |
| manual agent | Waiting/Working/In review | `/move` |
| manual agent | Done / cancel strip | confirm, then `/move` |
| manual agent | Needs answer | refused (“only an agent can ask”) |
| any | `done`/`cancelled` task | not draggable |

`done`/`cancelled` tiles are `draggable="false"`; everything else drags so the
refusal can explain itself. Valid zones get `.drop-valid`, invalid `.drop-invalid`;
the strip is injected in the markup (`#cancelStrip`) and only visible while
`body.is-dragging`. The same move set is mirrored for keyboards in the tile's `⋯`
menu (`movesFor`), which is the accessibility equivalent — keep them in sync.

**Typing is never interrupted.** `typingIn(node)` decides whether a push may
replace the task-modal panel (the guard also checks the focused element is inside
*that* dialog, so a settings dialog does not freeze the board). Submitting from the
modal sets `T.forceNext = true` so your own action always wins over the guard.

**Settings / team management.** `T.settingsTab` and `T.avatarPickerFor` keep the
settings dialog and an open avatar picker alive across repaints. The avatar editor
writes to `AV` and `localStorage` only — the daemon never sees avatars. Bulk add
posts `/api/mesh/agent` once per line and reports how many were accepted.

**Rename.** Inline on the team card (Enter saves, Esc cancels, empty resets) and in
settings. Display names are used *everywhere* an agent is shown (tiles, rail,
popup, menus, needs strip); the stable id is shown in small mono text when a
display name exists.

---

## 6. What this revision changed (already done — do not undo)

1. **Avatars** — colour + initials by default; emoji or uploaded image per agent
   (max 300 KB, stored as a data URI in `agentmesh.prefs.v1`). Shown on tiles, team
   cards, agent popup (xl), swimlane headers, settings rows, avatar filter chips.
2. **Settings dialog** (`⚙` in the top bar, or `G`) with three sections:
   *Team & avatars* (bulk add, per-agent avatar/nickname/workload), *Appearance*
   (theme, Comfortable/Compact density, cancelled lane, group by), *About &
   storage* (the exact endpoints and storage keys, plus “forget preferences”).
3. **Team management in the rail** — avatar filter chips (`All` + one per worker)
   that filter the board, workload mini-bars with `N open · M done`, live timer on
   the current task, launcher cards for agents that have not joined, Run / Copy
   start prompt / ✎ Rename buttons on every worker card.
4. **Time everywhere** — `⏱` time in the current status on every tile (amber after
   30 min), “last update” in the footer, a time block in the task popup (in status
   for / since / claimed / last update), live timers in team cards, popup and
   swimlanes, and an `idle > 30m` counter in the top-bar stats.
5. **Interaction & motion** — per-status accent colours on tiles, columns, badges,
   timers and workload bars; slide-in cancel strip; hover lift; `.tile-enter` /
   `.tile-flash` / `.event-enter` animations that only fire for genuinely new or
   changed items; pop-in dialogs and toasts; compact density mode.
6. **Robustness fixes** — textarea `value` (was set as an attribute and ignored, so
   the Edit tab looked empty), `dropAction` zone names (`col:working` was compared
   against `working`), demo cancel/move/launch now re-render through `push()`,
   timers keep their glyph when refreshed, avatar picker survives repaints.
7. **Tests + mock daemon** — `tests/` (114 checks) and `server.py` now implements
   `/launch`, `/move` and `launchers`, plus an agent that “starts” and settles.

---

## 7. Verify any change

```bash
npm install                 # once (jsdom only)
npm test                    # 93 checks: demo mode, drag rules, manual moves, team/time — no server needed
npm test -- --e2e           # 114: also boots a fresh mock daemon on :5200 and drives real POSTs + WebSocket
AGENTMESH_E2E_PORT=5210 npm test -- --e2e     # if the default port is busy
```

| Suite | Covers |
|---|---|
| `tests/01-core-demo.js` | columns, tiles, needs strip, `document.title`, modals, tabs, action bars, new task, search, counts, demo badge |
| `tests/02-drag-and-manual.js` | every drag rule incl. refusals with reasons, cancel strip, manual `/move` + confirmations, run/rename from the rail, keyboard menu, typing guard |
| `tests/03-e2e-daemon.js` | real WebSocket + real POSTs: approve, create, 400 → toast, answer, rename, launch, launcher joining, manual move |
| `tests/04-team-and-time.js` | settings dialog, emoji/colour avatars + persistence, bulk add, rename from settings, density, theme, timers, workload bars, avatar chips, animations |

Run the suites after **any** change to `live.html`; each suite prints `PASS`/`FAIL`
lines and exits non-zero on failure. The suites drive the real DOM through jsdom —
if you add a control, add a check for it.

---

## 8. Pending / natural next steps

Roughly in order of value. None of these block using the board today.

1. **Avatars are browser-local.** They live in `agentmesh.prefs.v1`. If the team
   should look the same on every machine, add an `avatar: {emoji, color}` field to
   `Agent` in the daemon payload and read it in `avatarCfg()` (keep localStorage as
   the fallback). Suggested: `PATCH /api/mesh/agent/:id` `{avatar}`.
2. **No endpoint to remove or re-role an agent.** The rail/settings can add, rename
   and run; deleting or switching worker ↔ manual needs `DELETE /api/mesh/agent/:id`
   (and a matching demo branch in `demoApi`).
3. **Manual agents cannot be created with a different `id`/`label`.** `POST
   /api/mesh/agent` is called with `label === id`; if the daemon supports friendly
   labels, pass the label typed in settings.
4. **Touch drag & drop.** Native HTML5 drag events do not fire on touch. The `⋯`
   menu covers the moves, but a long-press drag would be nicer: add a
   `pointerdown`-based drag that reuses `dropAction`/`markZones` (they are already
   pure functions of task + zone).
5. **`review` tasks occupied too long** are only tinted amber. A real notification
   (title flash, `Notification` API behind a permission prompt) would help.
6. **Progress chart.** `progressLog` timestamps are enough to draw a per-task
   timeline sparkline or a cycle-time stat per column; `statusSince()` already
   computes the durations.
7. **Column ordering / WIP limits** — columns are fixed by status. A per-column WIP
   badge (`count/limit`) is a small render change in `columnEl`.
8. **Demo data maintenance.** `demoState()` must keep covering every status, a
   violation, a blocked question, a manual task awaiting a paste, a manual task with
   a deliverable, at least one launcher not joined, and both real and manual tiles —
   the suites assert most of this.

---

## 9. Conventions for the next agent

- Keep it **one file**. `live.html` must stay standalone (inline CSS/JS, no
  external requests). It is currently ~2,400 lines; keep new code tight and inside
  the existing numbered sections.
- Reuse `el()`; if you need a new helper, put it in section 1 and say why in a
  short comment.
- Any new string from the server is untrusted: `textContent` only.
- Keep the DOM stable across renders: key by `task.id` via `dataset.id`, avoid
  rebuilding containers that hold focus, and keep column/activity scroll positions
  (`captureScroll`/`restoreScroll`, `data-scrollkey`).
- Match the existing visual language: CSS variables in `:root` (light, dark and
  `prefers-color-scheme`), status colours from `--acc-*`, 120–180 ms transitions,
  no heavy gradients.
- When you change behaviour, extend the suite in `tests/` and update the table in
  §6/§7 of this file.
