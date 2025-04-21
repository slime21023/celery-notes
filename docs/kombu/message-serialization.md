# 消息序列化：消息如何打包？

## 什麼是消息序列化？

在網絡上傳輸數據時，我們需要把 Python 中的數據（比如一個字典）轉換成能在網絡上傳送的格式，然後在另一端再把它轉回來。這個過程就叫做「序列化」（打包）和「反序列化」（解包）。

在 Kombu 中，消息序列化就是把您的數據（比如任務參數）打包成能在網絡上傳輸的格式，然後在接收端解包還原。

## Kombu 支持哪些打包方式？

Kombu 提供了幾種不同的「打包工具」，每種都有自己的特點：

- **JSON**：這是最常用的格式，容易讀懂，適合大多數情況。特別是當您需要與其他語言的系統交互時，JSON 是個好選擇。
- **Pickle**：這是 Python 專用的打包方式，能處理很複雜的數據，但有安全風險（不建議用於不受信任的環境）。
- **YAML**：類似 JSON，但更適合人類閱讀，速度稍慢。
- **MessagePack**：這是一種高效的格式，數據小、速度快，適合需要高性能的場景。

## 如何選擇打包方式？

選擇打包方式就像選擇寄包裹的方式：

- 如果包裹要送到外國（跨語言系統），用 JSON，因為大家都認識。
- 如果包裹很小但要快（高性能需求），用 MessagePack。
- 如果包裹是給自己人（純 Python 環境），可以用 Pickle，但要小心安全問題。

## 簡單實踐：用 JSON 打包消息

以下是一個簡單的例子，展示如何用 JSON 格式發送消息：

```python
from kombu import Connection, Producer, Exchange

# 建立連接
conn = Connection('amqp://localhost')
exchange = Exchange('my_exchange', type='direct')

# 創建生產者，用 JSON 打包
producer = Producer(conn, exchange, serializer='json')

# 發送消息
producer.publish({'name': 'Alice', 'age': 25}, routing_key='greeting')
print("消息已發送！")
```

接收端也需要用同樣的方式解包：

```python
from kombu import Connection, Consumer, Exchange, Queue

conn = Connection('amqp://localhost')
exchange = Exchange('my_exchange', type='direct')
queue = Queue('my_queue', exchange, routing_key='greeting')

def process_message(body, message):
    print(f"收到消息：{body}")
    message.ack()  # 確認收到

# 創建消費者，指定接受 JSON 格式
with Consumer(conn, queues=queue, callbacks=[process_message], accept=['json']):
    conn.drain_events()  # 開始接收
```

## 注意事項

- **一致性**：發送和接收要用同樣的打包方式，不然對方看不懂。
- **安全性**：不要隨便用 Pickle 接收陌生消息，可能會有危險代碼。
- **大小與速度**：如果消息很多或很大，考慮用 MessagePack 節省空間和時間。

通過這個章節，您應該已經明白了消息打包的基本概念。接下來我們會探索更多 Kombu 的功能！
