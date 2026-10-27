# 留在場外 V1.8.12｜台灣每天 21:00 同步、取消 36 小時賠率限制

## 修改內容
- Cloudflare Cron 設為 `0 13 * * *`（UTC 13:00 = 台灣 21:00），每天**一次**，同步 LOL / NBA 賠率並執行現有已啟用的結果／票券結算邏輯。
- 同時更新 `backend/src.js` 裡辨識 `controller.cron` 的條件。**只改 wrangler.toml 不夠**：否則 21:00 只會跑結算，不會同步賠率。
- 同時移除 `index.html` 與 `backend/src.js` 的 36 小時報價年齡限制，避免前端可按、後端拒絕。
- 仍拒絕已開賽、已完賽或開賽時間無效的賽事；現有會員停用／餘額／每日限額與賠率選項驗證仍生效。
- 前台顯示報價擷取時間與「歷史報價可能已變動」提示。取消有效期限表示舊價格也可投入，**不代表這是 Pinnacle 當前報價**。
- 不修改 OddsPapi 請求額度、勝負算法、資料庫、Cloudflare Secrets、會員點數或 admin.html。

## 適用版本
需已安裝 `backend/src.js` V1.8.11，且沿用標準 V1.8.11 的投注判斷與排程程式。會安全檢查，比對失敗就停止，不會覆蓋自訂程式碼。

## Windows CMD 安裝

把此 ZIP 裡的兩個檔案放到實際專案的 `sports-sim\scripts`。不要以 ZIP 中的範例設定覆蓋原本 `wrangler.toml`。

```bat
cd /d "C:\Users\vb096\OneDrive\桌面\sports-sim-v1.3-members\sports-sim\backend"
python ..\scripts\01-set-21-and-unlock-odds-v1812.py
node ..\scripts\02-verify-21-and-unlock-odds-v1812.mjs
```

如顯示 PASS，部署正式 Worker：

```bat
npx wrangler deploy
curl.exe -sS "https://outside-line-api.vb09612.workers.dev/health" | python -m json.tool
```

`/health` 應顯示 `version: 1.8.12`。這個健康檢查無法直接證實 Cron 已執行；可至 Cloudflare Dashboard > Worker > Settings > Triggers 查 Cron 是否為 `0 13 * * *`。Cloudflare Cron 更動可能需要約 15 分鐘傳播。

更新 GitHub Pages：

```bat
cd ..
git add index.html
git commit -m "Use nightly 21 Taiwan sync and unlock quote-age filter"
git push
```

如果 GitHub Pages 使用**不同**儲存庫，請手動把新的 `index.html` 上傳至那個 Pages 專案，不能只推送本機後端儲存庫。

## 重要說明
- 這是每天**一次同步執行**的排程設定，**不保證 Oddspapi 真的回傳新 NBA/LOL 賽事**。如果來源沒有新盤、API 逾時或已觸發當天同步鎖，結果仍可能為零。
- 如果當天早上已執行每日同步，即使改為晚上九點，某些項目仍有當天已嘗試的記錄；首次成功的晚間每日同步可能從隔天開始。
- 原本凌晨/早上的 Cron 及 23:00 結算觸發將由這個唯一 21:00 Cron 取代。夜間 21:00 同步時仍會執行現有的結果同步與票券結算。
- 手動管理 `/api/sync` 不受本修補強迫停止，使用者主動呼叫仍可另外執行。
- 取消 36 小時限制對虛擬模擬較方便，但可能讓舊報價維持可用，務必保留明顯的報價時間顯示；不要將它視為可交易的實時盤口。

## 回復
自動備份存放在 `sports-sim\backend\backups\v1812-...`，包含 `src.js`、`index.html`、`wrangler.toml`。如果要回復，可用備份蓋回原檔並重新部署 Worker／Pages。
