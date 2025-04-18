# 消息持久化

在分散式任務處理中，確保消息在伺服器故障或重啟時不丟失至關重要。RabbitMQ 提供了消息持久化機制來保證可靠性。本文將介紹持久化的原理與配置方法。

## 什麼是消息持久化？

消息持久化是指確保消息和隊列結構在伺服器重啟或故障時得以保留的功能。RabbitMQ 通過以下兩方面實現持久化：

### 1. 隊列持久化
- **設定**：宣告隊列時設置 `durable=True`，確保隊列結構在重啟後保留。
- **注意**：僅持久化隊列結構，不包括其中的消息，需與消息持久化結合使用。

### 2. 消息持久化
- **設定**：發送消息時設置 `delivery_mode=2`（或在 `pika` 中使用 `persistent=True`），將消息標記為持久化。
- **原理**：消息會被寫入磁盤，而非僅存於記憶體，只有在磁盤寫入完成後才算成功發送。

## 配置示例（Python `pika`）

以下是使用 Python `pika` 程式庫配置持久化隊列與消息的簡單示例：
```python
import pika

# 建立連接
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# 宣告持久化隊列
channel.queue_declare(queue='durable_queue', durable=True)

# 發送持久化消息
channel.basic_publish(
    exchange='',
    routing_key='durable_queue',
    body='Hello, RabbitMQ!',
    properties=pika.BasicProperties(delivery_mode=2)  # 持久化
)

print("持久化消息已發送")
connection.close()
```

## 確認機制與持久化的結合

持久化確保消息不因伺服器故障而丟失，但無法保證消息是否到達隊列。RabbitMQ 提供發送方確認（Publisher Confirms）機制，確保消息成功存儲：
- **設定**：啟用 `confirm_delivery` 模式，RabbitMQ 會回傳確認訊息。
- **示例**：
  ```python
  channel.confirm_delivery()
  if channel.basic_publish(exchange='', routing_key='durable_queue', body='Test'):
      print("消息確認成功")
  else:
      print("消息發送失敗")
  ```

## 持久化的注意事項

- **性能影響**：持久化消息需要寫入磁盤，會降低吞吐量，適合可靠性要求高的場景。
- **隊列與消息一致性**：隊列與消息均需設置持久化，否則重啟後可能導致數據不一致。
- **磁盤空間**：持久化消息可能佔用大量磁盤空間，需定期清理或設置 TTL。

## 小結

消息持久化是 RabbitMQ 確保可靠性的核心功能，通過配置持久化隊列與消息，您可以避免因故障導致的數據丟失。下一章節，我們將介紹「RabbitMQ 監控與管理工具」，探索如何優化與維護 RabbitMQ 系統。

> **提示**：在高可靠性需求的場景中，建議結合持久化與確認機制使用。
