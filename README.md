# RealNews

RealNews tracks how U.S. senators and representatives have voted over the last 4 years, with official summaries, related news and personal follows. It was created at a hackathon by a 3-person team.

Project scope, architecture and rules are in [AGENTS.md](AGENTS.md).

## Features

- **Explore by state:** a U.S. map with Senate and House views, including clickable congressional district maps.
- **Member and vote pages:** each lawmaker's voting record, and how every member voted on a roll call.
- **Policy pages:** a Summary built from the official record, the votes in both chambers, and related news.
- **Latest from Congress:** a news feed on the home page.
- **Google sign-in:** follow lawmakers and get a personal homepage with news about them.
- **ZIP-to-district search:** matches are shown as choices to confirm, because a ZIP code can cross district lines.

## Run locally

Use two terminals: start the backend first, then the frontend. Fill in [Settings](#settings) before starting.

### Backend (FastAPI, Python 3.11+)

macOS or Linux:

```bash
cd backend
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # first run only, then fill in the values
uvicorn app.main:app --reload
```

Windows PowerShell:

```powershell
cd backend
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
Copy-Item .env.example .env # first run only, then fill in the values
.\.venv\Scripts\python.exe -m uvicorn app.main:app --reload
```

The API docs are at http://localhost:8000/docs. If you run the backend without `--reload`, restart it after changing code or `backend/.env`.

### Frontend (Next.js)

Create `frontend/.env.local` with:

```
GOOGLE_CLIENT_ID=your-google-oauth-client-id
NEXT_PUBLIC_API_URL=http://localhost:8000
```

Then:

```bash
cd frontend
npm ci
npm run dev
```

Open http://localhost:3000.

Run only one frontend dev server at a time. Two servers in this folder share `frontend/.next` and corrupt each other's build; if pages stop loading, stop the server, delete `frontend/.next` and start it again. Restart the frontend after changing `frontend/.env.local`.

### Quick checks

- http://localhost:8000/api/health shows whether the backend is up.
- http://localhost:8000/api/states/UT/members shows whether member data is loaded and Firestore is reachable.

## Settings

Never commit real values. `backend/.env` and `frontend/.env.local` are ignored by git.

| Variable | File | Purpose | Required? |
|---|---|---|---|
| `FIREBASE_PROJECT_ID`, `FIREBASE_CLIENT_EMAIL`, `FIREBASE_PRIVATE_KEY` | `backend/.env` | Firestore access, from a Firebase service account | Yes. Data pages don't work without them |
| `FRONTEND_ORIGIN` | `backend/.env` | The frontend address allowed to call the API | Optional. Defaults to `http://localhost:3000` |
| `CONGRESS_API_KEY` | `backend/.env` | Official bill summaries and details during data loading | Optional. Without it, policies have basic titles only |
| `NEWS_API_KEY` | `backend/.env` | GNews articles | Optional. Without it, news sections show an unavailable message |
| `GOOGLE_CLIENT_ID` | `backend/.env` and `frontend/.env.local` (same value) | Google sign-in | Optional. Without it, the public site works but sign-in, follows and personal feeds don't |
| `SESSION_SECRET` | `backend/.env` | Signs 30-day sessions. At least 32 characters; try `openssl rand -hex 32` | Optional. Without it, sign-ins last about 1 hour |
| `HUD_API_KEY` | `backend/.env` | ZIP-to-district lookup | Optional. Without it, people choose their state and district by hand |
| `DEEPSEEK_BASE_URL`, `DEEPSEEK_API_KEY`, `DEEPSEEK_MODEL`, `DEEPSEEK_TIMEOUT_SECONDS`, `DEEPSEEK_MAX_CONCURRENT` | `backend/.env` | The AI endpoints, which the site doesn't currently use | Optional. Everything on the site works without them |
| `NEXT_PUBLIC_API_URL` | `frontend/.env.local` | The backend address | Optional. Defaults to `http://localhost:8000` |

### Google sign-in setup

In the same Google Cloud project as Firebase:

1. Create an OAuth client of type **Web application**.
2. Add `http://localhost:3000` and `http://localhost` as **Authorized JavaScript origins**.
3. Add your Google accounts as **test users** on the OAuth consent screen.
4. Put the client ID in both `backend/.env` and `frontend/.env.local`, then restart both servers.

## Data

Run these from `backend/` with its virtual environment active:

```bash
python -m scripts.ingest --dry-run --limit 3              # check the data sources; writes nothing
python -m scripts.ingest --congress 119 --limit 10        # load a small sample
python -m scripts.ingest                                  # full load of the 4-year window
```

**The full load needs the Firebase Blaze plan.** It's roughly 1.1 million Firestore writes, well over the free plan's daily limit. Use `--limit` for testing.

Deploy the Firestore security rules and indexes once from the repository root, using the [Firebase CLI](https://firebase.google.com/docs/cli):

```bash
firebase deploy --only firestore:rules,firestore:indexes
```

The project comes from `.firebaserc`; add `--project your-project-id` to use a different one.

## Current status

- AI summaries are not on the site. The backend AI endpoints remain and need a private DeepSeek or Ollama server. The AI chat character has been removed; it's available in the git history.
- The database may contain only a sample of votes, depending on what has been loaded.
- Ashley Moody and Jon Husted show as former senators, because the source data lists their appointed terms as ending on November 3, 2026.
