# Bot 03 雲端部署

本版使用 Google Compute Engine 常駐執行，保留 SQLite 與非同步 webhook 工作佇列，不依賴個人電腦或 Quick Tunnel。

## 資源

| 項目 | 設定 |
| --- | --- |
| 專案 | `free-9000` |
| 主機 | `movie-bot-03`，`us-west1-b`，`e2-micro` |
| 磁碟 | 30 GB `pd-standard`，刪除主機時不自動刪除此磁碟 |
| 公開 IP | `136.67.154.252`（保留的固定 IPv4） |
| HTTPS | `https://movie-bot-03.136-67-154-252.sslip.io` |
| Webhook | 上述網址的 `/webhook` |
| LINE | Zona Movie，`@796qcwyz` |
| 程式來源 | `/opt/movie-bot/releases/<commit>`，`/opt/movie-bot/app` 為目前版本連結 |
| 票根與事件 | `/var/lib/movie-bot/movie-bot.sqlite` |
| 金鑰 | Secret Manager 的 `movie-bot-03-env`；不在 GitHub 或 VM metadata |
| 執行身分 | 獨立 service account，只授權讀取這一份 Secret Manager secret |
| 管理 | IAP SSH；對外只開 HTTP/HTTPS，SSH 不對網際網路開放 |

Google Maps key 限定 `places.googleapis.com` 且限定本機固定 IP 使用。Gemini、LINE 與 TMDB 設定沿用既有電影 Bot，Map key 為獨立建立。

HTTPS 由 Caddy 管理，使用 sslip.io 公開 DNS 與自動更新憑證。網域依賴第三方 DNS；日後可換成自己的網域，更新 VM 的 `app-domain` metadata、Caddy 設定與 LINE webhook。沒有再使用短期 tunnel。

## 啟動與管理

`deploy/gce-startup.sh` 為可重複執行的 VM startup script，從下列非機密 metadata 讀取設定：`app-project`、`app-release`、`app-domain`、`app-secret`、`node-version`。應用程式來源固定到完整 commit，Node.js 官方套件經 SHA-256 驗證。

```bash
gcloud compute ssh movie-bot-03 --project=free-9000 --zone=us-west1-b --tunnel-through-iap
sudo systemctl status movie-bot caddy
sudo journalctl -u movie-bot --since '30 minutes ago'
sudo systemctl restart movie-bot
```

程序由 systemd 在開機自動啟動、失敗自動重啟，以獨立使用者執行。應用程式只監聽 loopback 的 3002，Caddy 處理外部 HTTPS。Secret Manager 內容由 VM 身分在 startup 時讀入 root-only 的 `/etc/movie-bot/environment`。

變更 Secret Manager 後，需要重跑 startup script 取得新內容，再重啟服務；不要把金鑰放入命令參數、聊天或版本控制。

```bash
sudo google_metadata_script_runner startup
```

更新版本時，先完成測試、備份資料，再將 `app-release` 設為已提交的完整 commit，更新 startup script metadata 並重跑。依賴不完整的版本不要切入；需回復時將 release 指回上一個已驗證的 commit。

## 備份與還原

票根在持久磁碟上。另有每日 SQLite 線上備份 timer，保留最近 7 份於 `/var/lib/movie-bot/backups/`；備份和主資料位於同一磁碟，不能取代異地災難備援。

```bash
sudo systemctl start movie-bot-backup.service
sudo systemctl status movie-bot-backup.timer
```

還原時先停止 `movie-bot`，備份現況，再將選定 SQLite 複本還原為主資料庫，確認擁有者 `moviebot:moviebot`、權限 600，最後啟動並核對票根。不要直接複製正在寫入的 SQLite；請使用 SQLite backup API。

## 費用

Google Free Tier 提供符合資格地區的每月一台 e2-micro 使用時數與 30 GB-month 標準磁碟，額度以帳戶合計，不保證此專案永遠免費。[官方免費額度](https://docs.cloud.google.com/free/docs/free-cloud-features)

使用中的一般 VM 外部 IPv4 定價為 US$0.005／小時，持續 730 小時約 US$3.65／月，另計超額傳輸與 Maps／Gemini API 費用。[官方網路費率](https://cloud.google.com/vpc/network-pricing)

若停止使用，應先備份票根，再停用服務、刪除不需要的 VM／磁碟／IP 並撤銷專用 Maps key。單純停止 VM 仍可能產生磁碟及保留 IP 費用。
