# animetsu-api (Animetsu is DEAD)

A small, fast, well-behaved REST wrapper around `animetsu.live`'s public
backend. It normalizes responses, caches the slow stuff, papers over a few
nasty quirks in the upstream MP4/HLS servers, and ships with a built-in
Swagger UI plus a working in-browser HLS demo player so you can sanity-check
everything without writing a single line of code.

Written in Go (Fiber v2). Single static binary, ~13 MB. Distroless Docker
image, zero runtime dependencies, runs on any free PaaS or a $4 VPS.

---

## Table of contents

- [Why this exists](#why-this-exists)
- [Quick start](#quick-start)
- [File tree](#file-tree)
- [Endpoints](#endpoints)
- [The HLS proxy (and why it exists)](#the-hls-proxy-and-why-it-exists)
- [Configuration (env vars)](#configuration-env-vars)
- [Deployment — 6 ways](#deployment--6-ways)
  - [1. Railway](#1-railway-easiest-button-deploy)
  - [2. Fly.io](#2-flyio)
  - [3. Render](#3-render)
  - [4. Koyeb](#4-koyeb)
  - [5. Docker (any host)](#5-docker-any-host)
  - [6. VPS / self-host with systemd + Caddy](#6-vps--self-host-with-systemd--caddy)
- [Using the API from your app](#using-the-api-from-your-app)
  - [JavaScript / TypeScript (web, hls.js)](#javascript--typescript-web-hlsjs)
  - [React Native (expo-av / react-native-video)](#react-native-expo-av--react-native-video)
  - [Kotlin / Android (ExoPlayer / Media3)](#kotlin--android-exoplayer--media3)
  - [Swift / iOS (AVPlayer)](#swift--ios-avplayer)
  - [Flutter (video_player / better_player)](#flutter-video_player--better_player)
  - [Python (yt-dlp / requests)](#python-yt-dlp--requests)
  - [curl + ffmpeg (download an episode)](#curl--ffmpeg-download-an-episode)
- [Security notes](#security-notes)
- [License](#license)

---

## Why this exists

`animetsu.live` already has a working backend, but talking to it directly
from a browser or mobile app is painful:

- It expects very specific `Referer` and `User-Agent` headers.
- It returns several stream "servers" (`pahe`, `kite`, `fsoft`) with
  inconsistent payload shapes and inconsistent reliability.
- All of those servers serve HLS (m3u8). Some return a master playlist,
  others return per-quality variants — the wrapper papers over the diff.
- HLS playlists embed AES-128 keys and segment URIs that a browser cannot
  hit directly because of CORS.

This wrapper fixes all of that, in one binary, behind a clean REST surface.

---

## Quick start

```bash
git clone <your fork>
cd animetsu-api
go run ./cmd/server   # listens on :8080
```

Then open:

- <http://localhost:8080/docs> — Swagger UI
- <http://localhost:8080/demo> — in-browser HLS player (paste an anime ID and hit Play)
- <http://localhost:8080/openapi.json> — raw OpenAPI 3 spec
- <http://localhost:8080/healthz> — liveness probe

Or with Docker:

```bash
docker build -t animetsu-api .
docker run --rm -p 8080:8080 animetsu-api
```

---

## File tree

```
animetsu-api-go/
├── cmd/server/main.go              ← entrypoint: load config, start Fiber, handle SIGTERM
├── internal/
│   ├── config/config.go            ← env-var parsing with sensible defaults
│   ├── logger/logger.go            ← zerolog JSON logger
│   ├── client/client.go            ← TWO HTTP clients: short-timeout for JSON,
│   │                                  no-body-timeout for streaming proxies
│   ├── cache/cache.go              ← in-memory TTL cache + single-flight + SWR
│   ├── middleware/middleware.go    ← API key auth, per-IP rate limit, request logger
│   ├── models/models.go            ← Envelope, WatchResponse, Source DTOs
│   ├── handlers/handlers.go        ← every JSON endpoint (home/search/watch/download/...)
│   ├── proxy/hls_proxy.go          ← /api/proxy/hls (playlist + segments + AES keys) + CORS preflight
│   ├── server/server.go            ← Fiber wiring: middleware, routes, error handler
│   └── docs/
│       ├── docs.go                 ← /docs (Swagger UI) + /demo (HLS player)
│       └── openapi.go              ← embedded OpenAPI 3 spec
├── pkg/hls/hls.go                  ← dependency-free m3u8 URI rewriter
├── deploy/
│   ├── fly.toml      ├── koyeb.yaml      ├── railway.json      └── render.yaml
├── Dockerfile                      ← multi-stage, distroless, ~13 MB final image
├── .env.example                    ← every supported env var
├── go.mod / go.sum
└── README.md
```

---

## Endpoints

All JSON responses are wrapped in:

```json
{ "success": true, "data": ... }
```

…or on error:

```json
{ "success": false, "error": { "code": "...", "message": "...", "status": 502 } }
```

### Discovery

| Method | Path | Notes |
|---|---|---|
| GET | `/api/home` | All home rails — cached 10 min, served stale-while-revalidate |
| GET | `/api/trending` `/api/season` `/api/popular` `/api/top-rated` `/api/upcoming` | Individual rails sliced from `/api/home` |
| GET | `/api/recent?page=1&per_page=12` | Recently updated episodes |
| GET | `/api/schedule` | Today's airing schedule |
| GET | `/api/random` | Random anime ID |
| GET | `/api/search?q=naruto&genres=Action,Drama&sort=POPULARITY_DESC&page=1` | AniList-style filters |
| GET | `/api/genres` `/api/formats` `/api/statuses` `/api/seasons` `/api/sorts` | Filter enums |

### Details

| Method | Path | Notes |
|---|---|---|
| GET | `/api/anime/:id` | Full metadata, trailer, next-airing |
| GET | `/api/anime/:id/episodes` | Episode list with thumbnails |
| GET | `/api/anime/:id/views/:ep` | View counter |

### Streaming

| Method | Path | Notes |
|---|---|---|
| GET | `/api/anime/:id/servers/:ep?source_type=sub` | Lists servers AND probes each in parallel — returns `working` + `latency_ms` |
| GET | `/api/anime/:id/watch/:ep?server=auto&source_type=sub&fallback=true` | Resolves playable HLS sources. `server=auto` tries `pahe → kite → fsoft` and returns the first one that actually serves bytes. Includes `subtitles[]` for soft-sub `<track>` wiring. |
| GET | `/api/watch?id=...&ep=...` | Same thing with query params (handier for non-RESTful clients) |
| GET | `/api/anime/:id/download/:ep?quality=1080p` | Best-quality proxied HLS URL + suggested `.mp4` filename + ready-to-paste `ffmpeg` mux command + soft-subtitle list |
| GET | `/api/anime/:id/downloads/:ep?source=pahe&quality=1080p&group=SubsPlease&type=sub` | **Real downloadable releases.** Switchable source — `source=pahe` (default, the small per-episode MP4s the animetsu.live Download dropdown shows: SubsPlease 360p/44MB · 720p/97MB · 1080p/154MB, one-click via pahe.win → kwik.cx) or `source=tosho` (p2p / magnet / NZB / DDL mirrors from Anime Tosho — full-quality, every fansub group). The `pahe` source is sourced via animetsu.live's public CDN-cached `dl` endpoint to bypass DDoS-Guard datacenter-IP blocks; if it ever fails the API silently falls back to `tosho` and sets `data.fallback_from = "pahe"`. Filters: `?quality=1080p`, `?group=Yameii`, `?type=sub\|dub\|raw`, `?limit=N`. |

#### Download response shape

```jsonc
{
  "success": true,
  "data": {
    "id": "6989b8a029cf95f4eb03b500",
    "episode": "1",
    "title": "Sousou no Frieren",
    "source": "animetosho",
    "query": "Sousou no Frieren 01",
    "groups": [
      {
        "group": "SubsPlease",
        "type": "sub",
        "qualities": ["1080p", "720p", "480p"],
        "releases": [
          {
            "title": "[SubsPlease] Sousou no Frieren - 01 (1080p) [...].mkv",
            "group": "SubsPlease",
            "quality": "1080p",
            "container": "mkv",
            "type": "sub",
            "language": "English Sub",
            "size_bytes": 1395864371,
            "size_human": "1.3 GB",
            "seeders": 153, "leechers": 4,
            "p2p_url": "https://storage.animetosho.org/p2p/.../...p2p",
            "magnet_uri":  "magnet:?xt=urn:btih:...",
            "nzb_url":     "https://storage.animetosho.org/nzbs/.../...nzb",
            "view_page":   "https://animetosho.org/view/...",     // ← page with every DDL mirror
            "nyaa_url":    "https://nyaa.si/view/1799935",
            "published_at":"2024-04-07T02:01:00Z",
            "info_hash":   "d2321a12ac71730eef9bca99435c8abe3ab7453e"
          }
        ]
      },
      { "group": "Yameii", "type": "dub", "...": "..." }
    ],
    "flat": [ /* every release, sorted quality desc → seeders desc */ ]
  }
}
```

> The legacy `/download/:ep` endpoint just hands back the HLS playlist URL and is fine for in-browser playback or `ffmpeg`-muxing to MP4. Use the new `/downloads/:ep` endpoint when you want **actual files** the user can save (the same SubsPlease / Yameii releases the animetsu.live download dropdown surfaces). It's a thin, cached wrapper over Anime Tosho's public JSON feed — no scraping, no headless browsers.


### Proxy

| Method | Path | Notes |
|---|---|---|
| GET / OPTIONS / HEAD | `/api/proxy/hls?url=<absolute>` | HLS playlist + segment + AES-128 key proxy. Streams segments without buffering so seeking is instant. |

### Meta

| Method | Path | Notes |
|---|---|---|
| GET | `/healthz` `/livez` `/readyz` `/health` `/ping` | Liveness probes |
| GET | `/docs` | Swagger UI |
| GET | `/demo` | In-browser HLS player |
| GET | `/openapi.json` | OpenAPI 3 spec |

---

## The HLS proxy (and why it exists)

### `/api/proxy/hls`

The HLS servers (`pahe`, `kite`, `fsoft`) return m3u8 playlists whose URIs
point straight at `mega-cloud.top`. A browser can't hit that host directly
because of CORS, and even server-side fetches need the right `Referer`. This
proxy:

1. Fetches the playlist with the right headers.
2. Rewrites every URI inside (segments, AES-128 keys, MAP init segments,
   sub-playlists) to point back at `/api/proxy/hls?url=<absolute>` on this host.
3. Streams segments and AES keys through unchanged via `fasthttp.SendStream`,
   never buffering a whole segment in memory — so seeking (which fires many
   concurrent segment fetches) stays smooth even on tiny VPS instances.
4. Tags segments with `Cache-Control: public, max-age=3600, immutable` so
   seeking back into already-played territory uses the browser cache instead
   of re-hitting the upstream.
5. Forces `Accept-Encoding: identity` upstream so segment bytes are never
   gzip-wrapped — that was the source of the v0.x `bufferAppendError` storm
   in hls.js (Content-Length lied about the body size after silent inflation).

### Subtitles (soft subs)

`/api/watch` returns a `subtitles` array (Animetsu sometimes returns it as
an object or `null` — the API normalizes the response, the demo player
normalizes the shape). Each entry has `{ label, url, lang, default }`.

To enable user-toggleable soft subs in any HTML5 player:

```html
<video controls crossorigin="anonymous">
  <track kind="subtitles" srclang="en" label="English" src="<sub.url>" default>
  <track kind="subtitles" srclang="es" label="Español"  src="<sub.url>">
</video>
```

The browser's native CC button toggles tracks on and off — exactly what
"soft subs" means. The demo player at `/demo` ships a "Soft subtitles"
language picker that drives `track.mode = 'showing' | 'disabled'` directly,
so you can flip languages without re-loading.

---

## Configuration (env vars)

Every value has a sane default. See `.env.example`.

| Var | Default | What it does |
|---|---|---|
| `PORT` | `8080` | TCP port (most PaaS injects this) |
| `API_KEY` | _empty_ | If set, every request needs `X-API-Key: <value>` (proxies + health stay open) |
| `RATE_LIMIT_RPM` | `60` | Per-IP requests/minute. Set `0` to disable |
| `CACHE_DEFAULT_TTL` | `10m` | In-memory cache TTL when a handler doesn't override |
| `UPSTREAM_TIMEOUT` | `15s` | Per-request timeout for JSON calls (NOT proxy streams) |
| `UPSTREAM_BASE` | `https://animetsu.live/v2` | Override only if Animetsu changes hostnames |
| `HLS_PROXY_BASE` | `https://mega-cloud.top/proxy` | Same |
| `UPSTREAM_REFERER` | `https://animetsu.live/` | Forwarded upstream |
| `UPSTREAM_USER_AGENT` | _Chrome 146 desktop_ | Forwarded upstream |
| `LOG_LEVEL` | `info` | `trace` / `debug` / `info` / `warn` / `error` |

---

## Deployment — 6 ways

### 1. Railway (easiest, button deploy)

`deploy/railway.json` (and the root `railway.toml`) tell Railway to use the
Dockerfile. Just push your fork and connect it on
<https://railway.app/new>. Set `PORT` to `8080` (Railway also injects its
own; the app reads `$PORT`).

Health check path: `/healthz`.

### 2. Fly.io

```bash
fly launch --copy-config --name animetsu-api --no-deploy
fly deploy
```

`fly.toml` already specifies the internal port, health checks, and
auto-stop/auto-start so the machine costs $0 when idle.

### 3. Render

Add a new **Web Service** pointing at your repo. Render reads
`deploy/render.yaml`. Plan: free. Health check: `/healthz`.

### 4. Koyeb

```bash
koyeb app init animetsu-api -i your-github/your-fork --ports 8080:http \
  --routes /:8080 --instance-type free --regions fra
```

Or import `deploy/koyeb.yaml` from the dashboard.

### 5. Docker (any host)

```bash
docker build -t animetsu-api .
docker run -d --name animetsu-api -p 8080:8080 \
  -e RATE_LIMIT_RPM=120 -e LOG_LEVEL=info \
  --restart unless-stopped animetsu-api
```

The image is `gcr.io/distroless/static-debian12:nonroot` — no shell, no
package manager, ~13 MB on disk.

### 6. VPS / self-host with systemd + Caddy

On any cheap VPS (Hetzner CPX11, OVH VPS Starter, Contabo, …):

```bash
# 1. Build the binary on your laptop and scp it over
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" \
  -o animetsu-api ./cmd/server
scp animetsu-api root@your-vps:/usr/local/bin/

# 2. Create a system user and a systemd unit on the server
sudo useradd --system --no-create-home --shell /usr/sbin/nologin animetsu
sudo tee /etc/systemd/system/animetsu-api.service >/dev/null <<'UNIT'
[Unit]
Description=Animetsu API
After=network.target

[Service]
ExecStart=/usr/local/bin/animetsu-api
User=animetsu
Restart=always
RestartSec=3
Environment=PORT=8080
Environment=RATE_LIMIT_RPM=120
Environment=LOG_LEVEL=info
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
UNIT

sudo systemctl daemon-reload
sudo systemctl enable --now animetsu-api
sudo systemctl status animetsu-api
```

Front it with Caddy for free TLS + HTTP/2:

```caddy
api.example.com {
    reverse_proxy 127.0.0.1:8080
    encode zstd gzip
}
```

That's it. `caddy reload` and you have HTTPS.

---

## Using the API from your app

The killer feature is `proxy_url` in every `/api/watch` source — a fully
absolute URL on **your own** host that any media player can hit without
custom headers and without CORS hoops.

### JavaScript / TypeScript (web, hls.js)

```ts
const r = await fetch(
  `https://your-api.example.com/api/watch?id=${id}&ep=${ep}&server=auto&source_type=sub`
).then(r => r.json());

const best = r.data.sources.find(s => /1080/.test(s.quality)) || r.data.sources[0];
const video = document.querySelector("video")!;

if (best.type.includes("mpegurl")) {
  // pin hls.js to 1.5.x — 1.6.x has a worker race with AES-128 keys
  const hls = new Hls({ enableWorker: false });
  hls.loadSource(best.proxy_url);
  hls.attachMedia(video);
} else {
  video.src = best.proxy_url;
}

// subtitles (when present)
for (const s of r.data.subtitles ?? []) {
  const t = document.createElement("track");
  t.kind = "subtitles"; t.label = s.label; t.srclang = s.lang || ""; t.src = s.url;
  if (s.default) t.default = true;
  video.appendChild(t);
}
```

### React Native (expo-av / react-native-video)

```tsx
<Video
  source={{ uri: source.proxy_url }}
  resizeMode="contain"
  shouldPlay
  textTracks={(subs ?? []).map(s => ({
    title: s.label, language: s.lang, type: "text/vtt", uri: s.url,
  }))}
/>
```

### Kotlin / Android (ExoPlayer / Media3)

```kotlin
val mediaItem = MediaItem.Builder()
    .setUri(source.proxyUrl)
    .setMimeType(if (source.type.contains("mpegurl"))
        MimeTypes.APPLICATION_M3U8 else MimeTypes.VIDEO_MP4)
    .setSubtitleConfigurations(subs.map {
        MediaItem.SubtitleConfiguration.Builder(Uri.parse(it.url))
            .setMimeType(MimeTypes.TEXT_VTT)
            .setLanguage(it.lang)
            .setLabel(it.label)
            .build()
    })
    .build()

val player = ExoPlayer.Builder(context).build()
player.setMediaItem(mediaItem)
player.prepare()
player.play()
```

No custom `OkHttpClient` needed — the proxy handles `Referer` for you.

### Swift / iOS (AVPlayer)

```swift
let url = URL(string: source.proxyURL)!
let asset = AVURLAsset(url: url)             // m3u8 OR mp4 — AVPlayer auto-detects
let item = AVPlayerItem(asset: asset)
let player = AVPlayer(playerItem: item)
player.play()
```

For sideloaded subtitle tracks, attach `AVMediaSelectionGroup` from a
custom `AVAsset` resource loader, or burn them in via ffmpeg server-side.

### Flutter (video_player / better_player)

```dart
final controller = BetterPlayerController(
  BetterPlayerConfiguration(autoPlay: true),
  betterPlayerDataSource: BetterPlayerDataSource(
    BetterPlayerDataSourceType.network,
    source.proxyUrl,
    videoFormat: source.type.contains("mpegurl")
        ? BetterPlayerVideoFormat.hls
        : BetterPlayerVideoFormat.other,
    subtitles: subs.map((s) => BetterPlayerSubtitlesSource(
      type: BetterPlayerSubtitlesSourceType.network,
      urls: [s.url], name: s.label, selectedByDefault: s.default_,
    )).toList(),
  ),
);
```

### Python (yt-dlp / requests)

```bash
yt-dlp "$(curl -s 'https://your-api.example.com/api/anime/<id>/download/<ep>?quality=1080p' \
  | jq -r '.data.proxy_url')" -o '%(title)s.%(ext)s'
```

### curl + ffmpeg (download an episode)

```bash
URL=$(curl -s 'https://your-api.example.com/api/anime/<id>/download/<ep>?quality=1080p' \
  | jq -r '.data.proxy_url')
ffmpeg -i "$URL" -c copy episode.mp4
```

For local soft-subs, fetch them from the same `/api/watch` payload and
pass each track to `ffmpeg` to produce an MKV with toggleable subtitles
(VLC / mpv / Infuse all honour the language flag):

```bash
ffmpeg -i "$URL" -i "https://.../sub.en.vtt" \
  -map 0 -map 1 -c copy -c:s srt \
  -metadata:s:s:0 language=eng \
  episode.mkv
```

---

## Security notes

- **API key is optional.** Set `API_KEY=...` and every request must include
  `X-API-Key: ...` (or `?key=...`). Liveness probes and `/api/proxy/*` stay
  open so embedded players keep working without leaking the key into the
  client bundle.
- **Rate limiting** is per-IP, in-memory, token-bucket. Fine for one node;
  put a real WAF in front if you expose it publicly at scale.
- The proxy will only fetch `http(s)://` URLs — no `file://` or `gopher://`
  shenanigans — but it does NOT validate the host. If you're worried about
  SSRF in your environment, add an allow-list to `proxy/hls_proxy.go`.

---

## License

MIT — see [LICENSE](LICENSE). Do whatever you want, just don't blame ME.
