# WebCat — Web Categorization API Service

A lightweight REST API that accepts an FQDN and returns an AI-generated
category report via OpenRouter.ai. Built with FastAPI + Python.

---

## Project Structure

```
webcat.py          # FastAPI application (main entry point)
webcat.conf        # Configuration: API keys, models, customers
requirements.txt   # Python dependencies
webcat.log         # Auto-created request/response log
webcat.db          # SQLite cache database (auto-created on first run)
```

---

## Quick Start

### 1. Install dependencies
```bash
pip install -r requirements.txt
```

### 2. Edit webcat.conf
Fill in your real values:

| Section        | Key           | Description                                       |
|----------------|---------------|---------------------------------------------------|
| `[openrouter]` | `api_key`     | Your OpenRouter.ai secret key                     |
| `[models]`     | `level_1/2/3` | Model strings for each detail tier                |
| `[customers]`  | `<key> = <n>` | Customer API key mapped to max detail level (1–3) |
| `[cache]`      | `cache_days`  | Days before a cached result is considered stale   |
| `[batch]`      | `threads`     | Concurrent workers for batch mode                 |
| `[batch]`      | `api_key`     | API key used for all batch processing calls       |
| `[batch]`      | `detail`      | Detail level for batch mode (1, 2, or 3)          |
| `[batch]`      | `backoff_*`   | Rate limit backoff settings                       |

### 3. Start the server
```bash
uvicorn webcat:app --host 0.0.0.0 --port 8000
```

Interactive docs are available at: `http://localhost:8000/docs`

---

## API Reference

### `GET /categorize`

**Query Parameters**

| Parameter     | Required | Type    | Default | Description                                              |
|---------------|----------|---------|---------|----------------------------------------------------------|
| `apiKey`      | ✅        | string  | —       | Customer API key (from webcat.conf)                      |
| `fqdn`        | ✅        | string  | —       | Fully-qualified domain name                              |
| `detail`      | ✅        | integer | —       | Output detail: `1`, `2`, or `3`                          |
| `preferFresh` | ❌        | integer | `0`     | Set to `1` to bypass cache and always call OpenRouter    |

All responses use the **IAB Content Taxonomy** for `tier1`, `tier2`, and `tier3`.
`tier2` and `tier3` will be an empty string when no applicable subcategory exists.

---

### Detail Level Examples

---

#### Level 1 — Simple
Returns IAB tier classification and confidence only. Suitable for high-volume,
low-latency lookups.

**curl:**
```bash
curl -G "http://localhost:8000/categorize" \
  --data-urlencode "apiKey=aaaa-1111-bbbb-2222" \
  --data-urlencode "fqdn=github.com" \
  --data-urlencode "detail=1"
```

**Response:**
```json
{
  "fqdn": "github.com",
  "tier1": "technology & computing",
  "tier2": "computing",
  "tier3": "software",
  "confidence": "high"
}
```

**curl (domain with tier1 only):**
```bash
curl -G "http://localhost:8000/categorize" \
  --data-urlencode "apiKey=aaaa-1111-bbbb-2222" \
  --data-urlencode "fqdn=weather.gov" \
  --data-urlencode "detail=1"
```

**Response:**
```json
{
  "fqdn": "weather.gov",
  "tier1": "news & politics",
  "tier2": "",
  "tier3": "",
  "confidence": "medium"
}
```

---

#### Level 2 — Detailed
Adds a safe-for-work flag and a one-sentence description on top of Level 1.

**curl:**
```bash
curl -G "http://localhost:8000/categorize" \
  --data-urlencode "apiKey=cccc-3333-dddd-4444" \
  --data-urlencode "fqdn=espn.com" \
  --data-urlencode "detail=2"
```

**Response:**
```json
{
  "fqdn": "espn.com",
  "tier1": "sports",
  "tier2": "sport articles",
  "tier3": "fantasy sports",
  "confidence": "high",
  "safe_for_work": true,
  "description": "A leading sports media network covering news, scores, highlights, and fantasy sports across all major leagues."
}
```

