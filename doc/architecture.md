# go-image-server - Architecture

> Back to [README](../README.md)

## Overview

```mermaid
graph TB
    Client[Client] --> Browser[Browser Cache<br>max-age 7d]
    Browser --> Worker[Cloudflare Worker<br>caches.default 7d]
    Browser --> Nginx[Nginx<br>proxy_cache 7d]
    Worker --> Nginx
    Nginx --> Gin[Gin Router]
    Gin --> Get[GetFromPath]
    Gin --> Post[PostToPath]
    Gin --> Del[DeleteFromPath]
    Gin --> Health[/check/state/]
    Get --> Vips[libvips Transform]
    Get --> Cache[(storage/image/cache)]
    Get --> Upload[(storage/image/upload)]
    Post --> Upload
    Del --> Trash[(upload/.trash/YYYY-MM-DD)]
```

## Module: routes

Maps HTTP methods and paths to handlers; unknown paths return `404 Not Found`.

```mermaid
graph LR
    subgraph routes
        SetRoutes --> SetGET
        SetRoutes --> SetPOST
        SetRoutes --> SetDELETE
        SetRoutes --> Set404[NoRoute → Set404]
    end
    SetGET --> H1["GET /check/state → ok"]
    SetGET --> H2["GET /c/img/*path → GetFromPath"]
    SetPOST --> H3["POST /upload/*path → PostToPath"]
    SetDELETE --> H4["DELETE /del/*path → DeleteFromPath"]
```

## Module: GetFromPath

Parses query parameters, serves a cache hit or transforms via libvips and writes back to the cache, then streams the response with an adaptive buffer.

```mermaid
graph TB
    subgraph GetFromPath
        Query["GetQuery<br>s w h o d b B t q"] --> TypeCheck[CheckImageType<br>invalid → webp]
        TypeCheck --> Header["Set Cache-Control / Expires 7d"]
        Header --> Special{".pdf / .svg / o=1?"}
        Special -- Yes --> StreamFile[streamFile original]
        Special -- No --> CachePath[GetFileCachePath]
        CachePath --> Hit{Cache file exists?}
        Hit -- Yes --> StreamImage[streamImage]
        Hit -- No --> Worker[goroutine + 30s timeout]
        Worker --> Size[GetNewSize]
        Size --> Resize[Resize Lanczos3]
        Resize --> Blur[GaussianBlur ← GetBlur]
        Blur --> Bright[Linear brightness]
        Bright --> Export["Export AVIF / WebP / JPG / PNG<br>← GetQuality"]
        Export --> Save[Write cache file]
        Save --> StreamImage
    end
    Worker -- Failure --> Err[HandleGetError<br>404 placeholder SVG]
    Worker -- Timeout --> Timeout[408 timed out]
    StreamImage --> Buffer["streamBuffer<br>≤50K:2K ≤500K:4K ≤2M:8K else:16K"]
    StreamFile --> Buffer
```

## Module: PostToPath

Writes the multipart file into the target folder and returns a directly usable image URL.

```mermaid
graph TB
    subgraph PostToPath
        Path{Empty path?} -- Yes --> E400[400 please assign a path first]
        Path -- No --> Mkdir[MkdirAll upload/path]
        Mkdir --> Form[FormFile filepath]
        Form --> Type{CheckUploadType}
        Type -- Supported --> Name["UUID(16)_millis.GetExtension"]
        Name --> SaveFile[SaveUploadedFile]
        SaveFile --> Src["GetDomain + /c/img/path/filename"]
    end
    Src --> R201[201 JSON]
    Config[configs<br>GO_ENV / PORT / DOMAIN] --> Src
```

## Module: DeleteFromPath

Replaces deletion with a move, bucketed by date with collision handling.

```mermaid
graph TB
    subgraph DeleteFromPath
        Path{Empty path?} -- Yes --> E400[400]
        Path -- No --> Stat[os.Stat for existence and dir check]
        Stat --> Mkdir["MkdirAll .trash/YYYY-MM-DD"]
        Mkdir --> Exists{Same name in trash?}
        Exists -- Yes --> Rename["Append _millis timestamp"]
        Exists -- No --> Move[os.Rename]
        Rename --> Move
    end
    Move --> R200["200 move path / folder to: ..."]
```

## Module: Edge Layer (Nginx + Cloudflare Worker)

Intercepts repeated requests in front of the Go service and only falls through to the origin on a miss.

```mermaid
graph LR
    subgraph Cloudflare Worker
        Ext{Extension is<br>jpg/jpeg/png/webp/svg?} -- No --> W400[400]
        Ext -- Yes --> Key[Build cache key from query string]
        Key --> CFHit{caches.default hit?}
        CFHit -- Yes --> HIT[CF-Cache-Status: HIT]
        CFHit -- No --> Fetch[Fetch origin and cache.put 7d]
    end
    subgraph Nginx
        Loc["/c/img/ → proxy_cache images_cache"]
        Up["/upload/ /del/ → pass-through, optional IP allowlist"]
    end
    Fetch --> Loc
    Loc --> Go[Go service :8080]
    Up --> Go
```

## Data Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant W as Cloudflare Worker
    participant N as Nginx
    participant G as GetFromPath
    participant FS as storage/image
    participant V as libvips
    C->>W: GET /c/img/a/b.jpg?s=480&t=avif
    alt Worker cache hit
        W-->>C: Image (HIT)
    else Miss
        W->>N: Fetch origin
        alt Nginx cache hit
            N-->>W: Image
        else Miss
            N->>G: proxy_pass
            G->>FS: Read cache/a/b_480___1.avif
            alt Cache file exists
                FS-->>G: Cached bytes
            else Not found
                G->>FS: Read upload/a/b.jpg
                G->>V: Resize / Blur / Linear / ExportAvif
                V-->>G: buffer
                G->>FS: Write cache file
            end
            G-->>N: Chunked stream
            N-->>W: Image (stored in proxy_cache)
        end
        W-->>C: Image (MISS, stored in caches.default)
    end
```

***

©️ 2025 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
