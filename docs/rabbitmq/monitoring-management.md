# RabbitMQ 監控與管理工具

RabbitMQ 的效能與可靠性很大程度上取決於有效的監控與管理。本文將介紹 RabbitMQ 提供的監控和管理工具，以及常見的最佳實踐，幫助您構建穩定的消息系統。

## RabbitMQ 管理界面

RabbitMQ 提供了一個內建的 Web 管理界面，方便查看系統狀態與配置資源。

- **啟用方式**：安裝後執行以下命令啟用管理插件：
  ```bash
  sudo rabbitmq-plugins enable rabbitmq_management
  ```
- **訪問方式**：通過瀏覽器訪問 `http://localhost:15672`，預設用戶名與密碼為 `guest`/`guest`。
- **功能**：
  - 查看隊列、交換機與綁定關係。
  - 監控消息速率與隊列長度。
  - 手動創建或刪除資源。
  - 管理用戶與權限。

## RabbitMQ 命令行工具（rabbitmqctl）

`rabbitmqctl` 是 RabbitMQ 提供的命令行工具，用於管理伺服器與資源。

- **查看系統狀態**：
  ```bash
  sudo rabbitmqctl status
  ```
- **列出隊列**：
  ```bash
  sudo rabbitmqctl list_queues
  ```
- **列出交換機**：
  ```bash
  sudo rabbitmqctl list_exchanges
  ```
- **管理用戶**：
  ```bash
  sudo rabbitmqctl add_user myuser mypassword
  sudo rabbitmqctl set_user_tags myuser administrator
  ```
- **管理虛擬主機**：
  ```bash
  sudo rabbitmqctl add_vhost myvhost
  sudo rabbitmqctl set_permissions -p myvhost myuser ".*" ".*" ".*"
  ```

## 監控指標與工具

為了確保 RabbitMQ 的穩定運行，需定期監控關鍵指標：

- **隊列長度**：過長可能表示消費者處理速度不足。
- **消息速率**：發送與消費速率是否平衡。
- **連接數量**：過多連接可能導致資源耗盡。
- **記憶體與磁盤使用**：避免資源過載導致系統崩潰。

### 第三方監控工具
- **Prometheus + Grafana**：通過 RabbitMQ 的 Prometheus 插件收集指標，並使用 Grafana 可視化。
  - 啟用插件：
    ```bash
    sudo rabbitmq-plugins enable rabbitmq_prometheus
    ```
  - 訪問指標：`http://localhost:15692/metrics`
- **Datadog/Zabbix**：支援 RabbitMQ 監控的商業解決方案。

## 最佳實踐

- **用戶安全**：禁用預設 `guest` 用戶，創建新用戶並設置複雜密碼。
  ```bash
  sudo rabbitmqctl delete_user guest
  ```
- **資源限制**：為隊列設置最大長度與 TTL，避免無限增長。
- **高可用性**：配置 RabbitMQ 集群，實現故障轉移（進階主題，後續介紹）。
- **日誌管理**：定期檢查 RabbitMQ 日誌（通常位於 `/var/log/rabbitmq/`），排查潛在問題。

## 小結

RabbitMQ 提供多種監控與管理工具，幫助您掌握系統狀態並及時解決問題。通過合理配置與監控，您可以確保消息系統的穩定性與高效性。本章節結束了「RabbitMQ 核心」的學習，下一階段我們將進入 Celery 的相關內容，探索如何基於 RabbitMQ 實現分散式任務處理。

> **提示**：定期監控隊列長度與系統資源是避免 RabbitMQ 問題的關鍵，建議結合自動化工具實現實時告警。
