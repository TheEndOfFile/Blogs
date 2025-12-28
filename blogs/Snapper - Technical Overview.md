

### 1) What this application does
Snapper discovers and stores public Instagram profile snapshots and selected post metadata over time for later analysis. It:
- Schedules and runs scrape jobs for Instagram profiles
- Captures profile header and page screenshots, basic profile metrics, bio and links
- Optionally scrapes the first N posts/reels and persists their details
- Maintains historical records (no overwrite) to enable growth analysis
- Exposes a REST API used by a simple, Instagram‑styled frontend

---

### 2) High‑level architecture
- Frontend: Static HTML/CSS/JS served via Nginx
- Backend API: Go (Gin) service exposing REST endpoints
- Scraper: Go service using chromedp (headless Chromium)
- Database: PostgreSQL for durable storage
- Object storage: S3-compatible (for images/screenshots/json artifacts)
- Docs: Swagger (swaggo) generated API documentation
- Orchestration: Docker Compose for dev, with `snapper-api-dev`, `snapper-postgres-dev`, `frontend` (Nginx)

```
Browser  <->  Nginx (static + proxy)  <->  Gin API  <->  PostgreSQL
                                     \->  chromedp (headless scraping)
                                     \->  S3 (artifacts)
```

---

### 3) Key technologies
- Language: Go 1.25
- Web framework: Gin
- Scraping: chromedp (+ anti-detection measures)
- DB: PostgreSQL
- SQL codegen: sqlc
- Logging: slog
- API docs: swaggo/swag
- Frontend: HTML/CSS/Vanilla JS, Chart.js
- Web server: Nginx
- Containers: Docker, Docker Compose

---

### 4) Data model (selected)
- `profiles`: historical snapshots of an IG profile (not unique on username); includes counts, bio, profile picture/screenshot/header screenshot URLs, scraped_at
- `posts`: posts associated with a profile snapshot; includes post_url, type, thumbnail, description, like/view counts, upload_date
- `jobs`: scraping jobs with `status`, `progress`, `options`, `job_type` ("profile" | "posts") and timestamps
- `scraping_logs`: aggregated run logs (success/failure, duration, cookies used)

Historical design: a new row is inserted on every successful scrape; uniqueness on `username` was dropped to preserve history.

---

### 5) Backend API
Base: `/api/v1`

- Health
  - `GET /health` – service liveness

- Jobs
  - `POST /scrape` – create a profile scraping job
  - `POST /scrape/posts` – create a posts‑only scraping job (first N posts)
  - `GET /jobs/{job_id}` – job status/result
  - `GET /jobs` – list jobs (optional status filter)

- Profiles
  - `GET /profiles?limit=&offset=` – recent profile snapshots (ordered by `scraped_at` desc)
  - `GET /profiles/{username}/history?limit=&offset=` – history for a username (desc)

- Posts
  - `GET /posts?limit=&offset=` – all posts with joined profile details
  - `GET /posts/{username}?limit=` – recent posts for a username

Swagger docs: `/swagger/index.html` (generated from annotations and `types.ProfileResponse` / `types.PostResponse`).

---

### 6) Scraping flow
1. Job creation: a client calls `POST /scrape` or `POST /scrape/posts` with `username` and options (e.g., `use_cookies`, `post_count`).
2. Job processing: the job service (DB‑backed in production) updates status/progress and runs scraping asynchronously.
3. Instagram session:
   - chromedp launches headless Chromium
   - Cookies (optional) are loaded from `/app/cookies/ig_cookie.json`
   - Anti‑detection measures and page readiness checks are applied
4. Profile scrape:
   - Navigate to `https://www.instagram.com/{username}/`
   - Capture header and full page screenshots
   - Extract username, follower/following/post counts, bio, links
   - Upload local artifacts to S3 (URLs stored in DB)
   - Insert a new `profiles` record preserving history
5. Posts scrape (if posts job):
   - Scroll and parse first N posts/reels from the profile grid
   - Extract `post_url`, `thumbnail_url`, description (alt), like/view counts (heuristics)
   - Insert `posts` records linked to the latest `profiles` entry for that username
6. Completion/logging:
   - Persist final status, result, and log duration/cookie usage

Resilience: individual failures (e.g., S3 upload or saving posts) are logged but do not fail the entire job if non‑critical.

---

### 7) Frontend
- Served by Nginx as a static site (configured in docker-compose)
- Uses the API via the `/api/v1` routes (proxied by Nginx)
- Key UI features:
  - Recent results: shows latest 6 unique usernames (from `GET /profiles?limit=20`), each card displays header screenshot and metrics
  - History: unique accounts with last scraped time and total scrapes
  - Profile modal: fetches `GET /profiles?limit=20`, filters by username for timeline
    - Growth chart (Chart.js): plots follower and posts counts over time (sorted ascending), tolerant to K/M/B notations and single‑data‑point cases
  - Cards use full header screenshots (object-fit: contain) as requested

---

### 8) Jobs and options
- Job types: `profile` (default) and `posts`
- Options (`ScrapingJobOptions`): `use_cookies`, `timeout`, `post_count`
- DB‑backed job service restarts incomplete jobs on boot and tracks progress via separate `started_at`/`completed_at` updates

---

### 9) Development & local run
- Prereqs: Docker Desktop
- Start stack (dev):
  - `docker-compose -f docker-compose.dev.yml up --build`
  - Services: `snapper-api-dev`, `snapper-postgres-dev`, `frontend` (Nginx)
