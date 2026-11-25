---
title: Docker
icon: Docker
description: How to install LibreChat locally with Docker, verify your setup, and configure custom endpoints
---

> [!NOTE]
> **官方原文副本** — 線上版：<https://www.librechat.ai/docs/local/docker>（中文版：<https://www.librechat.ai/zh/docs/local/docker>）
> 原始檔：[LibreChat-AI/librechat.ai `content/docs/local/docker.mdx`](https://github.com/LibreChat-AI/librechat.ai/blob/408527900701f755d2817a02d0c9c6befe0f57df/content/docs/local/docker.mdx)，commit `4085279`，2026-11-25 下載。
> 標示 **📝 本機附註** 的區塊是這台機器實際安裝時與官方不同、或官方沒寫到的地方；其餘為原文，未改動。
> `<Callout>`、`<Steps>` 等是官網的 MDX 元件，這裡會以原始標籤顯示。

For most scenarios, Docker Compose is the recommended installation method due to its simplicity, ease of use, and reliability.

## Prerequisites

- [`Git`](https://git-scm.com/downloads)
- [`Docker`](https://www.docker.com/products/docker-desktop/)

Docker Desktop is recommended for most users. For remote server installations, see the [Ubuntu Docker Deployment Guide](/docs/remote/docker_linux).

### System Requirements

LibreChat does not run AI models itself; it sends requests to the providers you configure, so it does not need a GPU.

LibreChat's containers need at least 1 GiB of RAM and 1 vCPU, the same [minimum](/docs/remote#minimum-requirements) as on a server. The default Docker Compose file starts MongoDB, Meilisearch (conversation search), and the RAG API with a PostgreSQL/pgvector database (file search) alongside LibreChat. It also defines the admin panel, which only starts once `ADMIN_PANEL_SESSION_SECRET` is set in `.env` (see [Admin Panel](/docs/features/admin_panel)). With everything running, make at least 2 GB of memory available to Docker for smoother operation.

If you use Docker Desktop, it runs containers inside its own virtual machine, so your computer also has to meet Docker Desktop's requirements. At the time of writing, those are at least 4 GB of system RAM on [macOS](https://docs.docker.com/desktop/setup/install/mac-install/) and [Linux](https://docs.docker.com/desktop/setup/install/linux/), and 8 GB on [Windows](https://docs.docker.com/desktop/setup/install/windows-install/), plus hardware virtualization. Check the linked pages for the current requirements. The memory available to containers is set in Docker Desktop's [resource settings](https://docs.docker.com/desktop/settings-and-maintenance/settings/) on macOS, Linux, and Windows with Hyper-V; with the Windows WSL 2 backend, it is set on the WSL 2 virtual machine instead (see [Docker's WSL guidance](https://docs.docker.com/desktop/features/wsl/best-practices/)). Docker Engine installed directly on Linux runs containers on the host and has no separate virtual machine.

If you connect LibreChat to models running on the same machine, such as [Ollama](/docs/configuration/librechat_yaml/ai_endpoints/ollama), size the hardware for the model you choose. That model runtime needs far more memory, and often a GPU, than LibreChat does.

<Callout type="warn" title="CPUs without AVX support">

The MongoDB image used by LibreChat's default Docker Compose setup requires a CPU with AVX instructions. On older x86 CPUs without AVX (and on some virtual machines that do not expose AVX to the guest), MongoDB crashes on startup.

**Fix:** Create a `docker-compose.override.yml` to use an older MongoDB image that does not require AVX:

```yaml filename="docker-compose.override.yml"
services:
  mongodb:
    image: mongo:4.4.18
```

See the [Docker Override guide](/docs/configuration/docker_override) for more details.

</Callout>

> [!NOTE]
> **📝 本機附註：前置需求檢查結果**
> - Git 2.55、Docker Desktop 29.9（WSL 2 backend）、Docker Compose v5.7。官方「Port Already in Use」裡的 `!override` 需要 Compose 2.24.4 以上，這版符合。
> - Docker 可用記憶體約 31 GB、24 核心，遠高於官方的 2 GB 建議。
> - x86 CPU 支援 AVX，上面的 AVX 注意事項不適用，`mongo:8.0.20` 可直接使用。

## Installation

<Steps>
  <Step>

### Clone the Repository

```bash
git clone https://github.com/LibreChat-AI/LibreChat.git
cd LibreChat
```

> [!NOTE]
> **📝 本機附註：跳過 clone，但多了一步「讓出容器名稱」**
> - 這個 repo 本身就是 LibreChat 的 fork，已在本機，所以不用 clone。
> - 官方沒寫到的衝突：`docker-compose.yml` 把 `container_name` 寫死（`LibreChat`、`chat-mongodb`、`admin-panel`、`vectordb`、`rag_api`、`chat-meilisearch`）。
>   Docker 的容器名稱在**所有容器（包含已停止的）**中必須唯一，所以本機另一套同樣用官方 compose 起的 LibreChat stack（libre-chat）只 `stop` 不夠，必須在它的目錄執行 `docker compose down`（**不加 `-v`**）把容器移除。
>   資料（bind mount 的 `data-node/`、`meili_data_*/`，以及 named volume）與映像都會保留，要恢復就回該目錄 `docker compose up -d --no-build`。
>   兩套 stack 因為容器名稱相同，**不能同時執行**。
> - 另一套 `libre-chat-sep` 在 override 裡把容器名稱加上 `-sep` 後綴、port 用 3081/3001，所以不衝突。這也是「同一台機器跑多套」的做法：用 override 改 `container_name` 與 port。

  </Step>
  <Step>

### Create Your Environment File

```bash
cp .env.example .env
```

The default `.env` file works out of the box for a basic setup. For in-depth configuration, see the [.env reference](/docs/configuration/dotenv).

<Callout type="info" title="Windows">

On Windows, use `copy .env.example .env` if `cp` is not available.

</Callout>

> [!NOTE]
> **📝 本機附註：「預設 `.env` 就能跑」背後發生了什麼**
> - 在 Git Bash 執行 `cp .env.example .env`。`.env` 已被 `.gitignore`（`.env*`）排除。
> - **`.env` 一定要先建立**：`docker-compose.yml` 用長語法 `type: bind` 把 `./.env` 掛進容器，檔案不存在時 compose 會直接報錯（bind source path does not exist），容器起不來。
> - `.env.example` 裡 `CREDS_KEY`、`CREDS_IV`、`JWT_SECRET`、`JWT_REFRESH_SECRET` 是**空的**。啟動時 log 出現：
>   ```text
>   warn: [credentials] No configured value was found for CREDS_KEY, CREDS_IV, JWT_SECRET, JWT_REFRESH_SECRET.
>         Temporary credentials from /app/data/.env.temp are being used.
>   info: [credentials] New database detected. Credential fingerprints were recorded for future key-drift checks.
>   ```
>   也就是 LibreChat 自己產生一組暫時金鑰，存在 named volume `librechat_librechat-data`（掛在容器 `/app/data`）。這就是官方說「開箱即用」的原因。
> - 這組金鑰用來加密使用者存的 API key、簽發登入 token。之後才改成正式值，key-drift 檢查會發現金鑰變了：先前加密的資料解不開、登入 session 失效。
>   **建議在註冊第一個帳號前就寫入正式值。** 官方產生器：<https://www.librechat.ai/toolkit/creds-generator>，或在 Git Bash 用 `openssl rand -hex 32`（`CREDS_IV` 用 `openssl rand -hex 16`）。
> - ✅ **本機實際做法**（[issue #3](https://github.com/JacobHsu/LibreChat/issues/3)）：帳號在設定正式金鑰前就註冊了，所以**沒有**產生新金鑰，而是把暫時金鑰原封不動固定到 `.env`。
>   - 為什麼不產生新的：第一次啟動時，LibreChat 已把金鑰指紋寫進 MongoDB 的 `librechatCredentialMetadata`（`packages/api/src/app/checks.ts`）。換成新金鑰會讓登入 session 失效，而且之後每次啟動都會出現 `Active fingerprints ... do not match the database credential record` 警告。程式碼明確要求「Do not overwrite the database marker」，所以不能靠刪掉那筆紀錄來消除警告。
>   - 為什麼可以直接沿用：暫時金鑰本來就是隨機產生的強金鑰，格式正確；寫進 `.env` 後指紋仍吻合。`.env` 的值優先於 `.env.temp`（`packages/api/src/credentials.ts` 的 `bootstrapCredentials`），之後 `.env` 就是唯一來源，刪掉 volume 也不會遺失金鑰。
>   - 做法：從容器讀 `/app/data/.env.temp`，用 `sed` 寫入 `.env` 的 4 行（不印出值），比對兩邊完全一致後，`docker compose up -d`（rag_api 會重建）再 `docker compose restart api`。
>   - 驗證：重啟後 log 沒有 `Temporary credentials` 和 fingerprint mismatch 警告；既有帳號可以登入 LibreChat 和 Admin Panel。
>   - **Git Bash 陷阱**：`docker compose exec api cat /app/data/.env.temp` 會失敗（`can't open 'C:/Program Files/Git/app/data/.env.temp'`），因為 Git Bash 把看起來像 Unix 路徑的參數自動轉成 Windows 路徑。要寫成 `docker compose exec api sh -c 'cat /app/data/.env.temp'`，或在指令前加 `MSYS_NO_PATHCONV=1`。
> - 另外 `.env.example` 預設 `SEARCH=false`：Meilisearch 容器會啟動，但對話搜尋預設不啟用；要用需改 `SEARCH=true` 並設定 `MEILI_MASTER_KEY`。

  </Step>
  <Step>

### Start LibreChat

```bash
docker compose up -d
```

The first launch pulls Docker images and may take a few minutes. Subsequent starts are much faster.

> [!NOTE]
> **📝 本機附註：實際啟動結果**
> - 只下載了 3 個 registry 映像（`librechat-dev`、`librechat-admin-panel`、`librechat-rag-api-dev-lite`）；`mongo:8.0.20`、`meilisearch:v1.35.1`、`pgvector` 本機已有快取。
> - 建立了 named volume `librechat_pgdata2`（向量資料庫）與 `librechat_librechat-data`（暫時金鑰等）。前綴 `librechat_` 是 compose project 名稱，取自資料夾名稱。
>   MongoDB / Meilisearch 資料則以 bind mount 存在 repo 根目錄的 `data-node/`、`meili_data_v1.35.1/`（都已被 gitignore）。
> - 每個 compose 指令都會印出 `The "UID" variable is not set` / `"GID"` 警告。在 Windows 的 Docker Desktop 上可忽略：`user: "${UID}:${GID}"` 變成空值，容器以映像預設使用者執行。
> - 預設只有 LibreChat（3080）和 admin-panel（3000）對外開 port；MongoDB、Meilisearch、vectordb、rag_api 只在 Docker 內部網路，主機連不到。要用 GUI 連 MongoDB 得在 override 加 `ports`（見 Docker Override 附註）。

  </Step>
  <Step>

### Verify and Log In

Open your browser and visit [http://localhost:3080](http://localhost:3080). You should see the LibreChat login page.

<Callout type="info" title="First Account = Admin in Single-Tenant Deployments">

In an unscoped single-tenant deployment, the first account you register becomes the admin account. There are no default credentials; you create your own username and password during registration. Tenant-scoped deployments do not auto-promote the first user in each tenant; provision tenant administrators through a trusted administrative flow.

</Callout>

Click **Sign up** to create your account, then follow [Your First Chat](/docs/quick_start/first_chat) to add a model API key and send your first message.

> [!NOTE]
> **📝 本機附註：驗證時看到的狀態**
> - `curl http://localhost:3080/` 回 HTTP 200，api log 出現 `Server listening on all interfaces at port 3080`、`Connected to MongoDB`、`RAG API is running and reachable at http://rag_api:8000`。
> - api log 有一行 `error: Config file YAML format is invalid: ENOENT: no such file or directory, open '/app/librechat.yaml'`。這是**預期中的**：基本安裝沒有掛載 `librechat.yaml`，LibreChat 改用內建預設設定。要自訂才需要做下面的「Mounting librechat.yaml」。
> - `admin-panel` 容器狀態為 `Restarting`，log：`SESSION_SECRET must be set to at least 32 characters (got 0). Refusing to start.` 也是預期中的，見 [features-admin_panel.md](features-admin_panel.md)。已在 [issue #1](https://github.com/JacobHsu/LibreChat/issues/1) 設定 `ADMIN_PANEL_SESSION_SECRET` 解決。
> - 第一個註冊的帳號會成為 ADMIN，admin panel 也是用這個帳號登入，所以註冊前先把金鑰設好（見上方 `.env` 附註）。
> - 登入前 api log 可能出現 `[refreshController] Invalid refresh token: invalid signature`，接著才是 `Login successful`。這不是故障：瀏覽器的 cookie 依主機名稱區分、**不分 port**，之前跑在 `localhost:3080` 的另一套 LibreChat 用不同金鑰簽發的 refresh token cookie 還留在瀏覽器裡，新的 stack 驗證不了，重新登入後就會被新的 cookie 取代。這個錯誤在固定金鑰之前（仍使用暫時金鑰時）就出現過，與 issue #3 無關。
> - Windows 查 log：`docker compose logs api --tail 50`；持續追蹤加 `-f`。

  </Step>
</Steps>

## Mounting librechat.yaml

To use a custom `librechat.yaml` configuration file with Docker, you need to mount it as a volume so the container can access it.

Copy the example override file and edit it:

```bash
cp docker-compose.override.yml.example docker-compose.override.yml
```

Ensure the librechat.yaml volume mount is uncommented in `docker-compose.override.yml`:

```yaml filename="docker-compose.override.yml"
services:
  api:
    volumes:
      - type: bind
        source: ./librechat.yaml
        target: /app/librechat.yaml
```

Restart for changes to take effect:

```bash
docker compose down && docker compose up -d
```

For full setup instructions including creating the file from scratch, see the [librechat.yaml guide](/docs/configuration/librechat_yaml). For more override options, see the [Docker override guide](/docs/configuration/docker_override).

> [!NOTE]
> **📝 本機附註：為什麼一定要「掛載」**
> - repo 的 `.dockerignore` 排除了 `librechat*`，而且預設 compose 用的是 registry 的預建映像，所以 `librechat.yaml` 不會出現在映像裡，只能用 volume 掛進去。
> - 上面用的是長語法 `type: bind`：來源檔案不存在時 compose 會直接報錯。Docker Override 頁的短語法 `./librechat.yaml:/app/librechat.yaml` 在來源不存在時會**建立一個同名資料夾**，之後 LibreChat 讀到的是資料夾、而不是設定檔。所以先從 `librechat.example.yaml` 建立 `librechat.yaml`，再啟動。
> - `librechat.yaml` 與 `docker-compose.override.yml` 都已被 `.gitignore` 排除。
> - 修改 `librechat.yaml` 後只需重啟 api：`docker compose restart api`；改了 override 或 `.env` 才需要 `docker compose up -d`（會重建受影響的容器）。
> - 寫完可用官方 YAML 檢查器驗證：<https://www.librechat.ai/toolkit/yaml-checker>（原文 Troubleshooting 的連結 `/toolkit/yaml_checker` 用底線，實際網址是連字號）。

## Updating LibreChat

The following commands will fetch the latest LibreChat project changes, including any necessary changes to the docker compose files, as well as the latest prebuilt images.

<Callout type="info" title="Permissions">

You may need to prefix commands with `sudo` according to your environment permissions.

</Callout>

<Callout type="info" title="Update script">

The repository's `npm run update:docker` script rebuilds locally built images but does not pull the prebuilt images that the default `docker-compose.yml` uses, so follow the manual steps below to update a default install.

</Callout>

```bash filename="Stop the running container(s)"
docker compose down
```

```bash filename="Remove all existing docker images"
# Linux/Mac
docker images -a | grep "librechat" | awk '{print $3}' | xargs docker rmi

# Windows (PowerShell)
docker images -a --filter "reference=registry.librechat.ai/librechat-ai/librechat*" --format "{{.ID}}" | ForEach-Object { docker rmi $_ }
docker images -a --filter "reference=ghcr.io/librechat-ai/librechat*" --format "{{.ID}}" | ForEach-Object { docker rmi $_ }
```

```bash filename="Pull latest project changes"
git pull
```

```bash filename="Pull the latest LibreChat image"
docker compose pull
```

```bash filename="Start LibreChat"
docker compose up
```

> [!NOTE]
> **📝 本機附註：這台機器更新時要注意**
> - 刪映像請用上面的 **Windows (PowerShell)** 版本，它只篩選官方 registry 的映像。Linux/Mac 版的 `grep "librechat"` 會一併刪掉本機其他 stack 自己建置的映像（例如 `librechat-custom:local`、`librechat-sep:local`）。
> - 這個 repo 是 fork，`git pull` 拉的是自己的 fork（`origin`）。要跟上官方需另外加 upstream remote 再合併，例如 `git remote add upstream https://github.com/LibreChat-AI/LibreChat.git`、`git fetch upstream`。
> - 最後一步 `docker compose up` 沒有 `-d`，會佔住終端機顯示 log，`Ctrl+C` 會停止容器；想在背景執行就加 `-d`。
> - `docker compose down` **不要加 `-v`**：加了會刪掉 named volume（向量資料庫與暫時金鑰）。

## Troubleshooting

### Port Already in Use

If you see an error like `bind: address already in use` for port 3080, another application is using that port.

Either stop the conflicting application, or change the port LibreChat uses. The default `docker-compose.yml` publishes `"${PORT}:${PORT}"`, and ports added in an override file are appended to that list instead of replacing it, so an override alone does not free port 3080.

The simplest fix is to set `PORT` in your `.env` file, which changes both the port LibreChat listens on and the published port:

```bash filename=".env"
PORT=3081
```

Alternatively, with Docker Compose 2.24.4 or later, replace the published port in `docker-compose.override.yml` with the `!override` tag (the container keeps listening on 3080):

```yaml filename="docker-compose.override.yml"
services:
  api:
    ports: !override
      - "3081:3080"
```

Restart with `docker compose down && docker compose up -d`, then visit `http://localhost:3081` instead.

> [!NOTE]
> **📝 本機附註：這台機器的 port 與名稱占用狀況**
> - 這裡沒有遇到 port 衝突。先前另一套 libre-chat stack 已把對外 port 移到 3085（LibreChat）與 127.0.0.1:27019（MongoDB），之後又整個 `down` 掉了。
> - 注意：本機的 `libre-chat-sep` 已占用 **3081** 與 3001。官方範例的 `PORT=3081` 在這台機器上會再撞到，要改 port 請挑別的（例如 3082）。
> - 官方只談 port 衝突，沒談**容器名稱衝突**（錯誤訊息：`Conflict. The container name "/LibreChat" is already in use`），處理方式見上方 clone 步驟的附註。

### Container Crashes on Startup

If containers exit immediately after starting, check the logs:

```bash
docker compose logs api
```

Common causes:

- Invalid `librechat.yaml` syntax: validate with the [YAML Validator](/toolkit/yaml_checker)
- Missing `.env` file -- ensure `.env` exists in the project root
- Docker not running -- ensure Docker Desktop is open and running

### Missing Environment Variables

If features are not working as expected, check that required environment variables are set in your `.env` file.

```bash
docker compose exec api env | grep -i "your_variable"
```

> [!NOTE]
> **📝 本機附註：** `grep` 在主機端執行，PowerShell 沒有，改用 `docker compose exec api env | Select-String -Pattern "your_variable"`，或在 Git Bash 執行原指令。

See the [.env reference](/docs/configuration/dotenv) for all available variables and their defaults.

## Next Steps

<Cards num={3}>
  <Cards.Card title="Custom Endpoints" href="/docs/quick_start/custom_endpoints" arrow>
    Add OpenRouter, Ollama, and other AI providers
  </Cards.Card>
  <Cards.Card title="Configuration Overview" href="/docs/configuration" arrow>
    Understand how LibreChat's config files work together
  </Cards.Card>
  <Cards.Card title="Authentication Setup" href="/docs/configuration/authentication" arrow>
    Configure OAuth, LDAP, and other login methods
  </Cards.Card>
</Cards>
