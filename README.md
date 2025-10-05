## Go + Redis URL Shortener

This repository contains a small URL shortener written in Go using the Fiber web framework and Redis as the backing store. It includes a simple rate-limiting mechanism and supports custom short codes and expiry times.

The project is organized with a lightweight API service (`/api`) and a Redis service (`/db`) and can be run locally with Go or with Docker / docker-compose.

---

## Quick summary

- Language: Go
- Web framework: Fiber (github.com/gofiber/fiber)
- Data store: Redis (github.com/go-redis/redis/v8)
- Purpose of Redis in this project:
  - Primary store (Redis DB 0) for mapping short codes -> original URLs
  - Secondary store (Redis DB 1) for rate-limiting per-client IP and a global redirect counter

Why Redis?
- Performance: Redis is an in-memory key-value store, so lookups and writes are fast — ideal for short-lived, low-latency lookups like URL resolution.
- TTLs: Redis supports per-key expirations directly, which makes implementing URL expiry and temporary rate limits simple.
- Simplicity: Using Redis keeps the architecture simple and avoids heavier relational schema for a small service.

---

## Repository layout

- `api/` - Go API service (main app)
  - `main.go` - application entrypoint; sets up Fiber, loads environment variables and routes
  - `Dockerfile` - multi-stage Dockerfile to build and run the Go binary
  - `database/database.go` - helper that creates Redis clients (uses env vars)
  - `routes/shorten.go` - endpoint to shorten URLs (POST `/api/v1`)
  - `routes/resolve.go` - endpoint to resolve short codes (GET `/:url`)
  - `helpers/` - small helpers used for URL validation and normalization
- `db/` - Dockerfile for Redis image (based on `redis:alpine`)
- `docker-compose.yml` - compose config to run the api + redis together
- `data/` - mounted Redis data directory (on host via docker-compose)

---

## How this app uses Redis (details)

- Redis DB 0 (default) : stores the mapping from short code -> original URL.
  - Key: short code (e.g. `4fgA1b`)
  - Value: full URL (e.g. `https://example.com/...`)
  - TTL: per-key expiry set when creating the short URL (default 24 hours in code)

- Redis DB 1 : used for ephemeral data
  - Rate limiting: stores client IP -> remaining calls with a TTL (example: 30 minutes)
  - Counter: a key named `counter` that is incremented on each successful redirect (for metrics)

Using separate logical Redis DB indices keeps persistent mapping data (DB0) separate from ephemeral counters and rate-limit keys (DB1). Both use the same Redis service but different databases inside it.

---

## Environment variables (used by the code)

- `APP_PORT` - address to bind the Fiber server. Example: `:3000` (recommended). The app reads this in `main.go`.
- `DB_ADDR` - Redis server address. When using docker-compose this should be `db:6379`.
- `DB_PASS` - Redis password (if you set one). Empty by default.
- `DOMAIN` - the domain used when returning shortened URLs (e.g. `http://localhost:3000`). The code concatenates this with the generated ID.
- `API_QUOTA` - number of allowed requests per IP in the quota window (used for rate-limiting). Example: `10`.

Notes:
- `main.go` expects `APP_PORT` to be set. If you run with docker-compose the `ports` mapping exposes `3000`.
- The Redis client in `database/database.go` uses `os.Getenv("DB_ADDR")` and `os.Getenv("DB_PASS")`.

---

## How to run

1) Run locally with Go (developer mode)

Make sure you have Go installed (matching your project's go.mod). From the project root:

```powershell
# set env vars for a local run (PowerShell example)
$env:APP_PORT=':3000'; $env:DB_ADDR='localhost:6379'; $env:DB_PASS=''; $env:DOMAIN='http://localhost:3000'; $env:API_QUOTA='10'

# build and run
cd api
go build -o main .
./main
```

Notes:
- If you use the `vendor` directory and the Dockerfile uses `-mod=vendor` during build, the local `go build` will still work if modules are available. If you prefer vendor mode locally, run `go env -w GOFLAGS='-mod=vendor'` or build with `go build -mod=vendor`.

2) Run with Docker Compose (recommended for reproducing environment)

```powershell
# from repo root (PowerShell)
docker-compose up --build -d

# view logs
docker-compose logs -f api

# stop
docker-compose down
```

Docker compose runs two services:
- `api` - the Go service built from `api/Dockerfile` and exposed on port 3000
- `db` - Redis server (image: `redis:alpine`) with port 6379 exposed

The compose file mounts `./data` to the Redis container path so Redis can persist data to the host. Make sure the `data/` folder exists and Docker has permissions to write to it.

