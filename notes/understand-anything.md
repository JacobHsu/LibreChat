# Understand-Anything 使用教學（LibreChat 學習用）

[Understand-Anything](https://github.com/Egonex-AI/Understand-Anything) 是一個 Claude Code plugin：
用多個 agent 掃描程式碼，建出「檔案／函式／類別／依賴」的知識圖，再提供互動式 dashboard、
問答、單檔解說與入門導覽。本筆記記錄它在這個 repo 的安裝方式，以及用它學習 LibreChat 的建議路線。

版本：2.9.8（MIT 授權）

---

## 1. 安裝狀態

| 位置 | 內容 |
|---|---|
| `.claude/settings.json` | 專案層級的啟用宣告（marketplace 來源 + `enabledPlugins`）。`.claude/` 已被 gitignore。 |
| `~/.claude/plugins/cache/understand-anything/` | plugin 本體（skills、agents、hooks），所有專案共用一份快取。 |
| `.gitignore` | 已加入 `.ua/` 與 `.understand-anything/`，產生的知識圖不會進 git。 |

安裝指令（已執行過，換機器或重裝時參考）：

```bash
claude plugin marketplace add Egonex-AI/Understand-Anything --scope project
claude plugin install understand-anything@understand-anything --scope project
```

在 Claude Code 內也可以用 `/plugin marketplace add ...`、`/plugin install ...`。
裝完要重開 session 或執行 `/reload-plugins` 才會載入。

其他管理指令：

```bash
claude plugin details understand-anything            # 看元件清單與 token 成本
claude plugin update understand-anything             # 更新
claude plugin uninstall understand-anything --scope project   # 移除
```

---

## 2. 指令一覽

| 指令 | 用途 | 前置條件 |
|---|---|---|
| `/understand [路徑] [選項]` | 掃描程式碼、建立知識圖（主流程） | 無 |
| `/understand-dashboard [路徑]` | 在瀏覽器開互動式依賴圖（本機 `127.0.0.1:5173`，網址帶 `?token=`） | 已有知識圖 |
| `/understand-onboard` | 產生入門導覽（會詢問是否存成 `docs/UA_ONBOARDING.md`） | 已有知識圖 |
| `/understand-chat <問題>` | 針對知識圖問答 | 已有知識圖 |
| `/understand-explain <檔案路徑>` | 深入解說單一檔案／函式 | 已有知識圖 |
| `/understand-domain [--full]` | 抽出業務領域與流程，產生領域流程圖 | 有知識圖時直接衍生（便宜），沒有時輕量掃描 |
| `/understand-diff` | 分析目前改動影響哪些模組、有什麼風險 | 已有知識圖 |
| `/understand-knowledge`、`/understand-figma` | 分析 LLM wiki 資料夾／Figma 檔 | 與本專案無關 |

`/understand` 常用選項：

| 選項 | 說明 |
|---|---|
| `--language zh-TW` | 摘要、說明、導覽都用繁中產生；會記到 `.ua/config.json`，之後增量更新沿用 |
| `--full` | 忽略既有知識圖，完整重建 |
| `--exclude "a/**,b/**"` | 額外排除檔案（**只對這次有效，不會保存**，長期範圍請用 `.understandignore`） |
| `--review` | 改用完整的 LLM 審查（更準、更貴） |
| `--auto-update` | 每次 commit 後提醒更新知識圖（本專案建議**不要開**，見第 6 節） |

---

## 3. 關鍵觀念：知識圖放在哪裡

- 知識圖存在「被分析目錄」底下的 `.ua/knowledge-graph.json`。
- `/understand packages/api` 會把圖寫到 `packages/api/.ua/`。
- 但 `/understand-chat`、`/understand-explain`、`/understand-onboard`、`/understand-diff`
  **只讀目前工作目錄（repo 根目錄）的 `.ua/`**，找不到就會叫你先跑 `/understand`。

所以建議做法是：**永遠在 repo 根目錄跑 `/understand`，用 `.ua/.understandignore` 控制要分析的範圍。**
這樣所有指令都讀同一份圖；之後放寬 ignore 規則再跑一次，就會增量把新範圍加進來。

> 只想看單一子目錄的 dashboard 時，`/understand packages/api` 搭配
> `/understand-dashboard packages/api` 也可以，只是 chat／explain 用不到那份圖。

---

## 4. 控制範圍：`.ua/.understandignore`

這個 repo 很大（`packages/api` 不含測試約 1,100 個原始檔，`client/src` 約 1,400 個），
一次全掃 token 成本很高。第一次跑 `/understand` 時，它會依 `.gitignore` 產生
`.ua/.understandignore` 並**暫停等你確認**；這時把下面的規則貼到檔案最後再確認繼續。

語法同 `.gitignore`。每個階段只要刪掉對應的那幾行，再跑一次 `/understand` 即可。

```gitignore
# ---- 學習範圍控制（由窄到寬，逐段刪除） ----

# 永遠排除：測試、建置產物、非程式碼
**/*.spec.ts
**/*.spec.tsx
**/*.test.ts
**/*.test.tsx
**/__tests__/
**/__mocks__/
vendor/
helm/
e2e/
examples/
redis-config/
otel/
search/
skill/
**/locales/

# 階段 3 再刪：前端
client/
packages/client/
packages/chat/

# 階段 4 再刪：舊後端與工具
api/
config/
scripts/
utils/
packages/channel-gateway/

# 階段 2 再刪：後端主體
packages/api/

# 階段 1 只留 packages/data-provider 與 packages/data-schemas
```

---

## 5. 學習路線

先讀 `AGENTS.md` 開頭兩段（各 package 的分工）與 `CONTEXT.md`（領域詞彙），再開始。

### 階段 1：共用契約（約 460 個檔，最便宜，先熟悉工具）

範圍：`packages/data-provider`（前後端共用型別、API 定義、`librechat.yaml` 的 `configSchema`）
與 `packages/data-schemas`（Mongoose schema 與資料庫 method）。

```
/understand --language zh-TW
/understand-dashboard
/understand-onboard
```

可以問：

```
/understand-chat librechat.yaml 的設定 schema 定義在哪裡？有哪些主要區塊？
/understand-chat 對話（conversation）和訊息（message）的資料模型長什麼樣子？
/understand-explain packages/data-provider/src/config.ts
```

### 階段 2：後端主體 `packages/api`

刪掉 ignore 裡 `packages/api/` 那一行，再跑一次 `/understand`（增量更新）。

```
/understand-chat 使用者送出一則訊息後，後端經過哪些模組才呼叫到 LLM provider？
/understand-chat Agent 和 MCP 工具是怎麼被載入和呼叫的？
/understand-chat 驗證流程在哪裡建立 req.user？auth 快取怎麼運作？
/understand-domain
```

### 階段 3：前端

刪掉「前端」那三行再跑。重點看 `client`（React app）、`packages/client`（共用 UI 元件）、
`packages/chat`（正在搬遷中的聊天狀態與訊息元件）。

```
/understand-chat 前端怎麼接收串流回應並更新訊息？
/understand-chat Recoil 和 Jotai 的狀態目前怎麼分工？
```

### 階段 4：舊後端與其餘部分

`api/` 是舊的 Express 接線，新邏輯不放這裡，最後再看即可。

### 之後改程式時

```
/understand-diff
```

看改動會波及哪些上下游模組。

---

## 6. 成本與注意事項

- **Token**：`/understand` 本身每次約 27k token，再加上每批檔案的分析；每個 session 固定多約 1.3k token
  （skill 描述）。範圍越窄越省。
- **`--auto-update` 先不要開**：它會在偵測到 commit 時提醒更新知識圖。本專案已有 husky 的
  pre-commit／pre-push 檢查，疊加起來容易拖慢流程。plugin 內建的兩個 hook 只在開了
  `autoUpdate` 時才動作，不開就沒有影響。
- **程式碼會送給模型**：分析是由 LLM agent 讀原始碼（使用你 Claude Code 的模型）。
  dashboard 本身只讀本機檔案、不呼叫 LLM。
- **worktree**：在 git worktree 裡跑時，它會把輸出導回主 checkout，避免 worktree 被清掉時圖也消失。
- **知識圖過期**：程式改動後，chat／explain 會提醒圖可能過期，重跑 `/understand` 會做增量更新。
- **`/understand-onboard` 存檔位置**：它預設提議存到 `docs/UA_ONBOARDING.md`，
  個人學習筆記可以改存到 `notes/`，避免混進官方文件目錄。
