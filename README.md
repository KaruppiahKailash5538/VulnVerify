# VulnVerify

VulnVerify is a local, evidence-led website configuration scanner. It stores account-owned projects and explicitly authorized targets, collects bounded HTTP/TLS evidence, classifies findings transparently, and supports same-target rescans and report exports.

## Safety scope

Only scan websites you own or have explicit permission to assess. Each target is saved under an account-owned project and authorization is reconfirmed when a scan or rescan starts. The scanner rejects private, loopback, reserved, mixed public/private DNS results, non-standard ports, and credentials in URLs. DNS is revalidated and connections are pinned to a validated public IP. Each scan uses at most six HTTP requests, a 512 KB response-body cap, five-second connection/read timeouts, same-origin GET requests only, and basic `robots.txt` disallow rules. Forms are counted, never submitted; redirects, payload tests, authentication bypasses, and exploit attempts are not performed. Rate-limit responses stop discovery. A missing header is an observed configuration gap, not proof of exploitability.

## Run locally

In PowerShell, from the workspace root:

```powershell
& ".\.venv\Scripts\python.exe" -m pip install -r ".\web-scanner-app\backend\requirements.txt"
Push-Location ".\web-scanner-app\backend"
& "..\..\.venv\Scripts\python.exe" -m uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

In a second terminal:

```powershell
Push-Location ".\web-scanner-app\frontend"
& "..\.\.venv\Scripts\python.exe" -m http.server 5173 --bind 127.0.0.1
```

For access from another device on the same Wi-Fi, bind both servers to `0.0.0.0` and allow your LAN origin in the backend CORS policy. The frontend automatically sends API calls to port 8000 on the hostname used to open the page. For example, if this computer's LAN IP is `10.56.49.92`, open <http://10.56.49.92:5173>.

Open <http://127.0.0.1:5173> for local access. Register an account using a password of at least 12 characters, create a project, add a website name and HTTP(S) host, and confirm authorization. After target validation, select the saved target, confirm authorization again, and start the safe web scan. Findings and before/after comparisons are based on observed target responses. API docs are at <http://127.0.0.1:8000/docs>.

## Persistence and tests

SQLite data is stored at `backend/vulnverify.sqlite3` by default. Set `VULNVERIFY_DB` or `DATABASE_URL` to change the database path for deployment environments. Run backend tests from `web-scanner-app/backend`:

```powershell
& "..\..\.venv\Scripts\python.exe" -m pytest
```

## Docker and production-oriented deployment

```powershell
Push-Location ".\web-scanner-app"
Copy-Item ".\prod.env.example" ".\.env"
# Update the file with your public domain and secret values, then:
docker compose up --build
```

The Compose stack launches the FastAPI API, frontend static site, and a local Redis instance, binds the API on port 8000, serves the frontend on port 8080, and keeps SQLite data under a named Docker volume. Environment variables such as `ALLOWED_ORIGINS`, `PUBLIC_BASE_URL`, `SECRET_KEY`, `DATABASE_URL`, and `REDIS_URL` can be overridden for hosted deployments without changing the application code.

### Production hosting notes

- Set `SECRET_KEY` to a strong random value before any public deployment.
- Set `PUBLIC_BASE_URL` to your actual HTTPS domain.
- Set `ALLOWED_ORIGINS` to your domain and any CDN or reverse-proxy origins.
- Place the app behind TLS termination (for example, a reverse proxy or managed ingress) before exposing it publicly.
- Keep the SQLite database in a persistent volume or migrate to PostgreSQL when the app scales beyond the local MVP.

Passwords use PBKDF2-HMAC-SHA256 and bearer session tokens expire after 12 hours. Account ownership is enforced on project, target, scan, finding, and report queries. Scan work is limited to two concurrent scans per user and 24 scans per saved target per 24 hours. This MVP does not yet include PostgreSQL, an external AI service, JavaScript-rendered page discovery, dependency analysis, authenticated testing, or active input testing. The local evidence explainer uses stored observations only and does not invent evidence.