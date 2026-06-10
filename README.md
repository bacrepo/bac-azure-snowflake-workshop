# BAC Azure + Snowflake Workshop

A hands-on teaching workshop for learning **Snowflake** (running on Microsoft Azure).
It is built to take someone from "never touched a cloud data warehouse" to
comfortably loading data, querying it, and managing compute — all through small,
runnable lessons.

> 📖 **Read the lessons here:** the workshop is published as a GitHub Page
> from the [`docs/`](docs/) folder. See `GITHUB_PAGE_URL` in `.env.public`.

---

## What's in this repo

| Folder        | Purpose                                                    | Published? |
| ------------- | --------------------------------------------------------- | ---------- |
| [`docs/`](docs/) | The teaching material — Markdown deployed as a GitHub Page | ✅ Yes      |
| `sql/`        | Internal SQL commands & scripts run during lessons         | ❌ No (gitignored) |
| `python/`     | Internal Python helpers (data gen, automation)             | ❌ No (gitignored) |

The split is deliberate:

- **`docs/` is the only thing students see.** It is the published, polished
  curriculum. Keep it clean, sequential, and copy-paste friendly.
- **`sql/` and `python/` are the instructor's working material.** They stay
  local (ignored by git) so the published page never leaks setup scripts,
  credentials, or work-in-progress.

---

## Getting started (instructor)

1. Copy the environment template and fill in your Snowflake credentials:
   ```bash
   cp .env.public .env
   # then edit .env — add SNOWFLAKE_USER and SNOWFLAKE_PASSWORD
   ```
   `.env` is gitignored; `.env.public` is the shareable template.

2. Lessons reference SQL kept in `sql/` (e.g. `sql/00_test.sql`). Run them in
   the Snowflake web UI (Snowsight) or via SnowSQL.

3. Edit teaching content under `docs/`. Pushing to the default branch
   redeploys the GitHub Page.

---

## Publishing the GitHub Page

Deployment is automated by [`.github/workflows/deploy-pages.yml`](.github/workflows/deploy-pages.yml):
every push to `main` that touches `docs/` builds the site with Jekyll and
publishes it.

**One-time setup:** Settings → Pages → **Source: GitHub Actions**.

> ⚠️ The workflow replaces the classic "Deploy from a branch" mode. Switch the
> Pages source to **GitHub Actions** so the workflow is allowed to deploy —
> otherwise GitHub keeps using the branch deploy and the workflow's deploy step
> will fail.

---

## License

[Strictly for BAC training purposes only.](LICENSE)
