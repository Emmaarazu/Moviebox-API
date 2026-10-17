<p align="center">
  <img src="https://h5-static.aoneroom.com/ssrStatic/mbOfficial/public/_nuxt/web-logo.apJjVir2.svg" alt="LOGO" width="200"/>
</p>

# MovieBox API Pro

A pure REST API wrapper around [moviebox.ph](https://moviebox.ph).

No scraping. No HTML parsing. No headless browser. It talks the exact same private JSON API the moviebox web player talks — `h5-api.aoneroom.com/wefeed-h5api-bff` — and re-exposes it as a clean, self-hosted REST surface.

Built for people who want to wire moviebox into their own apps, scripts, bots, dashboards, or media servers without reverse-engineering the network tab every time the site ships an update.

**Version 2.3.3** — full changelog at the bottom.

---

## What it does

- **Discovery** — banners, featured blocks, and every category row from the moviebox home feed
- **Sections** — list and browse any home section directly
- **Catalogs** — paginated movies, TV, animation (resolved from home sections)
- **Search** — full-text search + typeahead suggestions
- **Metadata** — full detail tree for any subject (episodes, seasons, languages, artwork)
- **Streams** — direct MP4 sources per resolution, plus HLS and DASH manifests
- **Stream proxy** — Range-aware MP4 passthrough for media servers
- **Subtitles** — full caption list per episode, SRT→VTT conversion inline
- **Diagnostics** — tab-ID probing, payload variant testing, raw upstream passthrough, health

## What it isn't

- Not a proxy for HLS/DASH manifests (MP4 only)
- Not a downloader (it hands you URLs, you decide what to do with them)
- Not affiliated with moviebox.ph
- Not a scraping tool — it depends on the upstream API staying reachable

---

## Quickstart

```bash
git clone https://github.com/faisaljs/Moviebox-API
cd Moviebox-API
pip install -r requirements.txt
python main.py
```

Server boots on `http://localhost:8000`. Open it in a browser and you get the dashboard, which lists every endpoint with a one-click test button.

For dev with autoreload:

```bash
RELOAD=1 python main.py
```

For a specific port:

```bash
PORT=9000 python main.py
```

Docker:

```bash
docker compose up -d
```

---

## Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/` | HTML dashboard |
| `GET` | `/health` | Service, cache, and tab state |
| `GET` | `/home` | Featured sections and banners |
| `GET` | `/home/all` | Flattened deduped feed |
| `GET` | `/sections` | List every home section |
| `GET` | `/browse/{section}` | Browse any home section directly |
| `GET` | `/movies` | Movies catalog (home-fallback) |
| `GET` | `/tv-series` | TV catalog (home-fallback) |
| `GET` | `/animation` | Animation catalog (home-fallback) |
| `GET` | `/search` | Full-text search |
| `GET` | `/search/suggest` | Typeahead (`?enrich=true` for slugs) |
| `GET` | `/detail/{slug}` | Full metadata for a subject |
| `GET` | `/api/stream/{subject_id}` | Direct video sources |
| `GET` | `/api/stream/{subject_id}/proxy` | Range-aware MP4 passthrough |
| `GET` | `/api/stream/{subject_id}/captions` | Subtitle list |
| `GET` | `/api/stream/{subject_id}/subtitles/{lang}` | SRT → VTT inline |
| `GET` | `/api/tabs` | Tab-ID discovery (diagnostic) |
| `GET` | `/api/tabs/variants` | Payload variant testing (diagnostic) |
| `GET` | `/api/debug/upstream` | Raw upstream passthrough |

---

### `GET /home`

Returns the operating list from the moviebox home feed, grouped into sections. Banner sections and `SUBJECTS_*` rows are both normalized into the same item shape.

```bash
curl http://localhost:8000/home | jq
```

```json
{
  "status": "success",
  "sections": [
    {
      "section": "Banner",
      "count": 11,
      "items": [
        {
          "name": "VisionQuest",
          "poster_url": "https://.../cover.jpg",
          "slug": "visionquest-hindi-0qVaGP1YHE4",
          "subject_id": "3908280950886646112",
          "badge": "Hindi"
        }
      ]
    },
    {
      "section": "🔥Cinema",
      "count": 14,
      "items": [ ... ]
    }
  ]
}
```

The item shape is consistent across banner rows and subject rows, so you can flatten everything into a single feed without branching on section type.

---

### `GET /home/all`

Flattened, deduped feed across every home section. Useful for a single scroll surface.

```bash
curl "http://localhost:8000/home/all" | jq
curl "http://localhost:8000/home/all?dedupe=false" | jq   # keep duplicates
```

**Response:** `{ count, items[] }`

---

### `GET /sections`

List every section the home feed exposes — name, type, item count. Section names can be used with `/browse/{section}` or the catalog endpoints.

```bash
curl "http://localhost:8000/sections" | jq
```

```json
{
  "sections": [
    { "section": "IN Banner", "type": "BANNER", "count": 12 },
    { "section": "🔥Hot Short TV", "type": "SUBJECTS_MOVIE", "count": 8 },
    { "section": "🔥Cinema", "type": "SUBJECTS_MOVIE", "count": 14 },
    { "section": "Bollywood", "type": "SUBJECTS_MOVIE", "count": 6 },
    { "section": "Top Anime", "type": "SUBJECTS_MOVIE", "count": 16 }
  ]
}
```

Section types you'll see:
- `BANNER` — the hero carousel
- `SUBJECTS_MOVIE` / `SUBJECTS_TV` / `SUBJECTS_ANIMATION` — content rows
- `SPORT_LIVE` — live sports widgets (usually empty)
- `FILTER` — category grid widget
- `APPOINTMENT_LIST` — coming-soon rows

Only the `SUBJECTS_*` and `BANNER` types carry usable content.

---

### `GET /browse/{section}`

Direct access to any home section by name (case-insensitive substring match).

```bash
curl "http://localhost:8000/browse/Cinema" | jq
curl "http://localhost:8000/browse/Top%20Anime" | jq     # URL-encode spaces
curl "http://localhost:8000/browse/Anime" | jq            # matches "Top Anime"
```

```json
{
  "section": "Cinema",
  "matched_titles": ["🔥Cinema"],
  "page": 1,
  "per_page": 24,
  "total": 14,
  "items": [
    {
      "name": "The Vvaan: Force of the Forrest [Hindi][CAM]",
      "poster_url": "https://pbcdn.aoneroom.com/image/.../7454ddb0.jpg",
      "slug": "the-vvaan-force-of-the-forrest-hindi-cam-yvxjyvRyyJ5",
      "subject_id": "4813212061729714704",
      "badge": "Hindi",
      "rating": "6.9"
    }
  ]
}
```

`matched_titles` shows exactly which section names the query landed on. If none matched, returns 404 with a pointer to `/sections`.

Pagination is client-side — the home feed only carries one page per section.

---

### `GET /movies`, `GET /tv-series`, `GET /animation`

Paginated catalogs. Each resolves by merging the matching home sections:

| Endpoint | Sections |
|----------|----------|
| `/movies` | Cinema, Bollywood, Hollywood, South Indian |
| `/tv-series` | Hot Short TV, Indian Drama, Western TV, Best Asian Series |
| `/animation` | Top Anime |

**Query parameters**

| Param | Type | Default | Notes |
|-------|------|---------|-------|
| `page` | int | `1` | 1-indexed |
| `sort` | string | `RECOMMEND` | Only used by the filter endpoint (see below) |
| `tab` | int | env `TAB_*` | Override tab id for the filter attempt |
| `source` | string | `auto` | `auto`, `filter`, `home` |

**How `source` works:**

- `auto` (default) — try `/subject/filter` first; if it returns a sentinel (see below), fall back to home sections
- `filter` — force the filter path (will return sentinel data on current upstream)
- `home` — skip filter, use home sections directly

**Response**

```json
{
  "page": 1,
  "per_page": 24,
  "total": 44,
  "total_sentinel": false,
  "source": "home_fallback",
  "sections": ["Bollywood", "Hollywood", "South Indian", "🔥Cinema"],
  "items": [ ... ]
}
```

`source` tells you where the data came from. `total_sentinel: true` means upstream returned the `1000000` "no-match" sentinel — usually because `/subject/filter` requires auth we don't have as a guest. When that happens, `auto` falls through to `home_fallback` and you get real content.

**Note on `/subject/filter`:** the endpoint requires auth beyond the guest token for real filter responses. Every payload shape we tested returns sentinel. `/api/tabs/variants` documents this. Home fallback is the correct path.

---

### `GET /search`

**Query parameters**

| Param | Type | Required | Notes |
|-------|------|----------|-------|
| `q` | string | yes | Min length 1 |
| `page` | int | no | Default `1` |

```bash
curl "http://localhost:8000/search?q=matrix" | jq
```

**Response:** `{ query, page, total, items[] }` — items have `name`, `poster_url`, `slug`, `subject_id`.

---

### `GET /search/suggest`

Upstream suggest is a **typeahead endpoint** — it doesn't join against the subject table. Response is `{title}` per item.

```bash
curl "http://localhost:8000/search/suggest?q=break" | jq
```

```json
{
  "query": "break",
  "count": 10,
  "suggestions": [{ "title": "Breaking Bad [Hindi]" }]
}
```

For enriched results (with `slug` and `subject_id`), pass `?enrich=true`. This fans out one `/search` per suggestion — costs ~10 upstream calls per request.

```bash
curl "http://localhost:8000/search/suggest?q=break&enrich=true" | jq
```

```json
{
  "suggestions": [
    { "title": "Breaking Bad [Hindi]", "slug": "breaking-bad-hindi-...", "subject_id": "..." }
  ]
}
```

---

### `GET /detail/{slug}`

Raw passthrough of the upstream detail endpoint. The response is the full tree — subject info, seasons, episodes, available languages, artwork variants.

```bash
curl "http://localhost:8000/detail/breaking-bad-ej6Bp0MCAo7" | jq
```

Top-level keys you'll see: `subject`, `stars`, `resource`, `metadata`, `isForbid`, `watchTimeLimit`.

---

### `GET /api/stream/{subject_id}`

Resolves direct playable sources for a subject (or specific episode).

**Path parameter**

- `subject_id` — from any discovery/search endpoint

**Query parameters**

| Param | Type | Default | Notes |
|-------|------|---------|-------|
| `detail_path` | string | required | The `slug` |
| `se` | int | `1` | Season number |
| `ep` | int | `1` | Episode number |

```bash
curl "http://localhost:8000/api/stream/6207982430134357800?detail_path=breaking-bad-ej6Bp0MCAo7&se=1&ep=1" | jq
```

```json
{
  "subject_id": "6207982430134357800",
  "se": 1,
  "ep": 1,
  "has_resource": true,
  "sources": [
    {
      "resolution": "1080p",
      "format": "MP4",
      "url": "https://.../1080p.mp4",
      "size": 1234567890,
      "duration": 2880,
      "codec": "h264"
    }
  ],
  "hls": [],
  "dash": [],
  "free_episodes": 3,
  "limited": false,
  "note": null
}
```

---

### `GET /api/stream/{subject_id}/proxy`

Range-aware MP4 passthrough. Drop into media servers that can't set custom headers (Jellyfin, Plex, Infuse, VLC).

**Query parameters**

- Same as `/api/stream` plus `res` — `best` (default), `1080p`, `720p`, `480p`, etc.

```bash
# Full file
curl -O "http://localhost:8000/api/stream/6207982430134357800/proxy?detail_path=breaking-bad-ej6Bp0MCAo7&res=1080p"

# Ranged seek
curl -H "Range: bytes=0-1048575" "http://localhost:8000/api/stream/.../proxy?..." -o chunk.bin
```

Forwards `Range` upstream and returns `206` with proper `Content-Range` when the client requests a partial. Full-file requests return `200` and stream through a 64 KB chunked iterator.

MP4 only. HLS/DASH proxy would need manifest URL rewriting — not implemented.

---

### `GET /api/stream/{subject_id}/captions`

Fetches the caption track list for the resolved stream.

```bash
curl "http://localhost:8000/api/stream/6207982430134357800/captions?detail_path=breaking-bad-ej6Bp0MCAo7&se=1&ep=1" | jq
```

```json
{
  "subject_id": "6207982430134357800",
  "se": 1,
  "ep": 1,
  "count": 1,
  "captions": [
    {
      "id": "7702756051481150960",
      "lan": "en",
      "lanName": "English",
      "url": "https://cacdn.hakunaymatata.com/subtitle/.../abc.srt?Policy=...",
      "size": "54233",
      "delay": 0
    }
  ]
}
```

---

### `GET /api/stream/{subject_id}/subtitles/{lang}`

Fetches the SRT for `lang`, converts to WebVTT inline, returns `text/vtt`. Drop-in for a `<track>` element.

```bash
curl "http://localhost:8000/api/stream/6207982430134357800/subtitles/en?detail_path=breaking-bad-ej6Bp0MCAo7" | head -20
```

```vtt
WEBVTT

00:00:17.833 --> 00:00:18.916
[Vision] <i>Who am I?</i>

00:00:19.541 --> 00:00:21.708
...
```

The converter strips numeric SRT index lines, swaps `,` timestamp separators to `.`, prepends `WEBVTT`, and collapses whitespace. SRT formatting tags (`<i>`, `<b>`, `{\an8}` positions) survive.

---

### `GET /api/tabs` and `GET /api/tabs/variants`

Diagnostic endpoints for the `/subject/filter` endpoint.

```bash
# Probe tab IDs 1-20 for sample titles
curl "http://localhost:8000/api/tabs?min_id=1&max_id=20" | jq

# Try every known payload variant against a single tab
curl "http://localhost:8000/api/tabs/variants?tab_id=2" | jq
```

**Current state:** every variant returns `total: 1000000` (sentinel). `/subject/filter` requires auth we don't have as a guest. The home fallback catalogs are the correct path. These endpoints stay for future upstream changes — if the auth situation shifts, `/api/tabs/variants` will tell you immediately.

---

### `GET /api/debug/upstream`

Raw passthrough to the BFF. Inspect upstream response shapes.

```bash
# Raw suggest response
curl -G "http://localhost:8000/api/debug/upstream" \
  --data-urlencode "path=/subject/search-suggest" \
  --data-urlencode "method=POST" \
  --data-urlencode 'payload={"keyword":"break"}' | jq

# Raw home response
curl -G "http://localhost:8000/api/debug/upstream" \
  --data-urlencode "path=/home?host=moviebox.ph" | jq
```

---

### `GET /health`

Liveness + cache diagnostics. Useful as a container healthcheck.

```json
{
  "status": "ok",
  "version": "2.3.3",
  "token_cached": true,
  "token_age_seconds": 142,
  "domain_cached": "https://mzfi.me",
  "domain_expires_in": 287,
  "client_pool_connections": 2,
  "tabs": { "movies": 2, "tv": 5, "animation": 8 },
  "catalog_sections": {
    "movies": ["Cinema", "Bollywood", "Hollywood", "South Indian"],
    "tv-series": ["Hot Short TV", "Indian Drama", "Western TV", "Best Asian Series"],
    "animation": ["Top Anime"]
  }
}
```

---

## Architecture

### Guest token

moviebox's BFF issues a guest JWT on the very first unauthenticated request. It arrives in an `x-user` response header, not the body. `_make_request` refreshes the cached token on every response that carries `x-user` — self-heals on each call. On cold start, `_get_bearer_token` serializes acquisition behind an `asyncio.Lock` so concurrent requests don't race.

### Player domain

Stream URLs live on a different host than the BFF. The host is fetched from `/media-player/get-domain` and can rotate. Cached for 300s, re-fetched on expiry. The referer for stream requests is built to match what the browser would send to the play page.

### Retry policy

`_make_request` retries 3 times with exponential backoff on transient 5xx, network errors, and honors `Retry-After` on 429. A single 401-retry fires after clearing the stale token and re-acquiring.

### Connection pool

One shared `httpx.AsyncClient` per process, 50 max connections / 20 keepalive, created in lifespan and closed on shutdown.

### Catalog source resolution

`/movies`, `/tv-series`, `/animation` first try `/subject/filter`. If upstream returns the `1000000` sentinel, they fall back to merging the matching home sections. Every response carries a `source` field so you know which path was taken.

### SRT → VTT

`/subtitles/{lang}` fetches the SRT, strips numeric index lines, converts `,` timestamps to `.`, prepends `WEBVTT`. Italics and other SRT tags survive.

---

## Project layout

```
.
├── main.py            # the whole app — routes, helpers, HTML dashboard
├── requirements.txt   # fastapi, uvicorn, httpx
├── verify.py          # live integration test — chains real requests
├── Dockerfile
├── docker-compose.yml
├── README.md
├── CONTRIBUTING.md
├── .env.example
└── .gitignore
```

No package structure, no ORM, no config layer. One file for the app because the app is one file's worth of logic.

---

## Verification

`verify.py` runs a live integration pass. It chains requests — pulls a real slug and subject_id from `/home`, feeds them into `/detail`, `/api/stream`, `/api/stream/.../captions`, `/proxy`, and `/subtitles`. Every route prints `PASS` or `FAIL` with a sample.

```bash
# Server must be running
python verify.py

# Or against a remote host
python verify.py https://your-host
```

17 blocks, exits non-zero on any failure. Drops straight into CI or a pre-commit hook.

---

## Environment

| Variable | Default | Notes |
|----------|---------|-------|
| `PORT` | `8000` | Listen port |
| `RELOAD` | `0` | `1` enables autoreload (dev) |
| `REQUEST_TIMEOUT` | `25` | Upstream timeout, seconds |
| `RETRY_ATTEMPTS` | `3` | Total tries per upstream call |
| `RETRY_BACKOFF` | `0.5` | Base backoff for exponential retry |
| `DOMAIN_TTL` | `300` | Player-domain cache TTL |
| `TAB_MOVIES` | `2` | Movies tab id (filter endpoint, unused when sentinel) |
| `TAB_TV` | `5` | TV tab id |
| `TAB_ANIMATION` | `8` | Animation tab id |

No config file. No `.env` required — the `.env.example` is a convenience.

---

## Deployment notes

- **Behind a reverse proxy** — forward the full path. Routes assume `/` as root.
- **Healthcheck** — point your orchestrator at `/health`.
- **Long-running** — the token is process-global. Multiple workers each acquire their own token on first request.
- **Rate limits** — upstream is unauthenticated for guests. Heavy polling of `/home` or catalogs will get you throttled. Cache your own reads if you're calling on a schedule.
- **ToS** — you're calling a third party's private API. Behave accordingly.

---

## Troubleshooting

### Catalog returns junk / music videos

The filter endpoint fell through to sentinel and the home fallback didn't match. Check `/sections` — the section names might have shifted. Update `CATALOG_SECTIONS` in `main.py` if so.

### `total_sentinel: true` on a catalog response

`/subject/filter` needs auth we don't have as a guest. The `auto` source mode falls through to home fallback; if you see `source: "filter_sentinel_no_fallback"`, no home section matched the keywords for that catalog. Either fix the keywords in `CATALOG_SECTIONS` or use `/browse/{section}` directly.

### Stream proxy returns 502

Upstream CDN might be blocking the referer. Check `/api/stream/{sid}` first — if that returns sources, the URLs are live and the proxy should work. If sources are empty, the episode may be gated.

### `/browse/Top Anime` returns 404

URL-encode the space: `/browse/Top%20Anime`. Or use a substring: `/browse/Anime`.

### Why does suggest return no slug?

Suggest is a typeahead endpoint — designed to return strings fast. It doesn't join against the subject table. Slug/sid live only in `/search` and `/detail`. Pass `?enrich=true` for merged results.

---

## Upgraded in v2.3.0

- Shared `httpx.AsyncClient` (connection pooling) via lifespan
- Async-locked token init — no cold-start race
- Retry with exponential backoff; single 401-retry after clearing stale token
- `/home/all` — flattened deduped feed
- `/api/stream/{sid}/proxy` — Range-aware MP4 proxy
- `/api/stream/{sid}/subtitles/{lang}` — SRT → WebVTT on the fly
- `/health` reports version, token age, client pool stats
- Dockerfile, docker-compose, .env.example, .gitignore

## Fixed in v2.3.1

- `/search/suggest` — honest about text-only upstream; `?enrich=true` opt-in for slug/sid fanout
- Catalogs — `total_sentinel` flag surfaces the `1000000` "no-match" sentinel; `?tab=` override; `TAB_*` env vars
- `/api/tabs` — tab-ID discovery endpoint
- `/api/debug/upstream` — raw passthrough for shape inspection

## Fixed in v2.3.2

- Catalogs fall back to home sections when filter returns sentinel (`source` field on every response)
- `/sections` — list every home section
- `/browse/{section}` — direct access to any home section
- `/api/tabs/variants` — try every known payload shape against a single tab
- `CATALOG_SECTIONS` mapping centralized in `main.py`

## Fixed in v2.3.3

- Re-added `/` dashboard route (dropped in v2.3.2 restructure)
- `/api/tabs/variants` confirmed: `/subject/filter` requires auth we don't have as guest; home fallback is the correct path
- `verify.py` restored `poster=YES/NULL` in sample output
- Full 17/17 PASS on live verification

---

## Extensions

Not implemented, but the shape of the code makes them easy:

- **HLS/DASH proxy** — needs manifest URL rewriting, more involved than the MP4 proxy
- **Persistent cache** — Redis or SQLite on the home feed so you're not hitting the BFF every request
- **Client SDK** — thin Python/JS wrapper for `pip install` / `npm install`
- **Front-end** — actual UI hitting your own API
- **Auth layer** — gate your own API with a key if you're deploying publicly
- **Search across catalogs** — client-side merge of `/search` with home-fallback data

PRs welcome — see [`CONTRIBUTING.md`](CONTRIBUTING.md).

---

Not affiliated with moviebox.ph. Provided as-is. You are responsible for how you use it.