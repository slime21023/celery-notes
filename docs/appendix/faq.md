# 常見問題解答 (FAQ)

本附錄整理了關於 Celery、Kombu 和 RabbitMQ 的常見問題及其解答，旨在幫助您快速解決在使用這些工具時遇到的問題。如果您有其他問題，建議查閱官方文檔或在社區中尋求幫助。

## Celery 相關問題

### 為什麼我的 Celery 任務沒有執行？
**可能原因：**

- Worker 沒有啟動或未正確連接到消息代理（Broker）。
- 任務隊列名稱與 Worker 監聽的隊列不匹配。
- 消息代理（如 RabbitMQ）未運行或連接配置錯誤。

**解決方法：**

- 檢查 Worker 是否運行：`celery -A your_module worker --loglevel=info`。
- 確認隊列名稱是否一致，使用 `-Q` 參數指定隊列：`celery -A your_module worker -Q your_queue`。
- 檢查 Broker 連接字符串（`CELERY_BROKER_URL`），確保 RabbitMQ 或其他後端服務正常運行。

### 如何查看 Celery 任務的執行狀態？
**解決方法：**

- 使用 Flower 監控工具：安裝 `pip install flower`，然後啟動 `celery -A your_module flower`。
- 配置結果後端（如 Redis），並使用 `AsyncResult` 獲取任務狀態：
  ```python
  from celery.result import AsyncResult
  result = AsyncResult(task_id)
  print(result.state)  # PENDING, STARTED, SUCCESS, FAILURE 等
  ```

### Celery 任務執行時間過長，如何設定超時？
**解決方法：**

- 在任務調用時設定 `time_limit` 和 `soft_time_limit`：
  ```python
  result = your_task.apply_async(time_limit=300, soft_time_limit=280)
  ```
- 全局配置超時（在 `celeryconfig.py` 中）：
  ```python
  task_time_limit = 300  # 硬性超時，單位：秒
  task_soft_time_limit = 280  # 軟性超時，單位：秒
  ```

## RabbitMQ 相關問題

### RabbitMQ 連接失敗，顯示 "Connection Refused" 錯誤？
**可能原因：**

- RabbitMQ 服務未啟動。
- 防火牆或網絡策略阻止了連接。
- 連接字符串中的主機或端口錯誤。

**解決方法：**

- 檢查 RabbitMQ 是否運行：`rabbitmqctl status`。
- 確認端口（默認 5672）是否開放：`telnet host 5672`。
- 檢查連接字符串是否正確，例如 `amqp://user:password@host:5672/vhost`。

### RabbitMQ 隊列中有消息積壓，如何處理？
**可能原因：**

- Worker 數量不足，無法及時消費消息。
- 任務處理時間過長，導致消息堆積。

**解決方法：**

- 增加 Worker 數量：`celery -A your_module worker --concurrency=10`。
- 檢查任務執行時間，優化代碼或設定超時。
- 使用 RabbitMQ 管理界面（端口 15672）查看隊列狀態，必要時手動清除積壓消息：`rabbitmqctl purge_queue queue_name`。

### 如何確保 RabbitMQ 消息不丟失？
**解決方法：**

- 配置持久化隊列和消息：
  ```python
  from kombu import Queue, Exchange
  queue = Queue('tasks', Exchange('tasks'), durable=True)
  producer.publish({'task': 'data'}, delivery_mode=2)  # 2 表示持久化
  ```
- 啟用發布者確認（Publisher Confirms），確保消息成功發送到隊列：
  ```python
  producer.publish({'task': 'data'}, confirm=True)
  ```

## Kombu 相關問題

### Kombu 連接 RabbitMQ 時出現認證錯誤？
**可能原因：**

- 用戶名或密碼錯誤。
- 虛擬主機（vhost）配置不正確。

**解決方法：**

- 檢查連接字符串中的用戶名、密碼和 vhost 是否正確。
- 在 RabbitMQ 管理界面中確認用戶權限：`rabbitmqctl list_permissions -p /vhost`。
- 如果需要，創建新用戶並授權：
  ```bash
  rabbitmqctl add_user user password
  rabbitmqctl set_permissions -p /vhost user ".*" ".*" ".*"
  ```

### 如何使用 Kombu 實現消息的發布與訂閱？
**解決方法：**

- 使用 `topic` 交換機實現發布/訂閱模式：
  ```python
  from kombu import Exchange, Queue, Producer, Consumer, Connection

  event_exchange = Exchange('events', type='topic')
  queue = Queue('event_queue', event_exchange, routing_key='event.*')

  # 發布消息
  with Connection('amqp://user:password@host:5672//') as conn:
      producer = Producer(conn)
      producer.publish({'data': 'event'}, exchange=event_exchange, routing_key='event.created')

  # 訂閱消息
  def callback(body, message):
      print(f"Received: {body}")
      message.ack()

  with Connection('amqp://user:password@host:5672//') as conn:
      consumer = Consumer(conn, queues=queue, callbacks=[callback])
      consumer.consume()
  ```

## 部署與效能問題

### 如何在大規模環境中避免 Celery Worker 資源爭用？
**解決方法：**

- 為不同任務類型配置專用 Worker 和隊列：
  ```bash
  celery -A your_module worker -Q high_priority --concurrency=5
  celery -A your_module worker -Q default --concurrency=10
  ```
- 使用 Kubernetes 或 Docker 限制每個 Worker 的資源使用，防止單一 Worker 過載系統。
- 監控 Worker 的 CPU 和內存使用，動態調整併發數。

### RabbitMQ 集群中節點同步失敗，如何排查？
**解決方法：**

- 檢查集群狀態：`rabbitmqctl cluster_status`。
- 確認節點間網絡連接是否正常，端口（4369、25672）是否開放。
- 查看日誌文件（通常位於 `/var/log/rabbitmq/`），查找具體錯誤信息。
- 如果問題持續，考慮重新加入節點：`rabbitmqctl join_cluster rabbit@other_node`。

## 其他常見問題

### 如何在 Celery 中實現定時任務？
**解決方法：**

- 使用 `celery beat` 插件：
  ```bash
  pip install django-celery-beat
  celery -A your_module beat
  ```
- 在 Django Admin 或程式碼中配置定時任務：
  ```python
  from celery.schedules import crontab
  from django_celery_beat.models import PeriodicTask, CrontabSchedule

  schedule, _ = CrontabSchedule.objects.get_or_create(minute='0', hour='*')
  PeriodicTask.objects.create(crontab=schedule, name='hourly_task', task='your_task')
  ```

### 如何清理 Celery 的結果後端數據？
**解決方法：**

- 如果使用 Redis 作為結果後端，可以手動清理：
  ```bash
  redis-cli -h host -p 6379 FLUSHDB
  ```
- 設定結果過期時間，避免數據積累：
  ```python
  app.conf.result_expires = 86400  # 結果1天後過期
  ```

如果您在以上問題中找不到解決方案，建議查閱官方文檔或在 Stack Overflow、RabbitMQ 社區等平台尋求幫助。
