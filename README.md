# GitHub Gists API

A small HTTP API that returns a GitHub user's public Gists.

```
GET /<username>   -> the user's public gists
GET /health       -> {"status": "ok"}
```

Built with Python 3.13 and Flask, served by gunicorn inside the container.

## Run with Docker (only Docker required)

```bash
docker build -t gists-api .
docker run --rm -p 8080:8080 gists-api
```

Then:

```bash
curl http://localhost:8080/octocat
curl "http://localhost:8080/octocat?page=1&per_page=5"
```

Example response (shortened to the first of octocat's 8 gists):

```json
{
  "user": "octocat",
  "page": 1,
  "per_page": 30,
  "count": 8,
  "gists": [
    {
      "id": "6cad326836d38bd3a7ae",
      "description": "Hello world!",
      "url": "https://gist.github.com/octocat/6cad326836d38bd3a7ae",
      "files": [
        {"name": "hello_world.rb", "language": "Ruby", "size": 175}
      ],
      "created_at": "2014-10-01T16:19:34Z",
      "updated_at": "2026-10-13T10:22:12Z"
    }
  ]
}
```

## Run the tests

Using a local virtual environment (nothing installed globally):

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt

pytest                             # unit tests, GitHub is mocked
RUN_INTEGRATION=1 pytest -m integration   # calls the real GitHub API for octocat
```

## Configuration

All configuration is through environment variables, e.g. `docker run -e GITHUB_TOKEN=... -p 8080:8080 gists-api`.

| Variable | Default | Purpose |
|---|---|---|
| `GITHUB_TOKEN` | unset | Optional token. Unauthenticated requests are limited by GitHub to 60/hour per IP. |
| `REQUEST_TIMEOUT_SECONDS` | `5` | Timeout for calls to GitHub. |
| `GITHUB_API_URL` | `https://api.github.com` | GitHub API base URL. |
| `LOG_LEVEL` | `INFO` | Application log level. |

## Behaviour

| Situation | Response |
|---|---|
| Success | `200` with the list of gists |
| Invalid `page` / `per_page` | `400` |
| User does not exist | `404` |
| GitHub rate limit hit (403/429) | `503` |
| GitHub unreachable or unexpected error | `502` |
| GitHub timed out | `504` |

Errors are returned as JSON: `{"error": "..."}`.

## Design decisions

- **Trimmed response.** Only the fields a client is likely to need are returned (id, description, url, each file's name, language and size in bytes, timestamps) rather than GitHub's full payload.
- **Pagination is passed through** to GitHub (`page`, `per_page` up to 100) instead of fetching every page, which keeps requests fast and limits rate-limit usage.
- **Unit tests mock GitHub** so they are fast and deterministic and can cover failure cases. A separate opt-in integration test checks the real API with `octocat`.
- **Container:** slim pinned base image, dependencies installed in a cached layer, runs as a non-root user, served by gunicorn rather than Flask's development server.

## Possible improvements

- Caching responses (e.g. short TTL, or ETag / conditional requests) to reduce GitHub rate-limit usage
- Prometheus metrics and structured (JSON) logs
- Following GitHub's `Link` header to expose next/previous page links
- CI pipeline to run tests and build the image on each push
