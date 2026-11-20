# StudyHub — Software Architecture Document (RUP)

| Field | Value |
| :---- | :---- |
| Project | StudyHub |
| Document | Software Architecture Document (RUP) |
| Phase | Elaboration |
| Authors | Ergens Hyso, Kristine Megrelishvili, Antonio Kanani |
| Status | Draft for Lifecycle Architecture Milestone |
| Related | [SRS](../SRS.md), [Use case diagram](../diagrams/use-case-diagram.puml) |

---

## 1. Introduction

This document describes the software architecture of StudyHub. It follows the Rational Unified Process (RUP) Software Architecture Document structure. The goal of Elaboration is to show that our React / Express / PostgreSQL design can carry the product through Construction.

StudyHub is a web application for student groups: courses and deadlines, shared tasks, meeting scheduling, notes, and reminders.

---

## 2. Architectural representation — Model–View–Controller

StudyHub uses the **Model–View–Controller (MVC)** pattern across a browser client and a Node.js API.

| Layer | Role in StudyHub | Where it lives |
| :---- | :---- | :---- |
| **View** | Screens the student sees and interacts with | React pages and layout in `client/src/pages`, `client/src/components` |
| **Controller** | Accepts HTTP requests, checks auth, maps input/output, calls domain rules and persistence | Express route modules in `server/src/routes` plus `server/src/middleware/auth.js` |
| **Model** | Domain rules and persistent data | Domain helpers in `server/src/services` (shared with Cucumber via root `src/`), PostgreSQL tables, access through `server/src/db.js` |

The React client is not a classic server-rendered MVC framework. The **View** runs in the browser; the **Controller** and most of the **Model** run on the server. The client talks to the API with REST/JSON (`client/src/api.js`).

```text
Browser (View)  --REST/JSON-->  Express (Controller)  -->  Services + PostgreSQL (Model)
```

High-level MVC diagram: [mvc-overview.png](../diagrams/mvc-overview.png) ([`.puml`](../diagrams/mvc-overview.puml))  
Class diagram with MVC packages: [mvc-class-diagram.png](../diagrams/mvc-class-diagram.png) ([`.puml`](../diagrams/mvc-class-diagram.puml))

---

## 3. Architectural goals, constraints, and MVC tooling

### 3.1 Goals

- One semester delivery for High- and Medium-priority SRS features.
- Same domain rules for Cucumber tests and the live API.
- Clear separation so UI changes do not rewrite persistence.

### 3.2 Constraints

- Stack fixed in Week 2: **React (Vite)**, **Node.js + Express**, **PostgreSQL**.
- No university SSO in the first release; email via SMTP only.
- GDPR-minded storage: salted password hashes, only needed personal data.

### 3.3 Our MVC tool / framework stack

We do **not** use a full-stack MVC generator such as Laravel, Django, Rails, or Spring MVC scaffolding.

| Part | Tool | Generates CRUD? | How we use it |
| :---- | :---- | :---- | :---- |
| View | React + Vite + React Router | No | We write pages and forms by hand |
| Controller | Express routers | No | We write REST routes by hand (`GET`/`POST`/`PATCH`/`DELETE`) |
| Model | PostgreSQL + `pg` + SQL migrations | No ORM scaffold | Schema in `db/migrations/001_init.sql`; queries in route handlers |

So: **our stack does not auto-generate CRUD**. We implement Create / Read / Update / Delete ourselves for tasks, groups, courses, meetings, notes, and related resources. That matches our Cucumber-first approach: domain validation lives in small service modules, not in a generated admin UI.

Architectural UML of the running system: [architectural-uml.png](../diagrams/architectural-uml.png) ([`.puml`](../diagrams/architectural-uml.puml))  
(Overlay this on the IDE-generated structure diagram from the last homework; see Week 7 blog.)

---

## 4. Use-Case view

The overall use-case diagram from the SRS is part of the architecture’s use-case view. It shows actors (Visitor, Student, Group Owner, Email Service) and the StudyHub use cases.

Embed / source: [use-case-diagram.puml](../diagrams/use-case-diagram.puml)  
Rendered: [use-case-diagram.png](../diagrams/use-case-diagram.png)

Architecturally significant use cases for Elaboration:

| Use case | Why it matters for architecture |
| :---- | :---- |
| Create a task | Validates View → Controller → Model path, membership checks, email side effect |
| Schedule meeting | Same path plus common-time logic over availability slots |
| Register / Log in | Session cookie auth middleware on all protected controllers |
| Send reminders | Asynchronous-style side effect via SMTP from the model/services boundary |

---

## 5. Logical view

### 5.1 MVC — high-level logical diagram

Packages map to MVC:

1. **View** — React SPA packages (`pages`, `components`, `auth` context, `api` client).
2. **Controller** — Express `routes/*` and `middleware/auth`.
3. **Model** — `services/*`, SQL schema, `db` pool.

See [mvc-overview.png](../diagrams/mvc-overview.png).

### 5.2 Class diagram with MVC sections

The class / module diagram groups concrete types into **View**, **Controller**, and **Model** packages and shows the main associations for the task and meeting flows.

See [mvc-class-diagram.png](../diagrams/mvc-class-diagram.png).

Example flow for *Create a task*:

1. `GroupDetail` (View) posts via `api` to `/api/groups/:id/tasks`.
2. `tasks` router (Controller) runs `requireAuth`, checks group membership, calls `validateCreateTask`.
3. Insert into `tasks` table (Model); optional email through `sendEmail`.
4. JSON response updates React state (View).

---

## 6. Process view

*Not applicable for this milestone.* StudyHub is a request/response web app without a separate multi-process concurrency design beyond Node’s event loop and PostgreSQL connections.

---

## 7. Deployment view

Client and server are separate deployable parts.

| Node | Artifact | Runtime |
| :---- | :---- | :---- |
| Student device | React SPA (static build from Vite) | Browser |
| Application host | Express API (`server/`) | Node.js 20+ |
| Data host | PostgreSQL 16 | Docker Compose locally (`db` service); managed Postgres in production |
| External | SMTP provider | Email for confirmations / reminders |

Deployment diagram (UML): [deployment.png](../diagrams/deployment.png) ([`.puml`](../diagrams/deployment.puml))

Even for local development the same shape holds: Vite on port 5173, API on port 4000, Postgres on host port 55432.

---

## 8. Implementation view

*Not applicable / light for this milestone.* Source layout:

```text
client/     View (React)
server/     Controller + Model access
db/         Schema migrations
features/   Cucumber (tests against shared domain rules)
src/        In-memory adapters for Cucumber (aligned with server services)
```

---

## 9. Data view (database model)

Persistent model is relational PostgreSQL. Core entities:

- `users`, `password_resets`, `notification_prefs`
- `courses`, `deadlines`
- `groups`, `group_members`, `invitations`
- `tasks`
- `availability_slots`, `meetings`, `meeting_responses`
- `notes`

ER / DB model diagram: [db-model.png](../diagrams/db-model.png) ([`.puml`](../diagrams/db-model.puml))  
Authoritative schema: [`db/migrations/001_init.sql`](../../db/migrations/001_init.sql)

Optional: regenerate an EER view in DataGrip / JetBrains Database tools from the live SQLite-or-Postgres connection and compare with our hand-drawn model.

---

## 10–11. Size / performance / other views

*Optional / n/a for this hand-in.* Quality targets remain those in the SRS (e.g. interactive pages under a few seconds on a normal connection). No separate sizing model is required yet.

---

## Document history

| Date | Change |
| :---- | :---- |
| 2026-11-19 | Initial SAD for Week 7 architecture homework |
