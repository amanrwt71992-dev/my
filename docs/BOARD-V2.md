# AgentMesh Board v2 — the contract

Owner: manager. Every v2 task points here. If a task and this file disagree, ask the manager (mesh_ask_manager). Do not guess.

Code lives in the project root:
`server/meshStore.ts` (state and rules), `server/tools.ts` (MCP tools), `server/index.ts` (REST), `server/live.html` (board page), `scripts/smoke.ts` (end to end checks), `tests/board/*.cjs` (board checks).

## 1. Columns and statuses

| Column label | Internal status | Meaning |
|---|---|---|
| Backlog | `backlog` | Not ready: waiting on other tasks (`dependsOn` unmet), or parked by the manager. Nobody can claim it. |
| To do | `todo` | Ready, waiting for pickup. May be reserved for one agent (`assignee`) or open to any worker. |
| In progress | `claimed`, `in_progress` | An agent holds it right now. |
| Needs answer | `blocked` | The agent asked the manager a question. |
| Code review | `review` | Reported, waiting for the manager. |
| Pending deployment | `pending_deploy` | Approved, not live yet. |
| Done | `done` | Finished (and deployed, or approved with no deployment). |
| (hidden) | `cancelled` | |

Rules:
- New tasks start as `todo`. They start as `backlog` only if `dependsOn` has an unfinished task or `parked: true` was sent.
- A `backlog` task moves to `todo` by itself the moment every task in `dependsOn` is `pending_deploy` or `done` ("approved"). Cancelled dependency: the task stays in Backlog and gets the flag `blockedByCancelled: true` for the manager to fix.
- Sent back after review, released, or returned by the stale-claim timer: status `todo`, still reserved for the same agent (`assignee`), feedback kept. (Today these use `backlog`.)
- Migration on load: every stored task with status `backlog` becomes `todo`. Do it once, in the store loader, and keep it idempotent.
- Only `todo` tasks can be claimed.

## 2. Stories and subtasks

A story is a task with `type: "story"`. It is a container, not work. Workers never claim it and never see it in `mesh_claim_task`.

New task fields:
- `parentId?: string` — the story this task belongs to. One level only (a story cannot have a parent; a subtask cannot be a story).
- `dependsOn?: string[]` — task ids that must be approved first (max 10, same workspace, no cycles; reject cycles with a clear error).
- `order?: number` — position among siblings.
- Story only: `mode: "parallel" | "sequential"`.

Mode behaviour (applied when subtasks are created):
- `sequential`: each subtask automatically gets `dependsOn = [previous sibling]`. If the story has an `assignee`, every subtask is reserved for it, so ONE agent does them in order. If a subtask names a different assignee, that is allowed (hand-off mid story).
- `parallel`: subtasks have no automatic dependencies. They may have different assignees or none. At creation the store compares `allowedFiles` of siblings and returns `warnings: [{a, b, overlap}]` for overlapping patterns (agents share one working tree).
- Explicit `dependsOn` is always allowed on top of either mode (for example "tests wait for both parallel parts").

Story status is DERIVED, never set by hand, recomputed after every change to a child:
- no children: `backlog`.
- every child `backlog` or `todo`, none started: `todo` (shown as To do).
- any child `claimed`, `in_progress`, `blocked`, `review`, or some done and some not: `in_progress`.
- every child approved (`pending_deploy` or `done`): `review` — final story review. The manager checks the story's own `acceptance` list across the combined result.
- Approve story: `pending_deploy` (or `done` with `noDeploy`/`deploy:false`), and its children follow (children that are `pending_deploy` become `done` when the story is marked deployed). Send story back: the manager adds a new subtask (or sends one child back), story returns to `in_progress`. Cancelling a story cancels every unfinished child.
- Progress for the board: `progress: {done, total}` where done = children `done` or `pending_deploy`.

`mesh_create_plan` accepts a story: `{feature_title, mode, assignee?, acceptance[], subtasks:[{key, title, brief, assignee?, dependsOn?: [key|taskId], allowedFiles, forbiddenFiles, acceptance, type, priority, labels}]}`. `key` is a local name used only inside one plan call so subtasks can reference each other. Old calls without `mode` keep working: they create independent `todo` tasks like today.

## 3. Handover (agent busy or stuck, another agent free)

Availability per agent, computed by the store and included in state:
- `busy` — holds a task. `idle` — registered, seen in the last 3 minutes, holds nothing. `offline` — otherwise.
- Manual agents are always `idle` (a human relays).

Rules:
- `todo` task: the manager can change `assignee` at any time (already true, keep it) or clear it to open the task to any worker.
- `claimed` / `in_progress` / `blocked` task: `handover(taskId, toAgent|null, note?)` releases it (as the Release button does today) and reserves it for `toAgent`. The new agent's claim reply includes `handover: {from, note, files_modified, last_progress}` taken from the previous holder's progress log. File locks held for the task transfer to the new agent.
- A worker can give up its OWN task: new MCP tool `mesh_release_task({agent_id, task_id, reason})` → status `todo`, `assignee = null` (open to any worker), reason stored as the note, event logged. It must say what is already done.
- Handover suggestions in state: `handoverHints: [{taskId, from, candidates:[agentId]}]` for `todo` tasks reserved for a `busy` agent in the same workspace while at least one other non-manual agent is `idle`, and the task has waited more than 2 minutes. Suggestion only; nothing moves by itself.
- REST: `POST /api/mesh/task/:id/handover {to, note}` (`to: ""` = open to any worker). Origin checks as for the other endpoints.

## 4. Type, priority, labels

- `type`: `feature` (default), `bug`, `test`, `docs`, `refactor`, `research`, `deploy`, `story`. Keep the existing `kind: "deploy"` working; `type: "deploy"` and `kind: "deploy"` mean the same, store them consistently.
- `priority`: `low`, `normal` (default), `high`, `urgent`. Claim order among eligible tasks: reserved-for-me first, then priority (urgent first), then oldest.
- `labels`: up to 5, each a lowercase slug of at most 24 chars (`[a-z0-9-]`), free text chosen by the manager (for example `ai`, `board`, `mcp`).
- All three are editable while the task is not finished, shown in `mesh_list_status` for the manager, and filterable in REST: `GET /api/mesh/state?type=&label=&priority=&assignee=`.

## 5. What the board shows on a tile

ID, type icon, title (2 lines), priority mark (only high/urgent), owner avatar + name with a state dot (busy / idle / offline), time in status, up to 2 labels plus "+n", and these chips when they apply:
- `Story TASK-x · 2/5` (links to the story)
- `after TASK-y` (dependency, muted while unmet)
- `Changes requested`
- `Hand over` button when `handoverHints` names this task
- a story tile shows a progress bar and its mode (`parallel` / `in order`), and can collapse its subtasks

Views: group by Status (default), by Agent, by Story (one lane per story, subtasks inside).

## 6. Rules for everyone working on this

- Never stop, restart or kill the daemon on port 3333, and never open `~/.agentmesh/state.json`. It is the live board other agents are using. `npm run smoke` starts its own server; use that.
- Checks to run before you report: `npm run build`, `npm run smoke`, `node scripts/check-board.mjs server/live.html`, and the board suites with `NODE_PATH=<path to a node_modules containing jsdom>`.
- Add checks for every rule you implement. Never delete, loosen or pad a check. In your reply, state the check counts before and after.
- Do not add npm dependencies.
