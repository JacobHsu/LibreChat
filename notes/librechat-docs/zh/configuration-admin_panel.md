---
title: 管理面板（Admin Panel）
icon: LayoutDashboard
description: 用內建的 Docker Compose 檔或以獨立服務方式執行 LibreChat 管理面板，並把它指向你的 LibreChat API。
---

> [!WARNING]
> **非官方翻譯**：官方沒有這一頁的中文版（中文網址 `/zh/docs/configuration/admin_panel` 會自動轉到英文版）。
> 本檔由英文原文 [../en/configuration-admin_panel.md](../en/configuration-admin_panel.md) 翻成繁體中文，原文出處：
> <https://www.librechat.ai/docs/configuration/admin_panel>（[原始檔](https://github.com/LibreChat-AI/librechat.ai/blob/408527900701f755d2817a02d0c9c6befe0f57df/content/docs/configuration/admin_panel.mdx)，commit `4085279`）。
> 正文只翻譯原文；另外在「與 LibreChat 一起部署」加了一段 📝 本機附註，說明 admin-panel 一直重啟時要怎麼設定。程式碼、指令、環境變數名稱、API 路徑保留英文；原文的 `<OptionTable>` 元件改寫成一般表格以便閱讀。內容有疑義時以英文原文為準。

[管理面板](/docs/features/admin_panel)是一個獨立服務，透過 LibreChat 的 `/api/admin` 端點運作。它需要 LibreChat v0.8.5 以上、面板能透過網路連到 LibreChat API，以及一個具有 `ADMIN` 角色或 `access:admin` 權限的帳號。在單租戶部署中，第一個註冊的帳號就是管理員（租戶範圍的部署則另外指派管理員）；要提升其他人為管理員，請見 [Access Control 設定](/docs/configuration/access_control#promote-another-admin)。

## 與 LibreChat 一起部署

官方的 `docker-compose.yml` 和 `deploy-compose.yml` 都包含 `admin-panel` 服務（映像 `registry.librechat.ai/librechat-ai/librechat-admin-panel:latest`），會跟整個 stack 一起啟動。

| Compose 檔 | 管理面板網址 | 提供方式 |
|---|---|---|
| `docker-compose.yml` | `http://localhost:3000` | 發布在主機 port `ADMIN_PANEL_PORT`（預設 `3000`） |
| `deploy-compose.yml` | `http://admin.localhost` | 經由內建的 nginx，以 `admin.localhost` 子網域轉送 |

沒有 session secret 時，面板會拒絕啟動。請在 LibreChat 的 `.env` 設定它；compose 檔會把它以 `SESSION_SECRET` 的名稱傳給面板：

```bash filename=".env"
# 至少 32 個字元。產生方式：openssl rand -hex 32
ADMIN_PANEL_SESSION_SECRET=replace-with-a-32-char-random-string

# 選填：預設 docker-compose.yml 使用的主機 port
# ADMIN_PANEL_PORT=3000

# 選填：面板透過 HTTPS 提供時設為 true
# ADMIN_PANEL_SESSION_COOKIE_SECURE=false
```

> [!NOTE]
> **📝 本機附註：admin-panel 一直重啟（Restarting）怎麼處理**
>
> **現象**：`docker compose ps` 顯示 `admin-panel` 為 `Restarting`，`docker compose logs admin-panel` 不斷出現：
> ```text
> [admin-panel] SESSION_SECRET must be set to at least 32 characters (got 0). Refusing to start.
> error: script "start" exited with code 1
> ```
>
> **原因**：
> 1. 從 `.env.example` 複製的 `.env` 中，`ADMIN_PANEL_SESSION_SECRET=` 是空的。
> 2. compose 把它以 `SESSION_SECRET` 傳給面板（`docker-compose.yml` 的 `SESSION_SECRET=${ADMIN_PANEL_SESSION_SECRET}`），面板啟動時發現長度是 0，就自行結束。
> 3. compose 對這個服務設了 `restart: always`，所以容器結束後立刻被重啟，形成迴圈。
>
> LibreChat 本身（http://localhost:3080）不受影響，只是 http://localhost:3000 打不開。
>
> **設定步驟**（在 repo 根目錄）：
> 1. 產生至少 32 個字元的隨機字串。要在 **Git Bash** 執行，PowerShell 沒有 `openssl`：
>    ```bash
>    openssl rand -hex 32
>    ```
>    會輸出 64 個十六進位字元。
> 2. 打開 repo 根目錄的 `.env`，找到 `ADMIN_PANEL_SESSION_SECRET=` 這一行（約在第 80 行），把剛產生的字串貼在等號後面：
>    ```bash
>    ADMIN_PANEL_SESSION_SECRET=<貼上剛產生的 64 個字元>
>    ```
>    不要加引號、前後不要留空白。這個值不要提交到 git（`.env` 已被 `.gitignore` 排除）。
> 3. 套用變更：
>    ```bash
>    docker compose up -d
>    ```
>    compose 偵測到 admin-panel 的環境變數改變，只會重建這個容器。注意 `docker compose restart admin-panel` **不會**重新讀取 `.env`，一定要用 `up -d`。
> 4. 確認：
>    ```bash
>    docker compose ps admin-panel        # 應變成 Up ... (healthy)
>    docker compose logs admin-panel --tail 20
>    ```
>    然後打開 http://localhost:3000。
>
> **登入**：使用在 LibreChat（http://localhost:3080）**第一個註冊**的帳號，它會自動成為 `ADMIN`。面板本身沒有註冊功能，所以要先到 LibreChat 註冊。
>
> **選填**：想在 LibreChat 的 **Settings > General** 看到 Admin Panel 連結，再加 `ADMIN_PANEL_URL=http://localhost:3000`。預設的 `docker-compose.yml` 不會幫你設這個值。
>
> **官方出處**：
> - 必須設 secret、compose 怎麼傳遞：本頁上方的 `.env` 範例（英文原文 [../en/configuration-admin_panel.md:23-34](../en/configuration-admin_panel.md#L23-L34)）
> - repo 的 `.env.example` 第 76-80 行註解：「Generate a unique value before starting the stack: openssl rand -hex 32」
> - `docker-compose.yml` 第 50-52 行註解：為什麼沒設時只有面板失敗，而不是整個 compose 失敗
> - 面板原始碼 [server.ts:12-25](https://github.com/LibreChat-AI/admin-panel/blob/2d03f110bde2dd1d3bd6c38f4e5e06da912981c4/server.ts#L12-L25)：印出這行錯誤訊息並結束的程式碼

其餘設定由 compose 檔負責：`API_SERVER_URL` 指向 `api` 服務，`VITE_API_BASE_URL` 跟隨 `DOMAIN_CLIENT`，用於瀏覽器端的重新導向；`deploy-compose.yml` 還會在 API 上把 `ADMIN_PANEL_URL` 設為 `http://admin.localhost`，讓管理員在 SSO 之後被導回面板。若不需要面板，可以移除 `admin-panel` 服務，或把它放到 Compose 的 [profile](https://docs.docker.com/compose/how-tos/profiles/) 之後。

瀏覽器會把 `*.localhost` 解析成 `127.0.0.1`，所以不用修改 hosts 檔，`admin.localhost` 就能使用。若要用真正的網域，請把 DNS 記錄指向主機、修改 `client/nginx.conf` 中 `admin.localhost` 的 `server_name`，並把 `ADMIN_PANEL_URL` 設成相符的值。

## 獨立部署

若要單獨執行面板，並指向其他地方的 LibreChat，可以直接使用映像：

```bash
docker run -p 3000:3000 \
  --add-host=host.docker.internal:host-gateway \
  -e SESSION_SECRET=replace-with-a-32-char-random-string \
  -e VITE_API_BASE_URL=http://localhost:3080 \
  -e API_SERVER_URL=http://host.docker.internal:3080 \
  -e SESSION_COOKIE_SECURE=false \
  registry.librechat.ai/librechat-ai/librechat-admin-panel:latest
```

面板需要兩個 LibreChat 位址。`VITE_API_BASE_URL` 是**瀏覽器**連到 LibreChat 的位址，用於登入時的重新導向。`API_SERVER_URL` 是**面板自己的伺服器**連到 LibreChat 的位址。在容器裡 `localhost` 指的是容器本身，所以 LibreChat 跑在同一台主機時，`API_SERVER_URL` 要用 `http://host.docker.internal:3080`（在 Linux 上需要上面的 `--add-host` 參數），而瀏覽器仍然用 `http://localhost:3080`。正式環境中，兩者都設成各自那一端能解析的 DNS 名稱。

[admin-panel repository](https://github.com/LibreChat-AI/admin-panel) 也有自己的 `docker-compose.yml`，以及用 Bun 從原始碼執行的說明（`bun install`，然後 `bun dev`）。

## 面板的環境變數

這些設在**管理面板服務**上，不是設在 LibreChat。

| 變數 | 型別 | 說明 | 範例 |
|---|---|---|---|
| `SESSION_SECRET` | string | Session 加密金鑰，至少 32 個字元。Docker 與正式環境必填；只有 `bun dev` 會退回內建的開發用 secret。內建 compose 檔中，這個值來自 `ADMIN_PANEL_SESSION_SECRET`。 | `SESSION_SECRET=replace-with-a-32-char-random-string` |
| `VITE_API_BASE_URL` | string | 瀏覽器端使用的 LibreChat API 網址，用於 OAuth 重新導向。Docker 必填。本機開發預設 `http://localhost:3080`。 | `VITE_API_BASE_URL=https://chat.example.com` |
| `API_SERVER_URL` | string | 面板伺服器呼叫 LibreChat 的網址，在與瀏覽器端網址不同時使用（例如內部的 Kubernetes service）。未設定時退回 `VITE_API_BASE_URL`。 | `API_SERVER_URL=http://librechat:3080` |
| `PORT` | number | 面板監聽的 port。預設：3000。 | `PORT=3000` |
| `VITE_BASE_PATH` | string | 讓面板在子路徑下提供服務。建置時與執行時必須相同，所以自訂子路徑需要自行建置映像。預設：`/`。 | `VITE_BASE_PATH=/adminpanel` |
| `ADMIN_SSO_ONLY` | boolean | 隱藏 email 與密碼表單，讓管理員只能用 SSO 登入。預設：false。 | `ADMIN_SSO_ONLY=false` |
| `ADMIN_SSO_ENABLED` | boolean | 設為 false 時，即使 LibreChat 已設定 OpenID，也隱藏 SSO 按鈕並跳過 SSO 自動導向。優先於 `ADMIN_SSO_ONLY`。預設：true。 | `ADMIN_SSO_ENABLED=true` |
| `ADMIN_SESSION_IDLE_TIMEOUT_MS` | number | 管理員 session 閒置多久（毫秒）後結束。預設：1800000（30 分鐘）。 | `ADMIN_SESSION_IDLE_TIMEOUT_MS=1800000` |
| `SESSION_COOKIE_SECURE` | boolean | session cookie 是否只在 HTTPS 傳送。正式環境預設為 true。只有純 HTTP 部署才設為 false，否則瀏覽器會丟棄 cookie，OpenID 登入無法完成。 | `SESSION_COOKIE_SECURE=false` |
| `ADMIN_PANEL_TRUSTED_ORIGINS` | string | 允許呼叫面板伺服器的公開來源，以逗號分隔。在純 HTTP、且反向代理會改寫 Host 標頭或去掉非預設 port 時需要；否則登入會回 403。 | `ADMIN_PANEL_TRUSTED_ORIGINS=http://admin.example.com:8080` |
| `ADMIN_PANEL_METRICS_SECRET` | string | 抓取 Prometheus `/metrics` 端點所需的 Bearer token。未設定時該端點回 401。 | `ADMIN_PANEL_METRICS_SECRET=` |

### 快取標頭

這些對應 LibreChat 自己的快取變數。每個變數的 `ADMIN_PANEL_` 版本優先；未設定時，面板會退回共用的名稱。

| 變數 | 型別 | 說明 | 範例 |
|---|---|---|---|
| `STATIC_CACHE_MAX_AGE` | number | `/assets/` 中帶 hash 檔案的瀏覽器 max-age（秒）。`ADMIN_PANEL_STATIC_CACHE_MAX_AGE` 會覆蓋它。預設：172800（2 天）。 | `STATIC_CACHE_MAX_AGE=172800` |
| `STATIC_CACHE_S_MAX_AGE` | number | 帶 hash 資源的 CDN s-maxage（秒）。`ADMIN_PANEL_STATIC_CACHE_S_MAX_AGE` 會覆蓋它。預設：86400（1 天）。 | `STATIC_CACHE_S_MAX_AGE=86400` |
| `INDEX_CACHE_CONTROL` | string | HTML 頁面的 Cache-Control 標頭。`ADMIN_PANEL_INDEX_CACHE_CONTROL` 會覆蓋它。 | `INDEX_CACHE_CONTROL=no-cache, no-store, must-revalidate` |
| `INDEX_PRAGMA` | string | HTML 頁面的 Pragma 標頭。`ADMIN_PANEL_INDEX_PRAGMA` 會覆蓋它。 | `INDEX_PRAGMA=no-cache` |
| `INDEX_EXPIRES` | string | HTML 頁面的 Expires 標頭。`ADMIN_PANEL_INDEX_EXPIRES` 會覆蓋它。 | `INDEX_EXPIRES=0` |

## LibreChat 端的設定

### 重新導向網址

面板和 LibreChat 使用不同網址時，請在 LibreChat 的 `.env` 把 `ADMIN_PANEL_URL` 設為面板的對外基礎網址，包含路徑前綴、結尾不加斜線。LibreChat 用它在 SSO 之後把管理員導回面板；設定之後，管理員也會在 **Settings > General** 看到 **Admin Panel** 連結。

```bash filename=".env"
ADMIN_PANEL_URL=https://admin.example.com/admin
```

使用 Helm chart 時，改在 values 檔設定 `librechat.adminPanelUrl`；chart 會把它轉成 `ADMIN_PANEL_URL`。

```yaml filename="values.yaml"
librechat:
  adminPanelUrl: https://admin.example.com/admin
```

### 單一登入（SSO）

管理員可以用 LibreChat 所使用的同一個 OpenID Connect provider 登入面板。請在身分提供者那邊，把 `${DOMAIN_SERVER}/api/admin/oauth/openid/callback` 註冊為額外的 redirect URI。provider 的其餘設定請見 [OpenID Connect](/docs/configuration/authentication/OAuth2-OIDC)。要依 IdP 的角色或群組授予 `ADMIN` 角色，使用 `OPENID_ADMIN_ROLE`（列在 [.env 參考](/docs/configuration/dotenv#openid-connect)）。

Google 登入面板使用 `GOOGLE_CLIENT_ID` 和 `GOOGLE_CLIENT_SECRET`。請向 Google 註冊 `${DOMAIN_SERVER}/api/admin/oauth/google/callback`。面板會要求離線存取權限，以便在管理員 token 過期時不必重新登入就能刷新；每次刷新都會再次檢查帳號的身分、停權狀態、允許的 email 網域、租戶與管理權限。

## 設定覆蓋（Configuration overrides）

覆蓋設定依主體（角色、群組或使用者）各自存成 profile，疊在 `librechat.yaml` 之上；另有一個套用到所有人的 Base profile。

**合併順序。** 不論優先序，Base profile 都最先套用，接著其餘 profile 依優先序由小到大套用；優先序相同時，依角色、群組、使用者、建立順序決定。建立時沒指定優先序的 profile，會取得該主體類型的預設值：角色 10、群組 20、使用者 100（Base 為 0）。之後的寫入若省略優先序，會保留已儲存的值。

**驗證。** 寫入（`PUT /api/admin/config/:principalType/:principalId` 與 `PATCH .../fields`）會套用在基礎設定之上，並以 `librechat.yaml` 的 schema 驗證。結果無效時以 HTTP 400 拒絕，什麼都不會儲存：

```json
{
  "error": "Invalid config override",
  "code": "CONFIG_OVERRIDE_INVALID",
  "issues": [{ "path": "interface.fileSearch", "code": "invalid_type" }]
}
```

每個 issue 只有路徑和代碼（schema 代碼如 `invalid_type`，或 `missing_merge_key`、`duplicate_merge_key`、`indexed_merge_key_write`、`union_dropped_key`、`invalid_document`），絕不包含送出的值。欄位 patch 只會因為它碰到或造成的問題而被拒絕。

之後才變成無效的已儲存覆蓋（例如升級後 schema 變嚴格）不會造成致命錯誤。解析設定時會丟掉無效欄位、改用下層的值，伺服器會為每個欄位記一行 log：

```text
[mergeConfigOverrides] Ignoring invalid override "<path>" for <principalType>/<principalId> (<code>)
```

不符合 [theme 規則](/docs/configuration/librechat_yaml/object_structure/theme#validation)的覆蓋主題，也會以同樣方式退回基礎主題。

**機密資料。** 透過 admin API 寫入的已註冊機密欄位會加密儲存，之後只以遮罩預覽回傳：Langfuse、OCR、語音、網頁搜尋、Assistants、Azure Assistants、MCP server 憑證、Azure 群組金鑰，以及每個 `endpoints.custom[].apiKey`。之後編輯時不填機密欄位會保留已儲存的值；但若寫入會改變已存金鑰的傳送目的地（例如 server URL 或 base URL），就必須重新填入金鑰。自訂 endpoint 的金鑰會保留 `${ENV_VAR}` 與 `user_provided` 原樣。自訂標頭這類通用 map 中的任意值不會加密，請在那裡使用環境變數引用。

**環境變數引用。** 覆蓋中的 `${ENV_VAR}` 只限於 `librechat.yaml` 已經引用的名稱、內建 endpoint 的憑證，以及列在 `CONFIG_OVERRIDE_ENV_ALLOWLIST` 中的名稱。其他引用在寫入時會被拒絕。

**只能寫在 YAML 的設定。** `type: stdio` 或帶有程序欄位（`command`、`args`、`env`、`cwd`、`stderr`）的 MCP server，以及 `langfuse.headers`，只能在 `librechat.yaml` 設定。admin API 會拒絕對它們的寫入，覆蓋也無法取代或移除它們。請見 [Langfuse: Authenticated Proxies and Gateways](/docs/configuration/langfuse#authenticated-proxies-and-gateways)。

**由部署決定的設定。** Base、角色、群組與使用者 profile 都不能更改 `persistence`、`clientLimits`、`telemetry.rum`、`telemetry.stackTraceLimit`、`conversationList`、`passwordHashing`、`interface.colorModeFromUrl`、`fileConfig.skills.deletionCleanupConcurrency`、`endpoints.agents.managementApi`、`endpoints.agents.channels.slackApp` 或 `mcpSettings.oauthFailureRetentionMs`。請在部署的 YAML 中設定它們。請查看管理設定目錄（catalog）以及寫入報告中「已套用／已略過」的清單：儲存 profile 並不代表僅限部署的值已經生效。

**依區段的存取權限。** 設定權限可以是全面的（`read:configs`、`manage:configs`），也可以只限一個頂層區段（`read:configs:<section>`、`manage:configs:<section>`）。建立、修改或刪除 Base profile 一律需要全面的 `manage:configs`；依區段的管理權限只適用於角色、群組與使用者 profile。請見 [Admin capabilities](/docs/configuration/access_control#admin-capabilities)。

## Admin API

面板會呼叫下列 LibreChat 端點。每個路由都會檢查管理權限與相應的能力。

| 掛載路徑 | 用途 |
|---|---|
| `POST /api/admin/login`、`/api/admin/oauth/*` | 管理員登入（本機帳號與 SSO） |
| `GET /api/admin/verify` | 驗證管理員 session |
| `/api/admin/users` | 使用者列表與搜尋 |
| `/api/admin/groups` | 群組與成員 |
| `/api/admin/roles` | 自訂角色、權限與成員 |
| `/api/admin/grants` | 系統能力授予 |
| `/api/admin/config` | Base 與各主體的設定 profile |
| `/api/admin/skills` | GitHub skill 同步狀態、執行紀錄與憑證 |
| `/api/admin/code-environments` | 配對與撤銷程式碼執行環境 worker |
| `/api/admin/langfuse` | Langfuse 連線設定與測試 |
| `/api/admin/audit-log` | 授權稽核紀錄：`GET /`（列表）、`GET /export.csv`、`GET /verify`（完整性檢查）、`GET /:id`。需要 `read:audit_log`；沒有寫入端點 |

設定讀取：`GET /api/admin/config/meta`（部署能力）、`/catalog`（範圍與模型選項）、`/env-status`（環境變數引用狀態）、`/effective`（呼叫者的實際生效設定），以及 `/resolved/:principalType/:principalId`（某主體解析後的設定）。讀取同樣維持授權檢查與機密遮罩。

`POST /api/admin/config/:principalType/:principalId/changes` 會以原子方式對單一 profile 套用多項變更，並回傳寫入報告。請檢查報告：已套用與已略過的位置分開列出，僅限部署的路徑可能被略過，但不會讓整批失敗。

`POST /api/admin/config/reload` 會重新載入本機設定，並廣播失效通知給其他實例。本機重新載入成功後，若廣播失敗，它可能回傳 `503`；這不代表已回滾，其他副本可能還沒重新載入。請見[升級指南](/docs/configuration/upgrade#admin-configuration-and-secrets)。

群組與使用者預覽、以及有效使用者的讀取，都在目標所屬的租戶內解析，包括該租戶的 Base profile。租戶範圍的呼叫者無法存取其他租戶，角色名稱也是指呼叫者自己的分區。
