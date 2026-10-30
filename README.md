# ShareLab

Trusted sharing of underused campus equipment and spaces, built to the ShareLab PRD. A zero-dependency Node.js server with SQLite and a single-page frontend. No `npm install` needed.

## Run locally

Needs **Node 22.13 or newer** (uses the built-in `node:sqlite`; Node 22.5 to 22.12 also works, the server relaunches itself with the required flag).

    npm start      # http://localhost:3000
    npm test       # 37 end-to-end checks on a temporary database

## Deploy

**Docker (any host):** `docker build -t sharelab . && docker run -p 3000:3000 -v sharelab-data:/data -e DEMO_PASSWORD=yourpass sharelab`

**Render:** push this folder to a Git repo, then New > Blueprint. `render.yaml` is included (free plan: data resets on redeploy; add a disk mounted at `/data` and use a paid plan to keep data).

**Railway / Fly.io / any Node host:** start command `node --disable-warning=ExperimentalWarning server.js`, Node 22+ (a `Procfile` and `.node-version` are included). Attach a volume and set `DB_PATH` to a file on it.

Health check: `GET /healthz`. The server binds `0.0.0.0:$PORT` and shuts down cleanly on SIGTERM.

### Environment variables (see `.env.example`)

| Variable | Purpose |
|---|---|
| `PORT`, `HOST` | Listen address (default 3000, 0.0.0.0) |
| `DB_PATH` | SQLite file (default `./data/sharelab.db`). If it cannot be opened the server falls back to the temp dir, then memory, and logs a warning instead of crashing |
| `DEMO_PASSWORD` | Password for demo accounts (default `demo1234`) |
| `SEED_DEMO=0` | Start empty with no demo users |
| `TRUST_PROXY=0` | Ignore `X-Forwarded-For` when not behind a proxy |
| `RESEND_API_KEY`, `MAIL_FROM` | Real email delivery (optional) |
| `ANTHROPIC_API_KEY`, `AI_MODEL` | AI assists (optional) |

### Troubleshooting "database error"

- `No such built-in module: node:sqlite` or a Node version message: install Node 22.13+ (or use the Dockerfile).
- `unable to open database file`: the folder is read-only or missing. Set `DB_PATH` to a writable path. The app now falls back automatically and prints a warning.
- Data disappears after redeploy: the host has no persistent disk. Mount one and point `DB_PATH` at it.
- Older database from a previous version: columns and tables are migrated automatically on start.

## Test credentials (password `demo1234`)

| Email | Role |
|---|---|
| student@demo | Student, verified for safety and electronics |
| student2@demo | Student, verified for safety, high usage (hits quota) |
| owner@demo | Verified resource owner |
| manager@demo | Lab manager (reviews, check-in, credentials) |
| admin@demo | Administrator (owner verification, audit, email outbox, flags) |
| owner2@demo | Owner awaiting verification |

## Judge walkthrough (about 8 minutes)

1. `owner2@demo` cannot list. As `admin@demo` open Admin and verify the owner; then list a resource as `owner2@demo`.
2. As `student@demo`, type what you need in the match box (for example "print an enclosure"), open the suggestion, pick a slot, watch the live checklist, submit.
3. As `manager@demo`, approve. As the student, check in with a photo, then return as "Worn" with a photo. Report a Damage incident with evidence.
4. As `owner@demo`, open Incidents, tap Summary draft, then decide with reasoning. Open History on the booking (hash-chained) and Export CSV.
5. Fairness: as `student2@demo`, request the same slot, get waitlisted with a reason and free-slot suggestions; cancel the original and watch the promotion.
6. Credentials: as the student, request verification with an uploaded certificate. As `manager@demo`, view the document, try AI assist, verify.
7. As `admin@demo`: verify the audit chain, see usage flags, the email outbox and automation status.

## Architecture

- `server.js`: HTTP server, SQLite schema and migrations, business rules, role checks, audit chain, background jobs (reminders, overdue), uploads, email queue, AI assists.
- `public/index.html`: single-page UI that renders server state and calls the API.
- All rules run on the server inside one transaction per action, so concurrent requests cannot double-book a slot.

API: `POST /api/login`, `/api/logout`, `GET /api/state` (filtered by role), `/api/precheck`, `/api/verify`, `/healthz`, `POST /api/upload`, `GET /api/file/:id`, `POST /api/match`, `GET /api/ai/summary`, `/api/ai/credential`, `POST /api/act` with `a` = submit, approve, reject, wl, cancel, ci, co, noshow, inc, resolve, credreq, credok, credno, vown, newres, mailpref, readnotif, reset.

## Features beyond the core loop

- **Photos and documents:** condition photos at check-in and return, incident evidence, certificate uploads. Images are resized in the browser, then validated by type, size (700 KB) and file signature. Access is checked per file (owner, manager, admin and the booking's student only).
- **Email:** every notification is queued to an outbox. With `RESEND_API_KEY` it is sent; otherwise it is logged only. Users can opt out.
- **AI assists (optional, advisory):** smart resource matching, incident summary drafts, certificate detail extraction. Without an API key, matching uses a keyword fallback and the others return a plain summary or a manual-review notice. Eligibility, approvals, penalties and credential verification are never decided by AI.
- **Usage flags:** rule-based flags (repeated cancellations, no-shows, late returns, holding most of a resource's upcoming hours) shown to reviewers. Flags never apply penalties.
- **Also:** no-show marking, alternative slot suggestions on conflicts, CSV export of a booking record.

## Security and privacy

scrypt password hashing, random session tokens with expiry, rate limiting on login (per IP and per email) and AI calls, server-side authorisation on every action, parameterised SQL, output escaping, strict CSP and security headers, uploaded files served sandboxed with `nosniff`, users receive only their own records. Demo data is fictional.

## Known limitations

Single institution, no SSO, no QR codes, no extension requests, no payments or deposits, fair-use formula is a simple heuristic, email recipients use the demo addresses unless real emails are set.

## Disclosure

No templates or pre-existing code. Google Fonts load from a CDN. Built with AI assistance (Claude); disclose this in the submission.
