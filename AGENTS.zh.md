# LibreChat 貢獻者與代理（agent）指引

> 本文件為 [`AGENTS.md`](AGENTS.md) 的繁體中文翻譯。如內容有出入，以英文版 `AGENTS.md` 為準。

`AGENTS.md` 是本儲存庫的貢獻者指引。後端程式碼位於 `packages/api`（TypeScript）與
`packages/data-schemas`（資料庫方法）；`api` 是舊有的 Express 接線層。共用的 API 型別與服務位於
`packages/data-provider`，React 應用程式位於 `client`，共用的基礎元件位於 `packages/client`。
聊天狀態、傳輸與訊息元件正逐步移至 `packages/chat`（`@librechat/chat`），該套件絕不匯入應用程式本身。

## 分支與 Pull Request

一般情況下從 `dev` 開分支並以 `dev` 為目標；`gh pr create` 預設目標是 `main`，因此請明確傳入
`--base dev`。`main` 是已發佈的分支，維持為 `dev` 的 fast-forward 並原樣同步——絕對不要開 backport
到 `main` 的 pull request，因為合併到 `dev` 的內容在下次同步時就會進入 `main`。以 `main` 為目標開啟的
pull request 會被自動改為其他目標分支。

**維護者指定的 canary 例外：** 實驗性的工作，或是已通過審查可合併、但尚不適合進入下一次 `main` 同步的
PR，可改以 `canary` 為目標。維護者可在工作開始前，或在審查一個停滯的 PR 時做此決定。新的 canary 工作請從
目前的 `origin/canary` 開分支，並明確傳入 `--base canary`；不要默默地更改既有 PR 的目標分支，也不要把
canary 的程式碼提升到 `dev`。若停滯的 PR 被改導向 canary，請先將其 base、diff 與已審查的 head 和目前的
canary 對照；任何 rebase 或新開 PR 都要與維護者協調。明確以 `canary` 為 base 的 PR 會留在那裡；
main 到 dev 的改目標 workflow 不會移動它。

仍然要在 PR 中連結相關的 issue（例如 `Related to #N`），讓工作保持可追溯。`Fixes #N` 在合併到 `dev`
或 `canary` 時不會關閉 issue——GitHub 只在預設分支上才會處理關閉關鍵字。合併後請手動關閉已解決的 issue。
多個 worktree 共用同一個 stash 堆疊，因此絕對不要使用不帶參數的 `git stash pop`。優先使用 WIP commit；
若非得使用 stash，請 apply 指定標記的那一筆。`target: main` 標籤與以發佈為目的的上游分支不受 main 到 dev
改目標 workflow 影響；不要用這兩者來 backport 一般的工作。

讓每個 PR 專注於一個可獨立審查的變更。當實作、審查或本地／自動化 e2e 測試發現另一個 bug 或邊界情況時，
若它能獨立出貨，請記錄下來並在發現當下盡快另開 PR。不要讓發現的缺口在目前的 PR 中累積。必要的修正、
其呼叫端與測試要放在一起；把大型變更拆成連貫、可審查的步驟，而不是把不相關的工作綁在一起。

描述要寫給沒有追蹤過這個分支的讀者：什麼東西壞了、什麼情況會觸發、變更後的行為如何，接著用一到兩種方式
呈現機制——聚焦的 diff、呼叫樹、淺層檔案樹，或 Mermaid 時序圖。只保留這次變更實際帶有的內容，並描述
程式碼現在的樣子，而不是敘述先前的 commit 或審查輪次。指出造成此 bug 的已合併 pull request 則不同；
那是讀者需要的歷史。格式與範例請見 `.github/pull_request_template.md`。

## 審查與完成

閱讀行內審查討論串本身——摘要留言或通知清單會遺漏發現的問題。依目前的程式碼逐一稽核，修正有效的部分，
並在回覆中說明理由以駁回已過時的部分。每一輪之後：執行聚焦的測試、在每個你修改過的 workspace 執行
`npx tsc --noEmit`、push，然後指名該 pull request 確切的遠端 head 請求下一輪審查——對較早 head 的乾淨
審查，無法說明你剛 push 的內容，而 CI 依它自己的時程執行。在每個已解決的討論串回覆解決它的 commit 與
佐證。經過兩輪有實際修改的審查後，停止逐串修補，改以不變量（invariant）的角度通讀整個子系統：視情況
涵蓋身分、授權、持久化、重試、取消、清理與混合版本行為。由哪個審查者、用什麼詞句觸發會改變；但審查必須
涵蓋確切 push 出去的 head，這點不會改變。

