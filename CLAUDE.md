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
