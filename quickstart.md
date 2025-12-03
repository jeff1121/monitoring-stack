# Quick Start

以最少步驟啟動整個監控堆疊。如果已經熟悉流程，可直接跳到「啟動堆疊」。

## 1. 先決條件
- macOS / Linux / WSL2 (建議 x86_64)
- Docker Desktop 4.29+ 或 Docker Engine 24+
- Docker Compose v2 (`docker compose version` 能正常執行)
- Git 與 GitHub CLI (僅在要 fork / push 至 GitHub 時需要)

## 2. 取得原始碼
```bash
git clone https://github.com/jeff1121/monitoring-stack.git
cd monitoring-stack
```
若已複製到本機，請確定目前分支為 `main` 且 `git pull` 為最新狀態。

## 3. 建立 .env
將 [.env.template](.env.template) 複製為 [.env](.env)，並調整符合需求的憑證：
```bash
cp .env.template .env
```
- `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD`：MinIO Console 及 API 登入
- `GF_SECURITY_ADMIN_USER` / `GF_SECURITY_ADMIN_PASSWORD`：Grafana 管理員帳密
- `GF_USERS_ALLOW_SIGN_UP`：是否允許自行註冊（預設 `false`）

## 4. 啟動堆疊
```bash
docker compose up -d
```
第一次啟動會自動抓取所有映像及建立 named volumes，所需時間依網路而定。可用 `docker compose ps` 確認所有服務皆為 `running`。

## 5. 初始化 MinIO Bucket
Thanos 需要名為 `thanos` 的 bucket：
1. 開啟 <http://localhost:9001>，登入帳密為 [.env](.env) 中設定（若沿用範本則為 `admin / change-me`）。
2. 使用 Console 新增 bucket，名稱輸入 `thanos`。

或透過 `mc` CLI：
```bash
mc alias set myminio http://localhost:9000 <MINIO_ROOT_USER> <MINIO_ROOT_PASSWORD>
mc mb myminio/thanos
```

## 6. 驗證服務
| 項目 | 驗證方式 |
|------|----------|
| Prometheus | <http://localhost:9090> → Status ▸ Targets，確認 `UP` |
| Thanos Query | <http://localhost:10902> → 嘗試 `up` 查詢 |
| Grafana | <http://localhost:3000> → 帳密為 [.env](.env) 中設定（範本預設 `admin / change-me`），首次登入請立即更換 |
| Alertmanager | <http://localhost:9093> |
| Node Exporter | <http://localhost:9100/metrics> |
| cAdvisor | <http://localhost:8080> |

登入 Grafana 後，可在 `Infrastructure/Node Exporter - 主機監控` 儀表板查看 CPU/Memory/Disk/Network/容器使用情況。

## 7. 常用操作
```bash
# 查看日誌
docker compose logs -f prometheus

# 熱更新 Prometheus 規則（修改檔案後）
docker kill --signal=SIGHUP prometheus

# 停止並移除容器
docker compose down

# 停止但保留 volume
docker compose down --volumes=false
```

## 8. 清理資源
若要完全移除所有資料與映像：
```bash
docker compose down -v
docker image prune --filter reference='prom/*|quay.io/thanos/*|grafana/grafana|minio/minio|gcr.io/cadvisor/cadvisor'
```
請注意這會刪除 Prometheus/Thanos/MinIO 中的歷史資料。

## 9. 下一步
- 參考 `prometheus/alerts.yml` 新增自訂告警。
- 在 `grafana/provisioning/dashboards/json/` 放入更多 JSON 儀表板。
- 更新 `alertmanager/alertmanager.yml` 對接真實通知渠道 (Email/Slack/Webhook)。

完成以上步驟後，即可使用完整的觀測性環境進行測試與練習。