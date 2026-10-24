# 留在場外 V1.3 — 每支手機首次建立 ID，自動登入

這是一款**只使用虛擬點數的戒賭模擬帳戶**：沒有真實投注、儲值、提款、現金獎金。賽事與賠率由 OddsPapi 在 Cloudflare Workers 後端定時同步，所有人共用快取。

## 功能

- **朋友只需開啟 GitHub Pages 網址**，第一次輸入 3～24 位英數字、底線或連字號建立 ID，即可獲得 10,000 虛擬點數，預設每日限額 500 點。
- **下一次開啟同一瀏覽器自動登入**。真正的登入憑證是伺服器生成的 256-bit 隨機 token，存放於該瀏覽器的 `localStorage`；**ID 只是帳戶名稱，不是密碼**。
- 建立帳戶時顯示一次 **64 字元復原碼**。務必複製保存；清除 Safari 網站資料、換手機、無痕模式結束後，可以在首頁用 **ID＋復原碼**重新登入。復原碼存於資料庫的是 SHA-256 雜湊，不保存明碼。
- **每個 ID 有獨立餘額、投注票券、流水、每日限額與 24 小時暫停**。其他人不會看到你的私人紀錄，朋友也不需要知道後端管理 Token。
- **OddsPapi 的賽事及賠率全站共用**，打開網站不會增加 OddsPapi API 次數。仍支援 NBA、MLB、LOL、羽球；開出的盤口依 OddsPapi 帳戶資料而定。
- **後端每日排程及完賽結果結算仍在 Worker 進行**，不需開啟手機；只有盤口結果為 WIN／LOSE／PUSH 才結算。
- **管理員同步／作廢操作不暴露在公開前端**，需要 Cloudflare Worker 的 `APP_ACCESS_TOKEN` 才能呼叫。

> 此版適合小群朋友測試。尚未加入防大量註冊、裝置遺失管理、完整帳戶自助支援與高強度營運風控。若要公開讓陌生人大量註冊，建議加上 Cloudflare Turnstile、Rate Limiting、帳戶異常偵測。`localStorage` 只供同一瀏覽器自動登入；請避免在公共電腦建立帳戶。
>
> OddsPapi 賠率來源能否公開轉載給朋友，仍應依你的 API 授權條件確認。

## 檔案清單

```text
sports-sim/
  index.html                    # 上傳 GitHub Pages 的公開前端
  README.md
  legacy-v1.html                # 舊本機版本（可選擇保留）
  backend/
    src.js                      # 多人帳戶驗證、下單、結算與管理 API
    oddspapi.js                 # 賠率與結算資料同步（延續 V1.2）
    wrangler.toml               # 排程、D1 綁定與環境設定
    migrations/
      0001_initial.sql
      0002_oddspapi.sql
      0003_members.sql          # 新增會員表及私人下注關聯，不刪除舊資料
    test/
      smoke-members.mjs
      smoke-oddspapi.mjs
```

## 第一步：更新 Cloudflare Worker 與 D1

### 已部署 V1.2 的使用者

1. **請先備份 D1**，並使用相同的 `outside-line-db`（不要建立新資料庫）。`wrangler.toml` 中的 `database_id` 一定要保留原本 UUID；`FRONTEND_ORIGIN` 繼續填 GitHub Pages 的 **origin**，例如 `https://YOUR_USERNAME.github.io`，**不要包含 `/sports-sim` 子路徑**。
2. 使用 ZIP 裡新版 `src.js`、`oddspapi.js`、`migrations` 等覆蓋本機 `sports-sim/backend` 原本內容。保留原本 `APP_ACCESS_TOKEN` 與 `ODDSPAPI_API_KEY` Cloudflare Secrets，不要直接放入 HTML。
3. 電腦在 `sports-sim/backend` 目錄執行：

```bash
npx wrangler login
npx wrangler d1 migrations apply outside-line-db --remote
npx wrangler deploy
```

### 首次部署的使用者

安裝 Node.js 20+，在 `sports-sim/backend` 執行：

```bash
npx wrangler login
npx wrangler d1 create outside-line-db
```

把建立資料庫時回傳的 UUID 填入 `wrangler.toml` 的 `database_id`，修改 `FRONTEND_ORIGIN` 為你的 GitHub Pages origin，然後：

