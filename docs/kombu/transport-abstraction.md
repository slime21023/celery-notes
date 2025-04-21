# 傳輸層抽象：如何適配不同工具？

## 什麼是傳輸層抽象？

Kombu 的一大特點是「傳輸層抽象」。這意味著您可以用統一的方式發送消息，而不用管底下用的是什麼工具（比如 RabbitMQ 或 Redis）。就像您可以用同一個手機 App 叫不同品牌的計程車，Kombu 幫您處理與不同「消息代理」的溝通細節。

簡單來說，Kombu 提供了一個「通用接口」，讓您專注於消息本身，而不用擔心工具的差異。

## Kombu 支持哪些工具？

Kombu 可以與很多不同的「消息代理」合作，包括：

- **RabbitMQ**：功能強大，支持複雜的分發規則，適合大多數場景。
- **Redis**：簡單快速，適合輕量級任務。
- **Amazon SQS**：雲端服務，適合在 AWS 上使用。
- **其他**：還有 Zookeeper、MongoDB 等，適應不同需求。

## 如何告訴 Kombu 用哪個工具？

您只需要在「連接字符串」中指定工具，Kombu 就會自動適配。以下是幾個例子：

```python
from kombu import Connection

# 用 RabbitMQ
conn_rabbitmq = Connection('amqp://user:pass@localhost:5672//')

# 用 Redis
conn_redis = Connection('redis://localhost:6379/0')

# 用 Amazon SQS
conn_sqs = Connection('sqs://aws_access_key:aws_secret_key@')
```

## 為什麼傳輸層抽象有用？

- **靈活性**：如果您原本用 RabbitMQ，後來想改用 Redis，只需改連接字符串，不用改其他代碼。
- **簡單性**：您不用學習每種工具的細節，Kombu 幫您處理。
- **一致性**：不管用哪種工具，發送和接收消息的方式都一樣。

## 注意事項

- 不同工具的功能不同：RabbitMQ 支持複雜規則，Redis 則比較簡單。選工具時要考慮您的需求。
- 性能差異：有些工具快，有些慢，根據您的消息量和速度要求選擇。

通過這個概念，您可以專注於消息本身，而不用擔心底層工具的差異。下一章我們會看看如何自己接收消息！