---

## API

1) Shorten URL

- Endpoint: POST /api/v1
- JSON body fields:
  - `url` (string) - the original URL to shorten. Must be a valid URL.
  - `short` (string, optional) - custom short code. If omitted, a 6-character UUID prefix is used.
  - `expiry` (number, optional) - expiry duration in hours (default in code: 24)

Example (curl):

```bash
curl -X POST "http://localhost:3000/api/v1" \
  -H "Content-Type: application/json" \
  -d '{"url":"https://example.com/very/long/path","expiry":24}'
```

Example (PowerShell):

```powershell
$body = @{ url = 'https://example.com/very/long/path'; expiry = 24 } | ConvertTo-Json
Invoke-RestMethod -Uri 'http://localhost:3000/api/v1' -Method Post -Body $body -ContentType 'application/json'
```

Sample successful response (JSON):

```json
{
  "url": "https://example.com/very/long/path",
  "short": "http://localhost:3000/4fgA1b",
  "expiry": 24,
  "rate_limit": 9,
  "rate_limit_reset": 29
}
```

Notes about shortening behavior:
- The service enforces HTTPS for stored URLs (see helper in `helpers/`).
- If a custom short code is provided and already exists, the API returns HTTP 403 "URL short already in use".
- The code uses Redis DB 0 to store the mapping and sets a TTL equal to `expiry` hours.
- Rate-limiting is implemented per-client-IP using Redis DB 1. A client is given `API_QUOTA` calls per window (the code uses 30 minutes as the window). Each shorten operation decrements the remaining quota stored under the client IP.

2) Resolve short URL

- Endpoint: GET /:url
- Example: GET /4fgA1b

Behavior:
- The handler looks up the short code in Redis DB 0; if found it redirects (301) to the original URL.
- On successful redirect the code increments a `counter` key in Redis DB 1 (useful for metrics).

Example (curl):

```bash
curl -v http://localhost:3000/4fgA1b
```

Example (PowerShell):

```powershell
Invoke-WebRequest -Uri 'http://localhost:3000/4fgA1b' -MaximumRedirection 0 -ErrorAction SilentlyContinue
```

If the short code does not exist the API returns HTTP 404 with a JSON error.

---

## Dockerfile notes

- `api/Dockerfile` is multi-stage:
  - Stage 1 (builder): builds the Go binary (supports `vendor` if present)
  - Stage 2 (runtime): copies the binary into an `alpine` image and runs it

- `db/Dockerfile` simply uses `redis:alpine` and exposes port 6379.

---

## Data persistence

The `docker-compose.yml` mounts `./data` into the Redis container. This allows Redis to persist snapshots to the host. If you want persistent Redis data outside of the compose directory you can change the volume mapping in `docker-compose.yml`.

---

## Troubleshooting & common issues

- Redis connection errors: ensure `DB_ADDR` is correct. In docker-compose `DB_ADDR` should be `db:6379` (service name + port). If running locally against host Redis use `localhost:6379`.
- App does not start: check `APP_PORT` is set and not already in use. Use `docker-compose logs -f api` to view logs.
- Permissions on `data/` volume: ensure Docker can write to the folder. On Windows you may need to share the drive or adjust permissions.
- Rate limit immediately exhausted: `API_QUOTA` might be set low or the same IP has many requests recorded in Redis DB 1. Inspect Redis keys to see the current value.

How to inspect Redis keys (using docker):

```powershell
# open a redis-cli session to the running container
docker exec -it <redis_container_name_or_id> redis-cli

# switch DB 1
SELECT 1
# list keys
KEYS *
# get a key
GET <some-key>
```

---

## Security considerations

- This example stores short codes without collision checks when auto-generating; in production you'd want stronger collision handling and possibly more entropy.
- The app trusts the `DOMAIN` env variable for creating public short links; ensure it's set to the correct origin.
- If you enable Redis authentication, set `DB_PASS` and never check secrets into source control.
- Consider adding HTTPS (TLS) termination in front of the service in production.

---

## Next steps / enhancements

- Add collision detection and retry for generated IDs.
- Add analytics endpoint to read the `counter` and per-key stats.
- Persist a mapping of created-by (user) to support user accounts and quotas.
- Add unit/integration tests for handlers.

---

## License

This repository does not include a license file. Add one if you plan to release or share the project publicly.

---

If anything in this README is incorrect or you want a different level of detail for any section (examples, architecture diagram, tests), tell me what you'd like changed and I will update the file.
