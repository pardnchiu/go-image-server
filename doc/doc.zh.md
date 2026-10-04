# go-image-server - 技術文件

> 返回 [README](./README.zh.md)

## 前置需求

- Docker 與 Docker Compose（建議部署方式）
- 自行編譯時：Go 1.21 以上、libvips（含 `pkg-config`），輸出 AVIF 需 libheif／libavif
- 404 佔位圖：`app/internal/assets/image/404-light.svg` 與 `404-dark.svg`（`.gitignore` 的 `**/image/` 規則會排除此目錄，需自行放置）
- 選用：Cloudflare 帳號（部署 `worker/index.js` 作為 CDN 層）

## 安裝

### Docker Compose

```bash
git clone https://github.com/pardnchiu/go-image-server.git
cd go-image-server
docker compose up -d
```

啟動後 Nginx 對外監聽 `8080`，Go 服務僅在內部網路 `frontend_network` 監聽 `8080`。圖片資料掛載於專案根目錄的 `./image`。

### 從原始碼編譯

```bash
git clone https://github.com/pardnchiu/go-image-server.git
cd go-image-server/app
go build -o main ./cmd/server
./main
```

上傳、快取、回收桶與 404 佔位圖路徑皆以**當前工作目錄**為根（`os.Getwd()`），必須在 `app/` 目錄下執行。

## 設定

### 環境變數

| 變數 | 必要 | 預設值 | 說明 |
|------|------|--------|------|
| `GO_ENV` | 否 | — | 設為 `development` 時，上傳回應的 `src` 使用 `http://localhost:{PORT}` |
| `PORT` | 否 | `8080` | Go 服務監聽埠 |
| `DOMAIN` | 正式環境必要 | — | 非 development 模式下，上傳回應的 `src` 使用 `https://{DOMAIN}` |

### 儲存目錄

| 路徑（相對於工作目錄） | 用途 |
|------------------------|------|
| `storage/image/upload/` | 原始上傳檔 |
| `storage/image/cache/` | 轉檔後的快取檔，目錄結構對應 `upload/` |
| `storage/image/upload/.trash/YYYY-MM-DD/` | 被刪除檔案／資料夾的回收位置 |
| `internal/assets/image/404-{light,dark}.svg` | 讀取失敗時回傳的佔位圖 |

### Nginx（`config/nginx/nodes.conf`）

| 路由 | 行為 |
|------|------|
| `/c/img/` | `proxy_cache` 快取 200／301／302／304 回應 7 天（`inactive=30d`、`max_size=2g`），回應加上 `X-Cache-Status` |
| `/upload/` | 不快取；可取消註解 `allow`／`deny` 限制來源 IP |
| `/del/` | 可取消註解 `allow`／`deny` 限制來源 IP |
| 全域 | `client_max_body_size 100M`；非 GET／HEAD／POST／DELETE／PUT／OPTIONS 直接回 `444` |

### Cloudflare Worker（`worker/index.js`）

部署前將 `new URL(url.pathname, "[URL]")` 中的 `[URL]` 換成圖片伺服器的來源網址。Worker 僅處理副檔名為 `.jpg`／`.jpeg`／`.png`／`.webp`／`.svg` 的路徑，並以完整 query string 作為快取 key 快取 7 天；其他路徑回 `400`。

## 使用方式

### 基礎：上傳與讀取

```bash
# 上傳至 storage/image/upload/blog/2025
curl -X POST \
  -F "filepath=@./photo.jpg" \
  http://localhost:8080/upload/blog/2025

# 以預設參數讀取（WebP、品質 75）
curl -o photo.webp \
  "http://localhost:8080/c/img/blog/2025/ERftP1gTS7WCTeJ8_1744080848530.jpg"
```

上傳成功回應（`201`）：

```json
{
  "success": 1,
  "filename": "ERftP1gTS7WCTeJ8_1744080848530.jpg",
  "type": "image/jpeg",
  "size": 2501808,
  "src": "http://localhost:8080/c/img/blog/2025/ERftP1gTS7WCTeJ8_1744080848530.jpg"
}
```

### 進階：轉檔參數

```bash
# 短邊 480 px、AVIF、品質 60
curl -o thumb.avif \
  "http://localhost:8080/c/img/blog/2025/ERftP1gTS7WCTeJ8_1744080848530.jpg?s=480&t=avif&q=60"

# 寬 800 px、模糊 10、亮度 0.8，失敗時回傳深色 404 佔位圖
curl -o cover.webp \
  "http://localhost:8080/c/img/blog/2025/ERftP1gTS7WCTeJ8_1744080848530.jpg?w=800&b=10&B=0.8&d=1"

# 回傳原始檔
curl -o origin.jpg \
  "http://localhost:8080/c/img/blog/2025/ERftP1gTS7WCTeJ8_1744080848530.jpg?o=1"
```

### 進階：刪除與還原

```bash
# 刪除單一檔案
curl -X DELETE \
  http://localhost:8080/del/blog/2025/ERftP1gTS7WCTeJ8_1744080848530.jpg

# 刪除整個資料夾
curl -X DELETE http://localhost:8080/del/blog/2025
```