乾淨的審查只是完成的一個訊號，不是「完成」的定義。要交付可觀察到的體驗——載入中、空狀態、成功、失敗、
取消、重試、恢復的工作階段——字串已在地化、無障礙性完好、預設值與已儲存的資料被保留，且沒有任何後端
能力缺少前端入口。修正要小，並測試先前遺漏的行為。回報 push 出去的 head、你在本地執行了什麼、CI 狀態、
該 head 的審查結果、任何你駁回的發現及其理由，以及你無法執行的檢查。

## 驗證

對於啟動、驗證（auth）、設定、檔案或訊息載入的變更，避免序列式的資料庫讀取，並重用已載入的請求資料。
完成前執行 `npm run lighthouse`：CI 管線會為每次 Mongo 查詢增加 250 ms，並檢查可見對話的 LCP。
請參閱[預算、重現方式與失敗診斷](e2e/lighthouse/README.md)。

建置成功不等於型別檢查通過：`packages/api`、`packages/client`、`packages/chat` 與 `packages/data-schemas`
使用 `tsdown` 建置，它只輸出而不檢查型別。請在你修改的 workspace 執行 `npx tsc --noEmit`。
`packages/client` 完全把 `*.spec.ts(x)` 與 `*.test.ts(x)` 排除在型別檢查之外。不帶參數的
`npm run sort-imports` 會改寫每個原始碼根目錄——請傳入你動到的路徑。執行
`npm run static-checks -- --against origin/dev` 以重現 PR 的靜態檢查；較慢的關卡請用
`npm run static-checks:full`。修正你所變更程式碼中的所有格式、lint 與 TypeScript 警告／錯誤。
pre-push hook 會針對要 push 的 commit 執行相同的檢查，包含慢速層級與型別檢查；它會拒絕髒的工作樹或
`HEAD` 以外的 ref，而不是檢查錯誤的內容，所以請先 commit，而不是繞過它。

## 模組邊界與設定

所有新的後端行為都屬於 `packages/api` 下的 TypeScript；資料庫相關的共用邏輯屬於 `packages/data-schemas`，
前後端共用的 API 邏輯屬於 `packages/data-provider`。請在專案根目錄以 `npm run build:data-provider`
建置 data-provider。

`/api` 只放接線，不放行為。當一個變更會在那裡的 CJS 檔案中加入邏輯——一個分支、一個輔助函式、一個驗證
步驟、一次服務呼叫——該邏輯應放在 `packages/api`，而 JS 檔案只保留 require、路由註冊與對 TS 模組的呼叫
（`MCPRequestContext.js` 就是這個樣貌，十三行的 re-export）。「最少」指的是 `/api` 增加了多少行為，
而不是 diff 有多小；此規則也適用於編輯既有的 CJS，而那是最常見的情況。

資料庫契約屬於 `packages/data-schemas`。不要讓 Mongoose 型別（`FilterQuery`、`Types.ObjectId`、
`Document`）出現在 `packages/api`、`packages/data-provider` 與 `client` 的匯出簽章中，因為那會讓儲存
引擎成為該模組公開 API 的一部分。接收並回傳純粹的型別化物件，並把查詢包在 data-schemas 方法之後。
這個邊界在 `packages/api` 中已經有洩漏，所以請停止擴大它，而不是重寫既有的部分；client 端完全沒有這類
洩漏，且必須維持如此。

新的調控參數要以可設定的形式出貨：在程式碼中引入的限制、逾時、開關或能力，都應在 `configSchema`
（`packages/data-provider/src/config.ts`）上有一個欄位，使其能在 `librechat.yaml` 中設定，並以重現
現行行為的值作為預設。寫死的常數與僅靠環境變數的開關需要理由。模組要接收依賴，而不是自己去抓取：
`packages/api` 中的程式碼從呼叫端接收其設定、資料庫方法與客戶端，就像 `createModels(mongoose)` 接收
應用程式的連線一樣，而不是匯入應用程式的單例或讀取全域狀態。整合（provider SDK、儲存後端、向量資料庫、
OAuth 伺服器）透過呼叫端提供的介面進來，讓第二種實作只是一個新參數，而不是一個新分支。
`packages/api/src/mcp` 下的靜態單例是應停止擴充的樣式，而非可仿效的模式。這是 client 狀態所有權的後端
那一半：傳進來，不要自己去抓。

## 服務失敗與面向使用者的錯誤

本節適用於後端服務（`packages/api`）與資料庫方法／儲存庫（`packages/data-schemas`），不適用於 React
元件。對於這些層中新增或大幅修改的程式碼，請在契約邊界決定失敗行為，而不是把每個不成功的結果都當成例外：