```bash
npx wrangler d1 migrations apply outside-line-db --remote
npx wrangler secret put ODDSPAPI_API_KEY
npx wrangler secret put APP_ACCESS_TOKEN
npx wrangler deploy
```

其中 `ODDSPAPI_API_KEY` 是你原本的 OddsPapi Key；`APP_ACCESS_TOKEN` 是**另一組**至少 20 字元的私密管理密碼，用來手動同步及作廢。**請不要分享給朋友**。

Cloudflare 排程預設 `0 0 * * *`（UTC 午夜 = 台灣每天 08:00），並遵守 OddsPapi 每日／每月呼叫上限；帳戶操作不會觸發外部賠率 API。

## 第二步：前端只改一行 Worker 網址

用文字編輯器打開 **`index.html`**，搜尋這一行：

```javascript
const API_BASE_URL='https://YOUR_WORKER_SUBDOMAIN.workers.dev';
```

把引號中的網址替換成部署成功後 Cloudflare 顯示的 Worker HTTPS 網址（**不要**附加 `/health` 或 `/api/state`）。例如：

```javascript
const API_BASE_URL='https://outside-line-api.your-subdomain.workers.dev';
```

> 這是示範網址，必須換成你自己的真實 Worker 網址。**公開 `index.html` 中絕對不能寫 `ODDSPAPI_API_KEY` 或 `APP_ACCESS_TOKEN`。**

將 `index.html` 上傳到原本 GitHub Pages 專案，再將網址加上 `?v=13` 清除舊版快取。

**此後你把這個網站網址分享給朋友，朋友只會看到「輸入 ID 建立帳戶」畫面，不需要再填 Worker 網址或任何 API Key。**

## 使用方式

1. 第一次用手機進入網站：輸入 ID（例如 `alice_01`）、按「建立我的模擬帳戶」。
2. 新帳戶初始餘額 10,000 點、每日限額 500 點。建立時畫面會顯示一組長復原碼，**請立即複製保存**。
3. 下次用同個 Safari 網站／瀏覽器再進入時，使用儲存的登入 token **自動登入，不需再輸入 ID**。
4. 想改用另一支手機，或清除網站資料：點「已經有 ID？換手機／重新登入」，輸入 ID 加上你保存的復原碼。
5. 「資料與同步」可檢視個人 ID、匯出個人資料與登出此裝置；好友不能以 ID 猜中你的私人資料。

**注意：** `localStorage` 是依照 **網站 origin＋瀏覽器** 儲存，而不是依手機硬體唯一編號。相同手機切換 Safari／Chrome、無痕模式或更換網址可能會視為新登入。遺失復原碼且裝置也沒保存登入狀態時，無法單靠 ID 找回原帳戶。

## 管理員選用：手動執行當天同步

一般會員的 HTML 不會提供管理員同步按鈕。若真的需要測試，**只有管理員**可透過終端機用管理密碼呼叫：

```bash
curl -X POST 'https://YOUR-WORKER.workers.dev/api/sync' \
  -H 'Authorization: Bearer YOUR_APP_ACCESS_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{}'
```

請替換 Worker 網址與管理密碼；勿把命令或密碼分享到公開聊天室或 GitHub。現有每日同步去重與免費額度保護仍有效。

## 舊版 V1.2 既有帳戶紀錄

**不會刪除**：遷移會保留 V1.2 原有的 `account`、舊票券、舊流水與賽事快取。為避免把舊帳戶錯誤指派給第一個碰巧選到某 ID 的朋友，**不會自動把原本單人帳戶內容轉移給任何新 ID**。新的多人成員 ID 都從 10,000 點開始；需要將舊的紀錄轉入指定 ID 時，應由資料庫管理員確認身分後另做一次性的資料移轉。**請勿直接刪除舊資料表。**

## 驗證

測試指令：

```bash
node --experimental-default-type=module backend/test/smoke-members.mjs
node --experimental-default-type=module backend/test/smoke-oddspapi.mjs
```

測試涵蓋：兩人互不查看／修改對方票券與餘額、共用賽事資料、每個人正確扣款與入帳、避免重複結算、復原碼登入、管理員權限與舊資料保留；也測試 OddsPapi 假資料同步。**這不代表已連線你的真實 Cloudflare 帳戶、真實 OddsPapi API 或實體 iPhone Safari，部署後仍需實際驗證。**