**curl (SFW false example):**
```bash
curl -G "http://localhost:8000/categorize" \
  --data-urlencode "apiKey=cccc-3333-dddd-4444" \
  --data-urlencode "fqdn=example-adult-site.com" \
  --data-urlencode "detail=2"
```

**Response:**
```json
{
  "fqdn": "example-adult-site.com",
  "tier1": "adult content",
  "tier2": "mature audiences",
  "tier3": "",
  "confidence": "high",
  "safe_for_work": false,
  "description": "A website hosting content intended for adult audiences only."
}
```

---

#### Level 3 — Complex
Full analysis including content tags, intended audience, and threat level.
Requires a Level 3 authorized API key.

**curl:**
```bash
curl -G "http://localhost:8000/categorize" \
  --data-urlencode "apiKey=eeee-5555-ffff-6666" \
  --data-urlencode "fqdn=nytimes.com" \
  --data-urlencode "detail=3"
```

**Response:**
```json
{
  "fqdn": "nytimes.com",
  "tier1": "news & politics",
  "tier2": "news",
  "tier3": "world news",
  "confidence": "high",
  "safe_for_work": true,
  "content_tags": ["journalism", "breaking news", "opinion", "politics", "culture"],
  "audience": "general",
  "threat_level": "none",
  "description": "The New York Times is a major American daily newspaper and digital news platform covering domestic and international news, politics, business, science, and culture. It is one of the most widely read English-language news sources in the world."
}
```

**curl (threat detected example):**
```bash
curl -G "http://localhost:8000/categorize" \
  --data-urlencode "apiKey=eeee-5555-ffff-6666" \
  --data-urlencode "fqdn=malicious-phishing-site.com" \
  --data-urlencode "detail=3"
```

**Response:**
```json
{
  "fqdn": "malicious-phishing-site.com",
  "tier1": "illegal content",
  "tier2": "cybercrime",
  "tier3": "phishing",
  "confidence": "high",
  "safe_for_work": false,
  "content_tags": ["phishing", "malware", "social engineering"],
  "audience": "general",
  "threat_level": "high",
  "description": "A domain associated with phishing campaigns designed to steal user credentials by impersonating legitimate services. Users should avoid visiting this site and report it to relevant authorities."
}
```

---

### Error Response Examples

All errors return a consistent JSON body with an `error_code` and a human-readable `detail` string:

```json
{
  "error_code": "WC-XXXX",
  "detail": "Description of the error."
}
```

---

**WC-1001 — Invalid customer API key (HTTP 401):**
```bash
curl -G "http://localhost:8000/categorize" \
  --data-urlencode "apiKey=bad-key" \
  --data-urlencode "fqdn=example.com" \
  --data-urlencode "detail=1"
```
```json
{"error_code": "WC-1001", "detail": "Invalid or missing customer API key."}
```

**WC-1002 — Detail level exceeds key authorization (HTTP 403):**
```bash
curl -G "http://localhost:8000/categorize" \
  --data-urlencode "apiKey=aaaa-1111-bbbb-2222" \
  --data-urlencode "fqdn=example.com" \
  --data-urlencode "detail=3"
```
```json
{"error_code": "WC-1002", "detail": "Detail level requested exceeds the maximum authorized for this API key."}
```

**WC-2002 — FQDN too long (HTTP 400):**
```json
{"error_code": "WC-2002", "detail": "FQDN exceeds the maximum allowed length of 512 characters."}
```

**WC-2003 — FQDN invalid characters (HTTP 400):**
```bash
curl -G "http://localhost:8000/categorize" \
  --data-urlencode "apiKey=eeee-5555-ffff-6666" \
  --data-urlencode "fqdn=bad_domain!.com" \
  --data-urlencode "detail=1"
```
```json
{"error_code": "WC-2003", "detail": "FQDN contains invalid characters. Only a-z, 0-9, dots, and hyphens are permitted."}
```

**WC-3001 — OpenRouter upstream error (HTTP 502):**
```json
{"error_code": "WC-3001", "detail": "Upstream categorization service (OpenRouter) returned a non-200 response."}
```

