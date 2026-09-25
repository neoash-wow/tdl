# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案概述

tdl 是以 Go 撰寫的 Telegram CLI 工具（下載、上傳、轉發、匯出訊息/成員），透過 [gotd/td](https://github.com/gotd/td) 直接使用 MTProto，而非 Bot API。使用者文件位於 `docs/`（Hugo 站點，線上版 https://docs.iyear.me/tdl/）。

## 常用指令                    

```bash
go build                                         # 建置（CI 使用此指令）
go test -v $(go list ./... | grep -v /test)      # 單元測試（CI 排除 ./test 的 e2e 測試）
go test ./pkg/kv -run TestXxx -v                 # 執行單一測試
golangci-lint run                                # Lint，設定見 .golangci.yaml
go generate ./...                                # 重新產生 *_enum.go（需安裝 go-enum）
go run main.go gen doc -d docs/content/en/more/cli   # 產生 CLI 文件（docs workflow 使用）
make build                                       # goreleaser snapshot 建置
```

- **Workspace**：`go.work` 將 `.`、`core`、`extension` 三個 module 綁在一起。`core/` 與 `extension/` 是**獨立發佈的 Go module**（各自有 `go.mod`），修改其公開 API 時要注意相容性；依賴升級需分別處理各 module。
- **格式化**：使用 `gofumpt` + `gci`，import 分組順序為：standard → default（第三方）→ `github.com/iyear/tdl` 前綴 → dot import。
- **E2E 測試**（`test/`，Ginkgo + Gomega）：需要本機執行的 teamgram-server（見 `.github/workflows/master.yml` 的 e2e job，目前在 CI 中停用，因不穩定）。執行方式：`ginkgo -v -r ./test`。`test/testserver` 會覆寫 `core/tclient` 的 DC/公鑰並呼叫 `dcpool.EnableTestMode()`，以連到測試伺服器。

## 架構

分層依賴方向：`cmd/` → `app/` → `core/`，`pkg/` 為 CLI 專用的共用工具。

- **`cmd/`**：cobra 命令定義，只負責 flag 解析與組裝 `Options`，再呼叫 `app/<feature>.Run`。
- **`app/`**：各功能的業務邏輯（`dl`、`up`、`forward`、`chat`、`login`、`migrate`、`extension`）。
- **`core/`**：不依賴 CLI 的可重用函式庫 —— `downloader` / `uploader` / `forwarder`（傳輸引擎）、`dcpool`（跨 DC 的 client 連線池）、`tclient`（建立 client 與 `RunWithAuth`）、`storage`（session / peers / state，建構在 KV 之上）、`logctx`（context 內的 zap logger）。
- **`extension/`**：第三方擴充套件 SDK。主程式透過環境變數 `TDL_EXTENSION`（JSON 格式的 `Env`：session、app id、proxy 等）把登入狀態傳給子程序形式的 extension。`pkg/extensions` 負責安裝/管理 extension；已安裝的 extension 會在 `cmd/root.go` 中被註冊為頂層子命令。

### 關鍵的橫向機制

- **Context 傳遞依賴**：`cmd/root.go` 的 `PersistentPreRunE` 初始化 logger 與 KV storage，並放入 context；之後一律用 `logctx.From(ctx)`、`kv.From(ctx)` 取出。`PersistentPostRunE` 負責關閉。需要 `cobra.EnableTraverseRunHooks = true` 才能讓每層的 hook 都被執行。
- **`tRun`（`cmd/root.go`）**：所有需要連線 Telegram 的命令都經由它建立 client，並在 `tclientcore.RunWithAuth` 內執行回呼，回呼參數為 `(ctx, *telegram.Client, storage.Storage)`。
- **KV storage（`pkg/kv`）**：可插拔的 driver（`bolt` 為預設、`file`、`legacy`），透過全域 flag `--storage type=...,path=...` 指定；以 namespace（`-n`）隔離不同帳號。首次執行時若無 bolt DB，會自動從 legacy 格式遷移（`migrateLegacyToBolt`）。
- **設定來源**：全域 flag 綁定到 viper，同時支援 `TDL_` 前綴的環境變數（`-` 轉為 `_`）。flag 名稱常數集中在 `pkg/consts`。
- **傳輸引擎的 Iter/Progress 模式**：`core/downloader`（uploader、forwarder 同理）接收 `Iter`（逐一產生待處理元素）與 `Progress`（`OnAdd`/`OnDone` 回呼）。`app/` 層負責實作這兩個介面（例如 `app/dl/iter.go`、`app/dl/progress.go`），並用 `errgroup` 以 `--limit` 控制任務並行數、`--threads` 控制單一檔案的分片並行數。單一元素失敗只會記錄 log，不會中斷整批；只有 `context.Canceled` 會中止全部。
- **Enum 程式碼產生**：帶有 `//go:generate go-enum` 的檔案會產生對應的 `*_enum.go`，請勿手動編輯產生出來的檔案。
- **表達式過濾**：`pkg/texpr` 以 `expr-lang/expr` 實作 forward/chat export 等功能的使用者過濾與路由表達式。

## 下載流程（`tdl dl`）

呼叫鏈：`cmd/dl.go` → `tRun` → `app/dl/dl.go` `Run` → `core/downloader` `Download`。上傳（`app/up` + `core/uploader`）與轉發（`app/forward` + `core/forwarder`）也採用相同的分層與 Iter/Progress 結構。

1. **`app/dl/dl.go` `Run`**：建立 `dcpool`，接著透過 `pkg/tmessage` 把來源轉成 `[]*tmessage.Dialog{Peer, Messages []int}`（`FromURL` 處理 `-u` 訊息連結，`FromFile` 處理 `-f` 官方客戶端匯出的 JSON）。加上 `--serve` 時改走 `serve.go`，以 HTTP 串流提供媒體，不寫入磁碟。
2. **`app/dl/iter.go`（實作 `downloader.Iter`）**：
   - 建構時會依 peer ID 與訊息 ID 排序 dialog（`--desc` 為反向），再對所有 peer/msg ID 計算 SHA256 作為 **fingerprint**。排序的目的是讓同一組輸入永遠得到同一個 fingerprint，續傳才能找到上次的進度。
   - 位置狀態分兩種：物理位置（`dialogIndex`/`messageIndex`）用來走訪；邏輯位置（`logicalPos`，整批中的全域序號）是續傳 `finished` map 的 key。被刪除、已完成或被過濾掉的訊息同樣要推進 `logicalPos`；相簿（`--group`）會一次推進整組的長度。修改迭代邏輯時必須維持這個不變式，否則續傳會錯位。
   - `processSingle`：`tmedia.GetMedia` → `-i`/`-e` 副檔名過濾 → 以 `text/template`（函式來自 `pkg/tplfunc`）產生檔名 → 處理 `--skip-same` → 建立 `<name>.tmp` → 把 `iterElem` 送進 buffered channel（`Value()` 從這個 channel 取值，用 channel 是因為相簿一次會產生多個 elem）。
   - 已被刪除的訊息（`tutil.ErrMessageDeleted`）會被跳過並記錄 ID，下載結束後統一提示使用者。
3. **續傳**：KV key 為 `key.Resume(fingerprint)`，value 是 `finished` 的 JSON。`Run` 的 defer 在發生錯誤（含 Ctrl+C）時呼叫 `saveProgress`，成功完成時則刪除該 key。`--continue`/`--restart` 可以略過詢問。
4. **`core/downloader`**：兩層並行 —— `errgroup.SetLimit(-l)` 控制同時處理的檔案數；每個檔案用 gotd downloader 以 1MB 分片（`MaxPartSize`，Telegram API 上限）並行下載，執行緒數為 `tutil.BestThreads(size, -t)`（依檔案大小分級：<1MB 用 1、<5MB 用 2、<20MB 用 4、<50MB 用 8，且不超過 `-t`）。若 `elem.AsTakeout()` 為真，改用 takeout session 的 client。
5. **`core/dcpool`**：依檔案所在 DC 取得 client。同一個 DC 的 client 採 lazy init 並快取，同時串上 middlewares；目前所在的 DC 用 `api.Pool`，其他 DC 用 `api.DC`（自動完成跨 DC 授權）。建立失敗時退回主 client，test mode 則一律使用主 client。
6. **進度與收尾**：`core/downloader/progress.go` 的 `writeAt` 包住 `WriteAt`，每寫完一片就回呼 `OnDownload`。`app/dl/progress.go` 的 `OnDone` 流程為：關閉檔案 → 失敗則刪除 `.tmp` → 成功則 `it.Finish(logicalPos)`，依 `--rewrite-ext` 以 MIME 修正副檔名，把 `.tmp` 改名為正式檔名，並將 mtime 設為訊息時間。
