---
plan_id: 001-persistent-tasks-in-postgres-with-a-web
spec: 001-persistent-tasks-in-postgres-with-a-web
feature: Persistent tasks in Postgres with a web UI to view and create them
gitops_app: ika100-demo/taskboard16-gitops
status: completed
repos:
- id: taskboard16-gitops
  shape: gitops-app
  summary: Declare the taskboard16 Postgres database, give it to taskboard16-api and point taskboard16-web
    at the API
  acs:
  - AC-001.1
  - AC-001.2
  - AC-001.3
  order: before-merge
  gitops:
  - resource: postgres
    type: database
  - uses: postgres
    component: taskboard16-api
  - env:
      API_URL: http://taskboard16-api
    component: taskboard16-web
  depends_on: []
  done: true
- id: taskboard16-api
  shape: service-python
  summary: Store tasks in Postgres so they survive restarts and are shared by every instance
  acs:
  - AC-001.1
  - AC-001.2
  - AC-001.5
  - AC-001.6
  - AC-001.7
  criteria:
  - 'Given a task created with `POST /tasks` `{"title": "Buy milk", "due_date": "2026-12-01"}`, when the
    service restarts against the same database and a client sends `GET /tasks`, then the response is 200
    and holds that task with the same `id`, `title` and `due_date` (system AC-001.1) (replaces AC-001.7)'
  - Given two service instances using the same `DATABASE_URL`, when a client creates a task with `POST
    /tasks` through one of them, then `GET /tasks` answered by either instance is 200 and holds that task,
    and the list is oldest first by creation time (system AC-001.2, AC-001.5)
  - 'Given a running service with a reachable database, when a client sends `POST /tasks` with `{"title":
    "Call Anna"}` and then `GET /tasks`, then the POST answers 201 with `"due_date": null` and the task
    is the last element of the GET array with that `id`, `title` and `due_date` (system AC-001.5)'
  - Given a running service, when a client sends `POST /tasks` whose `title` is empty, only whitespace
    or longer than 200 characters after trimming, then the response is 422 and `GET /tasks` returns the
    same array as before (system AC-001.6)
  - 'Given the database is unreachable, when a client sends `GET /tasks` or `POST /tasks`, then the response
    is 503 with the body `{"detail": "Task storage unavailable"}` within 5 seconds, and `GET /ready` answers
    503 (system AC-001.7)'
  - Given the database was unreachable while a client sent `POST /tasks` (answered 503), when the database
    is reachable again, then `GET /ready` answers 200 and `GET /tasks` does not hold the task sent during
    the outage (system AC-001.7)
  - Given an empty database on first start, when the service starts, then it creates its schema itself
    and `GET /tasks` answers 200 with `[]` (system AC-001.1)
  arguments: |
    Storage moves from the in-memory TaskStore to Postgres; connection settings come only from DATABASE_URL / PG*
    (injected by the gitops entry's `uses: postgres`). Remove the non-goal "Persistent storage" from the old spec's
    scope in the new spec. Ordering is by a server-side creation timestamp plus a tie-breaker (e.g. a bigserial),
    never by the uuid. Schema is created on startup idempotently (two instances may start at once). The request
    and response contract of POST/GET /tasks is unchanged (system A1). /health stays DB-independent.
  depends_on: []
  done: true
- id: taskboard16-web
  shape: web-nextjs
  summary: Show all tasks and a create form on the Taskboard start page, backed by the Task API
  acs:
  - AC-001.3
  - AC-001.4
  - AC-001.5
  - AC-001.6
  - AC-001.8
  criteria:
  - Given the Task API returns three tasks from `GET /tasks`, when a user opens the start page `/`, then
    the page lists the three tasks in the API's order (oldest first), each showing its title and its due
    date as `YYYY-MM-DD`, or "No due date" when `due_date` is null (system AC-001.3)
  - Given the Task API returns `[]` from `GET /tasks`, when a user opens the start page, then the page
    shows "No tasks yet" and the create form (system AC-001.4)
  - Given the start page, when a user enters the title "Call Anna" with or without a due date and submits
    the create form, then the web UI sends `POST /tasks` with that `title` and `due_date` (`YYYY-MM-DD`
    or null), and within 2 seconds the returned task appears at the end of the list without a manual reload
    and the form is cleared (system AC-001.5)
  - Given the create form, when a user submits a title that is empty, only whitespace or longer than 200
    characters after trimming, then the form shows "Title must be 1 to 200 characters", keeps the entered
    values and sends no `POST /tasks` (system AC-001.6)
  - Given the create form with a valid title, when the Task API answers `POST /tasks` with 422, then the
    form shows "Title must be 1 to 200 characters", keeps the entered values and the list is unchanged
    (system AC-001.6)
  - Given the Task API answers `GET /tasks` with 503, any other non-200 status, an unreadable body, or
    does not answer within 6 seconds, when a user opens the start page, then the page loads with status
    200 and shows "Tasks could not be loaded. Please try again later." instead of the list (system AC-001.8)
  - Given the Task API answers `POST /tasks` with 503, any status other than 201 or 422, or does not answer
    within 6 seconds, when a user submits the create form, then the form shows "The task could not be
    saved. Please try again." and keeps the entered values (system AC-001.8)
  - Given the Task API is down, when the platform calls `/api/ready` and `/api/health`, then both still
    answer 200, so the page stays reachable (system AC-001.8)
  arguments: |
    The start page replaces the starter page (system A3). The API is called only from the Next.js server
    (Server Component for the list, Server Action or route handler for create) using the server-side env var
    API_URL (no NEXT_PUBLIC_*, no browser-to-API call, no CORS). Document API_URL in docs/env-vars.md.
    Tests stub the Task API at API_URL; no login (system A10).
  depends_on: []
  done: true
gitops_pin: []
---

## Decomposition

Three entries, all in one level (no `depends_on`): the contract below fixes everything the web UI needs from the API,
so taskboard16-api and taskboard16-web build in parallel, and the gitops entry is applied before the merges
(`order: before-merge`) so the database and the wiring exist when the new images reach dev. No shared library is
needed: the only shared shape is the three-field task JSON, which the contract states.

## taskboard16-gitops — Declare the Postgres database and wire the components

**Shape:** `gitops-app` · **Criteria:** AC-001.1, AC-001.2, AC-001.3 · **Depends on:** []

The system has no resources yet. Persistence (AC-001.1) and sharing between instances (AC-001.2) need one Postgres
database of the taskboard16 system, used only by taskboard16-api (system A2): `resource postgres` (type `database`),
then `uses postgres` for taskboard16-api, which injects `DATABASE_URL` and `PG*`. The web UI showing tasks
(AC-001.3) needs to reach the API: `env API_URL=http://taskboard16-api` on taskboard16-web (the in-cluster Service
`taskboard16-api` listens on port 80). No replica or resource change: the spec decides no instance value, and
AC-001.2's two instances are a test condition of the API, not a deployed replica count. The database starts empty
in every environment (system A9).