**WC-3003 — Bad OpenRouter API key (HTTP 502):**
```json
{"error_code": "WC-3003", "detail": "Invalid or rejected OpenRouter API key. Check webcat.conf."}
```

---

### Full Error Code Reference

| Error Code | HTTP | Category      | Description                                                                 |
|------------|------|---------------|-----------------------------------------------------------------------------|
| `WC-1001`  | 401  | Auth          | Invalid or missing customer API key                                         |
| `WC-1002`  | 403  | Auth          | Detail level exceeds the maximum authorized for this API key                |
| `WC-1003`  | 403  | Auth          | API key is present but has been disabled                                    |
| `WC-2001`  | 400  | FQDN          | FQDN is empty                                                               |
| `WC-2002`  | 400  | FQDN          | FQDN exceeds 512 characters                                                 |
| `WC-2003`  | 400  | FQDN          | FQDN contains invalid characters (only a-z, 0-9, dots, hyphens allowed)    |
| `WC-2004`  | 400  | FQDN          | FQDN must contain at least two labels                                       |
| `WC-2005`  | 400  | FQDN          | FQDN contains consecutive dots (empty label)                                |
| `WC-2006`  | 400  | FQDN          | FQDN label violates RFC-1123 rules                                          |
| `WC-3001`  | 502  | Upstream      | OpenRouter returned a non-200 response                                      |
| `WC-3002`  | 502  | Upstream      | OpenRouter response could not be parsed                                     |
| `WC-3003`  | 502  | Upstream      | Invalid or rejected OpenRouter API key                                      |
| `WC-9001`  | 500  | Internal      | Unexpected internal error                                                   |

---

## Batch Mode

When `--inputfile` is supplied, the API server does **not** start. Instead, WebCat
reads a single-column CSV of FQDNs (no header) and categorizes them all, writing
every result into the SQLite cache.

### Running batch mode
```bash
python webcat.py --inputfile domains.csv
```

The input file should be a plain text file with one FQDN per line:
```
github.com
espn.com
nytimes.com
malicious-site.com
...
```

### How it works

1. FQDNs are read **one line at a time** — the entire file is never loaded into memory, so files with hundreds of thousands of entries are handled safely.
2. Each FQDN is validated. Invalid entries are logged and skipped.
3. The SQLite cache is checked first. If a fresh cached result exists, it is used and no OpenRouter call is made.
4. A pool of `threads` async workers (configured in `webcat.conf`) runs concurrently, each holding the semaphore only while an OpenRouter request is in flight.
5. If OpenRouter returns **HTTP 429 (rate limited)**, all workers pause, the backoff timer runs (with exponential growth up to `backoff_max`), and the affected FQDN is automatically re-queued for retry.
6. Progress is logged every 100 processed entries.
7. A final summary is printed on completion.

### Batch configuration (`webcat.conf`)

```ini
[batch]
threads            = 10    # concurrent OpenRouter requests
api_key            = ...   # API key used for all batch calls
detail             = 1     # detail level: 1, 2, or 3
backoff_initial    = 5     # seconds to wait on first 429
backoff_max        = 120   # maximum seconds to wait
backoff_multiplier = 2     # multiply wait time after each consecutive 429
```

### Progress log output

```
2025-01-01T12:00:00 | INFO | BATCH START | file=domains.csv | threads=10 | detail=1 | ...
2025-01-01T12:00:01 | INFO | CACHE HIT   | fqdn=github.com | detail=1 | auditDate=...
2025-01-01T12:00:02 | INFO | CACHE SET   | fqdn=espn.com | detail=1 | auditDate=...
2025-01-01T12:00:05 | WARN | BATCH BACKOFF | worker=3 | rate limited — pausing all workers for 5.0s
2025-01-01T12:00:10 | INFO | BATCH RESUME | next backoff ceiling=10.0s
2025-01-01T12:00:30 | INFO | BATCH PROGRESS | processed=100 | cache_hits=42 | openrouter=55 | skipped=2 | errors=1
2025-01-01T12:05:00 | INFO | BATCH COMPLETE | total_lines=1000 | processed=997 | cache_hits=420 | openrouter=574 | skipped=3 | errors=3
```

### Backoff behaviour

