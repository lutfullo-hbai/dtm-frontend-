# DTM AI Tutor — Frontend

Repository: `dtm-frontend-`. Web client for the DTM AI tutor: a personal tutor for DTM applicants that finds knowledge
gaps, builds a daily study plan and explains mistakes in Uzbek. In the build plan this repo is
called `dtm-tutor-frontend`.

## Status

**Documentation only until milestone M7.** The frontend was deliberately deferred; until M7
the API is demonstrated through Swagger and the backend's API tests. There is no application
code, package manifest or CI build in this repo yet.

M7 ("Frontend MVP") delivers a phone-browser client for the full student path: sign-up,
diagnostic test, daily plan, practice, explanation. The stack is chosen by an ADR at the start
of M7; the current recommendation is listed in [docs/frontend-plan.md](docs/frontend-plan.md).

## How this repo relates to the others

The project is split into three repositories, each built, tested and versioned on its own.

| Repo | Contains | Produces |
| --- | --- | --- |
| backend | FastAPI app, migrations, content pipeline, ADRs | Docker image and an `openapi.json` with every release |
| frontend (this repo) | Web app and browser tests (from M7) | Docker image |
| infra | Compose files, Caddy, monitoring, deploy workflow | Deployments |

Coordination rules:

- **Backend first.** A feature that touches both repos lands as a backend PR that adds the
  field or endpoint without breaking the old one, then a frontend PR that uses it.
- **The API is known from a file.** The frontend pins an exact backend release's
  `openapi.json` and generates its types from it.
- **Compatible versions are recorded in infra.** `envs/<env>/versions.env` pins the backend
  and frontend image tags; a deploy is a PR that changes that file, a rollback is its revert.

## Documents

- [CONTRIBUTING.md](CONTRIBUTING.md): branches, commits, PRs and checks for this repo.
- [docs/frontend-plan.md](docs/frontend-plan.md): M7 readiness: proposed stack, API
  consumption contract, per-screen API requirements, open questions for the backend and owner.
