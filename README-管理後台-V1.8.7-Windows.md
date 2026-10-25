# 留在場外 V1.8.7 — 管理員賽事後台（Windows CMD）

本版本基於 V1.8.6，每腿勝負圖示、Pinnacle / OddsPapi、ESPN NBA、PandaScore 預覽、Cloudflare D1 與會員點數不會被安裝腳本清空或重置。

## 1. Windows 專案位置

```text
C:\Users\vb096\OneDrive\桌面\sports-sim-v1.3-members\sports-sim\
├── index.html                    # 前台：自動顯示手動新增的運動分類
├── admin.html                    # 新增的管理頁面
├── backend\
│   ├── src.js                    # V1.8.7 /api/admin/fixtures 路由
│   ├── admin-fixtures.js         # 新增 / 修改賠率 / 比分預覽 / 確認寫入
│   └── wrangler.toml             # 請保留現有設定！
└── scripts\
    ├── 01-install-admin-v187.py
    ├── 02-verify-admin-v187.mjs
    └── v187-assets\              # 不能漏掉整個資料夾
        ├── admin.html
        ├── admin-fixtures.js
        └── admin-routes.txt
```

## 2. 建議使用「Windows 修補包」

將 ZIP 裡的 `sports-sim\scripts\` 資料夾**合併**至你目前專案的 `sports-sim\scripts\`，包括 `v187-assets` 子資料夾。不要使用這個 ZIP 取代整個現有專案。

在 Windows CMD 執行，每行各按一次 Enter：

```bat
cd /d "C:\Users\vb096\OneDrive\桌面\sports-sim-v1.3-members\sports-sim\backend"
python ..\scripts\01-install-admin-v187.py
node ..\scripts\02-verify-admin-v187.mjs
```

看到 `PASS` 再繼續：

```bat
npx wrangler deploy
curl.exe -sS "https://outside-line-api.vb09612.workers.dev/health" | python -m json.tool
```

`version` 應為 `1.8.7`。其他既有設定（例如 `idOnlyLoginEnabled`、`nbaAutoResultsEnabled`）會沿用你目前的 `wrangler.toml`，不會被修補程式改動。

## 3. 更新 GitHub Pages

GitHub Pages 所使用的儲存庫需要更新**兩個前端檔案**：

- `index.html`：讓前台的運動篩選依 D1 動態顯示，並將手動賠率與 OddsPapi/Pinnacle 報價分開標示。
- `admin.html`：新管理頁面。後台 API 與 Token 驗證均在 Cloudflare Worker，HTML 不包含管理員 Token。

若目前 `sports-sim` 就是 Git 儲存庫，從 `backend` 回上一層後可執行：

```bat
cd ..
git status
git add index.html admin.html backend/src.js backend/admin-fixtures.js
git commit -m "Add secured event admin panel v1.8.7"
git push
```

**如果 GitHub Pages 用另一個儲存庫**，只需將這兩個 HTML 上傳至 Pages 所使用的資料夾，請勿將 `wrangler.toml` 或管理員 Token 上傳 GitHub。

管理頁面網址為你的 GitHub Pages **前台 `index.html` 同一層**的 `admin.html`，例如：

```text
https://whchenpt15-source.github.io/<repo>/admin.html
```

`<repo>` 請改成前台實際使用的儲存庫路徑；如果是使用者主頁，網址可能沒有 `<repo>`。

## 4. 直覺操作管理頁面

1. 進入 `admin.html`，輸入既有的管理員 `APP_ACCESS_TOKEN`，點「驗證並載入賽事」。Token 只存在目前分頁的記憶體，不保存到 localStorage。
2. 「新增賽事」填寫**運動分類、聯賽、主客隊、台灣開賽時間、各自獲勝賠率**，可選填和局賠率。新增分類如「足球」不必修改資料表。
3. 「賽事管理清單」可依隊名／聯賽搜尋，切換「尚未結算」「最近賽事」「手動建立」，修改*僅手動建立且未開賽*的賠率。
4. 比賽正式完賽後，在清單按「輸入賽果」；**客隊比分／主隊比分**皆填寫正式結果，包括需要計算的加時賽。按「先預覽」會列出每腿勝負和涉及的票券數，不改 D1。
5. 逐項核對後按「確認比分並結算相關票券」。Worker 將正式比分存入 `fixtures`，補上需依供應商 ID 判定的 `ticket_legs.provider_result`，再呼叫原有 `settleTickets` 結算票券、`members.balance_cents` 和 `ledger`。
6. 如果賽果已完成但餘額尚未跟上，可在底部「補跑結算」進行**全站待結算**檢查；操作不會覆寫已完成賽事。

**重要限制**：賽事尚未開賽時拒絕手動提交正式比分；已經是 `completed` 的賽事不能透過此頁覆寫，以防重複發放點數。若比分和既有 `provider_result` 衝突也會拒絕；請人工核對資料來源。二選一勝負盤不支援和局賽果，若該運動可能和局，建立賽事時應填和局賠率。

## 5. 安全與保留項目

- 使用原本的 `APP_ACCESS_TOKEN` Wrangler Secret，沒有將 Token 寫入 HTML、URL、D1 或 ZIP。
- 後台 API 僅允許具有管理員 Bearer Token 的請求，且沿用 `FRONTEND_ORIGIN` 瀏覽器來源限制。
- 所有 SQL 都使用參數綁定；後端檢查比賽時間、比分、賠率和重複結算。
- 賠率修改只影響未來的投注；票券在 `ticket_legs.odds` 中保存下單時賠率。
- 未新增 D1 schema 或 migration；不需要更改 `wrangler.toml`，不需要修改現有 APP_ACCESS_TOKEN。
- 備份由修補腳本保留在 `backend\backups\admin-v187-<時間>\`；該路徑已在 `.gitignore` 裡。
- 尚未在你的 Cloudflare 線上環境部署；測試是使用 SQLite 內建工具模擬與實際後端 API 呼叫路徑測試。

## 6. 常見問題

- **403 需要管理員權限**：確認輸入的是 Worker Secret `APP_ACCESS_TOKEN` 而不是會員 ID/Session Token。
- **此網站來源未獲允許**：確認 `wrangler.toml [vars] FRONTEND_ORIGIN` 和 Pages 所在網域的 origin 完全一致，**不要包含 `/repo` 路徑**。
- **找不到 `/api/admin/fixtures`**：Worker 尚未部署到 `1.8.7`；請從 `backend` 重新執行 `npx wrangler deploy`。
- **前台沒有新增賽事**：網站的時間範圍預設 7 天，且只顯示尚未開賽的賽事，請切換 14 天並重新整理。
- **已輸入比分但部分串關還是灰色**：其他腿仍未正式確認時會保留待確認標誌；這是預期的行為。
