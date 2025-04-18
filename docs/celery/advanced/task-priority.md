# 任務優先級

任務優先級是 Celery 的一種進階功能，允許您為任務設置優先級，確保重要任務優先處理。本文將介紹任務優先級的配置與應用場景。

## 什麼是任務優先級？

任務優先級是指通過設定數值或規則，控制任務在隊列中的處理順序。高優先級任務會優先被工作者取出執行，從而減少等待時間。

- **作用**：確保關鍵任務優先處理，提升系統響應性。
- **應用場景**：業務中存在緊急任務或重要客戶需求時。

## 任務優先級配置

Celery 支援通過消息代理（如 RabbitMQ）實現任務優先級。以下以 RabbitMQ 為例，介紹配置方法。

### 步驟 1：啟用優先級隊列

在 RabbitMQ 中，需為隊列啟用優先級支援（最大優先級值通常為 0-9）：
```python
from celery import Celery

app = Celery('tasks', broker='amqp://guest:guest@localhost:5672//')

# 定義隊列，啟用優先級
app.conf.update(
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='UTC',
    task_default_queue='default',
    task_queues={
        'default': {
            'exchange': 'default',
            'routing_key': 'default',
            'queue_arguments': {'x-max-priority': 10},  # 支援 0-9 優先級
        },
    },
)
```

- **x-max-priority**：設置隊列支援的最大優先級值，範圍通常為 0-9，值越大優先級越高。

### 步驟 2：設置任務優先級

在任務調用時，通過 `apply_async` 指定優先級：
```python
from app import critical_task, normal_task  # 假設這些是已定義的任務

# 高優先級任務
result_high = critical_task.apply_async(args=[data], priority=9)

# 普通優先級任務
result_normal = normal_task.apply_async(args=[data], priority=0)
```

- **priority**：優先級值，值越大優先級越高，範圍應與隊列配置的 `x-max-priority` 一致。

### 步驟 3：工作者配置

確保工作者正確處理優先級隊列，無需額外配置，RabbitMQ 會自動按優先級分發任務：
```bash
celery -A app worker --loglevel=info --queues=default
```

## 應用場景

- **緊急任務處理**：為關鍵業務任務設置高優先級，確保快速響應。
  ```python
  emergency_task.apply_async(args=[data], priority=9)
  ```
- **客戶分級**：為 VIP 客戶的任務設置高優先級，提升用戶體驗。
- **資源分配**：在資源有限時，確保重要任務優先獲得處理。

## 注意事項

- **消息代理支援**：任務優先級依賴消息代理的支援，RabbitMQ 支援優先級隊列，而 Redis 等其他代理可能不支援或需要額外插件。
- **優先級範圍**：確保設置的優先級值不超過隊列配置的 `x-max-priority`，否則可能無效。
- **效能影響**：過多高優先級任務可能導致低優先級任務長期排隊，需合理設計優先級策略。

## 小結

任務優先級是 Celery 確保關鍵任務快速處理的重要功能，通過合理設置優先級值，您可以優化系統響應性與資源分配。下一章節，我們將介紹「並發控制」，探索如何限制工作者並發任務數量以避免資源過載。

> **提示**：任務優先級應謹慎使用，避免過多高優先級任務導致系統不平衡。
