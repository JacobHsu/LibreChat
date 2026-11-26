# 精簡清單（本 fork 移除了哪些上游內容）

這個 repo 是 [LibreChat-AI/LibreChat](https://github.com/LibreChat-AI/LibreChat) 的快照（初始 commit `609737a`），
為了學習與單機 Docker Compose 部署而精簡。這份清單記錄**刪了什麼、為什麼刪**，
從上游同步新版本後，照這份清單重新刪一次即可。

**精簡原則：** 只刪「執行路徑以外」的東西——部署變體、用不到的獨立服務、上游團隊專用的工具。
`api/`、`packages/`、`client/` 裡的功能程式碼與單元測試一律保留，即使某個功能目前沒用到。

---

## 1. 已移除

| 路徑 | 內容 | 為什麼刪 | 移除於 |
|---|---|---|---|
| `scripts/quality/` | 上游的「變更品質」PR 留言工具（預算、覆蓋率、bundle 分析等） | 驅動它的 workflow（`change-quality.yml`、`quality-decision.yml`）快照時就不在了，留著只剩失效的測試 | `aaf7a61` |
| `package.json` 的 `e2e:github` script | 用 `act` 在本機跑 `.github/workflows/playwright.yml` | 指向的 workflow 檔從未存在於此 repo | `aaf7a61` |
| `helm/` | Kubernetes Helm chart（`librechat`、`librechat-rag-api`） | 只用 Docker Compose 部署，不用 K8s | PR #12（`704f1d4`） |
| `.gitignore` 的 Helm 區塊 | `helm/librechat/Chart.lock` 等 4 行 | 隨 `helm/` 一起移除 | PR #12 |
| `otel/langfuse-fanout/` | 多租戶 Langfuse 追蹤分送閘道（Go + OpenTelemetry Collector） | 單一站點不需要把追蹤分送到多個租戶的 Langfuse 專案 | PR #13（A 組） |
| `docker-compose.langfuse-fanout.yml`、`deploy-compose.langfuse-fanout.yml` | 啟動上述閘道的 compose 疊加檔 | 隨閘道一起移除 | PR #13（A 組） |
| `.env.example` 的 fanout 閘道變數 | `LANGFUSE_FANOUT_LISTEN_ADDR`、`..._REDIS_*`、`..._CENTRAL_*` 等閘道專用設定 | 閘道已移除；LibreChat 本身會讀的 4 個 app 端開關（`LANGFUSE_FANOUT_ENABLED` 等）保留 | PR #13（A 組） |
| `playwright-mock.yml` 的 timings 讀取步驟 | `shard_timings` job 裡讀取 `e2e-shard-timings-*` 快取的步驟 | 寫入端 `playwright-timings.yml` 不在此 fork，永遠讀不到；shard 一律用 `e2e/timings.mock.json` | CI 修正 PR |

連帶修改（不是刪除，但同步後要一起處理）：

- `otel/langfuse-fanout/README.md`、`.env.example`：拿掉 Helm 相關敘述（PR #12；fanout README 已在 A 組整份移除）。
- `.oss-scanner/threat_model.md`：低風險路徑清單拿掉 `helm/`（PR #12）。
- `packages/api/src/protection/conformance/README.md`：Trace exports 那列原本指向
  `otel/langfuse-fanout/otelcol.yaml`，改成註明閘道不隨此 fork 提供（PR #13）。
- `.github/scripts/playwright-matrix.test.mjs`：刪掉檢查 `playwright-timings.yml` 的測試，
  因為該 workflow 不在保留清單內（CI 修正 PR）。
- `.github/workflows/playwright-mock.yml`：`shard_timings` 的註解改為說明此 fork 沒有
  timings 寫入端（CI 修正 PR）。
- `e2e/specs/mock/scenarios/ci-setup.spec.ts`：共用依賴快取的必要參與者清單拿掉
  `lighthouse.yml lighthouse`，原因同上（CI 修正 PR）。

## 2. 快照時就不包含的上游內容

以下是建立快照時就沒有複製進來的項目，以 2026-11-26 的上游 `dev` 分支比對得出。
上游在快照之後新增的檔案也可能出現在這裡，同步時以「這些本來就不要」看待即可。

**`.github/` 底下：**
`CODEOWNERS`、`CODE_OF_CONDUCT.md`、`CONTRIBUTING.md`、`FUNDING.yml`、`ISSUE_TEMPLATE/`、
`MAIN_PROMOTION.md`、`SECURITY.md`、`configuration-release.json`、`configuration-unreleased.json`、
`playwright.yml`、`readme-i18n.json`

**`.github/workflows/`：** 上游 43 個，本 repo 只保留 7 個
（`agents-integration-tests`、`backend-review`、`cache-integration-tests`、`frontend-review`、
`merge-gate`、`playwright-mock`、`static-checks`）。未包含的 36 個：

```
a11y.yml                      backend-windows-nightly.yml   change-quality.yml
channel-gateway.yml           client.yml                    codegraph-e2e-vote-gate.yml
codegraph-e2e-votes.yml       codegraph-select.yml          data-provider.yml
data-schemas.yml              dev-branch-images.yml         dev-images.yml
dev-staging-images.yml        docker-publish.yml            docker-smoke.yml
favorites-quality.yml         frontend-windows-nightly.yml  helmcharts.yml
langfuse-fanout.yml           lighthouse.yml                locize-i18n-sync.yml
main-image-workflow.yml       merge-gate-tests.yml          playwright-bombadil.yml
playwright-timings.yml        pr-retarget-dev.yml           promote-main.yml
promotion-tests.yml           quality-decision.yml          readme-translate.yml
repository-safety.yml         retry-docker-builds.yml       sync-helm-chart-tags.yml
tag-images.yml                windows-private-files.yml     workspace-acceptance.yml
```

**頂層：** `.claude/`、`README.ru.md`

### 被 `.gitignore` 誤擋、已補回的檔案

上游有追蹤，但快照時被本 repo `.gitignore` 的 `*.d.ts` 規則擋掉，導致 `@librechat/api` 型別檢查失敗。
已從上游 `dev` 原樣補回，並在 `.gitignore` 加上 `!` 例外規則，同步時不會再被擋：

- `packages/api/src/types/bcryptjs.d.ts`、`es2024-string.d.ts`、`proxy-from-env.d.ts`
- `packages/api/types/index.d.ts`
- `packages/chat/src/modules.d.ts`
- `packages/data-schemas/src/types/winston-transports.d.ts`

同樣被 `.gitignore` 擋掉的其他上游檔案：

| 檔案 | 擋掉它的規則 | 處理 |
|---|---|---|
| `examples/sdk-runtime/librechat.yaml` | `librechat.yaml` | **已補回**：SDK Agents 範例的設定片段，`docs/sdk/agents.md` 會引用；加了 `!/examples/sdk-runtime/librechat.yaml` 例外，根目錄自己的 `librechat.yaml` 仍被忽略 |
| `redis-config/certs/dump.rdb` | `dump.rdb` | **刻意不補回**：這是 Redis 的資料快照檔，上游在 2025-07（#8457）隨 Redis 重構一起 commit，放在 `certs/` 下，看起來是執行產物而非刻意追蹤的檔案；沒有它 `cache-integration` CI 照樣通過 |
| `.claude/skills/` 底下 9 個檔案 | `/.claude/` | 不補回：上游團隊的 Claude Code 技能設定 |

## 3. 刻意保留

| 路徑 | 為什麼留 |
|---|---|
| `notes/librechat-docs/` 裡提到 Helm 的句子 | 官方文件原文副本，不改動 |
| `scripts/readme/lib.mjs` 詞彙表的 `'Helm'` | 只讓 README 翻譯時不翻這個詞，沒有實際作用 |
| `packages/api/src/langfuse/` 等 fanout 相關的 app 程式碼 | LibreChat 本體的一部分；測試裡的 `langfuse-fanout-collector` 只是 URL 字串 |
| `redis-config/` | `cache-integration-tests`、`playwright-mock` 兩個 CI 會用到 |
| `bun.lock`、`.bun-version` | CI 和 `scripts/` 會用到 bun |
| `helmet` 套件 | Express 安全標頭套件，名字像但與 Helm 無關 |

## 4. 候選（尚未決定）

| 組別 | 內容 | 牽連 |
|---|---|---|
| B. 效能壓測 e2e | `e2e/benchmarks*`、`bombadil/`、`perf/`、`lighthouse/` 與對應 playwright config | `package.json` 約 20 個 script、`@antithesishq/bombadil` 依賴、`AGENTS.md` 的 lighthouse 要求 |
| C. Slack 頻道閘道 | `packages/channel-gateway`、`docker-compose.channels*.yml`、`docker-compose.slack-workspaces.yml` | 3 個 CI workflow 與 `scripts/sort-imports.mts` 有引用 |
| D. 雜項工具設定 | `.devcontainer/`、`.codex/`、`.oss-scanner/` | 低 |

---

## 從上游同步後重新精簡

同步完成後，在 repo 根目錄執行：

```sh
# 第 1 節的資料夾與檔案
git rm -r -q --ignore-unmatch \
  scripts/quality \
  helm \
  otel/langfuse-fanout \
  docker-compose.langfuse-fanout.yml \
  deploy-compose.langfuse-fanout.yml

# 第 2 節：只保留 7 個 workflow
keep='agents-integration-tests|backend-review|cache-integration-tests|frontend-review|merge-gate|playwright-mock|static-checks'
git ls-files .github/workflows | grep -v -E "/($keep)\.yml$" | xargs -r git rm -q
```

接著手動處理：

- `package.json`：刪掉 `e2e:github` script。
- `.gitignore`：刪掉 `# Helm` 區塊。
- `.env.example`：fanout 段落只留 4 個 app 端開關（見第 1 節）。
- `.oss-scanner/threat_model.md`：低風險路徑清單拿掉 `helm/`。
- `packages/api/src/protection/conformance/README.md`：Trace exports 列的 `otelcol.yaml` 路徑。
- `.github/scripts/playwright-matrix.test.mjs`、`e2e/specs/mock/scenarios/ci-setup.spec.ts`：
  移除對 `playwright-timings.yml`、`lighthouse.yml` 的依賴（見第 1 節）。
- `.gitignore`：保留 `*.d.ts` 與 `!/examples/sdk-runtime/librechat.yaml` 的 `!` 例外規則；
  用新的上游 `.gitignore` 覆蓋時要加回去，否則那 6 個型別宣告檔與範例設定片段會再次消失。
- 第 2 節 `.github/` 底下的其他檔案與頂層項目：視需要刪除。

最後搜尋殘留引用，確認沒有指向已刪路徑的設定：

```sh
git grep -n -E "scripts/quality|helm/|otel/langfuse-fanout|langfuse-fanout\.yml"
```
