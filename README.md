<div align="center">

# [psync](https://getpsync.vercel.app)

**Synchronized lyrics API. Word-level, line-level, or plain.**

A single serverless endpoint that resolves lyrics from seven providers, normalises every payload into one shape, and serves it as JSON, enhanced LRC, TTML, or plain text.

[![TypeScript](https://img.shields.io/badge/TypeScript-5.7-3178c6?logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![Vercel](https://img.shields.io/badge/Vercel-Functions-000?logo=vercel&logoColor=white)](https://vercel.com/docs/functions)
[![Node](https://img.shields.io/badge/Node-20+-339933?logo=node.js&logoColor=white)](https://nodejs.org/)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](./LICENSE)

</div>

---

## Overview

`psync` is a lyrics backend for music clients. It takes track metadata, walks a tiered list of providers until something validates, and returns timings your player can render against.

Built to survive provider churn rather than depend on any single one. many sources, three timing grades, and a transport layer that assumes every provider will eventually rate-limit, fingerprint, or change its schema.

## Features

- **Word-level sync** — per-word `start`/`end` timings for karaoke rendering, with backing-vocal flags
- **Graceful degradation** — `word` → `line` → `plain` instead of an empty response
- **Four output formats** — JSON, enhanced LRC, TTML, plain text
- **Seven providers** — lyrics-plus, lrc.red, Unison, Deezer, Better Lyrics, Musixmatch, LRCLIB
- **Resilient transport** — rotating fingerprints, cookie persistence, jittered retries, per-host circuit breakers
- **Validated before cached** — monotonic timing checks and duration envelopes keep bad payloads out
- **Token gated** — bearer auth on every request
- **Zero runtime dependencies** — Node built-ins only


## API

### `GET /api/lyrics`

```bash
curl -H "Authorization: Bearer $PSYNC_TOKEN" \
  "https://getpsync.vercel.app/api/lyrics?title=Hello&artist=Adele&duration=295"
```

### Parameters

| Name | Type | Required | Notes |
| --- | --- | --- | --- |
| `title` / `track` | string | yes* | Track title, 200 chars max |
| `artist` / `a` | string | yes* | Artist name, 200 chars max |
| `duration` / `dur` | number | no | Seconds. Significant accuracy gain. |
| `album` | string | no | Disambiguation hint |
| `isrc` | string | no* | Exact catalog key |
| `v` / `videoId` | string | no* | YouTube video ID |
| `format` | enum | no | `json`, `lrc`, `lrc-plain`, `ttml`, `plain` |
| `words` | `1` | no | Word-synced only, `404` otherwise |
| `refresh` | `1` | no | Bypass cache |

\* Satisfy at least one of (`title` + `artist`), `isrc`, or `v`.

### Response

```json
{
  "source": "lyricsplus",
  "kind": "word",
  "title": "Hello",
  "artist": "Adele",
  "duration": 295,
  "isrc": "GBBKS1500214",
  "plain": "Hello, it's me\nI was wondering if after all these years you'd like to meet",
  "lines": [
    {
      "start": 6047,
      "end": 10344,
      "text": "Hello, it's me",
      "words": [
        { "start": 6047, "end": 7279, "text": "Hello, " },
        { "start": 8899, "end": 9244, "text": "it's " },
        { "start": 9244, "end": 10344, "text": "me" }
      ]
    }
  ]
}
```

Timings are integer milliseconds from track start. `kind` is `word`, `line`, or `plain`. `words[].background` marks backing vocals. `lines` is empty when `kind` is `plain`; `plain` is always populated.

### Formats

| `format` | `Content-Type` | Body |
| --- | --- | --- |
| `json` | `application/json` | Full response object |
| `lrc` | `text/plain` | Enhanced LRC with `<mm:ss.xx>` word tags |
| `lrc-plain` | `text/plain` | Standard line-level LRC |
| `ttml` | `application/ttml+xml` | Apple-style TTML |
| `plain` | `text/plain` | Text only |

### Status codes

| Status | Meaning |
| --- | --- |
| `200` | Success |
| `400` | Missing or invalid parameters |
| `401` | Bad or missing token |
| `404` | No lyrics, or no word sync when `words=1` |
| `405` | Wrong method |
| `429` | Rate limited, see `Retry-After` |

All errors are JSON. `404` bodies include an `attempted` array naming the providers tried.

### Response headers

| Header | Values |
| --- | --- |
| `X-Psync-Source` | Provider that won |
| `X-Psync-Kind` | `word`, `line`, `plain` |
| `X-Psync-Cache` | `HIT` or `MISS` |
| `X-Psync-Store` | `redis` or `memory` |
| `X-Request-Id` | Request UUID |
| `X-RateLimit-Limit` / `-Remaining` | Budget state |

## Resolution order

Word tier, then line, then plain. The first validated hit wins and the walk stops.

| # | Source | Tiers |
| --- | --- | --- |
| 1 | Lyrics+ — `v2` → `v1` → `ttml`, mirrors staggered 1.2s | word, line |
| 2 | lrc.red — ISRC direct, then scored catalog search | word, line |
| 3 | Unison — video ID, then metadata, three formats | word, line, plain |
| 4 | Deezer — GraphQL pipe, JWT refresh on rejection | word, line, plain |
| 5 | Better Lyrics — TTML | word, line |
| 6 | Musixmatch — rotated guid, subtitle endpoint | line, plain |
| 7 | LRCLIB — exact get, then scored search | line, plain |

## Resilience

Providers rate-limit, fingerprint, and change schemas. The transport layer assumes all three.

- Six rotating browser identities, each with a coherent `sec-ch-ua`, `sec-fetch-*`, and `Accept-Language`
- Per-host header profiles with plausible `referer` and `origin`
- Persistent cookie jar per host, so sessions look stateful
- Jittered exponential retry on `403`, `429`, `5xx`, and network faults, honouring `Retry-After`
- Per-host circuit breaker: three consecutive faults open the host for 45s, doubling to 10 minutes
- Sources reordered by live health, so a known-good path is always tried first
- Multi-strategy fallbacks inside every source, plus a second parse pass per payload shape
- Payload validation before caching: line count, monotonic timings, duration envelope, junk filtering
- 18s global deadline, 8s per request — dead sources are skipped without consuming the budget
- Optional proxy pool with sticky per-host assignment, rotating only on trip

## Caching

Successful responses cache for 24h under a key of `title|artist|album|duration|isrc|videoId`, and carry `s-maxage=86400` with `stale-while-revalidate=604800`. Misses are CDN-cached for 600s only.

Send consistent parameters. A request with `duration` and one without are separate cache entries.

## Health

```
GET /api/health
```

```json
{ "status": "ok", "store": "redis", "gate": "private", "proxies": 0 }
```

`gate: "open"` after deploy means `PSYNC_TOKEN` did not bind and the endpoint is public.


## Credits
- [YouLy+](https://github.com/ibratabian17/YouLyPlus) and [LyricsPlus](https://github.com/ibratabian17/LyricsPlus) by [@ibratabian17](https://github.com/ibratabian17)
- [Better Lyrics API](https://github.com/better-lyrics/api) by [@boidushya](https://github.com/boidushya)
- [lrc.red](https://lrc.red) by w4v
- [Unison](https://github.com/better-lyrics/unison)
- [LRCLIB](https://github.com/tranxuanthang/lrclib)
- [Musixmatch](https://www.musixmatch.com), [Deezer](https://www.deezer.co)

**For token, join our discord server - https://discord.gg/GguYzY9d3**