## taskboard16-api — Store tasks in Postgres

**Shape:** `service-python` · **Criteria:** AC-001.1, AC-001.2, AC-001.5, AC-001.6, AC-001.7 · **Depends on:** []

Owns the storage half of every criterion: tasks survive restarts and are shared by instances (AC-001.1, AC-001.2),
a created task is readable via `GET /tasks` (AC-001.5) and a rejected one is not (AC-001.6), and the outage
behaviour with 503 within 5 seconds, a 503 readiness and no half-created task (AC-001.7). Its component spec
supersedes AC-001.7 of `001-create-and-list-tasks-in-memory` ("tasks are gone after a restart") through the line
marked `(replaces AC-001.7)`. The HTTP contract of `POST /tasks` and `GET /tasks` stays as built (system A1).

## taskboard16-web — Task list and create form on the start page

**Shape:** `web-nextjs` · **Criteria:** AC-001.3, AC-001.4, AC-001.5, AC-001.6, AC-001.8 · **Depends on:** []

Owns everything the user sees: the list oldest first with due dates or "No due date" (AC-001.3), the empty state
(AC-001.4), creating without a reload within 2 seconds (AC-001.5), the title check before sending and on a 422
(AC-001.6, system A5), and the two error messages when the API or its database is down (AC-001.8). Wording is
English and exactly as quoted (system A8).

## Contract

Base URL for the web UI: the server-side env var `API_URL` (`http://taskboard16-api` in the cluster). Only the
Next.js server calls the API; the browser talks to the web UI only. All bodies are JSON (`Content-Type:
application/json`), UTF-8. No authentication (system A10).

Task object (unchanged from taskboard16-api spec 001):

