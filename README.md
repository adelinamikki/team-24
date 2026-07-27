# Orderly

A web app that recommends dishes based on your preferences, budget, allergies, and the photo of a restaurant menu.

📖 **[Full documentation](https://orderly-team24.github.io/team-24/)** — architecture, testing, quality requirements, and more.

## Live demo

| Service | URL |
|---------|-----|
| Frontend (Vercel, auto-deploy from `main`) | https://team-24-navy.vercel.app |
| Recommender API (Render) | https://team-24.onrender.com |
| Upload / OCR API (Render) | https://team-24-1.onrender.com |

> Free-tier services on Render may take 5–15 s to wake up after inactivity.

## User guide

How to actually use Orderly, step by step:

1. **Create an account.** Go to [`/register`](https://team-24-navy.vercel.app/register), enter an email, username, and password. You're logged in automatically right after.
   - **Already have an account?** Go to [`/login`](https://team-24-navy.vercel.app/login) instead and sign in with your email and password.
2. **Set your food preferences.** On first login you'll be asked for allergies, dislikes, and likes (all optional, plain comma-separated text — e.g. "peanuts, shellfish"). You can change these any time from **Profile**.
3. **Set a budget and (optionally) a menu photo.** On the next step, enter your budget and either upload a photo of a real restaurant menu or skip it to use a sample menu. You can also type what you're in the mood for (e.g. "something for breakfast", "I don't want steak") — the AI takes this into account alongside your saved preferences.
4. **Get a recommendation.** The app shows one recommended dish that fits your budget, respects your allergies/exclusions, and (if you gave one) matches your mood.
   - **"I'll order it"** saves the dish to your order history.
   - **"Another option"** asks for a different dish — it won't repeat one you've already seen in this session.
   - **"End session"** clears the current menu/budget/mood so you can start fresh (you stay logged in).
5. **Check your order history.** Click **History** in the navigation bar to see everything you've ordered. Click **Dislike** on any dish you didn't like — disliked dishes are never recommended again.
6. **Manage your account.** The **Profile** page lets you update preferences or permanently delete your account.

**Things Orderly guarantees:** it will never recommend a dish containing something you've listed as an allergy or explicitly said you don't want (even in free text, e.g. "I don't want steak") — this is enforced in code, not just requested from the AI. It also never recommends a plain drink (water, soda, coffee, etc.) as "the dish."

### Troubleshooting

- **The site seems stuck loading, or the first request times out.** The backend services are on Render's free tier and spin down after inactivity — the first request after a while can take 5–15 s to wake them up. Reload once and try again.
- **The menu photo didn't scan correctly.** Scanning handles single- and multi-column printed menus in several currencies, but not handwritten specials boards or unusual layouts — see [docs/customer-handover.md](docs/customer-handover.md) for the current, up-to-date list of known limitations.
- **Something else looks broken.** Check [docs/customer-handover.md](docs/customer-handover.md) first — most known gaps are already documented there. If it's not listed, open a GitHub issue.

**Things Orderly does its best on, but doesn't guarantee:** open-ended mood requests ("something warm and comforting") and menu-photo scanning for layouts beyond a standard single- or two-column menu (e.g. 3+ columns, uneven column widths, or handwritten specials boards). See [docs/customer-handover.md](docs/customer-handover.md) for the current, up-to-date list of known limitations.

## Project structure

```
team-24/
├── src/
│   ├── backend/              # Recommender service (FastAPI + Python)
│   ├── upload-menu-backend/  # OCR / photo upload service (FastAPI + Tesseract)
|   ├── new-frontend/         # React SPA (deployed via Vercel)
│   └── db/                   # SQLAlchemy models + Alembic migrations (shared DB layer)
├── docs/                     # Design docs, quality requirements, testing strategy
├── reports/                  # Weekly reports and customer notes
├── CHANGELOG.md
└── README.md
```

## Getting started (local development)

### Prerequisites

- Python 3.12+
- Node.js 18+ and npm

### Backend services

The recommender service requires `DATABASE_URL` (PostgreSQL in production, see
[ADR-002](docs/architecture/adr/ADR-002-postgresql-sqlalchemy.md)) — it will not start without it.
For local development, SQLite works fine since it's just a SQLAlchemy connection string:

```bash
# Create and activate a virtualenv
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

# Install dependencies for both backend services
pip install -r src/backend/requirements.txt
pip install -r src/upload-menu-backend/requirements.txt

# Point at a local SQLite file (swap for a real postgres:// URL to match production)
export DATABASE_URL="sqlite:///$(pwd)/local.db"

# Any value works locally — just needs to be set and consistent between runs
export JWT_SECRET_KEY="local-dev-secret"

# Create the tables
cd src/db
alembic upgrade head
cd ../..

# Run the recommender service (http://localhost:8003)
cd src/backend
AI_BACKEND=stub uvicorn main:app --reload --port 8003

# In a separate terminal — run the OCR upload service (http://localhost:8002)
# Requires tesseract-ocr to be installed on the host:
#   Ubuntu/Debian: sudo apt-get install tesseract-ocr
#   macOS:         brew install tesseract
cd src/upload-menu-backend
uvicorn main:app --reload --port 8002
```

### Frontend (React)

```bash
cd src/new-frontend
npm install
REACT_APP_API_URL=http://localhost:8003 REACT_APP_UPLOAD_URL=http://localhost:8002/upload-menu npm start
# Opens http://localhost:3000
```

### Running tests

```bash
# Recommender backend
cd src/backend
pytest tests/ -v --cov=. --cov-report=term-missing

# Upload backend
cd src/upload-menu-backend
pytest tests/ -v --cov=. --cov-report=term-missing
```

## Documentation

- [Contributing](CONTRIBUTING.md) — how to set up, branch, test, and submit a change
- [AGENTS.md](AGENTS.md) — guidance for AI coding agents working in this repo
- [Customer handover](docs/customer-handover.md) — current transition/handover state, access, known limitations
- [Hosting transfer guide](docs/hosting-transfer-guide.md) — step-by-step: moving Render/Vercel hosting to a new account
- [User stories](docs/user-stories.md)
- [Architecture documentation](docs/architecture/README.md) — component, sequence, and deployment views, plus ADRs
- [Development process](docs/development-process.md) — git workflow, PR process, CI pipeline, configuration management
- [Quality requirements](docs/quality-requirements.md)
- [Quality requirement tests](docs/quality-requirement-tests.md)
- [Testing strategy](docs/testing.md)
- [Definition of Done](docs/definition-of-done.md)
- [User acceptance tests](docs/user-acceptance-tests.md)
- [Roadmap](docs/roadmap.md)
- [MVP v0 Report](reports/week2/mvp-v0-report.md)
- [CHANGELOG](CHANGELOG.md)
- [Week 2 report](reports/week2/README.md)
- [Week 3 report](reports/week3/README.md)
- [Week 4 report](reports/week4/README.md)
- [Week 5 report](reports/week5/README.md)
- [Week 6 report](reports/week6/README.md)
- [Week 7 report](reports/week7/README.md)

Hosted (browsable) version of the docs above: https://orderly-team24.github.io/team-24/

## License

[MIT](LICENSE)
