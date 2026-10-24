# 留在場外 V1.1 — 後端快取賠率 / 雲端虛擬帳戶

**只有虛擬點數，不會向 Pinnacle 或任何平台進行真實投注。**

這一版將 V1.0 的單機本機帳戶改成「GitHub Pages 前端 + Cloudflare Worker + D1」的**單一使用者**原型：

- The Odds API 的 Key **只存在 Cloudflare Secret**，不會被下載到 HTML。
- 固定台灣時間每日 **08:00** 同步一次：NBA (`basketball_nba`)、MLB (`baseball_mlb`)、英超 (`soccer_epl`) 的 Pinnacle 賠率及最近三天的完賽比分。
- 預設抓 `h2h`（勝負）、`spreads`（讓分）、`totals`（大小分）。遇到 Pinnacle 暫無報價時不產生假數據。
- 開網頁只呼叫自己的 Worker `/api/state`，由 D1 取快取，不會呼叫 The Odds API。
- 下注與點數扣款由後端處理；押注當下的賠率固定存到 D1。支援單場和串關、每日虛擬投入上限、24 小時暫停、結算點數及 CSV 以外的 JSON 備份。
- 賽事 `completed=true` 且具有雙方最終分數時，後端自動計算勝負、讓分、大小分；走水退款、未完成賽事保留待結算。後端在每天同步時會結算，不需要開啟網頁。
- 一次排程通常消耗 `3 聯賽 × (3 市場 + 2 比分 credits) = 15 credits`；以 30 天估計 450 credits。額外查詢、失敗重試、資料供應商政策變更需另計，實際用量以 The Odds API 回傳的回應標頭為準。
- 用 D1 每日唯一鍵保證同一台灣日期各聯賽、各資料種類最多「嘗試」一次，手動按同步不會因此重覆扣額度。失敗時當日**不自動重試**，須待次日或由管理員手動修復紀錄後再抓。

> ⚠ 注意：此版本尚未部署到你的 Cloudflare 帳戶，必須按照下方步驟部署。你**不需要把 API Key 傳給任何人**。

## 0. 安裝需求

需要電腦（Windows PowerShell / macOS Terminal / Linux）和 Node.js 20+，並登入你自己的 Cloudflare 帳戶，且已有 The Odds API 的 API Key。

解壓 ZIP 後，進入 `sports-sim/backend` 資料夾。Windows PowerShell 例如：

```powershell
cd .\sports-sim\backend
```

Node.js 官網： https://nodejs.org/
Cloudflare Workers： https://dash.cloudflare.com/

## 1. 登入 Cloudflare

```bash
npx wrangler login
```

瀏覽器會開啟 Cloudflare 的授權畫面。

## 2. 建立 D1 資料庫

```bash
npx wrangler d1 create outside-line-db
```

指令會顯示 D1 資料庫的 UUID。打開 `backend/wrangler.toml`，找到：

```toml
[[d1_databases]]
binding = "DB"
database_name = "outside-line-db"
database_id = "REPLACE_WITH_YOUR_D1_DATABASE_UUID"
```

將 `database_id` 改成實際 UUID（只改這一處）。

然後設定 `FRONTEND_ORIGIN` 為你的 GitHub Pages 網站**來源（origin）**。例如你的站是 `https://abc.github.io/sports-sim/`，應填：

```toml
FRONTEND_ORIGIN = "https://abc.github.io"
```

**不要**加 `/sports-sim/`，也不要加尾端斜線。若使用自訂網域就填自己的 `https://domain.example`。

## 3. 建立資料表

```bash
npx wrangler d1 migrations apply outside-line-db --remote
```

系統會根據 `migrations/0001_initial.sql` 建立持久化資料表。

## 4. 部署 Worker

```bash
npx wrangler deploy
```

會得到類似下面的後端網址：

```text
https://outside-line-api.你的-cloudflare-subdomain.workers.dev
```

開啟 `https://...workers.dev/health` 應得到 JSON `{ "ok": true, ... }`。`/health` 不需密碼，也不呼叫 The Odds API。

## 5. 設定兩個 Cloudflare Secret（**非常重要**）

第一個 Secret 是你**已經擁有**的 The Odds API Key：

```bash
npx wrangler secret put ODDS_API_KEY
```

終端機詢問時貼入 Key，**不要把它寫進 HTML / GitHub / wrangler.toml**。

第二個 Secret 是你自己新設的「網站存取密碼」。**和 The Odds API Key 不同**，長度至少 20 字元，建議 32 位以上的隨機密碼，請另外保管：

```bash
npx wrangler secret put APP_ACCESS_TOKEN
```

這個密碼稍後輸入到網頁「第一次連線」表單，以 Bearer Token 傳送給 Worker。網頁會存在本機 LocalStorage 以便下次自動登入；在公用設備使用後請按「清除此裝置連線設定」。

Cloudflare Secrets 官方文件：https://developers.cloudflare.com/workers/configuration/secrets/

## 6. 上傳 GitHub Pages 前端

建立**獨立的 `sports-sim` Repository**，並將下列檔案放到儲存庫的**根目錄**（不要把後端 Secret 或其他開發測試檔放進 GitHub）：

- `sports-sim/index.html`（新版）
- `sports-sim/legacy-v1.html`（保留舊版，供原瀏覽器回看舊紀錄）

