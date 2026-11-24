# Contributing

These rules come from the project build plan (section "GitHub workflow rules"). They are the
same in the backend and frontend repos. Shared issue and PR templates live in the shared
`.github` repository and apply here too.

**Repo language is English**: code, commits, PRs, issues and docs. A foreign employer must be
able to read everything. User-facing interface text is Uzbek; that is product content, not
repo language.

## Branches

Nobody pushes directly to `dev`, `staging` or `main`, including the owner.

| Branch | Purpose | Created from | Merges into | Merge method |
| --- | --- | --- | --- | --- |
| `main` | Code in production | - | only via PR from `staging` | merge commit |
| `staging` | Release candidate | - | only via PR from `dev` | merge commit |
| `dev` | Integration | - | receives task branch PRs | squash |
| `feat/…`, `fix/…`, `docs/…`, `chore/…` | One task | `dev` | `dev` | squash |
| `hotfix/…` | Urgent production bug | `main` | `main`, then `main` -> `dev` | squash |

A task branch is squashed into `dev`, so one task is one commit on `dev`. `dev` -> `staging`
and `staging` -> `main` use a normal merge commit; squashing there makes the branch histories
diverge and produces false conflicts in later PRs.

## The path of one task

Every change follows issue -> branch -> PR -> CI -> merge.

1. Open an issue from the task template: goal, acceptance criteria, milestone, labels. The task
   ID is the one in the plan, e.g. `M7-03`.
2. Branch from an up-to-date `dev`, named `<type>/<task-id>-<short-slug>`:
   ```
   git switch dev && git pull
   git switch -c feat/M7-03-auth-pages
   ```
3. Make small commits in Conventional Commits format (below).
4. Push after the first commit and open a **draft PR**. Title in commit format with the task
   ID, e.g. `feat(auth): login and register pages (M7-03)`. Body contains `Closes #<issue>`.
5. Fill in the PR template checklist and read the whole diff yourself.
6. When CI is green, squash merge and delete the branch.
7. When the milestone is done: PR `dev` -> `staging`, run the demo scenario on staging, then
   PR `staging` -> `main` and tag.

## Commits

- Format: `<type>(<scope>): <imperative short description>`, e.g.
  `fix(practice): show result after a repeated answer`.
- Types: `feat`, `fix`, `refactor`, `perf`, `test`, `docs`, `ci`, `chore`.
- Subject line at most **72 characters**. The body explains **why**, not what.
- The format is checked by `commitizen` in a pre-commit hook and in CI.
- Scope (proposal, not in the plan): the screen or area the change touches, matching the
  `area:` labels, e.g. `auth`, `practice`, `plan`, `stats`, `explanation`, `api-client`.

## Pull requests

- One PR closes one task and changes **at most 400 lines**. Lock files and generated code (for
  example types generated from `openapi.json`) do not count. If it is larger, split the task.
- The description has four parts:
  1. **What** changed.
  2. **Why**.
  3. **How it was tested**.
  4. **How to roll it back**.
- If the PR moves the pinned `openapi.json` to a new backend release, link that release and
  show the schema diff in the PR.
- All review comments must be resolved before merge.

## Tests

Until M7 there is no code, so there are no tests. From M7, the plan's general test rules
(section "Test strategy", rules) apply here where they fit a frontend:

- A test name states the behaviour, e.g. `shows_retry_time_after_429`.
- Arrange -> Act -> Assert; one test fails for one reason.
- Fake only the outer boundary (the backend API, from its pinned `openapi.json`), not your own
  code.
- Time and randomness are passed in, so a test gives the same result every run.
- Every bug fix starts with a test that fails, then the fix.
- No meaningless tests written to raise coverage.

The backend-specific rules (database transactions, `polyfactory`, `FakeLLMProvider`, the
`mastery`/`planner` coverage floor) do not apply. From **M7-09**, a Playwright end-to-end
scenario (register -> diagnostic -> plan -> practice -> explanation) runs in CI on every PR.
A frontend coverage floor is not set by the plan: **open question** for the owner.

## Cross-repo changes

A feature that needs a new or changed API is two PRs, always in this order:

1. **Backend PR first.** It adds the field or endpoint without breaking the existing one, and
   ships in a backend release that publishes `openapi.json`.
2. **Frontend PR second.** It moves the pinned `openapi.json` to that release, regenerates the
   types and uses the new API.

The frontend never codes against an endpoint that is not in a released `openapi.json`. Which
backend and frontend image tags run together is recorded in the infra repo's
`envs/<env>/versions.env`.

## Branch protection (GitHub rulesets)

- On `dev`, `staging` and `main`: PR required, force push and deletion forbidden.
- All PR conversations must be resolved.
- Required approvals: 0 (GitHub does not let you approve your own PR); the PR checklist and
  strict CI take its place.
- Required status checks in the plan: `lint`, `typecheck`, `test`, `migrations`, `build`.
  For this repo:
  - **Until M7** there is no application code, so there is nothing to lint, type-check or
    build. Which checks are required on a documentation-only repo (for example only the
    commit-message check) is an **open question** for the owner.
  - **M7-01** adds the frontend `lint`, `typecheck` and `build` checks.
  - **M7-09** adds the Playwright end-to-end scenario to CI.
  - `migrations` is a backend check and does not apply here.

## Issues, labels, board

- GitHub Milestones M0-M13; every issue belongs to exactly one milestone. Frontend work is in
  M7 (later milestones such as M8 and M12 add frontend tasks of their own).
- Labels: `type:feat`, `type:fix`, `type:docs`, `type:chore`; `area:…`; `prio:high`.
- Project board: Backlog -> Ready -> In progress -> Review -> Done. At most 2 tasks in
  progress at once.
- A new idea becomes a backlog issue; it is not added to the current milestone.

## Releases

- SemVer. Before v1.0.0: `v0.<milestone>.<patch>`, e.g. `v0.3.0` when M3 is done.
- The changelog is generated from commits (`git-cliff` or `release-please`).
- Every tag gets a GitHub Release: what was added, whether a migration is involved, and how
  to roll back.
- Proposal (not in the plan): a frontend release also names the backend release whose
  `openapi.json` it was built against, so the pair in `versions.env` is easy to check.
