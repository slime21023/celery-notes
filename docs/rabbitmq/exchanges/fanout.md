# Fanout Exchange（扇形交換機）

Fanout Exchange（扇形交換機）是 RabbitMQ 中用於廣播消息的交換機類型，適合發布/訂閱模式。本文將介紹其工作原理與應用場景。

## 工作原理

- **路由規則**：Fanout Exchange 無視路由鍵（Routing Key），將消息分發到所有綁定的隊列。
- **分發邏輯**：只要隊列與交換機綁定，所有消息都會被複製並發送到這些隊列。
- **特點**：簡單高效，適合廣播通信。

## 配置示例（Python `pika`）

以下是使用 Python `pika` 程式庫配置 Fanout Exchange 的簡單示例：
```python
import pika

# 建立連接
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# 宣告 Fanout Exchange
channel.exchange_declare(exchange='fanout_notifications', exchange_type='fanout')

# 宣告隊列
channel.queue_declare(queue='notification_queue1')
channel.queue_declare(queue='notification_queue2')

# 綁定隊列到交換機（無需路由鍵）
channel.queue_bind(exchange='fanout_notifications', queue='notification_queue1')
channel.queue_bind(exchange='fanout_notifications', queue='notification_queue2')

# 發送消息
channel.basic_publish(exchange='fanout_notifications', routing_key='', body='System notification')

print("消息已發送")
connection.close()
```

## 應用場景

- **系統通知**：將通知消息廣播到所有訂閱者，如應用更新或系統維護公告。
- **即時更新**：將狀態變化通知所有相關組件，如價格變動或庫存更新。

## 小結

Fanout Exchange 通過廣播機制實現簡單高效的消息分發，適合發布/訂閱模式的場景。下一章節，我們將介紹「Headers Exchange」，探索基於標頭的路由機制。

> **提示**：Fanout Exchange 適合需要廣播的場景，但可能導致消息過多，需合理控制訂閱者數量。
