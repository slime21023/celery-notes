# 自定義生產者：如何發送消息？

## 什麼是生產者？

在 Kombu 中，「生產者」（Producer）是負責發送消息的一方。就像您把信投入郵筒，生產者把消息送到「交換機」，然後系統會幫忙分發到正確的地方。

## 為什麼要自定義生產者？

雖然 Celery 提供了簡單的方式（比如 `@app.task`）來發送任務，但自定義生產者讓您能更精細地控制消息，比如決定消息去哪裡、如何打包、是否需要持久保存等。

## 簡單實踐：創建一個生產者

讓我們從最簡單的例子開始：發送一條消息。

```python
from kombu import Connection, Producer, Exchange

# 建立連接
conn = Connection('amqp://localhost')
exchange = Exchange('my_exchange', type='direct')

# 創建生產者
producer = Producer(conn, exchange, routing_key='greeting')

# 發送消息
producer.publish({'name': 'Alice', 'message': 'Hello!'}, serializer='json')
print("消息已發送！")
```

在這個例子中：

1. 我們連接到消息代理。
2. 定義了一個交換機，告訴系統消息要去哪個「分發中心」。
3. 創建生產者，並指定消息的「地址」（routing_key）。
4. 發送消息，系統會幫我們分發。

## 更進一步：確保消息不丟失

有時候，消息代理可能會重啟，我們希望消息不要丟失。這時可以設置「持久化」：

```python
# 發送持久化消息
producer.publish(
    {'name': 'Alice', 'message': 'Important!'},
    serializer='json',
    routing_key='greeting',
    delivery_mode=2  # 2 表示持久化
)
```

## 小技巧

- **重試機制**：如果發送失敗，可以設置 `retry=True` 讓 Kombu 自動重試。
- **優先級**：可以設置 `priority` 讓重要消息優先處理（如果消息代理支持）。
- **多個交換機**：可以根據需求把消息發送到不同的交換機。

## 注意事項

- 確保交換機和隊列已經設置好，不然消息可能送不到。
- 消息太大可能影響速度，建議適當壓縮或拆分。
- 記錄發送失敗的情況，方便排查問題。

通過這個章節，您已經學會了如何發送消息。結合前面的內容，您可以開始構建自己的消息系統了！
