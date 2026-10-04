# go-image-server - 架構

> 返回 [README](./README.zh.md)

## 概覽

```mermaid
graph TB
    Client[客戶端] --> Browser[瀏覽器快取<br>max-age 7 天]
    Browser --> Worker[Cloudflare Worker<br>caches.default 7 天]
    Browser --> Nginx[Nginx<br>proxy_cache 7 天]
    Worker --> Nginx
    Nginx --> Gin[Gin 路由]
    Gin --> Get[GetFromPath]
    Gin --> Post[PostToPath]
    Gin --> Del[DeleteFromPath]
    Gin --> Health[/check/state/]
    Get --> Vips[libvips 轉檔]
    Get --> Cache[(storage/image/cache)]
    Get --> Upload[(storage/image/upload)]
    Post --> Upload
    Del --> Trash[(upload/.trash/YYYY-MM-DD)]
```

## Module: 路由（routes）

將 HTTP 方法與路徑對應到 handler，未知路徑回 `404 Not Found`。

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

## Module: 讀取與轉檔（GetFromPath）

解析 query 參數、命中快取檔或交由 libvips 轉檔並寫回快取，最後以自適應 buffer 串流回應。

```mermaid
graph TB
    subgraph GetFromPath
        Query["GetQuery<br>s w h o d b B t q"] --> TypeCheck[CheckImageType<br>不合法 → webp]
        TypeCheck --> Header["設定 Cache-Control / Expires 7 天"]
        Header --> Special{".pdf / .svg / o=1？"}
        Special -- 是 --> StreamFile[streamFile 串流原始檔]
        Special -- 否 --> CachePath[GetFileCachePath]
        CachePath --> Hit{快取檔存在？}
        Hit -- 是 --> StreamImage[streamImage]
        Hit -- 否 --> Worker[goroutine + 30 秒逾時]
        Worker --> Size[GetNewSize]
        Size --> Resize[Resize Lanczos3]
        Resize --> Blur[GaussianBlur ← GetBlur]
        Blur --> Bright[Linear 亮度]
        Bright --> Export["Export AVIF / WebP / JPG / PNG<br>← GetQuality"]
        Export --> Save[寫入快取檔]
        Save --> StreamImage
    end
    Worker -- 失敗 --> Err[HandleGetError<br>404 佔位 SVG]
    Worker -- 逾時 --> Timeout[408 timed out]
    StreamImage --> Buffer["streamBuffer<br>≤50K:2K ≤500K:4K ≤2M:8K 其餘:16K"]
    StreamFile --> Buffer
```

## Module: 上傳（PostToPath）

將 multipart 檔案寫入指定資料夾並回傳可直接引用的圖片網址。

```mermaid
graph TB
    subgraph PostToPath
        Path{path 為空？} -- 是 --> E400[400 please assign a path first]
        Path -- 否 --> Mkdir[MkdirAll upload/path]
        Mkdir --> Form[FormFile filepath]
        Form --> Type{CheckUploadType}
        Type -- 支援 --> Name["UUID(16)_毫秒.GetExtension"]
        Name --> SaveFile[SaveUploadedFile]
        SaveFile --> Src["GetDomain + /c/img/path/filename"]
    end
    Src --> R201[201 JSON]
    Config[configs<br>GO_ENV / PORT / DOMAIN] --> Src
```

## Module: 刪除（DeleteFromPath）

以搬移取代刪除，依日期分桶並處理同名衝突。

```mermaid
graph TB
    subgraph DeleteFromPath
        Path{path 為空？} -- 是 --> E400[400]
        Path -- 否 --> Stat[os.Stat 判斷存在與是否為資料夾]
        Stat --> Mkdir["MkdirAll .trash/YYYY-MM-DD"]
        Mkdir --> Exists{回收桶已有同名？}
        Exists -- 是 --> Rename["名稱附加 _毫秒時間戳"]
        Exists -- 否 --> Move[os.Rename]
        Rename --> Move
    end
    Move --> R200["200 move path / folder to: ..."]
```

## Module: 邊緣層（Nginx + Cloudflare Worker）

在 Go 服務前攔截重複請求，僅未命中時回源。

```mermaid
graph LR
    subgraph Cloudflare Worker
        Ext{副檔名為<br>jpg/jpeg/png/webp/svg？} -- 否 --> W400[400]
        Ext -- 是 --> Key[以 query string 組快取 key]
        Key --> CFHit{caches.default 命中？}
        CFHit -- 是 --> HIT[CF-Cache-Status: HIT]
        CFHit -- 否 --> Fetch[回源並 cache.put 7 天]
    end
    subgraph Nginx
        Loc["/c/img/ → proxy_cache images_cache"]
        Up["/upload/ /del/ → 直通，可設 IP 白名單"]
    end
    Fetch --> Loc
    Loc --> Go[Go 服務 :8080]
    Up --> Go
```

## 資料流

```mermaid
sequenceDiagram
    participant C as 客戶端
    participant W as Cloudflare Worker
    participant N as Nginx
    participant G as GetFromPath
    participant FS as storage/image
    participant V as libvips
    C->>W: GET /c/img/a/b.jpg?s=480&t=avif
    alt Worker 快取命中
        W-->>C: 圖片（HIT）
    else 未命中
        W->>N: 回源
        alt Nginx 快取命中
            N-->>W: 圖片
        else 未命中
            N->>G: proxy_pass
            G->>FS: 讀取 cache/a/b_480___1.avif
            alt 快取檔存在
                FS-->>G: 快取內容
            else 不存在
                G->>FS: 讀取 upload/a/b.jpg
                G->>V: Resize / Blur / Linear / ExportAvif
                V-->>G: buffer
                G->>FS: 寫入快取檔
            end
            G-->>N: chunked 串流
            N-->>W: 圖片（寫入 proxy_cache）
        end
        W-->>C: 圖片（MISS，寫入 caches.default）
    end
```

***

©️ 2025 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
