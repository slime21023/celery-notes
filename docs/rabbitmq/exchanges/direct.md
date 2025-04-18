# Direct Exchange（直接交換機）

Direct Exchange（直接交換機）是 RabbitMQ 中最簡單的交換機類型，用於精確匹配路由鍵的消息分發。本文將介紹其工作原理與應用場景。

## 工作原理

- **路由規則**：Direct Exchange 根據消息的路由鍵（Routing Key）與隊列綁定時指定的鍵進行完全匹配。
- **分發邏輯**：只有當路由鍵與綁定鍵完全一致時，消息才會被分發到對應隊列。
- **特點**：簡單高效，適合點對點通信。

## 配置示例（Python `pika`）

以下是使用 Python `pika` 程式庫配置 Direct Exchange 的簡單示例：
```python
import pika

# 建立連接
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# 宣告 Direct Exchange
channel.exchange_declare(exchange='direct_tasks', exchange_type='direct')

# 宣告隊列
channel.queue_declare(queue='email_queue')

# 綁定隊列到交換機，指定路由鍵
channel.queue_bind(exchange='direct_tasks', queue='email_queue', routing_key='email')

# 發送消息
channel.basic_publish(exchange='direct_tasks', routing_key='email', body='Send email task')

print("消息已發送")
connection.close()
```

## 應用場景

- **任務分配**：將特定類型的任務（如 `email` 或 `report`）發送到專用隊列，由特定消費者處理。
- **簡單路由**：需要精確控制消息流向的場景，避免不必要的廣播。

## 小結

Direct Exchange 通過精確匹配路由鍵實現簡單高效的消息分發，適合需要點對點通信的場景。下一章節，我們將介紹「Topic Exchange」，探索更靈活的模式匹配路由機制。

> **提示**：Direct Exchange 是最常用的交換機類型之一，適合大多數簡單任務分發需求。
