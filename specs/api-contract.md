# API Contract — Telusko Workflow Engine

Derived directly from `frontend/app.js`. This is the HTTP contract a backend
must implement for the existing frontend to work unmodified — no frontend
changes are assumed.

All endpoints are called relative to the page origin (`/api/...`). Every
call except login attaches `Authorization: Bearer <token>` when a token is
present (see [Auth & error handling](#auth--error-handling)).

## Task object shape

Wire format is camelCase:

```json
{
  "id": 1,
  "title": "Introduction to Spring Boot",
  "description": "Cover project setup, auto-configuration, and starter dependencies.",
  "assignedRole": "Content",
  "state": "code_ready",
  "createdAt": "2026-05-01"
}
```

- `assignedRole` — one of `Admin | Content | Editor | Uploader`.
- `state` — one of `code_ready | recorded | editing | uploaded | published`
  (order matters — it's the pipeline sequence `STATES` in app.js).
- `createdAt` — `YYYY-MM-DD`.

## `POST /api/auth/login`

Not one of the four `api.*` methods, but required for everything else to work.

- Request: `application/x-www-form-urlencoded` body `username=...&password=...`
  (not JSON).
- Success response: `application/json`, must include `access_token` (a JWT).
  The frontend decodes the JWT payload client-side (`parseJwtPayload`) and
  reads a `role` claim with values `admin | content_team | video_editor |
  uploader`, mapped to `Admin | Content | Editor | Uploader` respectively.
- Failure response: **must** be `application/json` with a `detail` field.
  A non-JSON error response (e.g. a plain-text 404 from a static file
  server) is interpreted by the frontend as "no backend attached" and
  triggers local offline-demo mode instead of showing the error — so a real
  backend must always return JSON, even on failure, or logins with
  non-default credentials will show a misleading error.

## `getTasks` → `GET /api/tasks`

- Request: no body, no query/path params.
- Success response: `200`, JSON array of task objects (shape above).
- Error handling is unique to this method: **it never throws to the
  caller** (except on 401). Any other failure (`!res.ok`, network error) is
  caught internally, shown as a toast (`Failed to load tasks: <message>`),
  and the method returns a `structuredClone` of the current in-memory
  `tasks` array — the board keeps showing stale data rather than going
  blank.

## `createTask` → `POST /api/tasks`

- Request: `Content-Type: application/json`, body:
  ```json
  { "title": "...", "description": "...", "assignedRole": "Content", "state": "code_ready" }
  ```
  Note: no `id` or `createdAt` — the backend must generate both.
- Success response: `200`/`201`, JSON body of the full created task object
  (including server-assigned `id` and `createdAt`).
- Failure response: any `!res.ok` status. The frontend throws
  `Error("Server error <status>")` with `.status` and `.data` (the parsed
  JSON body, if any) attached. The caller displays `data.detail` if
  present, else the generic message — **so validation/permission errors
  must be returned as JSON with a `detail` string.**

## `updateTask` → `PUT /api/tasks/{id}`

- Request: `Content-Type: application/json`, body is a **partial** update —
  e.g. `{ "state": "recorded" }` for a drag/advance move, or the full
  `{ title, description, assignedRole, state }` set from the edit modal.
  The backend must merge partial updates rather than requiring a full
  object.
- Success response: JSON body of the full updated task object.
- Failure response: same contract as `createTask` — `.status` + `.data`
  with a `detail` string, surfaced as `Blocked: <detail>` in the UI.

## `deleteTask` → `DELETE /api/tasks/{id}`

- Request: no body.
- Success response: any `res.ok` status; body is ignored (the frontend
  just returns `true`).
- Failure response: same `.status` + `.data` contract as above, surfaced as
  `Blocked: <detail>`.

## Auth & error handling

- **Envelope**: every non-401 response is normalized by `apiFetch` to
  `{ ok, status, data }`, where `data` is the parsed JSON body if the
  response's `content-type` includes `application/json`, otherwise `null`.
  A backend that returns errors without a JSON content-type will have
  `data` come back `null`, and callers that expect `data.detail` will fall
  back to a generic message.
- **Auth header**: `Authorization: Bearer <token>` is attached to every
  call when a token exists in `localStorage`.
- **401 handling**: a `401` response short-circuits inside `apiFetch`
  itself — it clears the stored token, forces the login form back open,
  shows a `"Session expired. Please sign in again."` toast, and throws
  `Error("Unauthorised — redirected to login")`. This happens **before**
  any method-specific error handling, so `getTasks` correctly rethrows it
  instead of swallowing it, but `createTask`/`updateTask`/`deleteTask`
  callers will show a second, redundant toast (`Save failed: Unauthorised…`
  or similar) on top of the session-expired one — existing behavior, not a
  bug to fix on the backend side, just something to expect.

### DISCREPANCY

CLAUDE.md states role-based stage-advancement rules
(`ROLE_CONFIG.canAdvance`) are "UX-only guards, not a trust boundary —
there is no server in this repo to re-enforce them." Once a real backend
exists, it **must** independently re-validate that the authenticated
user's role is permitted to make the requested state transition on
`updateTask`, returning a `4xx` with a JSON `detail` message on violation
(the frontend already has a path to surface this via `Blocked: <detail>`)
— otherwise the permission model exists only in the client and is
trivially bypassable via direct API calls.