- 成功時回傳純值。`null` 只用於有文件記載的「不存在」（例如查詢成功但沒有紀錄），絕不用來掩蓋查詢失敗。
- 對於呼叫端可以處理、不需中止操作的預期失敗，優先使用型別化的可辨識聯集結果，例如
  `{ ok: true; value: T } | { ok: false; error: { code: string; message?: string } }`。若更改既有的
  領域專屬結果形狀會破壞呼叫端，請保留它；不要在同一個服務中引入可互換的 `ok`、`valid` 與裸
  `{ message }` 契約。當呼叫端需要區分失敗時，代碼應是穩定、機器可讀的識別字。內部驗證輔助函式可沿用
  既有的在地模式。
- 對非預期的營運失敗（資料庫、provider、網路）與被違反的不變量要拋出例外。當邊界必須辨識特定失敗時，
  使用型別化／帶代碼的錯誤。在能夠復原或轉譯的地方攔截；否則讓它到達所屬的請求或工作邊界。不要攔截
  例外後改回傳一個 truthy 的紀錄、`null` 或 `{ message }`：呼叫端可能會把服務中斷誤認為資料，或以
  HTTP 200 送出。對既有契約的遷移要獨立進行，連同其呼叫端與測試，而不是順帶改變傳輸格式。
- 在 HTTP、串流或工作邊界，把失敗對應到既有的狀態碼與回應契約。只揭露刻意核准的代碼與有幫助且對使用者
  安全的資訊。錯誤揭露是一道安全邊界：絕不把原始例外、堆疊、查詢文字、憑證、請求內容、標頭、provider
  回應內容，或任意的 `error.message` 轉送給客戶端或寫入使用者可見的持久化狀態。清理診斷資訊，並只記錄
  該路徑可安全保留的內容。對於可能回顯使用者提交內容的錯誤，優先使用 `getSafeErrorMetadata` 而非錯誤
  文字；`packages/api/src/utils/errors.ts` 記載了何時適合使用經遮蔽的錯誤文字。
- 在地化是 UI 的責任，不是資料庫方法的回傳形狀。當失敗到達面向使用者的介面時，提供一個安全、穩定的
  代碼，讓 UI 對應到在地化、可採取行動的文案及備援文字。請參閱
  `client/src/components/Messages/Content/Error/registry.ts` 與
  `client/src/components/Chat/Trace/Viewer.tsx`。任何刻意揭露的 provider 細節，都要維持在其既有的保護
  政策之後，而不是把任意的上游文字視為安全。

以下是在地契約的範例，並非要求複製其形狀：`packages/api/src/admin/auditLog.ts` 使用解析結果，
`packages/api/src/agents/openai/service.ts` 在對應成 OpenAI 相容回應前先驗證，
`packages/api/src/skills/sync/errors.ts` 以穩定代碼命名同步失敗。
`packages/data-schemas/src/methods/prompt.ts` 目前在查詢失敗時回傳 `{ message }`；不要延伸這個模式。
請測試「不存在」與「失敗」的區別、邊界的非成功回應，以及敏感診斷資訊無法到達使用者可見的介面。

## RUM 隱私

當新增、重新命名或修改一個接收或回傳密碼、token、API 金鑰或其他憑證的端點時，請在同一個變更中檢視
[RUM 隱私規則](client/src/lib/rum/privacy.ts)。檢查每條路由與別名，包括部署時的 base path。當端點需要從
遙測中移除請求與回應內容時，更新 `CREDENTIAL_ROUTES`。引入憑證標頭時檢查 `CREDENTIAL_HEADER`。一般聊天
與 API 請求則保留內容擷取。

在[隱私測試](client/src/lib/rum/privacy.spec.ts)中加入回歸案例。驗證請求與回應中的機密皆不存在，且一般
內容仍被保留。若變更遙測 SDK、擷取設定或匯出路徑，也要檢查[匯出器](client/src/lib/rum/exporter.ts)及其
[測試](client/src/lib/rum/exporter.spec.ts)。執行 [RUM 瀏覽器測試](e2e/rum/privacy.spec.ts)，透過 proxy
與直接匯出兩種方式驗證 SDK 的實際輸出。

## 前端主題與樣式

前端工作請先組合既有的 `@librechat/client` 基礎元件與變體，再考慮加入功能專屬的樣式。顏色與共用外觀使用
語意化的主題／Tailwind 角色；不要引入原始調色盤 utility、寫死的顏色或任意的主題 CSS。若系統無法表達某個
可重用的設計需求，請深化共用的基礎元件或有版本的主題 token 註冊表，而不是把 class 複製到功能中。主題定義
選擇的是語意化的顏色與外觀，而非選擇器、應用程式行為或替代版面。真正屬於版面與行為的部分保留在本地，並
說明為何任何新的自訂 CSS 無法使用共用基礎元件。支援淺色／深色與減少動態效果；保留預設值，並為可重用的
新變體測試一個刻意不同的參考主題。