GitHub > Settings > Pages > Deploy from branch（通常 `main` + `/root`），開啟網站。

網站第一次開啟會要你填：
1. **Worker 網址**（例如 `https://outside-line-api.xxx.workers.dev`）
2. **網站存取密碼**（剛設定的 `APP_ACCESS_TOKEN`，*不是* Odds API Key）

連線成功後可建立虛擬帳戶（預設 10,000 點、每日上限 500 點）。

> 如果看到 CORS / Forbidden / Network Error，優先檢查 `wrangler.toml` 的 `FRONTEND_ORIGIN` 是否等於瀏覽器網站的 `https://網域`。改好要重新 `npx wrangler deploy`。瀏覽器不要以 `file://` 直接開啟正式前端：正式前端預期由 GitHub Pages 提供 HTTPS origin。

## 7. 第一次抓賽事（不用等明天早上）

進入網站「**資料與同步**」，點擊「**執行今日同步（首次測試）**」。系統會依序抓預設的三個聯賽，每種資料每日最多嘗試一次。

之後 Cron 每天 UTC 00:00（台灣時間 08:00）同步；若今天已按過手動同步，該天 Cron 會略過同日既有的抓取紀錄。

每日同步狀態和回傳的實際 credits 可在網站「資料與同步」頁查看。抓不到資料的可能原因：該聯賽休賽、Pinnacle 沒有開出盤口、你的 API 訂閱沒有該資料、供應商比分覆蓋不足。無資料不代表前端壞掉。

如想節省額度，只抓勝負：將 `backend/wrangler.toml` 的 `MARKETS` 改為 `"h2h"`，重新部署。但同一天已抓取的資料不會重新抓，待翌日同步生效。

## 8. 保存與結算規則

- 只要 D1 不被人為刪除，關掉 Safari 或清除前端快取不會刪掉**雲端**下注紀錄；若清除瀏覽器儲存，需要重新輸入 Worker 網址和存取密碼。
- 同一筆投注在資料庫中使用唯一 ID；後端只更新 `pending` 的票券一次。押注時間、金額、賠率固定留存，不會因日後賠率更新而回溯更改。
- 勝負：比較主隊／客隊最終比分；足球英超允許和局。
- 讓分：比較 `(選取隊得分 - 對方得分) + 讓分`。相等算走水、該腿賠率改為 1。
- 大小：比較總分與盤口門檻。相等算走水、退款。
- 串關：有一腿輸即全輸；其餘腿還沒完賽就等待；走水腿從乘積移除。整張都走水則退回本金。
- Score API 僅能回查最近 3 天，且不保證所有運動都有完整比分；若缺結果會保留 `pending`，**絕不臆測**。超過開賽 72 小時仍未有結果，可於「資料與同步」選擇人工**作廢退款**（不是自行宣稱勝負）。
- 運動項目的結算細節（足球加時賽、賽事腰斬、棒球提前結束及市場專用條款）不一定完全等於 Pinnacle 官方的真實下注規則。此為虛擬教育原型，不可用來核算真實投注。
- **不是即時賠率**：每日一次的價格快照，超過 36 小時自動禁止作為新模擬投注報價。

## 原有 V1.0 紀錄與遷移

新 V1.1 是**全新的雲端帳戶**。為避免直接將無法核對的舊版手動結算／示範賽事納入真實賽事的自動結算，本版**不會自動把 V1.0 的 localStorage 合併到 D1**。

`legacy-v1.html` 是原版備份頁面：若部署在原網站相同的 domain/path 下，通常可以繼續讀到瀏覽器內原本的 V1.0 資料。請先於舊版的「資料管理」匯出 JSON 備份，再開啟新版本。重新部署、瀏覽器資料清除等情形可能造成舊版瀏覽器資料遺失，請務必備份。

## 成本試算與限制

- 3 聯賽 ×（3 種盤口 × 1 區域當量 + Scores 回查 2）= **15 credits/日**，30 天約 **450 credits**。如果 API 沒有賠率賽事，部分 calls 可能不扣，但仍要依實際回應標頭確認。
- 官方說明： https://the-odds-api.com/liveapi/guides/v4/
- Cloudflare Cron（UTC 時區）： https://developers.cloudflare.com/workers/configuration/cron-triggers/
- Cloudflare D1： https://developers.cloudflare.com/d1/get-started/
- 本版為**單人私用、單一帳戶**，只有一組 `APP_ACCESS_TOKEN`。不適合作為多人公開下注平台。如果未來要讓多位使用者分開儲存，需要另外增加真正的帳號驗證、每個帳號的 row 級授權與防濫用。
- 本地語法／模擬測試已執行；尚未取得你的 Cloudflare 帳戶權限，也未曾使用你的實際 Odds API Key，因此**不聲稱已實際部署或取得真正報價**。

## 檔案內容

```text
sports-sim/
  index.html                 # GitHub Pages 前端，無 Key
  legacy-v1.html             # 舊版純本機備份
  backend/
    wrangler.toml             # Worker 與排程設定，無 Key
    src.js                   # API、同步、結算、權限邏輯
    migrations/
      0001_initial.sql       # D1 資料表
    test/
      smoke.mjs              # Node SQLite 煙霧測試
      browser-check.py       # Chromium 模擬互動測試
  README.md
```