| Field | Type | Notes |
|---|---|---|
| `id` | string | opaque, server-assigned (uuid4); never sent by the client |
| `title` | string | 1 to 200 characters after trimming; stored and returned exactly as sent |
| `due_date` | string `YYYY-MM-DD` or `null` | calendar date without time; `null` = no due date |

`GET /tasks`
- `200`: JSON array of task objects, every task, oldest first by creation time, no limit; `[]` when there are none.
- `503`: the database is unreachable (see Errors).

`POST /tasks`
- Request: `{"title": string, "due_date": "YYYY-MM-DD" | null}`; `due_date` optional; unknown fields ignored. The
  web UI sends `due_date: null` when the date field is empty.
- `201`: the created task object; it is committed, so a following `GET /tasks` from any instance holds it as the
  last element.
- `422`: validation failed (title missing, not a string, empty, whitespace-only, over 200 characters after
  trimming; `due_date` not a real `YYYY-MM-DD` date); FastAPI standard body `{"detail": [{"type", "loc", "msg",
  "input", ...}]}`. Nothing is stored.
- `503`: the database is unreachable; nothing is stored, and the task never appears later.

`GET /ready` (taskboard16-api): `200 {"status": "ready"}` when a `SELECT 1` against the database succeeds within 1
second, else `503`. `GET /health` stays `200` regardless of the database.

`/api/ready` and `/api/health` (taskboard16-web): `200` regardless of the Task API, so the start page can show its
error message while the API is down.

Database: one Postgres database (resource `postgres`), reached by taskboard16-api only, via `DATABASE_URL` / `PG*`.
The API owns its schema and creates it idempotently at startup.

### Errors

- Validation errors: `422` with FastAPI's standard `detail` array. The web UI does not parse the array; any `422` on
  `POST /tasks` shows "Title must be 1 to 200 characters" and keeps the entered values (the web UI checks the date
  field is empty or a `YYYY-MM-DD` date picker value, so a date 422 does not occur from the form).
- Storage unavailable: `503` with `{"detail": "Task storage unavailable"}` on `GET /tasks` and `POST /tasks`.
- Any status the contract does not list (including `500`, `502`, `504`), a body that is not the listed JSON shape, a
  connection error or a timeout is treated by the web UI like `503`: on `GET /tasks` the page shows "Tasks could not
  be loaded. Please try again later." instead of the list; on `POST /tasks` the form shows "The task could not be
  saved. Please try again." and keeps the entered values. The web UI never retries a `POST` on its own (no duplicate
  tasks); the user resubmits.
- The web UI logs every failed API call through `@/lib/logger` with the status (or `timeout` / `connection_error`)
  and never shows API error bodies to the user.

### Timeouts

- taskboard16-api to Postgres: connect timeout 2 seconds, statement timeout 2 seconds, pool checkout wait at most 1
  second; on any of them the request answers `503` so that `GET /tasks` and `POST /tasks` answer within 5 seconds
  while the database is down (AC-001.7). Readiness check: 1 second. A `POST` whose transaction did not commit is
  rolled back, so no task created during an outage appears afterwards.
- taskboard16-web to taskboard16-api: 6 seconds per call (`GET /tasks` on page render, `POST /tasks` on submit), longer
  than the API's 5-second bound so the API's `503` arrives first; after that the call counts as failed (see Errors).
  With the API healthy, `POST /tasks` plus updating the list stays under 2 seconds (AC-001.5).
- When the API is down or not ready, Kubernetes stops routing to it (readiness `503`); the web UI keeps serving its
  start page with the error message, and recovers on the next page load or submit with no restart.

## gitops-app PR

The gitops entry is applied before either component PR merges: it adds the `postgres` database resource, gives it to
taskboard16-api (`DATABASE_URL`, `PG*`) and sets `API_URL=http://taskboard16-api` on taskboard16-web, all in dev
(the only instance of both components). After Argo reconciles, verify: the Postgres resource is healthy, the
taskboard16-api pod has `DATABASE_URL` and `/ready` answers 200, the taskboard16-web pod has `API_URL`. After the
component PRs merge, dev tracks `main`: the dev smoke tests run, and `http://taskboard16-web.taskboard16-dev.localhost/`
lists the tasks created via `http://taskboard16-api.taskboard16-dev.localhost/tasks`, which survive a
`kubectl rollout restart` of taskboard16-api.

No `gitops_pin`: neither component has a staging or prod instance, and the spec requires access control (a follow-up
spec) before staging or prod is exposed (system A10).
