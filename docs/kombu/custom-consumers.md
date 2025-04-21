# 自定義消費者：如何接收消息？

## 什麼是消費者？

在 Kombu 中，「消費者」（Consumer）是負責接收消息並處理它的一方。就像您從郵箱取信並閱讀，消費者從「隊列」中取出消息，然後做一些事情（比如執行任務）。

## 為什麼要自定義消費者？

雖然 Celery 提供了現成的消費者（Worker），但自定義消費者讓您能完全控制消息的處理方式，比如決定什麼時候確認消息、如何處理錯誤等。這在需要特殊邏輯時特別有用。

## 簡單實踐：創建一個消費者

讓我們從最簡單的例子開始：接收一條消息並打印出來。

```python
from kombu import Connection, Consumer, Exchange, Queue

# 建立連接
conn = Connection('amqp://localhost')
exchange = Exchange('my_exchange', type='direct')
queue = Queue('my_queue', exchange, routing_key='greeting')

# 定義如何處理消息
def process_message(body, message):
    print(f"收到消息：{body}")
    message.ack()  # 告訴系統：我已經處理完了

# 創建消費者並開始接收
with Consumer(conn, queues=queue, callbacks=[process_message], accept=['json']):
    conn.drain_events()  # 等待消息（會一直運行）
```

在這個例子中：

1. 我們連接到消息代理（這裡用 RabbitMQ）。
2. 定義了一個交換機和隊列，告訴系統我們要從哪裡取消息。
3. 定義了一個函數 `process_message`，告訴系統收到消息後要做什麼。
4. 創建消費者，開始等待消息。

## 更進一步：處理錯誤

有時候，處理消息可能會出錯（比如數據格式不對）。我們可以告訴系統，如果出錯，應該把消息放回去重新處理：

```python
def process_message(body, message):
    try:
        print(f"處理消息：{body}")
        # 模擬錯誤
        if 'error' in body:
            raise ValueError("出錯了！")
        message.ack()  # 成功就確認
    except Exception as e:
        print(f"錯誤：{e}")
        message.reject(requeue=True)  # 出錯就放回去
```

## 小技巧

- **確認消息**：處理完一定要調用 `message.ack()`，不然消息會一直留在隊列中。
- **多個消息**：可以設置 `prefetch_count` 一次處理多條消息，加快速度。
- **停止接收**：可以用 `consumer.cancel()` 停止消費者。

## 注意事項

- 不要忘記確認消息，否則隊列會越積越多。
- 如果消費者運行很久，可能會占用很多內存，記得偶爾重啟。
- 記錄錯誤信息，方便查找問題。

通過這個章節，您已經學會了如何接收並處理消息。下一章我們會看看如何發送消息！
