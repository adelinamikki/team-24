# Customer Handover

_Last updated: 2026-07-17 (Week 7, Sprint 5)_

## 1. Current Product Status & Handover Scope

- **Repository:** retained by the team with customer given read access, because the team needs to finalize remaining Sprint 5 work and the MVP v3 release.
- **Deployment/hosting:** Deployment remains on the team's account. Customer has been granted viewer access via invite to monitor and manage the deployment. Full account transfer was not done because the work on the product is still being done.
- **Domain / hosting service account:** the team owns it now.
- **CI/CD, monitoring, or other service accounts:** The team own the project until the final presentation.

> "Repo ownership retained by team; customer added as Reader;
> deployment stays on team's account until the final presentation."

## 2. How the Customer Accesses and Uses the Product

- **Production URL:** https://team-24-navy.vercel.app
  (backing services: recommender API at https://team-24.onrender.com,
  upload/OCR API at https://team-24-1.onrender.com — customer doesn't need
  to interact with these directly)

- **Login / access method:** Customer creates an account (email + password)
  to use the product. No credentials are included in this document — see
  §4 (Configuration & Secrets-Handling) for how account/service credentials
  are shared and handled.

- **Link to current product access artifact:** [Live demo](https://team-24-navy.vercel.app)

- **Brief walkthrough of the main user flow:** After signing up and logging
  in, the user The user sets preferences (budget, allergies, dietary
  restrictions), uploads a photo of a restaurant menu, which the system reads
  via OCR. The app returns dish recommendations from that menu.
  Past recommendations are saved to the user's History page for later
  reference.

## 3. Installation or Deployment Instructions

The product is currently hosted by the team (see §1 for ownership details).
If you ever need to deploy your own instance — e.g. after course support
ends — here is a summary of what's involved; full local-development steps
are in [README.md](../README.md#getting-started-local-development).

### Prerequisites
- Python 3.12+
- Node.js 18+ and npm
- A PostgreSQL database (or compatible) for the recommender backend —
  the service will not start without a working `DATABASE_URL`
- `tesseract-ocr` installed on the host running the upload/OCR service
  (`sudo apt-get install tesseract-ocr` on Ubuntu/Debian, `brew install
  tesseract` on macOS)
- (Optional) an OpenAI API key, only needed if running the OpenAI-backed
  recommendation engine rather than the default stub/local-LLM options
- Accounts on your chosen hosting providers for the three services
  (currently: Vercel for the frontend, Render for both backend APIs)

### Key steps
1. Provision a PostgreSQL database and set `DATABASE_URL` for the
   recommender service, then run the database migrations
   (`cd src/db && alembic upgrade head`) to create the required tables.
2. Deploy the two backend services (`src/backend` — recommender,
   `src/upload-menu-backend` — OCR) as separate services, each with its
   Python dependencies installed (`pip install -r requirements.txt`) and
   required environment variables set (see §4).
3. Deploy the frontend (`src/new-frontend`) as a static/Node build, pointing
   `REACT_APP_API_URL` at your deployed recommender API URL and
   `REACT_APP_UPLOAD_URL` at your deployed OCR service's `/upload-menu`
   endpoint (both build-time variables — see `src/new-frontend/src/config.js`).
4. Verify all three services are reachable and the frontend can complete a
   full flow (sign up, upload a menu, get recommendations) end to end.

> For the specific case of moving hosting from the team's Render/Vercel
> accounts to your own (including migrating existing data), see
> [docs/hosting-transfer-guide.md](hosting-transfer-guide.md) — a detailed
> walkthrough rather than this general-purpose summary.

> Note: free-tier hosting (as currently used) may sleep after inactivity,
> adding 5–15s latency to the first request — factor this in when choosing
> a hosting tier.

Full local-development setup (for testing before deploying, including a
SQLite-based quick start) is documented in
[README.md](../README.md#getting-started-local-development).
## 4. Configuration & Secrets-Handling Expectations

Required environment variables (names only — actual values shared via
private channel, see below):

**Recommender backend:**
- `DATABASE_URL` — required; connection string for the PostgreSQL database
- `JWT_SECRET_KEY` — required; signing key for access/refresh tokens
- `AI_BACKEND` — `stub` (default, no external calls) / `openai` / `lmstudio`
- `OPENAI_API_KEY` — required only if `AI_BACKEND=openai`
- `OPENAI_BASE_URL`, `OPENAI_MODEL` — optional, have defaults
- `ALLOWED_ORIGINS` — optional CORS setting
- `NEGATION_LLM_EXTRACTION` — optional, `true`/`false`

**Upload/OCR backend:**
- `TESSERACT_PATH` — optional
- `ALLOWED_ORIGINS` — optional

Current values are configured directly in the Render dashboard for each
service and are not committed to the repository. [Customer contact /
instructor] can request current values through [private channel].

> Note: when transferring service ownership, generate a new
> `JWT_SECRET_KEY` value rather than reusing the team's existing one, so
> that previously issued tokens and any prior access are invalidated.

## 5. Operational Notes for Normal Use

### Periodic tasks
- **Database migrations:** when the product schema changes (tracked via
  Alembic in `src/db/`), someone with access to the recommender service's
  environment needs to run `alembic upgrade head` against the production
  database after deploying new code. This is currently a manual step, not
  automated in CI/CD.
- **Backups:** there is currently no automated backup solution for the
  production PostgreSQL database. This is a known gap — see §7.
- **Dependency/service renewals:** no paid subscriptions are currently in
  use (all three services run on free tiers); no renewal action is needed,
  but free-tier limits should be monitored if usage grows (see §7).

### Manual steps not yet automated
- Database migrations (above) are applied manually rather than as part of
  the deploy pipeline.
- There is no automated redeploy trigger beyond Vercel's auto-deploy from
  `main` for the frontend; the two backend services on Render must be
  redeployed manually (or via Render's own auto-deploy setting, if enabled)
  when backend code changes.

### Monitoring / logs
- Application logs are available through each service's hosting dashboard
  (Render dashboard for both backend APIs, Vercel dashboard for the
  frontend). Access to these dashboards depends on the account-ownership
  arrangement described in §1.
- There is no dedicated uptime/error monitoring or alerting service
  configured; issues are currently caught by manual checking or user
  reports rather than automated alerts. This is a known limitation — see §7.
- CI (`.gitlab-ci.yml`) runs a link-checker (`lychee`) against `main` to
  catch broken documentation links, but this does not cover application
  health or runtime errors.

## 6. Troubleshooting & Support

### Common issues and fixes

| Symptom | Cause / Fix |
|---|---|
| First request after inactivity takes 5–15 seconds | Backend services run on Render's free tier, which sleeps after inactivity. This is expected — just wait for the first response. |
| Menu photo doesn't produce recommendations / OCR fails | Ensure the photo is clear, well-lit, and the menu text is readable. Very low-quality photos may not parse correctly — see known limitations in §7. |
|

### Contact for support
For issues beyond the above, contact Daria Gorshkova (dayeon761, d.gorshkova@innopolis.university).
The team provides active support until end of course grading, 31.07.2026. 
After this date, the customer will need to rely on self-service troubleshooting and the issue
tracker below.

### Reporting problems
Problems can be reported directly via the project's GitHub Issues tracker:
[[link to your GitHub Issues page](https://github.com/Orderly-Team24/team-24/issues)]. This remains available as a reporting
channel independent of the active-support period above.

## 7. Known Limitations, Unfinished Areas, and Risks

### Product limitations
- **3+ columns** The service doesnt handle more than 3 colums in menus
- **Open-ended mood requests** (e.g. "something warm and comforting") are
  best-effort — the AI takes them into account but matching quality is not
  guaranteed the way allergy/exclusion filtering is.
- The frontend has **no automated test suite**; UI changes are verified
  manually rather than by CI-enforced tests
### Security / operational risks
- **No password reset / "forgot password" flow** — a user who loses their
  password currently has no self-service way to regain account access.
- **No email verification** on signup — accounts are usable immediately
  with an unverified email address.
- **No rate limiting** on the API endpoints, including login/signup —
  a risk for abuse (credential stuffing, spam accounts) if usage grows
  beyond the current small scale.

## 8. Current Handover Status

**Handover level reached:** `Ready for independent use` 

**Customer-confirmation status:** 'Accepted with follow-up items'

The product works correctly.

## 9. Remaining Actions

| Action | Owner (team / customer / external) | Blocks full transition? |
|---|---|---|
| Fix order history bug | Team | Yes |
| Re-record product demo video (showing current/fixed functionality) | Team | Yes |
| Transfer repository ownership to customer | Team | Yes |
| Send customer a written summary of what was done + demo video | Team | Yes |
| Request customer's written confirmation of final acceptance | Team → Customer | Yes |

**Customer-confirmation status:** `Accepted with follow-up items`

The customer confirmed the handover level as `Ready for independent use`
after trying the product directly during the Week 7 meeting. However, full
acceptance is conditional on the following follow-up items:

- **What follow-up items remain:** the order history feature needs to be
  fixed, the product demo video needs to be re-recorded to reflect the
  current (fixed) state of the product, and repository ownership needs to
  be transferred to the customer.
- **Whose side the blocker is on:** currently on the **team's side** — all
  three follow-up items are engineering/administrative tasks the team
  still needs to complete. Once done, the blocker shifts to the
  **customer's side**: the team will send a written summary of what was
  done plus the updated demo video, and ask the customer to confirm
  acceptance in writing.
- **What evidence of readiness/usefulness was still obtained:** the
  customer independently tried the product during the Week 7 meeting and
  explicitly confirmed the handover level as `Ready for independent use`;
  the only reason full acceptance was withheld was the specific,
  named follow-up items above — not a broader concern about the product's
  overall readiness.
- **What is still needed for full acceptance:** the team must (1) fix the
  order history bug, (2) re-record and share the demo video, (3) transfer
  repository ownership, (4) send the customer a written summary of the
  completed work together with the video, and (5) obtain the customer's
  written confirmation of acceptance in response.

## 10. Related Documentation

- [README.md](../README.md)
- [CONTRIBUTING.md](../CONTRIBUTING.md)
- [AGENTS.md](../AGENTS.md)
- [Hosted documentation site](<link>)
- [docs/user-acceptance-tests.md](./user-acceptance-tests.md)
- [docs/roadmap.md](./roadmap.md)
