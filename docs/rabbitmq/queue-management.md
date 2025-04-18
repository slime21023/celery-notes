# 隊列（Queue）管理

隊列（Queue）是 RabbitMQ 中存儲消息的緩衝區，消費者從隊列中取出消息進行處理。本文將介紹隊列的基本屬性、配置方法及管理技巧，幫助您設計高效的消息處理系統。

## 什麼是隊列？

隊列是 RabbitMQ 中用於暫存消息的組件，生產者通過交換機將消息分發到隊列，消費者訂閱隊列以處理消息。每個隊列可以有不同的屬性與行為，適應各種應用需求。

## 隊列的核心屬性

- **耐久性（Durable）**：隊列是否在伺服器重啟後保留。設為 `durable=True` 可持久化隊列結構（不包括消息）。
- **自動刪除（Auto-Delete）**：隊列在最後一個消費者斷開後是否自動刪除，適合臨時隊列。
- **排他性（Exclusive）**：隊列是否僅限單一連接使用，連接斷開後自動刪除，適合臨時會話。
- **最大長度（Max Length）**：限制隊列中消息數量，避免內存過載。
- **消息生存時間（Message TTL）**：設置消息在隊列中的最長存活時間，過期後自動移除。
- **死信交換機（Dead Letter Exchange, DLE）**：當消息被拒絕或過期時，轉發到指定交換機進行處理。

## 隊列配置示例（Python `pika`）

以下是使用 Python `pika` 程式庫配置隊列的簡單示例：
```python
import pika

# 建立連接
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# 宣告持久化隊列
channel.queue_declare(queue='task_queue', durable=True)

# 宣告帶 TTL 和最大長度的隊列
args = {'x-message-ttl': 60000, 'x-max-length': 100}  # TTL 60秒，最大長度100
channel.queue_declare(queue='limited_queue', arguments=args)

print("隊列已宣告")
connection.close()
```

## 隊列管理技巧

- **隊列命名**：使用有意義的名稱（如 `email_tasks`、`error_logs`），便於識別與調試。
- **資源限制**：設置最大長度或 TTL，避免隊列無限增長導致記憶體問題。
- **死信處理**：配置死信交換機與隊列，處理失敗或過期的消息。
  ```python
  args = {'x-dead-letter-exchange': 'dlx_exchange'}
  channel.queue_declare(queue='main_queue', arguments=args)
  ```
- **隊列監控**：定期檢查隊列長度與消息處理速度，及時調整消費者數量。

## 隊列操作命令（RabbitMQ CLI）

使用 `rabbitmqctl` 工具管理隊列：
- **列出所有隊列**：
  ```bash
  sudo rabbitmqctl list_queues
  ```
- **刪除隊列**：
  ```bash
  sudo rabbitmqctl delete_queue task_queue
  ```
- **清除隊列消息**：
  ```bash
  sudo rabbitmqctl purge_queue task_queue
  ```

## 小結

隊列是 RabbitMQ 中消息存儲與處理的基礎，通過合理配置隊列屬性與管理策略，您可以設計高效可靠的消息系統。下一章節，我們將介紹「消息持久化」，探索如何確保消息在故障時不丟失。

> **提示**：隊列屬性配置需根據業務需求謹慎選擇，避免過度限制或資源浪費。
