# Bot 03 驗證紀錄

日期：2026-10-26（Asia/Taipei）。基礎來源為 bot-02 的影城擴充版本 `66691a1f1994889baec628ea3d499c46a76e0b91`，新增位置功能於獨立 bot-03 目錄，沒有修改 bot-02 服務或資料庫。

## 依賴與自動化測試

使用 Node.js 24.22.1，以本版 `package-lock.json` 完成乾淨安裝（`npm ci --ignore-scripts`）。保留 `@google/adk` 2.3.0，未新增執行期套件。

執行 `npm test`，**74 項全部通過**，涵蓋既有電影、票根、分館、場次、圖片來源、清單分頁、LINE 簽章與去重，以及新增：

- Places API (New) 請求格式、必要欄位、台灣語系與距離排序。
- 停業／重複／半徑外地點、缺 key、額度、HTTP 錯誤與異常回應。
- 地址／地標多候選，以及唯一完整名稱解析。
- 精確分館配對、錯縣市／行政區、無法配對與人工 Place ID 對應。
- 最多三館場次、單館失敗隔離、日期／時間／片名參數。
- LINE 定位事件欄位驗證、私訊隔離、重送不重查、事件處理後內容清空。
- 真正 ADK Runner 呼叫新 FunctionTool、模型虛構時間不採用、模型後續失敗保留已取得結果。
- 位置使用者隔離、15 分鐘期限與自動清除、清除位置指令。
- 原始 LINE 經緯度不進模型文字、長 LINE 回覆保留最後一館來源。

完整套件新增 GitHub Actions workflow，在 push／pull request 上執行乾淨安裝與測試。

## 真實 Gemini／ADK：已執行

以 `gemini-3.5-flash-lite` 執行 `smoke:maps:adk:fixture`。此測試 **Google Maps 回應為明確的替代資料**，Gemini／ADK 與開眼場次讀取為真實服務。

1. 模擬 LINE 傳入公開示範位置：模型呼叫 `nearby_cinemas`，地圖 adapter 回傳京站威秀，程式對應 `vieshow-qsquare`，讀取當天剩餘場次。
2. 文字「我在台北車站，附近三公里今晚六點以後有哪些電影可以看？」：模型呼叫同工具，正確帶入地標與 `18:00`，先解析位置再查影院，輸出真實開眼時間與來源。

測試使用記憶體 SQLite，不傳 LINE 訊息、不寫正式票根，驗證後清除定位。通過此測試只能證明真實模型選工具與場次整合，不能當成真實 Google Maps 測試。

## 真實 Google Maps 與手機：待設定

既有設定沒有 `GOOGLE_MAPS_API_KEY`，因此未執行真實 Places API 查詢，也未測量實際 Google 地點名稱的自動配對覆蓋率。

已提供 `smoke:maps` 與 `smoke:maps:adk`。設定已啟用 Places API (New) 的 Maps key 後可執行；不同分館名稱如未可靠配對，會提供地圖連結而不生成場次。

本次未切換既有 LINE webhook、未停止 bot-02，也未將 bot-03 部署成常駐服務。手機傳定位與 LINE 回覆需在設定／切換後實際確認。
