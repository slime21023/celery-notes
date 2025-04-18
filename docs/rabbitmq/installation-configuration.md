# RabbitMQ 安裝與配置

RabbitMQ 是一個功能強大的開源消息隊列系統，本文將指導您如何在本地環境中安裝和配置 RabbitMQ，為後續學習與實踐做好準備。

## RabbitMQ 安裝步驟

以下是以 Ubuntu 系統為例的安裝指南，其他系統（如 Windows 或 macOS）可參考官方文件（[RabbitMQ 安裝指南](https://www.rabbitmq.com/download.html)）。

### 步驟 1：更新系統包
```bash
sudo apt update
sudo apt upgrade
```

### 步驟 2：安裝 RabbitMQ 伺服器
```bash
sudo apt install rabbitmq-server
```

### 步驟 3：啟動 RabbitMQ 服務
```bash
sudo systemctl start rabbitmq-server
sudo systemctl enable rabbitmq-server
```

### 步驟 4：檢查服務狀態
```bash
sudo systemctl status rabbitmq-server
```
如果服務正常運行，會顯示 `active (running)`。

### 步驟 5：啟用管理插件（可選）
RabbitMQ 提供一個 Web 管理界面，方便查看隊列與消息狀態。
```bash
sudo rabbitmq-plugins enable rabbitmq_management
```
啟用後，通過瀏覽器訪問 `http://localhost:15672`，預設用戶名與密碼為 `guest`/`guest`。

## 基本環境測試

安裝完成後，可以使用以下命令測試 RabbitMQ 是否正常運行：
```bash
rabbitmqctl status
```
輸出中應包含節點資訊與運行狀態。

## RabbitMQ 基本配置

RabbitMQ 的配置文件通常位於 `/etc/rabbitmq/rabbitmq.conf`（具體路徑依系統而異）。以下是一些常見配置項：

- **監聽端口**：預設為 5672（AMQP 協議）和 15672（管理界面）。
  ```ini
  listeners.tcp.default = 5672
  ```
- **用戶與權限**：預設用戶為 `guest`，建議創建新用戶並設置權限。
  ```bash
  # 創建新用戶
  sudo rabbitmqctl add_user myuser mypassword
  # 設置管理員權限
  sudo rabbitmqctl set_user_tags myuser administrator
  # 設置虛擬主機權限
  sudo rabbitmqctl set_permissions -p / myuser ".*" ".*" ".*"
  ```
- **虛擬主機（Virtual Host）**：用於隔離不同應用環境，預設為 `/`。
  ```bash
  sudo rabbitmqctl add_vhost myvhost
  ```

## 小結

通過安裝與基本配置，您已經搭建了一個本地 RabbitMQ 環境。下一章節，我們將介紹「交換機（Exchange）類型與用途」，探索 RabbitMQ 中消息分發的核心機制。

> **提示**：記錄安裝過程中的任何問題（如端口衝突），可參考官方文件或社群資源解決。
