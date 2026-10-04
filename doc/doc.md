# go-image-server - Documentation

> Back to [README](../README.md)

## Prerequisites

- Docker and Docker Compose (recommended deployment)
- For building from source: Go 1.21+, libvips with `pkg-config`; AVIF output requires libheif / libavif
- 404 placeholder images: `app/internal/assets/image/404-light.svg` and `404-dark.svg` (the `**/image/` rule in `.gitignore` excludes this directory, so provide them yourself)
- Optional: a Cloudflare account to deploy `worker/index.js` as the CDN layer

## Installation

### Docker Compose

```bash
git clone https://github.com/pardnchiu/go-image-server.git
cd go-image-server
docker compose up -d
```

Nginx listens on `8080` publicly; the Go service listens on `8080` inside `frontend_network` only. Image data is mounted from `./image` in the project root.

### From Source

```bash
git clone https://github.com/pardnchiu/go-image-server.git
cd go-image-server/app
go build -o main ./cmd/server
./main
```

Upload, cache, trash, and 404 placeholder paths all resolve against the **current working directory** (`os.Getwd()`), so run the binary from `app/`.

## Configuration

### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `GO_ENV` | No | — | When `development`, the upload response `src` uses `http://localhost:{PORT}` |
| `PORT` | No | `8080` | Go service listen port |
| `DOMAIN` | Yes in production | — | Outside development mode, the upload response `src` uses `https://{DOMAIN}` |

### Storage Layout

| Path (relative to working directory) | Purpose |
|--------------------------------------|---------|
| `storage/image/upload/` | Original uploads |
| `storage/image/cache/` | Transformed cache files, mirroring `upload/` |
| `storage/image/upload/.trash/YYYY-MM-DD/` | Destination for deleted files / folders |
| `internal/assets/image/404-{light,dark}.svg` | Placeholder returned on read failure |

### Nginx (`config/nginx/nodes.conf`)

| Route | Behavior |
|-------|----------|
| `/c/img/` | `proxy_cache` stores 200 / 301 / 302 / 304 responses for 7 days (`inactive=30d`, `max_size=2g`) and adds `X-Cache-Status` |
| `/upload/` | No caching; uncomment `allow` / `deny` to restrict source IPs |
| `/del/` | Uncomment `allow` / `deny` to restrict source IPs |
| Global | `client_max_body_size 100M`; methods other than GET / HEAD / POST / DELETE / PUT / OPTIONS return `444` |

### Cloudflare Worker (`worker/index.js`)

Replace `[URL]` in `new URL(url.pathname, "[URL]")` with the image server origin before deploying. The worker handles only paths ending in `.jpg` / `.jpeg` / `.png` / `.webp` / `.svg`, caches them for 7 days keyed by the full query string, and returns `400` for everything else.

## Usage

### Basic: Upload and Read

```bash
# Upload to storage/image/upload/blog/2025
curl -X POST \
  -F "filepath=@./photo.jpg" \
  http://localhost:8080/upload/blog/2025

# Read with default parameters (WebP, quality 75)
curl -o photo.webp \
  "http://localhost:8080/c/img/blog/2025/ERftP1gTS7WCTeJ8_1744080848530.jpg"
```

Successful upload response (`201`):

```json
{
  "success": 1,
  "filename": "ERftP1gTS7WCTeJ8_1744080848530.jpg",
  "type": "image/jpeg",
  "size": 2501808,
  "src": "http://localhost:8080/c/img/blog/2025/ERftP1gTS7WCTeJ8_1744080848530.jpg"
}
```

### Advanced: Transform Parameters

```bash
# Short edge 480 px, AVIF, quality 60
curl -o thumb.avif \
  "http://localhost:8080/c/img/blog/2025/ERftP1gTS7WCTeJ8_1744080848530.jpg?s=480&t=avif&q=60"

# Width 800 px, blur 10, brightness 0.8, dark 404 placeholder on failure
curl -o cover.webp \
  "http://localhost:8080/c/img/blog/2025/ERftP1gTS7WCTeJ8_1744080848530.jpg?w=800&b=10&B=0.8&d=1"

# Return the original file
curl -o origin.jpg \
  "http://localhost:8080/c/img/blog/2025/ERftP1gTS7WCTeJ8_1744080848530.jpg?o=1"
```

### Advanced: Delete and Restore

```bash
# Delete a single file
curl -X DELETE \
  http://localhost:8080/del/blog/2025/ERftP1gTS7WCTeJ8_1744080848530.jpg

# Delete an entire folder
curl -X DELETE http://localhost:8080/del/blog/2025
```

