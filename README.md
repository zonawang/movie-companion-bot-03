# 電影夥伴 LINE Bot 03：附近影院與電影場次

延續 [movie-companion-bot-02](https://github.com/zonawang/movie-companion-bot-02)，在既有 Google ADK Agent 加入位置查詢。傳送 LINE 定位，或輸入地址／地標，即可查附近影院，再沿用既有來源查詢實際電影與場次。

**2026-10-26 已部署並切換 LINE：[Zona Movie（@796qcwyz）](https://line.me/R/ti/p/@796qcwyz)**。Maps key 已設定，原有 1 筆票根已完整搬移；服務不再依賴個人電腦或臨時 tunnel。主機與管理方式見 [部署說明](docs/deployment.md)。

本版使用 **Google Maps Platform Places API (New) + ADK 自訂 FunctionTool** 實作 map grounding；沒有使用 Gemini 內建 Google Maps grounding，也不需要 Grounding Lite MCP 或改用 Vertex AI。

## 實作文章

- [從查分館到找附近：用 Google ADK 與 Google Maps，讓電影夥伴依位置找場次](medium.md)

## 可以這樣問

- 直接傳送 LINE 的「位置」：找半徑 3 公里內最近 3 間影院，查今天尚未開演場次。
- 「我在台北車站，附近三公里今晚六點後有哪些電影可以看？」
- 「台北市信義區市府路 1 號附近有哪些電影院？只看影院，不用查場次。」
- 傳定位後再問：「改找五公里內的影院。」
- 「京站威秀明天晚上六點後有哪些場次？」
- 「清除位置」：清除目前 ADK 對話與定位，已確認票根保留。

電影簡介、評論、觀影票根草稿與確認、影城清單分頁及單館／多館場次查詢均沿用 bot-02。

## 回覆與限制

回覆列出影院名稱、地址、Google Maps 連結、直線距離，以及可靠對應分館後的片名、版本與時間。場次和地點來源分開列出，模型不重寫場次或猜剩餘座位。

- 搜尋範圍 100 公尺至 10 公里，預設 3 公里；列出 1～5 間，預設 3 間，最多自動查 3 館場次。
- 地標有多個候選時先請補城市／完整地點或傳定位，不擅選第一筆。
- 以精確別名、縣市與已有行政區核對分館。無法可靠對應時提供地圖入口，說明尚無法確認片單。
- 地圖結果可能超出原有 108 個已登錄據點。場次仍受前版覆蓋限制：102 個有結構化場次、6 個只有圖片入口；這是 2026-10-26 來源快照，非完整營業名冊。
- 誠品使用官方公開資料，其他影城使用開眼第三方時刻表。有場次不代表仍有票；沒有票價、即時座位或代訂票。
- 距離為直線估算，不提供步行／開車時間或即時交通。
- 沒有 Maps 金鑰時會明確提示，原有指定影城、電影與票根功能仍可使用。

## 安裝與設定

需要 Node.js 24.13 以上、npm 11.8 以上。

```bash
git clone https://github.com/zonawang/movie-companion-bot-03.git
cd movie-companion-bot-03
npm ci --ignore-scripts
cp .env.example .env
```

沿用既有 Gemini、LINE 與選用 TMDB 設定，另加地圖金鑰：

```dotenv
GEMINI_API_KEY=你的_Gemini_key
GEMINI_MODEL=gemini-3.5-flash-lite
LINE_CHANNEL_SECRET=你的_LINE_Channel_secret
LINE_CHANNEL_ACCESS_TOKEN=你的_LINE_Channel_access_token
MOVIE_DATA_MODE=tmdb
TMDB_READ_ACCESS_TOKEN=你的_TMDB_Read_Access_Token
GOOGLE_MAPS_API_KEY=你的_Google_Maps_Platform_key
PORT=3002
SQLITE_PATH=./data/movie-bot.sqlite
```

Google Cloud 專案需啟用 **Places API (New)** 與帳務，金鑰需有該 API 權限；Gemini key 不會自動具有地圖權限。Text Search 同時解析地址與地標，不另需 Geocoding API。請設定符合伺服器環境的金鑰限制與用量監控，金鑰僅填本機 `.env`。

```bash
npm run chat  # 終端機地標測試，不需要 LINE 設定
npm start     # LINE webhook 本機服務
```

本次已完成雲端切換；以下安裝流程不會自動更動其他 LINE 帳號。自行取代舊版本前，先備份原 SQLite、停止舊服務，將資料搬至新目錄並核對，再設定 HTTPS tunnel／webhook；不要讓兩個服務同時使用同一資料庫。見 [設定與切換](docs/line-setup.md)。

`MOVIE_DATA_MODE=demo` 時，TMDB 電影工具為明確標示的虛構示範資料；地圖與指定影院場次仍使用各自真實來源。場次工具不依賴 TMDB key。

## 驗證

```bash
npm test                       # 離線回歸及本機 HTTP webhook，不呼叫付費 API
npm run smoke:maps             # 真實 Maps + 場次，需 Maps key，會使用 API 額度
npm run smoke:maps:adk         # 真實 Gemini + Maps + 場次，需 Gemini/Maps key
npm run smoke:maps:adk:fixture # 真實 Gemini + 明確替代地圖資料 + 真實場次
```

smoke scripts 使用記憶體資料庫，不傳送 LINE 訊息、不寫入正式票根。地圖替代資料不能視為真實 Maps 驗證。執行紀錄見 [bot-03 驗證](docs/verification-03.md)；[前版驗證](docs/verification.md) 保留。

## 程式結構

| 檔案 | 職責 |
| --- | --- |
| `src/maps.mjs` | Places Text Search／Nearby Search、座標驗證、距離與錯誤處理 |
| `src/nearby.mjs` | 地點與分館配對、最多三館場次查詢、固定格式輸出 |
| `src/maps-place-ids.json` | 人工核對後的 Place ID → 分館 ID |
| `src/agent.mjs` | ADK Agent、10 個 FunctionTool、位置隔離與期限 |
| `src/line.mjs` / `src/app.mjs` | LINE 定位、簽章、去重、清除位置與長訊息拆分 |
| `src/cinema.mjs` | 既有單館／多館場次及時間篩選 |
| `src/cinema-registry.mjs` | 影城登錄、別名、縣市及品牌查詢 |
| `src/store.mjs` | SQLite 票根確認、使用者隔離與事件佇列 |

LINE 精確座標不放入模型文字或票根，最多於記憶體保留 15 分鐘；待處理事件暫存 SQLite，完成後清空。文字地址會交給 Gemini 處理。見 [設計與設定](docs/map-grounding.md)、[資料來源](docs/credits.md)、[隱私說明](docs/privacy.md) 與 [使用條款](docs/terms.md)。
