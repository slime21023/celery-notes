# 監控與日誌

在生產環境中，監控 Celery、Kombu 和 RabbitMQ 的運行狀態以及記錄詳細的日誌是確保系統穩定性和快速排查問題的關鍵。本章節將介紹如何設定監控工具和日誌系統，以便您能及時發現問題並優化系統性能。

## Celery 監控工具

Celery 提供了多種監控工具，幫助您追蹤任務狀態、Worker 運行狀況和隊列健康。

### 使用 Flower 進行監控

Flower 是一個基於 Web 的 Celery 監控工具，提供即時的任務狀態、Worker 狀態和隊列資訊。

#### 安裝與啟動

```bash
pip install flower
celery -A your_module flower --port=5555
```

啟動後，您可以在 `http://localhost:5555` 訪問 Flower 的 Web 介面。

#### 主要功能
- **任務狀態**：查看任務的執行狀態（成功、失敗、重試中）。
- **Worker 監控**：檢查 Worker 的在線狀態和處理任務數量。
- **隊列資訊**：查看隊列中的任務數量，避免積壓。

#### 最佳實踐
- 在生產環境中，設定 Flower 的認證功能，避免未授權訪問：

```bash
celery -A your_module flower --basic_auth=user:password
```

- 定期檢查隊列長度，設定告警以避免任務積壓。

### 使用 Celery Events

Celery Events 是一個內建的事件系統，允許您監聽任務和 Worker 的事件。

#### 使用示例

```python
from celery import Celery
from celery.events import EventReceiver

app = Celery('tasks', broker='amqp://localhost')

def my_monitor(app):
    state = app.events.State()
    
    def on_event(event):
        state.event(event)
        task = state.tasks.get(event['uuid'])
        print(f"任務 {task.name} 狀態：{task.state}")
    
    with app.connection() as connection:
        recv = EventReceiver(connection, handlers={'*': on_event})
        recv.capture(limit=None, timeout=None, wakeup=True)

if __name__ == '__main__':
    my_monitor(app)
```

#### 最佳實踐
- 將事件記錄到日誌系統或資料庫中，以便長期分析。
- 使用事件監聽來觸發告警，例如任務失敗時發送通知。

## RabbitMQ 監控

RabbitMQ 提供了管理插件和 API，幫助您監控隊列、交換機和連接狀態。

### RabbitMQ 管理插件

啟用 RabbitMQ 管理插件：

```bash
rabbitmq-plugins enable rabbitmq_management
```

啟動後，您可以在 `http://localhost:15672` 訪問管理介面（默認用戶：guest，密碼：guest）。

#### 主要功能
- 查看隊列長度、消息速率和消費者數量。
- 檢查交換機和綁定關係。
- 監控伺服器資源使用情況（如記憶體和 CPU）。

#### 最佳實踐
- 設定資源限制，避免單個隊列佔用過多資源。
- 定期檢查未確認消息數量，確保消費者正常工作。

### 使用 Prometheus 和 Grafana

RabbitMQ 支持 Prometheus 指標匯出，可以與 Grafana 整合進行可視化監控。

#### 設定步驟
1. 啟用 `rabbitmq_prometheus` 插件：

```bash
rabbitmq-plugins enable rabbitmq_prometheus
```

2. 配置 Prometheus 抓取 RabbitMQ 指標。
3. 在 Grafana 中匯入 RabbitMQ 儀表板。

#### 最佳實踐
- 設定告警規則，例如隊列長度超過閾值時發送通知。
- 長期儲存指標數據，分析系統趨勢。

## 日誌記錄

詳細的日誌記錄是排查問題的基礎，Celery、Kombu 和 RabbitMQ 都支持日誌配置。

### Celery 日誌

設定 Celery 的日誌級別和輸出位置：

```python
from celery import Celery
import logging

app = Celery('tasks', broker='amqp://localhost')

# 設定日誌
logging.basicConfig(filename='celery.log', level=logging.INFO)
app.conf.task_track_started = True  # 記錄任務開始時間
app.conf.task_send_sent_event = True  # 記錄任務發送事件
```

### Kombu 日誌

Kombu 的日誌可以幫助排查消息傳輸問題：

```python
import logging

logging.getLogger('kombu').setLevel(logging.DEBUG)
```

### RabbitMQ 日誌

RabbitMQ 的日誌文件通常位於安裝目錄下，檢查日誌以了解伺服器錯誤或連接問題。

#### 最佳實踐
- 使用日誌輪轉工具（如 `logrotate`）管理日誌文件大小。
- 將日誌集中到日誌管理系統（如 ELK Stack）進行分析。

## 總結與注意事項

- **選擇合適的監控工具**：Flower 適合 Celery 任務監控，RabbitMQ 管理插件適合消息代理監控，Prometheus 和 Grafana 適合長期指標分析。
- **設定告警**：對於關鍵指標（如隊列長度、任務失敗率），設定告警以便及時響應。
- **記錄詳細日誌**：確保 Celery、Kombu 和 RabbitMQ 的日誌級別適當，方便排查問題。
- **定期審查**：定期檢查監控數據和日誌，發現潛在問題並優化系統。

通過有效的監控和日誌記錄，您可以確保分散式任務系統的穩定運行，並快速解決問題。