| Event                     | Action                                                     |
|---------------------------|------------------------------------------------------------|
| OpenRouter returns 429    | All workers pause; one drives the backoff timer            |
| First hit                 | Wait `backoff_initial` seconds (default: 5)                |
| Consecutive hits          | Wait multiplied by `backoff_multiplier` each time          |
| Ceiling reached           | Wait capped at `backoff_max` seconds (default: 120)        |
| Successful response       | Backoff timer resets to `backoff_initial`                  |
| FQDN that triggered 429   | Automatically re-queued and retried after backoff          |

---

## SQLite Cache

Results are cached in `webcat.db` (created automatically on first run) keyed by `fqdn` + `detail` level.

**Cache behaviour:**

- On a successful OpenRouter response, the result is stored with the current UTC timestamp in the `auditDate` column.
- Subsequent requests for the same `fqdn` + `detail` check the database first.
- If the cached record is **younger** than `cache_days` (set in `webcat.conf`), it is returned immediately — no OpenRouter call is made.
- If the cached record is **older** than `cache_days`, it is treated as stale and a fresh OpenRouter call is made, updating the database.
- Setting `preferFresh=1` bypasses the cache entirely, always calls OpenRouter, and updates the database with a fresh result and a new `auditDate`.

**Cache configuration (`webcat.conf`):**
```ini
[cache]
cache_days = 30
```

**Response fields added by the cache:**

| Field       | Type    | Description                                              |
|-------------|---------|----------------------------------------------------------|
| `source`    | integer | `100` = served from local database, `200` = fetched from OpenRouter |
| `auditDate` | string  | ISO 8601 UTC timestamp of when the result was stored     |

**curl — force fresh result:**
```bash
curl -G "http://localhost:8000/categorize" \
  --data-urlencode "apiKey=test123" \
  --data-urlencode "fqdn=github.com" \
  --data-urlencode "detail=1" \
  --data-urlencode "preferFresh=1"
```

**Response (fresh from OpenRouter):**
```json
{
  "fqdn": "github.com",
  "tier1": "technology & computing",
  "tier2": "computing",
  "tier3": "software",
  "confidence": "high",
  "source": 200,
  "auditDate": "2025-01-01T12:00:00+00:00"
}
```

**Response (served from cache):**
```json
{
  "fqdn": "github.com",
  "tier1": "technology & computing",
  "tier2": "computing",
  "tier3": "software",
  "confidence": "high",
  "source": 100,
  "auditDate": "2025-01-01T12:00:00+00:00"
}
```

---

## Security Design

- **FQDN max length**: 512 characters — requests exceeding this are rejected.
- **FQDN character whitelist**: Only `a-z`, `0-9`, `.`, `-` are accepted.
  Any other character (including uppercase, URL-encoded values, etc.) is rejected.
- **Label validation**: Each dot-separated label must be 1–63 chars,
  start and end with an alphanumeric character, following RFC-1123.
- **API keys**: Validated on every request; partial key logged for audit trail.
- **Detail level enforcement**: Each customer key has a ceiling level;
  requests above it are rejected with HTTP 403.
- **OpenRouter key**: Stored only in `webcat.conf` — never exposed in responses or logs.

---

## Logging

Every request and response is appended to `webcat.log` in the format:

```
2025-01-01T12:00:00+00:00 | INFO | REQUEST  | ts=... | ip=... | apiKey=aaaa1111**** | fqdn=github.com | detail=2 | model=...
2025-01-01T12:00:00+00:00 | INFO | RESPONSE | ts=... | ip=... | fqdn=github.com | detail=2 | tier1=technology & computing | tier2=computing | tier3=software
```

Rejected requests are logged at `WARNING` level with the rejection reason.

---

## OpenRouter Token Efficiency

- `temperature: 0` — deterministic, no creative overhead.
- `max_tokens: 256` — tight cap; the JSON schema is small.
- `response_format: {"type": "json_object"}` — enforces pure JSON output.
- System prompt is minimal and instructional only.
- User prompt is a single concise sentence with the exact expected schema inline.
- No chain-of-thought, no reasoning steps, no preamble.
