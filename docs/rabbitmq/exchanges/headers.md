# Headers Exchange（標頭交換機）

Headers Exchange（標頭交換機）是 RabbitMQ 中基於消息標頭進行路由的交換機類型，適合需要高靈活性的場景。本文將介紹其工作原理與應用場景。

## 工作原理

- **路由規則**：Headers Exchange 根據消息的標頭（Headers）而非路由鍵進行匹配。
- **分發邏輯**：隊列綁定時指定標頭條件，消息標頭符合條件時會被分發到對應隊列。
- **匹配模式**：
  - `x-match=all`：消息標頭必須完全符合所有指定條件。
  - `x-match=any`：消息標頭符合任一條件即可。
- **特點**：靈活性高，但性能相對其他交換機類型稍低。

## 配置示例（Python `pika`）

以下是使用 Python `pika` 程式庫配置 Headers Exchange 的簡單示例：
```python
import pika

# 建立連接
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# 宣告 Headers Exchange
channel.exchange_declare(exchange='headers_tasks', exchange_type='headers')

# 宣告隊列
channel.queue_declare(queue='priority_tasks')

# 綁定隊列到交換機，指定標頭條件
bind_args = {'x-match': 'all', 'priority': 'high', 'type': 'urgent'}
channel.queue_bind(exchange='headers_tasks', queue='priority_tasks', arguments=bind_args)

# 發送消息，設置標頭
headers = {'priority': 'high', 'type': 'urgent'}
channel.basic_publish(
    exchange='headers_tasks',
    routing_key='',
    body='Urgent high-priority task',
    properties=pika.BasicProperties(headers=headers)
)

print("消息已發送")
connection.close()
```

## 應用場景

- **複雜條件路由**：根據多個消息屬性（如優先級、類型）進行精細分發。
- **業務邏輯分發**：根據消息元數據（如來源、版本）決定處理路徑。

## 小結

Headers Exchange 通過標頭匹配實現高度靈活的消息分發，適合複雜條件路由的場景。雖然性能稍低，但在需要基於多屬性決策時非常有用。

> **提示**：Headers Exchange 適合特定需求場景，若路由需求簡單，建議優先考慮其他交換機類型以提升性能。