- Live reload (API): air (installed in Dockerfile.dev)
- SQL codegen: `sqlc generate` (uses `pkg/postgres/sqlc.yaml`)
- Swagger generate: `swag init -g main.go` (outputs to `docs/`)

---

### 10) Configuration
Environment (via `env` / `env.template`):
- Database connection
- S3 credentials/bucket
- Cookie path (hardcoded default `/app/cookies/ig_cookie.json` in job processing)
- Timeouts/ports

---

### 11) Security & notes
- Scraping uses a headless browser and may require valid session cookies; store cookies securely and rotate if needed
- Public‑only content scraping; respect website ToS and rate limits
- Avoids storing secrets in repo; rely on environment variables for credentials

---

### 12) Known limitations / future work
- Post grid parsing is heuristic and may need maintenance when IG changes DOM
- Additional post details (comments, timestamps) could be added via per‑post navigation
- Authentication & user management are out of scope for now
- Pagination/filters in frontend for larger datasets

---

### 13) Key source paths
- API: `internal/api/handlers.go`, `internal/api/router.go`
- Services: `internal/services/*` (job, profile, logging)
- Scraper: `pkg/scraper/instagram_service.go`, `pkg/scraper/extraction_methods.go`
- Repos (sqlc): `internal/repositories/*`, SQL in `db/queries/*.sql`
- Migrations: `db/migrations/*.sql`
- Frontend: `frontend/index.html`, `frontend/css/style.css`, `frontend/js/app.js`
- Swagger: `docs/`

---

### 14) Step‑by‑step: from dev to production

#### A. Development setup
1. Prerequisites
   - Docker Desktop (or Docker Engine + Docker Compose)
   - Make sure ports 8080 (API), 3000 (frontend), and Postgres port are available
2. Clone repository and copy env
   - `cp env.template env`
   - Fill DB, S3 credentials, and any overrides
3. Start the stack (dev)
   - `docker-compose -f docker-compose.dev.yml up --build`
   - Services: Postgres, API with live reload (air), Nginx static frontend
4. Run SQL code generation when editing SQL
   - Inside repo: `sqlc generate`
5. Generate Swagger on API changes
   - `swag init -g main.go` (outputs to `docs/`)
6. Apply DB migrations (dev)
   - Dev compose typically runs migrations at start; to re-run manually use your migration tool or `psql` (files under `db/migrations/`)
7. Provide Instagram cookies (optional but recommended)
   - Put an exported cookie JSON at `cookies/ig_cookie.json` (mapped in the container at `/app/cookies/ig_cookie.json`)
8. Verify
   - API: `http://localhost:8080/health`
   - Swagger: `http://localhost:8080/swagger/index.html`
   - Frontend: `http://localhost:3000/`

#### B. Local functional test
1. Start a profile job
   - `curl -X POST http://localhost:8080/api/v1/scrape -H 'Content-Type: application/json' -d '{"username":"<user>"}'`
2. Start a posts‑only job (first N posts)
   - `curl -X POST http://localhost:8080/api/v1/scrape/posts -H 'Content-Type: application/json' -d '{"username":"<user>","post_count":12}'`
3. Track progress
   - `curl http://localhost:8080/api/v1/jobs/<JOB_ID>`
4. View data
   - Profiles: `GET /api/v1/profiles?limit=20`
   - Posts: `GET /api/v1/posts/<username>`
5. Frontend
   - Check Recent, History; open a profile card to view the growth chart

#### C. Pre‑production checklist
1. Environment & secrets
   - Provide production `.env` (DB creds, S3, any proxies)
   - Provide production Instagram cookies (if needed)
2. Storage
   - S3 bucket exists; IAM policy restricted to the needed paths
3. Database
   - Provision managed PostgreSQL; apply all migrations under `db/migrations/`
4. Images
   - Build production images:
     - API: `docker build -t your-reg/snapper-api:prod -f Dockerfile .`
     - Frontend: Nginx serves `frontend/` (the dev compose mounts; for prod bake an image with the assets copied)
5. Observability
   - Configure logs shipping (stdout or GELF); health endpoint monitored
6. Networking & TLS
   - Put API and frontend behind a reverse proxy with TLS (nginx/traefik/ALB)

#### D. Production deployment (Compose example)
1. Write a minimal compose (prod) with:
   - API container (env from secrets/config), depends_on DB, exposes 8080
   - Nginx container serving static `frontend/` and proxy `/api/` to API
   - External managed Postgres (no DB container)
2. Deploy
   - `docker compose -f docker-compose.prod.yml pull && docker compose -f docker-compose.prod.yml up -d`
3. Run migrations
   - Trigger a one‑off job/container or use your migration tool to apply `db/migrations/`

#### E. Kubernetes (optional)
1. Build & push images to a registry
2. Manifests/Helm basics
   - Deployments for API and Nginx
   - Service per deployment; Ingress with TLS
   - Secrets for env (DB URL, S3 credentials)
   - ConfigMap for Nginx
3. Jobs/cron (optional) to re‑trigger scraping on cadence

#### F. Operations & maintenance
1. Rotating cookies
   - Update the cookie file/secret; restart API pods/containers if necessary
2. Monitoring
   - Watch 5xx in API logs, job failure rates, and scrape durations
3. Scaling
   - API and Nginx are stateless; scale horizontally
   - DB and S3 scale independently (managed services recommended)
4. Backup/retention
   - Use managed Postgres automated backups; S3 lifecycle policies for artifacts
5. DOM changes on Instagram
   - Post grid selectors are heuristic; adjust `pkg/scraper/*.go` if parsing degrades