```json
{
  "success": 1,
  "message": "move path to: /go/src/app/storage/image/upload/.trash/2026-10-04/ERftP1gTS7WCTeJ8_1744080848530.jpg"
}
```

The trash keeps only the **file / folder name** (not the original parent path); a same-day name collision appends a millisecond timestamp. To restore, move the item back to its original location under `storage/image/upload/`.

### Error Handling Example

```bash
# Check the upload result by HTTP status code
status=$(curl -s -o response.txt -w "%{http_code}" \
  -X POST -F "filepath=@./photo.jpg" \
  http://localhost:8080/upload/blog/2025)

if [ "$status" != "201" ]; then
  echo "upload failed ($status): $(cat response.txt)" >&2
  exit 1
fi
```

## API Reference

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/c/img/*path` | Read and transform by parameters, writing the result to the local cache |
| `POST` | `/upload/*path` | Upload via `multipart/form-data` into the given folder |
| `DELETE` | `/del/*path` | Move a file or folder into the trash |
| `GET` | `/check/state` | Health check, returns `ok` |
| Any | Other paths | `404` with `404 Not Found` |

`*path` may contain multiple `/` segments.

### `GET /c/img/*path` Parameters

| Param | Alias | Default | Description |
|-------|-------|---------|-------------|
| `o` | `origin` | `0` | `1` returns the original file and ignores all transform parameters |
| `s` | `size` | — | Short-edge length (px); takes precedence over `w` / `h` |
| `w` | `width` | — | Width (px) |
| `h` | `height` | — | Height (px) |
| `q` | `quality` | `75` (`50` for AVIF) | Output quality, clamped to 0–100; ignored for PNG |
| `t` | `type` | `webp` | `avif` / `webp` / `jpg` / `jpeg` / `png`; invalid values fall back to `webp` |
| `b` | `blur` | `0` | Gaussian blur sigma, clamped to 0–100 |
| `B` | `bright` | `1` | Brightness multiplier (linear) |
| `d` | `dark` | — | `1` or `dark` switches error responses to the dark 404 placeholder |

**Sizing rules:**

| Condition | Behavior |
|-----------|----------|
| `s` present | Short edge scaled to `min(s, original short edge)`, aspect ratio kept |
| Only `w` or only `h` | That edge scaled to `min(value, original)`, aspect ratio kept |
| Both `w` and `h` | Scale factor derived from `min(w, original width)`, aspect ratio kept |
| None, and short edge > 1024 px | Long edge scaled to 1024 px |
| Otherwise | Original dimensions |

No rule ever upscales the original.

**Special files:** `.pdf` / `.svg` paths always stream the original file without transformation.

**Response headers:** `Cache-Control: public, max-age=604800` with a matching `Expires` (7 days).

**Cache file naming:** `storage/image/cache/{path without extension}_{size}_{q}_{b}_{B}.{t}`, where the size segment is `{s}`, `{w}_{h}`, `{w}_auto`, `auto_{h}`, or `auto_auto` depending on the parameters.

**Errors:** A missing original or failed transform returns the 404 placeholder SVG (`Cache-Control: no-cache`); processing longer than 30 seconds returns `408 timed out`.

### `POST /upload/*path`

| Field | Type | Description |
|-------|------|-------------|
| `filepath` | file | File to upload; the type is taken from the part's `Content-Type` |

Supported types: `image/jpeg`, `image/jpg`, `image/png`, `image/webp`, `image/svg+xml`, `application/pdf`. Filenames follow `{16 alphanumerics}_{millisecond timestamp}.{ext}`.

| Status | Body | Case |
|--------|------|------|
| `201` | JSON (`success` / `filename` / `type` / `size` / `src`) | Upload succeeded |
| `400` | `please assign a path first` | No folder specified |
| `400` | `can not get file form request` | Missing `filepath` field |
| `400` | `can not create folder: ...` / `can not save file` | Folder creation or write failed |
| `500` | gin Recovery default response | Unsupported file type (the error branch currently carries a nil `err`, triggering a panic) |
| `408` | `timed out` | Processing exceeded 30 seconds |

### `DELETE /del/*path`

| Status | Body | Case |
|--------|------|------|
| `200` | `{"success":1,"message":"move path to: ..."}` | File moved to trash |
| `200` | `{"success":1,"message":"move folder to: ..."}` | Folder moved to trash |
| `400` | `please assign a path first` | No path specified |
| `400` | `path not found: ...` | File or folder does not exist |
| `400` | `can not create folder: ...` / `can not move file: ...` | Trash creation or move failed |
| `408` | `timed out` | Processing exceeded 30 seconds |

***

©️ 2025 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
