# 術語表

本附錄整理了與 Celery、Kombu 和 RabbitMQ 相關的常用術語及其簡要定義，幫助您快速理解這些工具的核心概念。術語表採用表格格式，方便閱讀和查詢。

| 術語                  | 定義                                                                 |
|-----------------------|----------------------------------------------------------------------|
| **AMQP**              | 先進消息隊列協議，RabbitMQ 使用的標準協議，用於消息傳輸。              |
| **Broker**            | 消息代理，負責接收和分發消息，Celery 使用 RabbitMQ 或 Redis 作為 Broker。 |
| **Celery**            | 基於 Python 的分散式任務隊列框架，用於處理異步任務和定時任務。          |
| **Concurrency**       | Celery Worker 能同時處理的任務數量，與硬體資源相關。                  |
| **Durable**           | RabbitMQ 中的屬性，確保隊列或交換機在服務重啟後仍然存在。              |
| **Exchange**          | RabbitMQ 中的組件，負責將消息分發到隊列，類型包括 direct、topic 等。   |
| **Flower**            | Celery 的監控工具，提供 Web 界面查看任務和 Worker 狀態。              |
| **Kombu**             | Python 庫，Celery 的底層消息傳輸層，用於與 RabbitMQ 等後端交互。      |
| **Message**           | 在 RabbitMQ 和 Celery 中傳輸的數據單位，通常包含任務信息。            |
| **Queue**             | RabbitMQ 中的組件，用於儲存消息，等待消費者處理。                     |
| **Result Backend**    | Celery 中用於儲存任務結果的後端，如 Redis，用於狀態查詢。            |
| **Routing Key**       | RabbitMQ 中用於決定消息分發到哪個隊列的標識。                        |
| **Task**              | Celery 中的工作單位，表示需要異步執行的函數或操作。                   |
| **Worker**            | Celery 中的進程，負責從隊列中獲取任務並執行。                        |

如果您需要對某個術語有更深入的了解，建議查閱官方文檔或相關教程。