```json
{
  "success": 1,
  "message": "move path to: /go/src/app/storage/image/upload/.trash/2026-10-04/ERftP1gTS7WCTeJ8_1744080848530.jpg"
}
```

回收桶僅保留**檔名／資料夾名**（不保留原始父路徑）；同日同名時自動附加毫秒時間戳。還原時將檔案移回 `storage/image/upload/` 下的原位置即可。

### 錯誤處理範例

```bash
# 以 HTTP 狀態碼判斷上傳結果
status=$(curl -s -o response.txt -w "%{http_code}" \
  -X POST -F "filepath=@./photo.jpg" \
  http://localhost:8080/upload/blog/2025)

if [ "$status" != "201" ]; then
  echo "upload failed ($status): $(cat response.txt)" >&2
  exit 1
fi
```

## API 參考

### 端點

| 方法 | 路徑 | 說明 |
|------|------|------|
| `GET` | `/c/img/*path` | 讀取並依參數轉檔，結果寫入本地快取 |
| `POST` | `/upload/*path` | 以 `multipart/form-data` 上傳至指定資料夾 |
| `DELETE` | `/del/*path` | 將檔案或資料夾移入回收桶 |
| `GET` | `/check/state` | 健康檢查，回傳 `ok` |
| 任意 | 其他路徑 | `404`，回傳 `404 Not Found` |

`*path` 可包含多層 `/`。

### `GET /c/img/*path` 參數

| 參數 | 別名 | 預設值 | 說明 |
|------|------|--------|------|
| `o` | `origin` | `0` | `1` 時直接回傳原始檔，忽略其他轉檔參數 |
| `s` | `size` | — | 短邊長度（px），優先於 `w`／`h` |
| `w` | `width` | — | 寬度（px） |
| `h` | `height` | — | 高度（px） |
| `q` | `quality` | `75`（AVIF 為 `50`） | 輸出品質，限制於 0–100；PNG 不套用 |
| `t` | `type` | `webp` | `avif`／`webp`／`jpg`／`jpeg`／`png`；不合法值回退為 `webp` |
| `b` | `blur` | `0` | 高斯模糊 sigma，限制於 0–100 |
| `B` | `bright` | `1` | 亮度倍率（線性乘數） |
| `d` | `dark` | — | `1` 或 `dark` 時，錯誤回應使用深色 404 佔位圖 |

**尺寸規則：**

| 條件 | 行為 |
|------|------|
| 有 `s` | 短邊縮至 `min(s, 原短邊)`，等比例縮放 |
| 只有 `w` 或只有 `h` | 該邊縮至 `min(值, 原長度)`，等比例縮放 |
| `w` 與 `h` 皆有 | 以 `min(w, 原寬)` 計算縮放比，等比例縮放 |
| 皆無且短邊 > 1024 px | 長邊縮至 1024 px |
| 其他 | 維持原尺寸 |

所有規則皆不放大原圖。

**特殊檔案：** 副檔名 `.pdf`／`.svg` 一律直接串流原始檔，不經轉檔。

**回應標頭：** `Cache-Control: public, max-age=604800` 與對應的 `Expires`（7 天）。

**快取檔命名：** `storage/image/cache/{原路徑去副檔名}_{尺寸}_{q}_{b}_{B}.{t}`，尺寸段依參數為 `{s}`、`{w}_{h}`、`{w}_auto`、`auto_{h}` 或 `auto_auto`。

**錯誤：** 原檔不存在或轉檔失敗時回傳 404 佔位 SVG（`Cache-Control: no-cache`）；處理超過 30 秒回 `408 timed out`。

### `POST /upload/*path`

| 欄位 | 型別 | 說明 |
|------|------|------|
| `filepath` | file | 上傳檔案；依該 part 的 `Content-Type` 判斷類型 |

支援類型：`image/jpeg`、`image/jpg`、`image/png`、`image/webp`、`image/svg+xml`、`application/pdf`。檔名格式為 `{16 位英數}_{毫秒時間戳}.{副檔名}`。

| 狀態碼 | 內容 | 情境 |
|--------|------|------|
| `201` | JSON（`success`／`filename`／`type`／`size`／`src`） | 上傳成功 |
| `400` | `please assign a path first` | 未指定資料夾 |
| `400` | `can not get file form request` | 缺少 `filepath` 欄位 |
| `400` | `can not create folder: ...`／`can not save file` | 建立資料夾或寫檔失敗 |
| `500` | gin Recovery 預設回應 | 不支援的檔案類型（目前錯誤分支未帶 `err`，觸發 panic） |
| `408` | `timed out` | 處理超過 30 秒 |

### `DELETE /del/*path`

| 狀態碼 | 內容 | 情境 |
|--------|------|------|
| `200` | `{"success":1,"message":"move path to: ..."}` | 檔案已移入回收桶 |
| `200` | `{"success":1,"message":"move folder to: ..."}` | 資料夾已移入回收桶 |
| `400` | `please assign a path first` | 未指定路徑 |
| `400` | `path not found: ...` | 檔案或資料夾不存在 |
| `400` | `can not create folder: ...`／`can not move file: ...` | 回收桶建立或搬移失敗 |
| `408` | `timed out` | 處理超過 30 秒 |

***

©️ 2025 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
