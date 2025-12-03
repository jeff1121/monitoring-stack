# Monitoring Stack

一組以 Prometheus + Thanos 為核心、搭配 Alertmanager、Grafana、Node Exporter、cAdvisor 及 MinIO 的觀測性範例專案，適合作為在單一主機上驗證完整監控鏈的起點。透過 docker-compose 一鍵啟動，即可同時取得短期與長期度量資料、統一告警出口與現成儀表板。

## 功能重點
- **短期 + 長期度量**：Prometheus 保存近兩天資料，Thanos 透過 MinIO 進行長期儲存並提供集中查詢。
- **自動告警**：包含基礎設施、Thanos、MinIO、Grafana、主機與容器資源等多組告警規則，並經由 Alertmanager 路由。
- **即時視覺化**：Grafana 預先配置資料源與 Node Exporter 儀表板，啟動後即可觀察本機硬體與容器狀態。
- **可擴充**：以 Docker Compose 組態為基礎，方便加入其他 exporter、資料源與儀表板。

## 專案結構
```
monitoring-stack/
├── docker-compose.yml                     # 主要編排檔
├── prometheus/
│   ├── prometheus.yml                     # Prometheus 設定
│   └── alerts.yml                         # 告警規則
├── alertmanager/
│   └── alertmanager.yml                   # Alertmanager 設定與路由
├── grafana/
│   └── provisioning/
│       ├── datasources/datasource.yml     # Grafana 資料源
│       └── dashboards/
│           ├── dashboards.yml             # 儀表板供應設定
│           └── json/node-exporter-overview.json
├── thanos/
│   └── bucket.yml                         # Thanos 物件儲存設定
└── data/                                  # 由 volume 掛載的資料目錄 (MinIO/Prometheus)
```

## 服務清單
| 服務 | 容器 | 埠號 | 說明 |
|------|------|------|------|
| Prometheus | `prometheus` | 9090 | 短期 TSDB、抓取與規則引擎 |
| Thanos Sidecar | `thanos-sidecar` | 10902 | 對接 Prometheus 的 gRPC/HTTP 端點 |
| Thanos Store | `thanos-store` | 10901 | 從 S3 讀取區塊，供 Query 節點使用 |
| Thanos Compactor | `thanos-compact` | - | 壓縮與維護歷史區塊 |
| Thanos Query | `thanos-query` | 10902 | Prometheus 相容查詢端點與 UI |
| MinIO | `minio` | 9000 / 9001 | 提供 S3 相容儲存 Thanos 區塊 |
| Alertmanager | `alertmanager` | 9093 | 告警路由與通知 |
| Grafana | `grafana` | 3000 | 視覺化與儀表板管理 |
| Node Exporter | `node-exporter` | 9100 | 主機硬體度量 |
| cAdvisor | `cadvisor` | 8080 | 容器層級度量 |

## 先決條件
- Docker 24+ 與 Docker Compose v2
- 4 GB 以上 RAM（建議），並允許容器掛載主機 `/proc`、`/sys`、`/var/lib/docker` 等路徑

## 環境變數
將 [.env.template](.env.template) 複製為 [.env](.env) 並設定 MinIO 與 Grafana 憑證，Docker Compose 會自動載入同層 [.env](.env)。範本預設值僅供測試，建議在正式環境改用強密碼並妥善保護檔案權限。

## 使用方式
1. 請依照 [quickstart.md](quickstart.md) 完成 [.env](.env) 建置與堆疊啟動。
2. 初次啟動後須於 MinIO (http://localhost:9001) 建立 `thanos` bucket，登入帳密為 [.env](.env) 中的 `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD`，或改用 `mc` CLI 指令。
3. Grafana 介面位於 http://localhost:3000，使用 [.env](.env) 中的 `GF_SECURITY_ADMIN_USER` / `GF_SECURITY_ADMIN_PASSWORD` 登入後即可看到「Node Exporter - 主機監控」儀表板。
4. 若要新增自訂 exporter 或告警規則，可直接修改對應目錄並重新載入/重啟服務。

## 延伸調整建議
- **安全性**：於生產環境啟用 TLS、使用 Vault 或 Secrets 管理敏感憑證，並限制網路存取。
- **多節點部署**：可為每個 Prometheus 節點加上不同的 `external_labels`，並將 `thanos-sidecar`、`thanos-store` 指向共同的物件儲存。
- **告警整合**：更新 `alertmanager.yml` 中的 webhook、Email 或 Slack 設定，對接實際通知渠道。
- **儀表板與資料源**：依照 Grafana provisioning 架構，加入更多 JSON 儀表板或 datasources。