## 後端驗證快取

新增或修改會變更使用者文件的程式碼時，請為受影響的使用者讓驗證用的使用者文件快取失效，包括大量的角色與
使用者變更。否則突發快取的 JWT 請求可能在快取到期前提供過時的 `req.user`。

## Client 狀態所有權

client 正從 Recoil 遷移到 Jotai。新的狀態一律使用 Jotai，即使檔案中已經匯入了 Recoil；許多檔案同時匯入
兩者，因此混合匯入無法說明該用哪一個。對既有狀態而言，轉換的單位是一個 atom 加上所有讀寫它的檔案，因為
atom 無法只轉換一半——轉換你動到的區域，而不是整個 store。
依所有權劃分：功能自己既寫又讀的狀態屬於該功能，因此要轉換為 Jotai 並保留在功能內；功能只是取用的
應用程式全域偏好設定與外殼狀態（`maximizeChatSpace`、`showScrollButton`、`enterToSend`、artifact
可見性）必須透過 props 或由宿主提供的小型 context 傳入，而不是透過 `~/store` 去抓取；當取用者位於你正在
修改的功能之外時，讓該 atom 留在 Recoil 並將其傳入。傳入的做法讓功能日後能移到自己的 workspace 而不需
重寫，並讓 Jotai 轉換的範圍限定在功能所擁有的狀態。持久化的 atom 請使用
`client/src/store/jotai-utils.ts` 中的輔助函式。

## 程式碼風格與效能

**在意義仍清楚的前提下，變數、物件、參數與函式優先使用簡潔的單字名稱。** 讓作用域與型別提供脈絡：在
session 專屬的程式碼中，若不會失去區別，就用 `provider` 而不是 `sessionProvider`。去掉多餘的修飾詞，
不要堆疊名詞；也不要用難懂的縮寫替代，或為了省 token 而犧牲清晰度。盡可能使用簡短的單字檔名；把相關
模組放在一個單字命名的目錄中。

優先使用扁平的提早回傳、純函式與明確的型別。避免 `any`、寬泛的 `unknown`／型別轉換、重複的型別與動態
匯入。抽出真正重複的邏輯，而不是複製它；只為不明顯的行為或公開 API 契約撰寫註解。

匯入分為三個區段：套件值（最短的行在前，`react` 最前面）、`import type`（最長的行在前，套件型別在本地
型別之前），以及本地／專案值（最長的行在前）。使用獨立的 `import type { ... }`，不要在值匯入中使用行內
`type`。對你修改的檔案執行範圍限定的匯入排序工具。

避免對共用的訊息陣列做額外的遍歷與不必要的配置。對於啟動與請求路徑，重用已載入的使用者／設定資料，避免
序列式的資料庫讀取，並平行啟動彼此獨立的讀取。不要削弱授權或租戶檢查，也不要在這些檢查成功前寫出回應。

在熱路徑上增加工作之前，先評估效能影響。在 PR 中指出新增的資料庫或網路呼叫、序列等待、記憶體配置與
client 端工作。對有意義的變更，量測受影響的延遲或資源使用的前後差異，視情況包含復原與並行請求；若有未
量測的部分要明說，而不是在沒有證據的情況下稱其影響可忽略。

## 前端規則

使用聊天的分享／匯出動作選單模式：`@librechat/client` 的 `DropdownPopup` 搭配 `Ariakit.MenuButton`
（參見 `HeaderMenu.tsx` 與 `useExportShare.tsx`）。絕不為應用程式的動作或排序選單引入或重新引入 Radix
`DropdownMenu` 系列。當選單動作會開啟對話框時，保留分享／匯出對話框項目的契約（`hideOnClick: false`、
項目 ref、button render，以及對話框的 `triggerRef`）。

所有可見文案都使用 `useLocalize()`，且只更新 `client/src/locales/en/translation.json` 中的英文鍵值。使用
語意化 HTML、鍵盤操作行為與 ARIA 標籤。API 互動使用 React Query，並在 mutation 後讓相關查詢失效；查詢與
mutation 的鍵定義在 `packages/data-provider/src/keys.ts`。共用的端點定義、型別與資料服務放在
`packages/data-provider`，並對動態 URL 參數進行編碼。在聚焦的元件測試中涵蓋載入、成功與失敗狀態。

## 測試

從所屬的 workspace 執行聚焦的 Jest 測試，而不是整個 monorepo。優先使用真實邏輯與 spy，而非 mock 內部
實作；資料庫查詢使用 `mongodb-memory-server`，MCP 行為使用真正的 MCP SDK。只 mock 外部 HTTP API 或無法
控制的服務。前端測試請使用元件同層的 `__tests__`，並在適用時使用 `test/layout-test-utils`。
