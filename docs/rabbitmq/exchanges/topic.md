# Topic Exchange（主題交換機）

Topic Exchange（主題交換機）是 RabbitMQ 中用於模式匹配的消息分發組件，適合複雜路由需求。本文將介紹其工作原理與應用場景。

## 工作原理

- **路由規則**：Topic Exchange 根據路由鍵的模式匹配進行消息分發，支持通配符 `*`（匹配單詞）和 `#`（匹配多詞）。
- **分發邏輯**：隊列綁定時指定模式，消息的路由鍵符合模式時會被分發到對應隊列。
- **特點**：靈活強大，適合多層次的事件分類。

## 通配符說明

- `*`：匹配單個詞，例如 `log.*` 可匹配 `log.error` 或 `log.info`，但不匹配 `log.error.critical`。
- `#`：匹配零個或多個詞，例如 `user.#` 可匹配 `user.login` 或 `user.login.success`。

## 配置示例（Python `pika`）

以下是使用 Python `pika` 程式庫配置 Topic Exchange 的簡單示例：
```python
import pika

# 建立連接
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# 宣告 Topic Exchange
channel.exchange_declare(exchange='topic_logs', exchange_type='topic')

# 宣告隊列
channel.queue_declare(queue='error_logs')

# 綁定隊列到交換機，指定模式
channel.queue_bind(exchange='topic_logs', queue='error_logs', routing_key='*.error')

# 發送消息
channel.basic_publish(exchange='topic_logs', routing_key='app.error', body='Error log message')

print("消息已發送")
connection.close()
```

## 應用場景

- **日誌系統**：根據日誌等級或來源分發消息，如 `app.error` 或 `server.warning`。
- **事件通知**：根據事件類型或層次結構分發消息，如 `user.login.success` 或 `user.logout`。

## 小結

Topic Exchange 通過模式匹配實現靈活的消息分發，適合需要分類或層次化路由的場景。下一章節，我們將介紹「Fanout Exchange」，探索廣播式消息分發機制。

> **提示**：Topic Exchange 是處理複雜路由需求的最佳選擇，但模式設計需謹慎以避免過度匹配。
