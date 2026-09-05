# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## Project

Telusko Workflow Engine (repo: `content-tracker`) — a static, frontend-only Kanban board
for tracking video production through a pipeline: `Code & Examples Ready → Recorded →
Editing → Uploaded (Pending Verify) → Verified & Published`. Each task is assigned to a
role (`Admin`, `Content`, `Editor`, `Uploader`), and role determines which stage
transitions a user is allowed to perform.

Everything lives in `frontend/` (`index.html`, `app.js`, `styles.css`). There is no
backend, no build step, no package manager, and no test suite.

## Tech stack

- **HTML5** — `frontend/index.html`, single page, no templating.
- **CSS3** — `frontend/styles.css`, hand-written, no preprocessor or framework.
- **Vanilla JavaScript (ES6+)** — `frontend/app.js`, no framework (no React/Vue/etc.),
  no bundler, no transpilation. Uses native `fetch`, `localStorage`, and plain script tags.
- **Browser `localStorage`** — the only persistence layer: stores the JWT (`telusko_token`)
  and the offline-demo dataset (`telusko_offline`, `telusko_offline_tasks`).
- No package manager (no `package.json`), no CI config, no Dockerfile.

## Running locally

Serve `frontend/` with any static server, e.g.:

```
python -m http.server 8080 --directory frontend
```

or open `frontend/index.html` directly in a browser. Changes are picked up on reload —
no bundler, no dev server needed.

The app expects a backend answering `/api/auth/login` and `/api/tasks*` at relative
paths. Without one reachable, use offline demo mode (below) to exercise the UI.

## Architecture — `frontend/app.js`

Single-file vanilla-JS SPA.

- **`STATES`** — ordered pipeline array (`code_ready → recorded → editing → uploaded →
  published`). Order defines advancement; keep in sync with the `<select id="task-state">`
  options in `index.html`.
- **`ROLE_CONFIG`** — per-role `canAdvance` whitelist of stages each role may push a card
  forward from. Drag-and-drop is Admin-only; non-admins use the "Move Forward" button,
  gated by `canCurrentRoleAdvance`.
- **`ROLE_MAP`** — maps JWT `role` claim values (`admin`, `content_team`, `video_editor`,
  `uploader`) to `ROLE_CONFIG` keys.
- **`tasks`** — in-memory array, re-rendered via `renderBoard()` after every API call
  resolves. Never mutate `tasks` directly outside that flow.

### Auth

Login posts form-encoded credentials to `/api/auth/login`, stores the returned JWT in
`localStorage` (`telusko_token`), and derives `currentRole` from the token's `role` claim
(`applyRoleFromToken`, via `parseJwtPayload` — client-side decode only, not a security
check). `apiFetch` attaches `Authorization: Bearer <token>` to every call and forces
re-login on a 401.

### Offline demo mode

If `/api/auth/login` is unreachable or returns a non-JSON response (i.e. no backend is
attached) and the credentials are exactly `admin` / `admin123`, the app falls into a
local-only demo mode instead of failing the login:

- `enterOfflineMode()` sets a `telusko_offline` flag and seeds `telusko_offline_tasks` in
  `localStorage` from `SEED_TASKS` if not already present.
- All four `api.*` methods branch on `isOfflineMode()` and read/write
  `telusko_offline_tasks` instead of calling `apiFetch`.
- `logout()` clears both the token and offline mode.

Any other credentials against an unreachable backend show a real error instead.

### API layer (`api.{getTasks,createTask,updateTask,deleteTask}`)

Wraps `apiFetch`, which wraps `fetch` and normalizes the result to
`{ ok, status, data }` (`data` is parsed JSON if the response has a JSON content-type,
else `null`). `getTasks()` is the only method that swallows errors and falls back to the
current in-memory `tasks` for graceful degradation; the others throw and let the caller
show a toast.

Task object shape (camelCase on the wire): `id`, `title`, `description`, `assignedRole`
(`Admin|Content|Editor|Uploader`), `state` (one of the `STATES` keys), `createdAt`
(`YYYY-MM-DD`).

### Rendering

`renderBoard()` is the single re-render entry point — full rebuild, no diffing. Cards and
columns are recreated from `tasks` each call. Drag-and-drop state lives in the
module-level `draggedTaskId`, reset on `dragend` and after every drop.

### Roles & permissions

- Admin: free drag-and-drop across all columns, can delete, can edit any field including
  stage in the modal.
- Content / Editor / Uploader: can only advance from stages listed in their
  `ROLE_CONFIG.canAdvance`; the stage `<select>` in the modal is disabled for non-Admins.

These are UX-only guards, not a trust boundary — there is no server in this repo to
re-enforce them.

## Rules

- Keep the response envelope (`{ ok, status, data }`) and camelCase task field names if a
  backend is ever wired up — `app.js` depends on both.
- Don't add a build step, framework, or package manager without an explicit request —
  the project is intentionally dependency-free.
  