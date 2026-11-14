---
spec_id: 001-persistent-tasks-in-postgres-with-a-web
title: Persistent tasks in Postgres with a web UI to view and create them
status: draft
priority: P1
shape: gitops-app
---

# 001 — Persistent tasks in Postgres with a web UI to view and create them

## Problem

The Task API keeps tasks in memory only, so every restart or redeploy of taskboard16-api loses all tasks, and a second replica would see different tasks. The Taskboard web UI is still the starter page, so people can use the taskboard only by calling the API by hand. Now that both components run in dev, the system should keep tasks in a Postgres database and give people a page where they see and create tasks.

## Stories

As a taskboard user, I want the tasks I recorded to still be there after the system restarts or is redeployed, so that I can rely on the taskboard as my list of what needs doing.
**Repos:** taskboard16-api, taskboard16-gitops

As a taskboard user, I want to open the Taskboard web UI and see all tasks with their due dates, so that I do not have to call the API by hand.
**Repos:** taskboard16-web, taskboard16-api, taskboard16-gitops

As a taskboard user, I want to create a task with a title and an optional due date in the web UI, so that I can write down what needs doing from the browser.
**Repos:** taskboard16-web, taskboard16-api

## Acceptance criteria

- **AC-001.1** Given a task created with `POST /tasks` `{"title": "Buy milk", "due_date": "2026-12-01"}`, when every taskboard16-api instance restarts (or is redeployed) and a client sends `GET /tasks`, then the response is 200 and holds that task with the same `id`, `title` and `due_date`.
- **AC-001.2** Given two taskboard16-api instances running against the same database, when a task is created through one of them, then `GET /tasks` answered by either instance holds that task.
- **AC-001.3** Given three tasks created one after another, when a user opens the start page of the Taskboard web UI, then the page lists the three tasks oldest first, each showing its title and its due date as `YYYY-MM-DD`, or "No due date" when it has none.
- **AC-001.4** Given no task exists, when a user opens the start page of the web UI, then the page shows "No tasks yet" and the create form.
- **AC-001.5** Given the start page of the web UI, when a user enters the title "Call Anna" with or without a due date and submits the create form, then within 2 seconds the task appears at the end of the list on the page without a manual reload, the form is cleared, and `GET /tasks` on the API holds the task with that title and due date.
- **AC-001.6** Given the create form, when a user submits a title that is empty, only whitespace or longer than 200 characters, then the form shows "Title must be 1 to 200 characters", keeps the entered values, and no task is created (the list and `GET /tasks` are unchanged).
- **AC-001.7** Given the database is unreachable, when a client sends `GET /tasks` or `POST /tasks`, then the response is 503 within 5 seconds, `GET /ready` answers 503, and no task created during the outage appears once the database is back.
- **AC-001.8** Given the API or its database is unavailable, when a user opens the start page of the web UI, then the page loads and shows "Tasks could not be loaded. Please try again later." instead of the list; and when a user submits the create form in that state, the form shows "The task could not be saved. Please try again." and keeps the entered values.

## Non-goals

- Editing, completing or deleting tasks, in the API or in the web UI.
- Users, login or per-user task lists: anyone who can reach the environment may see and create tasks; access control is a follow-up spec before staging or prod is exposed.
- Filtering, sorting options, search or pagination of the list.
- Fields beyond title and due date.
- Live updates of the list when someone else creates a task (a reload shows them).
- Carrying over tasks held in memory at the moment of the switch (see A7).
- Backups and restores of the database.

## Assumptions

- **A1** The API contract of `POST /tasks` and `GET /tasks` (fields, validation, 201/422, oldest first) stays as the API defines it today; only the storage changes (other option: extend the contract in this spec; affects AC-001.1, AC-001.2, AC-001.5).
- **A2** One Postgres database is declared as a resource of the taskboard16 system and is used only by taskboard16-api; the web UI reaches tasks only through the API, never the database (other option: the web UI reads the database directly; affects AC-001.1, AC-001.2, AC-001.3).
- **A3** The task list and the create form are on the web UI's start page, which replaces the starter page (other option: a separate `/tasks` page; affects AC-001.3 to AC-001.6, AC-001.8).
- **A4** The list shows every task, oldest first, with no limit, matching the API's order (other option: newest first or a cap; affects AC-001.3, AC-001.5).
- **A5** The web UI checks the title (1 to 200 characters after trimming) before sending and also shows the same message when the API rejects it with 422 (other option: rely on the API only; affects AC-001.6).
- **A6** When the database is unreachable the API answers 503 and reports not ready, so the platform stops routing to it; timeout 5 seconds (other option: 500, or a longer timeout; affects AC-001.7).
- **A7** Tasks held in memory by the running API when the database version is deployed are not carried over; they are lost as on any restart today (other option: a one-off copy; affects AC-001.1).
- **A8** UI wording is English and exactly as quoted in the criteria (other option: other wording or German; affects AC-001.4, AC-001.6, AC-001.8).
- **A9** The database starts empty in every environment; no seed tasks (other option: a few example tasks in dev; affects AC-001.4).
- **A10** No login: anyone who can reach the environment may see and create tasks, accepted for dev; access control (login) is a follow-up spec required before staging or prod is exposed (other option: login in this spec; affects AC-001.3, AC-001.5) (answered by the user 2026-11-14).

## Notes for the architect

- AC-001.7 of the taskboard16-api component spec `001-create-and-list-tasks-in-memory` ("tasks are gone after a restart") and its non-goal "Persistent storage" are contradicted by AC-001.1 of this spec; the API's component spec derived from this one must supersede that criterion (`replaces AC-001.7` in that repo).

## Open questions

- ~~Who may see and create tasks? The web UI and the API are exposed through the Gateway with no login, so anyone who can reach them can read every task and create tasks. (suggested: no login in this spec, accepted for dev; access control is a follow-up spec before staging or prod is exposed; affects AC-001.3, AC-001.5, new criterion if login is wanted)~~ Answered 2026-11-14: no login in this spec; anyone who can reach the environment may see and create tasks, accepted for dev; access control (login) is a follow-up spec before staging or prod is exposed (A10, Non-goals).

## Changelog

- 2026-11-14 created
- 2026-11-14 amended: answered the access question (no login, accepted for dev; login is a follow-up spec before staging/prod), folded into A10 and Non-goals
