# Frontend plan: M7 readiness

Status: draft, written before M7 so the backend milestones (M2-M6) can deliver what the frontend
will need. Nothing here is a final decision. The stack is decided by an ADR at the start of M7,
and screens are designed together with the owner during M7.

## How to read this document

Every requirement cites its source in the build plan, for example `[M3-06]` for a milestone
task or `[API rules]` for the plan's "API design rules" section. Anything else is labelled:

- **Proposal**: a suggested shape. Not agreed; the backend issue decides.
- **OPEN QUESTION (OQ-n)**: something the plan does not specify. Endpoint paths are never
  invented here; where the plan names none, it is an open question.

All open questions are collected in [section 6](#6-open-questions).

## 1. Scope of M7

Source: plan, M7 "Frontend MVP". A student completes the whole path in a phone browser:
sign-up, diagnostic, plan, practice, explanation. M7 completes the MVP.

| ID | Task | Backend dependency |
| --- | --- | --- |
| M7-01 | ADR and skeleton; frontend lint, typecheck, build in CI | none |
| M7-02 | API client generated from `openapi.json`; data fetching | every backend release |
| M7-03 | Login and register pages; access token in memory; auto refresh on 401 | M2 |
| M7-04 | Question screen (KaTeX, options, result); shared by diagnostic and practice | M3, M4 |
| M7-05 | Home: today's tasks, streak, days until the exam | M5 |
| M7-06 | Weak topics; stats page; Telegram reminders setting | M4, M5 |
| M7-07 | Explanation: SSE stream, waiting state, "useful" / "wrong" buttons | M6 |
| M7-08 | Loading, empty and error states on every screen; Uzbek error text by `code` | all |
| M7-09 | Playwright E2E: register -> diagnostic -> plan -> practice -> explanation | all |
| M7-10 | Frontend container in compose, Caddy routes, Lighthouse mobile check | infra |

Out of MVP scope [Goal and scope]: mobile app, payments, parent/teacher accounts, free chat,
a second subject. i18n arrives in M12 [M12-02].

## 2. Proposed stack (pending ADR at M7-01)

The plan lists these as **its own recommendation** (M7 introduction), to be confirmed or
replaced by the M7-01 ADR. None of it is installed or decided yet.

| Concern | Recommended | Source |
| --- | --- | --- |
| Framework | Next.js | [M7-01] |
| Language | TypeScript | [M7-01] |
| Styling | Tailwind CSS | [M7-01] |
| Components | shadcn/ui | [M7-01] |
| API types and client | `openapi-typescript` | [M7-02] |
| Data fetching and cache | TanStack Query | [M7-02] |
| Math rendering | KaTeX | [M7-04] |
| E2E tests | Playwright | [M7-09] |

Points the ADR should settle (proposal):

- **Rendering model.** The access token lives only in memory [M7-03], so the Next.js server
  cannot fetch the student's data on their behalf. Authenticated screens are therefore
  rendered and fetched on the client. The ADR should state whether server rendering is used
  at all (for example only for public pages) or whether a static export is enough.
- **Package manager, Node version, lint/format tools**: not in the plan.
- **Current maintenance status** of each library, checked at ADR time, not recalled.

## 3. API consumption contract

### 3.1 Where the schema comes from

- The backend publishes `openapi.json` with every release [Repos]. In M0 it is exported to
  `docs/api/openapi.json` in the backend repo by a deterministic script (backend M0 contract).
- The frontend pins an **exact backend release** and takes the file from that release
  [Repos, M7-02]. It never follows `dev` or `main` of the backend.
- Types (and a client) are generated from the pinned file with `openapi-typescript` [M7-02].
- When the pinned version is bumped and the schema no longer matches the frontend's usage,
  **the frontend build fails** [M7-02].
- Generated code does not count toward the 400-line PR limit [PR rules].

Proposal for the mechanics (to be decided in M7-02):

- Commit the pinned file plus a small record of its backend tag and checksum, so a bump is a
  reviewable diff and CI can verify the file matches the release.
- CI regenerates the types from the pinned file and fails if they differ from what is
  committed, then `typecheck` fails on any code that uses a removed or changed field.

### 3.2 Rules the frontend relies on

| Rule | Frontend consequence | Source |
| --- | --- | --- |
| Prefix `/api/v1`; breaking changes only in `/api/v2` | Base path is fixed per major version | [API rules] |
| JSON fields `snake_case`; times ISO 8601 UTC; IDs UUID strings | Generated types use snake_case as is (no renaming layer). **Proposal:** times are shown in the student's local zone | [API rules] |
| Errors are RFC 9457 `application/problem+json` with stable `code` and `request_id` | Error text is chosen by `code`, never by `detail` or `title` | [API rules, M7-08] |
| 422 `validation_error` carries `errors: [{loc, message, type}]` | Field errors are mapped to form fields by `loc` | backend M0 contract |
| 401 = no or invalid token; 403 = no permission; 404 also for another user's resource | 401 triggers refresh; 404 shows "not found", never "forbidden" | [API rules] |
| 409 = state conflict (e.g. session already finished) | Treated as a recoverable state. **Proposal:** the screen reloads its data | [API rules, M3-06, M3-07] |
| 429 with `Retry-After` | The UI shows when the action can be retried | [API rules, M2-07] |
| 202 for background work, with a status URL | Explanation requests show a waiting state | [API rules, M6-04] |
| `X-Request-ID` on every response; `request_id` in error bodies | Error screens show the request ID so a report can be traced in logs | [API rules] |
| Repeatable-harm POSTs (e.g. answers) need `Idempotency-Key`; replay returns the stored response for 24 h | One key is generated per answer attempt and reused on every retry of that attempt | [API rules, M3-06] |
| Cursor pagination `?limit=&cursor=`, response `{items, next_cursor}`, `limit` max 100 | Lists use "load more" / infinite scroll, never page numbers | [API rules] |
| Every endpoint documents `response_model`, error responses and an example | Generated types include error bodies; examples drive mocks for tests | [API rules] |

## 4. What each screen needs from the API

Each row lists the endpoints the plan names. Where the plan names none, the need is stated as
an open question.

### 4.1 Auth (M7-03) <- backend M2

Endpoints named in the plan [M2-01]: `POST /auth/register`, `POST /auth/login`,
`POST /auth/refresh`, `POST /auth/logout`, `GET /me`, `GET /me/sessions`,
`DELETE /me/sessions/{id}`.

Known behaviour:

- Access token 15 min (`sub`, `role`, `jti`); refresh token 30 days [M2-04].
- Access token sent as `Authorization: Bearer`; refresh token only in an `httpOnly`,
  `Secure`, `SameSite=Strict` cookie [API rules]. The frontend never sees the refresh token.
- Access token kept in memory only; automatic refresh on 401 [M7-03].
- Every refresh rotates the token; reuse of a revoked token revokes the whole family [M2-05].
- Logout from one device and from all devices [M2-06].
- Register returns the same response whether or not the email is taken [M2-03].
- Login is rate limited to 5 per minute per IP and per email, 429 + `Retry-After` [M2-07].

Frontend flow (proposal):

1. On app start there is no access token in memory, so the app calls `POST /auth/refresh`
   once; success means "signed in", 401 means "show login".
2. On a 401 from any call: one refresh, then retry the original request once. Concurrent 401s
   share a single in-flight refresh. If refresh fails, clear state and go to login.
3. Register success screen must not reveal whether the email already existed [M2-03].

Requirements and questions for the backend:

- **OQ-1 Multi-tab refresh race.** With rotation and reuse detection [M2-05], two tabs (or a
  reload during a refresh) can send the same refresh cookie at once; the second request uses
  a revoked token and the whole family is revoked, logging the student out everywhere. Does
  the backend allow a short grace window for the immediately previous token, or must the
  frontend serialise refreshes across tabs (proposal: Web Locks API / BroadcastChannel)?
  Either way it must be written in the M2 contract.
- **OQ-2 Cookie scope and topology.** `SameSite=Strict` only works if the frontend and API
  are the same site. Proposal: serve both from one origin behind Caddy (`/api/*` routed to the
  backend), which needs no CORS and keeps the cookie first-party; scope the cookie with
  `Path=/api/v1/auth`. If separate subdomains are chosen instead, CORS with credentials and a
  cookie `Domain` decision are needed. Decide with infra before M7-10 (Cloudflare Tunnel ->
  Caddy [Infrastructure]); staging and production need distinct cookie names or hosts so one
  never overwrites the other.
- **OQ-3 Register response.** Does register sign the student in (return an access token and
  set the cookie) or require a separate login? Is there email verification? Not in the plan.
- **OQ-4 Login response shape**: field name for the access token and whether `expires_in` is
  returned (lets the client refresh just before expiry instead of waiting for a 401).
- **OQ-5 CSRF on cookie-authenticated endpoints** (`/auth/refresh`, `/auth/logout`):
  `SameSite=Strict` is the plan's mitigation; confirm no extra token is expected.
- Error codes needed (names are the backend's to choose): invalid credentials, rate limited,
  weak or invalid password (with the password rules exposed so the form mirrors them), refresh
  token expired, refresh token reused / family revoked.

### 4.2 Question screen (M7-04) <- backend M3 (practice), M4 (diagnostic)

One component serves both diagnostic and practice [M7-04]: question card, formulas with KaTeX,
options, result after answering.

Endpoints named in the plan:

- Practice [M3-05..M3-07]: `POST /practice-sessions` (topic, question count) -> 201, questions
  returned **without** the correct answer; `GET /practice-sessions/{id}`;
  `POST /practice-sessions/{id}/answers` with `Idempotency-Key`, returns the result and the
  correct option, second answer to the same question -> 409;
  `POST /practice-sessions/{id}/finish` -> summary (correct count, time spent), repeat -> 409.
- Diagnostic [M4-05]: `POST /diagnostics`, `POST /diagnostics/{id}/answers`. Adaptive,
  25-30 questions [M4].
- The correct answer never appears before the student answers [M3-10].

Requirements and questions for the backend:

- **OQ-6 One question schema.** Proposal: practice and diagnostic return the same question and
  option schema (one `QuestionRead`-style model), so the shared component has one type.
- **OQ-7 Formula encoding.** How is math embedded in question and option text: LaTeX with
  which delimiters (`$…$`, `\(…\)`), or a structured format? Can text contain images or
  tables? KaTeX rendering depends on this.
- **OQ-8 Diagnostic sequencing.** Because the diagnostic is adaptive, does the answer response
  include the next question (proposal: yes, to avoid a second round trip per question)? How
  does the client know the diagnostic has ended, and does the diagnostic reveal the correct
  option after each answer like practice does?
- **OQ-9 Idempotency on diagnostic answers.** The plan requires `Idempotency-Key` for practice
  answers [M3-06]; confirm the same for `POST /diagnostics/{id}/answers`.
- **OQ-10 Resuming.** After a reload or lost connection, can the client fetch the current
  position (which questions are answered) for both a practice session (`GET
  /practice-sessions/{id}` [M3-05] suggests yes) and a diagnostic (no read endpoint named)?
  How does the client find an unfinished session or diagnostic to resume?
- Error codes needed: session already finished (`session_already_finished` is the plan's
  example [API rules]), question already answered, session not found.

### 4.3 Today's plan / home (M7-05) <- backend M5

Shows today's tasks, the streak, and days until the exam [M7-05].

Endpoints named in the plan: `GET /me/plan/today`, `POST /me/plan/tasks/{id}/complete`,
`GET /me/progress` (streak, weekly result) [M5-07]; `PUT /me/goal` (exam date, target score,
minutes per day) [M5-01].

Requirements and questions for the backend:

- **OQ-11 Reading the goal.** Only `PUT /me/goal` is named. Days until the exam needs the exam
  date: is there a `GET`, or does `GET /me/plan/today` include it?
- **OQ-12 Home in one request.** The home screen needs plan + streak + days left. Proposal:
  `GET /me/plan/today` includes the streak and days left, or the backend confirms three
  parallel calls are acceptable. Either is fine; it should be decided in the M5 contract.
- **OQ-13 Task -> practice.** What does a plan task carry (type: new topic / review / mixed
  test [M5-02]; topic id and name; question count; minutes)? Does starting a task create a
  practice session with the right parameters, and is the task completed automatically when
  that session finishes, or only by `POST /me/plan/tasks/{id}/complete`?
- **OQ-14 "Today" in which time zone?** Plans, streaks and "days left" depend on where the day
  boundary is. Server times are UTC [API rules]; the students are in Uzbekistan (UTC+5). Does
  the backend use the student's zone (stored per user?) or a fixed zone?
- **OQ-15 First run.** Before the diagnostic and goal exist, what does `GET /me/plan/today`
  return (empty plan, 404, or a specific `code`)? The UI needs a stable signal to send a new
  student to the diagnostic or goal setup.

### 4.4 Weak topics and stats (M7-06) <- backend M4, M5

Weak topics with mastery level, root causes first; stats page with growth line, accuracy per
topic and an estimated score **range** [M7-06].

Endpoints named in the plan: `GET /me/weak-topics` (ordered by root cause, base topics first)
and `GET /me/mastery` (graph and values) [M4-06]; `GET /me/stats/overview`,
`GET /me/stats/topics`, `GET /me/stats/timeline`, score returned as a range [M5-11]; topic
metadata from `GET /subjects`, `GET /subjects/{id}/topics`, `GET /topics/{id}` with `ETag`
[M3-04].

Requirements and questions for the backend:

- **OQ-16 Display names inline.** Proposal: weak-topic and stats items include the topic name
  (and subject), so the screen does not call `GET /topics/{id}` per row (client-side N+1).
- **OQ-17 Mastery scale.** The model is Elo-style [M4-01], an unbounded number. What should a
  student see? Proposal: the API also returns a normalised value (0-1) or a level band, so the
  meaning of the number stays in the backend.
- **OQ-18 Timeline parameters**: range (last 7 / 30 days?), granularity, and whether days with
  no snapshot are returned as gaps.
- **OQ-19 Pagination.** Are `weak-topics` and `stats/topics` cursor-paginated lists
  `{items, next_cursor}` [API rules], or bounded lists returned whole? One subject's topic
  graph is small; stating it in the contract avoids guessing.

### 4.5 Telegram reminders setting (M7-06) <- backend M5

Endpoints named in the plan: `PUT /me/notifications/preferences` (channel, preferred time,
quiet hours, max per day; any type can be turned off) [M5-12]. Linking: the app gives a bot
link with a one-time token; the bot binds `chat_id` when the student presses `/start`
[M5-13]. Default cap: 2 reminders per day [M5].

Requirements and questions for the backend:

- **OQ-20 Link endpoint.** No path is named for issuing the one-time bot link. What is it, how
  long is the token valid, and what does it return (a `t.me` deep link)?
- **OQ-21 Link status.** How does the app learn that linking succeeded (poll a status field,
  e.g. on `GET /me`), and how does the student unlink?
- **OQ-22 Reading preferences.** Only `PUT` is named; the settings form needs the current
  values. Is there a `GET /me/notifications/preferences`? In which time zone are "preferred
  time" and quiet hours expressed (see OQ-14)?

### 4.6 Explanation via SSE (M7-07) <- backend M6

Show the SSE stream, a waiting state, "useful" and "wrong" buttons [M7-07].

Known behaviour: the request goes to a worker queue; the API returns 202 and a status URL
[M6-04]; if the queue is full the student sees an estimated wait [M6-04];
`GET /explanations/{id}/stream` is Server-Sent Events, and a cached explanation is returned in
full at once [M6-05]; the explanation ends with its source [M6-02];
`POST /explanations/{id}/feedback` with "useful" or "wrong", and "wrong" removes it from the
cache [M6-08]; daily limit per student [M6-10]; if Ollama is down the app keeps working and a
clear message replaces the explanation [M6].

Requirements and questions for the backend:

- **OQ-23 Requesting an explanation.** No path is named for the request that returns 202.
  What creates an explanation (keyed by answer id? by question + chosen option?), and what
  does the status URL return?
- **OQ-24 Auth on the stream.** The browser `EventSource` API cannot send an `Authorization`
  header, and the access token is in memory only [M7-03]. Proposal: the frontend reads the
  stream with `fetch` and a streaming body parser, sending the Bearer header as usual, so no
  token goes into a URL (URLs end up in logs). The backend should confirm the stream endpoint
  accepts Bearer auth like every other endpoint.
- **OQ-25 Event format.** Proposal, to be fixed in the M6 contract:
  `event: queued` (position or estimated wait), `event: delta` (text chunk), `event: done`
  (explanation id, source), `event: error` (a problem body with `code`). Also: is the text
  plain, Markdown, or Markdown with LaTeX (same delimiters as OQ-7)? Is the source a
  structured field or part of the text?
- **OQ-26 Proxy behaviour.** The stream passes through Cloudflare Tunnel and Caddy
  [Infrastructure]. Proxies may buffer or close idle streams. Proposal: the backend sends a
  heartbeat comment every ~15 s while queued, sets `Cache-Control: no-cache`, and infra
  configures Caddy to flush immediately on this route. Needs a check on staging in M6.
- **OQ-27 Reconnect.** If the stream drops, does reconnecting replay from the start (fine for
  short texts) or support `Last-Event-ID`?
- **OQ-28 Feedback body** (field names, allowed values, optional comment?) and whether feedback
  can be changed or sent twice.
- Error codes needed: explanation unavailable (LLM down) [M6], daily limit reached [M6-10],
  queue full (with estimated wait) [M6-04].

### 4.7 States and errors on every screen (M7-08) <- all backend milestones

- Every screen has loading, empty and error states [M7-08].
- Error text is Uzbek and chosen by the API's `code` [M7-08].
- **OQ-29 Error code catalogue.** Proposal: the backend keeps one list of every `code` with its
  status and meaning (for example in `docs/api/` or as an enum in `openapi.json`), updated in
  the same PR that adds a code. The frontend keeps a matching `code` -> Uzbek message table and
  a generic fallback for unknown codes. Codes known today (backend M0 contract):
  `validation_error`, `not_found`, `method_not_allowed`, `http_<status>`, `internal_error`,
  `not_ready`; plan example: `session_already_finished`.
- **OQ-30 Validation messages.** 422 `errors[].message` is presumably English. Proposal: the
  frontend maps `errors[].type` (+ `loc`) to Uzbek text; the backend keeps `type` values stable
  and documents them.

### 4.8 Not needed by M7

- `GET /me/practice-sessions` (history, cursor-paginated) [M3-08]: no M7 screen lists history.
  Whether a history screen is in M7 is a decision for the owner (section 7).
- `GET /me/sessions` / `DELETE /me/sessions/{id}` [M2-01]: no M7 screen lists devices; "log out
  everywhere" may still be wanted in settings (section 7).

## 5. Non-functional targets

| Target | Source |
| --- | --- |
| The full path works on a **360 px wide** phone screen | [M7 done criteria] |
| Lighthouse mobile speed check | [M7-10] |
| Loading, empty and error states on every screen | [M7-08] |
| A person who does not know the project finishes the diagnostic and sees their plan unaided | [M7 done criteria] |
| Playwright E2E scenario green in CI | [M7 done criteria] |
| Uzbek interface and error text | [Goal, M7-08] |
| **Proposal:** WCAG 2.2 AA (contrast 4.5:1 for text, full keyboard path, visible focus, labelled inputs; touch targets at least 24 px per 2.5.8) | not in the plan |
| **Proposal, beyond AA:** 44 px touch targets (the AAA level, WCAG 2.2 criterion 2.5.5) | not in the plan |
| **Proposal:** Core Web Vitals "good" on mobile (LCP <= 2.5 s, INP <= 200 ms, CLS <= 0.1) and a Lighthouse threshold fixed in CI | the plan names the check, not the threshold (OQ-31) |
| **Proposal:** formulas render without layout shift; KaTeX loaded only on screens that show math | not in the plan |
| **Proposal:** a network drop during practice keeps the answered state; retries reuse the same `Idempotency-Key` | follows from [API rules] |

## 6. Open questions

For the backend (answer in the relevant milestone's contract issue):

| ID | Question | Milestone |
| --- | --- | --- |
| OQ-1 | Multi-tab refresh race vs reuse detection: grace window or client-side lock? | M2 |
| OQ-2 | Same origin vs subdomains; cookie `Path`/`Domain`; CORS; staging vs prod cookie isolation | M2 + infra |
| OQ-3 | Does register sign the student in? Email verification? | M2 |
| OQ-4 | Login response fields; `expires_in`? | M2 |
| OQ-5 | Any CSRF token beyond `SameSite=Strict` on refresh/logout? | M2 |
| OQ-6 | One question/option schema for practice and diagnostic | M3/M4 |
| OQ-7 | Formula encoding (LaTeX delimiters), images/tables in questions | M3 |
| OQ-8 | Diagnostic: next question in the answer response? end signal? correct option shown? | M4 |
| OQ-9 | `Idempotency-Key` on diagnostic answers | M4 |
| OQ-10 | Resuming an unfinished session or diagnostic | M3/M4 |
| OQ-11 | How to read the goal (exam date) | M5 |
| OQ-12 | Home screen in one request or three parallel calls | M5 |
| OQ-13 | Plan task fields; how a task starts practice and gets completed | M5 |
| OQ-14 | Time zone that defines "today", streaks and reminder times | M5 |
| OQ-15 | First-run signal from `GET /me/plan/today` | M5 |
| OQ-16 | Topic names inline in weak-topic and stats items | M4/M5 |
| OQ-17 | Mastery scale shown to students (normalised value or band) | M4 |
| OQ-18 | Timeline range, granularity, gaps | M5 |
| OQ-19 | Pagination of weak-topics and stats lists | M4/M5 |
| OQ-20 | Endpoint and lifetime of the Telegram one-time link | M5 |
| OQ-21 | Telegram link status and unlinking | M5 |
| OQ-22 | Reading notification preferences; their time zone | M5 |
| OQ-23 | Endpoint that requests an explanation (202) and its status URL | M6 |
| OQ-24 | Bearer auth on the SSE stream (fetch-based client) | M6 |
| OQ-25 | SSE event names and payloads; text format; source field | M6 |
| OQ-26 | Heartbeats and buffering through Cloudflare Tunnel and Caddy | M6 + infra |
| OQ-27 | Stream reconnect behaviour | M6 |
| OQ-28 | Feedback body and repeat rules | M6 |
| OQ-29 | Error `code` catalogue location and upkeep | M2 onward |
| OQ-30 | Stable 422 `errors[].type` values for Uzbek field messages | M2 onward |

For the owner:

| ID | Question |
| --- | --- |
| OQ-31 | Lighthouse mobile threshold that fails CI (the plan names the check, not the number) |
| OQ-32 | Required status checks on this repo before M7 (documentation only) |
| OQ-33 | Uzbek script for the interface: Latin or Cyrillic |

## 7. Decisions to make with the owner at M7 start

The plan says screens are drawn together with the owner first, then coded [M7]. Agenda:

1. **Confirm or change the stack** (ADR, M7-01), including the rendering model (section 2).
2. **Hosting topology** with infra: same origin or subdomains (OQ-2), decided before M7-10.
3. **Screen list and flow**: sign-up -> diagnostic -> plan -> practice -> explanation. Where
   does the goal (exam date, target score, minutes per day [M5-01]) get entered: before or
   after the diagnostic? Is a practice history screen or a "log out everywhere" setting in
   M7?
4. **Screen sketches** for each M7-03..M7-08 screen, drawn together before coding.
5. **Visual direction and design tokens** (spacing, type scale, colours, radius), proposed by
   the frontend and reviewed by the owner on the sketches.
6. **Interface language details** (OQ-33): Latin or Cyrillic Uzbek script (the plan does not
   say), tone of address, and the wording of error messages per `code`.
7. **Quality bar**: accept WCAG 2.2 AA and the Core Web Vitals targets in section 5, and fix
   the Lighthouse threshold (OQ-31).
8. **How mastery is shown** to a student (OQ-17): numbers, levels or colours plus text.
