# LibreChat 官方文件（本機安裝用副本）

照官方 Docker 教學在本機安裝 LibreChat 時「有用到的」官方文件原文。
每份都從官網的原始碼 repo [LibreChat-AI/librechat.ai](https://github.com/LibreChat-AI/librechat.ai)
下載，對應 commit `4085279`（2026-11-25），內容未改動。

與官方不同、或官方沒寫到的地方，以下面這種區塊插在原文對應位置：

> [!NOTE]
> **📝 本機附註：** ……

要找所有附註，搜尋 `📝 本機附註` 即可。

---

## 檔案與官方連結

```text
librechat-docs/
├── README.md   本頁：索引、官方連結、安裝進度
├── en/         英文原文 + 📝 本機附註（以此為準）
└── zh/         中文翻譯（僅供對照，無附註；官方缺的頁面為自行翻譯，檔頭有標示）
```

兩個資料夾的檔名一一對應，例如 `en/local-docker.md` ↔ `zh/local-docker.md`。

### `en/`：英文原文，有附註

| 檔案 | 內容 | 官方網址 |
|---|---|---|
| [en/local-docker.md](en/local-docker.md) | **主教學**：Docker 本機安裝、掛載 `librechat.yaml`、更新、排錯 | <https://www.librechat.ai/docs/local/docker> |
| [en/features-admin_panel.md](en/features-admin_panel.md) | Admin Panel 是什麼、能做什麼、怎麼登入 | <https://www.librechat.ai/docs/features/admin_panel> |
| [en/configuration-admin_panel.md](en/configuration-admin_panel.md) | Admin Panel 的部署方式、環境變數、Admin API | <https://www.librechat.ai/docs/configuration/admin_panel> |
| [en/configuration-docker_override.md](en/configuration-docker_override.md) | 用 `docker-compose.override.yml` 自訂設定 | <https://www.librechat.ai/docs/configuration/docker_override> |
| [en/configuration-librechat_yaml.md](en/configuration-librechat_yaml.md) | `librechat.yaml` 的建立、掛載、驗證、排錯 | <https://www.librechat.ai/docs/configuration/librechat_yaml> |
| [en/features-scheduled_chats.md](en/features-scheduled_chats.md) | 排程對話是什麼（實驗性、本機未開啟） | <https://www.librechat.ai/docs/features/scheduled_chats> |
| [en/configuration-scheduled_chats.md](en/configuration-scheduled_chats.md) | 排程對話的設定與部署需求；**附註：啟動 log 的 `scheduler NOT started`** | <https://www.librechat.ai/docs/configuration/scheduled_chats> |

### `zh/`：中文翻譯，僅供對照

| 檔案 | 官方網址 | 備註 |
|---|---|---|
| [zh/local-docker.md](zh/local-docker.md) | <https://www.librechat.ai/zh/docs/local/docker> | 2026-09-24 翻譯，缺「System Requirements」、admin panel 說明，排錯段落沒有 `!override` |
| [zh/configuration-docker_override.md](zh/configuration-docker_override.md) | <https://www.librechat.ai/zh/docs/configuration/docker_override> | — |
| [zh/configuration-librechat_yaml.md](zh/configuration-librechat_yaml.md) | <https://www.librechat.ai/zh/docs/configuration/librechat_yaml> | 與英文版同一天更新（2026-11-17），內容是最新的 |
| [zh/features-admin_panel.md](zh/features-admin_panel.md) | <https://www.librechat.ai/zh/docs/features/admin_panel> | **已過時**：仍寫「預覽」狀態，原始碼連結是舊的 `ClickHouse/librechat-admin-panel` |
| [zh/configuration-admin_panel.md](zh/configuration-admin_panel.md) | 無（中文網址會轉到英文版） | **非官方翻譯**：官方沒有中文版，由英文原文（commit `4085279`）翻成繁體中文。**含一段附註：admin-panel 一直重啟的設定步驟** |
| — | — | `features-scheduled_chats`、`configuration-scheduled_chats` 官方沒有中文版 |

中文版是簡體機器翻譯，而且比英文版舊（英文版 docker 頁更新於 2026-10-23、admin panel 頁更新於 2026-10-29）。兩者不同時以英文版為準。

### 教學有提到、但還沒用到（未下載）

| 主題 | 官方網址 | 何時會用到 |
|---|---|---|
| `.env` 完整參考 | <https://www.librechat.ai/docs/configuration/dotenv> | 設定金鑰、provider、登入方式 |
| 金鑰產生器 | <https://www.librechat.ai/toolkit/creds-generator> | 產生 `CREDS_KEY`、`CREDS_IV`、`JWT_SECRET`、`JWT_REFRESH_SECRET` |
| YAML 檢查器 | <https://www.librechat.ai/toolkit/yaml-checker> | 驗證 `librechat.yaml` |
| 第一次對話 | <https://www.librechat.ai/docs/quick_start/first_chat> | 註冊後加 API key |
| 自訂 endpoint | <https://www.librechat.ai/docs/quick_start/custom_endpoints> | 接 OpenRouter、Ollama 等 |
| 權限控管 | <https://www.librechat.ai/docs/features/access_control> | Admin Panel 的角色與群組 |
| Admin Panel 原始碼 | <https://github.com/LibreChat-AI/admin-panel> | — |

用到時再下載進來並加附註。

---

## 安裝進度

依 [en/local-docker.md](en/local-docker.md) 的步驟：

- [x] 前置需求：Git、Docker Desktop
- [x] Clone：跳過，這個 repo 就是 fork；改為先讓出另一套 stack 占用的容器名稱
- [x] 建立 `.env`：`cp .env.example .env`，未修改任何值
- [x] 啟動：`docker compose up -d`，http://localhost:3080 回 HTTP 200
- [x] 固定正式金鑰（`CREDS_*`、`JWT_*`）：因為已先註冊，改為沿用暫時金鑰（[issue #3](https://github.com/JacobHsu/LibreChat/issues/3)）
- [x] 設定 `ADMIN_PANEL_SESSION_SECRET`，啟用 http://localhost:3000（[issue #1](https://github.com/JacobHsu/LibreChat/issues/1)）
- [x] 註冊第一個帳號（自動成為 ADMIN），已驗證可登入 LibreChat 與 Admin Panel
- [x] 掛載 `librechat.yaml`：最小設定（[issue #6](https://github.com/JacobHsu/LibreChat/issues/6)）
- [ ] 加 API key，完成第一次對話

## 常用指令（在 repo 根目錄執行）

| 目的 | 指令 |
|---|---|
| 啟動／套用 `.env` 或 override 的變更 | `docker compose up -d` |
| 只重啟 LibreChat（改了 `librechat.yaml` 後） | `docker compose restart api` |
| 看狀態 | `docker compose ps` |
| 看 log | `docker compose logs api --tail 50`（追蹤加 `-f`） |
| 停止並移除容器（**保留資料**） | `docker compose down` |
| 連資料一起刪（**無法復原**） | `docker compose down -v`，並手動刪 `data-node/`、`meili_data_*/` |
| 檢查合併後的 compose 設定 | `docker compose config` |
