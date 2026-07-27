# Hosting Transfer Guide — Render + Vercel

Practical, step-by-step instructions for moving Orderly's hosting from the team's accounts to the customer's own accounts, recreating the two backend services fresh under the customer's own free-tier Render account (a personal Render workspace can't have members invited — that's a paid Render Team feature — so services are recreated rather than transferred in place).

This is separate from the GitHub repository transfer, which is already complete (`docs/customer-handover.md` §1).

## Prerequisites

- Customer has their own Render account (free tier is fine).
- Customer has their own OpenAI account + API key (do **not** hand over the team's key — billing for AI requests would stay on the team's card otherwise).
- Repository is already accessible to the customer (already done — see `docs/customer-handover.md`).

## Step 1 — Deploy the two backend services on the customer's Render account

Both services deploy the same way as documented in `src/backend/README.md` § "Deploy to Render":

### Recommender service (`src/backend`)
1. Render Dashboard → **New → Web Service** → select the repository.
2. **Root Directory:** leave empty (repository root) — the service imports `src/db/` via a relative path, and Docker needs both directories in the same build context.
3. **Dockerfile Path:** `src/backend/Dockerfile`
4. **Runtime:** Docker · **Instance type:** Free
5. Environment variables — see the table below.
6. Create Web Service. Render will build and deploy; the Dockerfile's `CMD` already runs `alembic upgrade heads` automatically before starting the server, so a fresh database gets its schema created with no manual migration step.
7. **Where to find the public URL:** once the deploy finishes, it's shown at the top of the service's page, directly under its name — looks like `https://<service-name>.onrender.com`. You'll need this for Step 4.

### OCR / upload service (`src/upload-menu-backend`)
Same steps, but:
- **Dockerfile Path:** `src/upload-menu-backend/Dockerfile`
- Only needs `ALLOWED_ORIGINS` (and optionally `TESSERACT_PATH`) — no database, no JWT.

### Environment variables

| Variable | Service | Value |
|---|---|---|
| `DATABASE_URL` | Recommender | New Postgres connection string (Step 2 below) |
| `JWT_SECRET_KEY` | Recommender | **Generate a new one** — `openssl rand -hex 32`. Never reuse the team's old value; anyone with the old value could otherwise forge session tokens even after transfer. |
| `AI_BACKEND` | Recommender | `openai` |
| `OPENAI_API_KEY` | Recommender | Customer's **own** OpenAI key, not the team's |
| `OPENAI_BASE_URL` | Recommender | Optional, has a default |
| `ALLOWED_ORIGINS` | Both | The Vercel frontend URL, once known (Step 4) |
| `TESSERACT_PATH` | OCR | Optional |

## Step 2 — Create a fresh PostgreSQL database

1. Render Dashboard → **New → PostgreSQL** (free tier).
2. **Where to find the connection strings:** open the new database → **Info** (or **Connect**) tab. Both **Internal Database URL** and **External Database URL** are listed there, each behind a "reveal"/copy icon (they contain the password, so Render hides them by default — click to copy, don't just eyeball and retype). Copy the **Internal** one for the recommender service's `DATABASE_URL` (internal is fine since both live in the same Render account), and the **External** one for the data migration in Step 3 (that runs from a machine outside Render's network, so it needs the externally-reachable address).

## Step 3 — Migrate data from the old database

The customer has no access to the team's existing Render account, so this step is split: the **team** exports the old data into a file, and hands over that **file** (never a connection string or password) to whoever performs the restore into the new, customer-owned database.

### 3a. Team side — export (run by whoever has access to the team's Render account)

```bash
# Get OLD_EXTERNAL_DATABASE_URL from the team's Render dashboard:
# the existing Postgres service → Info (or Connect) tab → External Database URL
# (behind a "reveal"/copy icon — copy it, don't retype it by hand)
pg_dump "OLD_EXTERNAL_DATABASE_URL" -F c -f orderly_backup.dump
```

Send `orderly_backup.dump` itself (e.g. as a file attachment) to whoever does Step 3b — not the connection string used to produce it.

### 3b. New-account side — restore (run by whoever has access to the customer's new Render account, from Step 2)

```bash
# Get NEW_EXTERNAL_DATABASE_URL the same way, but from the *new* Postgres
# service created in Step 2 (customer's own Render dashboard → the new
# database → Info/Connect tab → External Database URL)

# Schema already exists from `alembic upgrade heads` on first boot of the
# recommender service (Step 1) — --data-only avoids conflicting with it.
# If restoring before the recommender service's first deploy, there's no
# schema yet, so plain pg_restore (without --data-only) also works.
pg_restore --data-only --no-owner -d "NEW_EXTERNAL_DATABASE_URL" orderly_backup.dump
```

Do not paste either connection string into chat/Slack/etc. — treat them like passwords. `orderly_backup.dump` itself contains real user data (emails, order history) — handle/delete it like any other data export, not a throwaway file.

## Step 4 — Point the frontend at the new backend URLs

The frontend's backend URLs are a single config point (`src/new-frontend/src/config.js`), overridable via build-time environment variables — no source code changes needed:

| Vercel environment variable | Value |
|---|---|
| `REACT_APP_API_URL` | New recommender service URL, e.g. `https://<new-name>.onrender.com` |
| `REACT_APP_UPLOAD_URL` | New OCR service URL + `/upload-menu`, e.g. `https://<new-name>.onrender.com/upload-menu` |

Set these in the Vercel project (Settings → Environment Variables) — either on the team's Vercel project before transferring it, or on the customer's own Vercel project if they're hosting the frontend themselves too. Redeploy after setting them (env var changes require a new build to take effect, since Create React App bakes them in at build time — a saved env var alone does **not** update an already-built deployment).

**Where to find the frontend's own live URL** (to put in the two backend services' `ALLOWED_ORIGINS` above): Vercel project's **Overview** page, shown right under the project name — looks like `https://<project-name>.vercel.app`. Also repeated at the top of every entry under the **Deployments** tab.

## Step 5 — Smoke test

On the new URLs, end to end:
1. Register a new account, log in.
2. Set preferences, upload a menu (or skip to the sample menu).
3. Get a recommendation, click "I'll order it".
4. Open **History** — the order should appear and survive a page reload.
5. Click **Dislike** on it — reload again, it should still show "Disliked ✓" (requires the fix from issue #395, once merged).

## After transfer

- Update `docs/customer-handover.md` §1 (who owns what) and §9 (remaining actions) to reflect the new hosting.
- The team can decommission the old Render services once the customer confirms the new ones work.
