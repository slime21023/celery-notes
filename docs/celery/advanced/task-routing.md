# 任務路由

任務路由是 Celery 的一種進階功能，允許您根據任務類型、屬性或其他條件將任務分發到特定的隊列或工作者。本文將介紹任務路由的配置與應用場景。

## 什麼是任務路由？

任務路由是指通過設定規則，將不同類型的任務發送到不同的消息隊列，進而由特定的工作者處理。這有助於任務分類管理與資源分配優化。

- **作用**：將任務分發到指定隊列，實現任務分類與工作者專用化。
- **應用場景**：不同任務需要不同資源或優先級時，如 CPU 密集型任務與 I/O 密集型任務分開處理。

## 任務路由基本配置

Celery 支援通過配置文件或任務裝飾器指定任務路由規則。以下以 RabbitMQ 作為消息代理為例。

### 步驟 1：定義隊列與路由規則

在 Celery 應用配置中定義隊列與路由規則：
```python
from celery import Celery

app = Celery('tasks', broker='amqp://guest:guest@localhost:5672//')

# 定義隊列與路由規則
app.conf.update(
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='UTC',
    task_default_queue='default',  # 預設隊列
    task_queues={
        'default': {'exchange': 'default', 'routing_key': 'default'},
        'cpu_intensive': {'exchange': 'tasks', 'routing_key': 'cpu_intensive'},
        'io_intensive': {'exchange': 'tasks', 'routing_key': 'io_intensive'},
    },
    task_routes={
        'app.cpu_task': {'queue': 'cpu_intensive'},
        'app.io_task': {'queue': 'io_intensive'},
    },
)
```

- **task_queues**：定義多個隊列及其交換機與路由鍵。
- **task_routes**：指定任務到隊列的映射規則。

### 步驟 2：定義任務與路由

在任務定義時，可以通過裝飾器指定隊列，也可依賴配置中的路由規則：
```python
@app.task(queue='cpu_intensive')
def cpu_task(data):
    # CPU 密集型任務
    return heavy_computation(data)

@app.task
def io_task(data):
    # I/O 密集型任務，依賴配置路由到 io_intensive 隊列
    return fetch_data_from_network(data)
```

### 步驟 3：啟動專用工作者

為不同隊列啟動專用工作者，確保任務由合適的工作者處理：
```bash
# 處理 cpu_intensive 隊列的工作者
celery -A app worker --loglevel=info --queues=cpu_intensive

# 處理 io_intensive 隊列的工作者
celery -A app worker --loglevel=info --queues=io_intensive

# 處理預設隊列的工作者
celery -A app worker --loglevel=info --queues=default
```

- **--queues**：指定工作者監聽的隊列名稱。

## 動態路由

除了靜態配置，您也可以在任務調用時動態指定隊列：
```python
# 動態指定隊列
result = cpu_task.apply_async(args=[data], queue='high_priority_cpu')
```

## 應用場景

- **任務分類**：將 CPU 密集型任務與 I/O 密集型任務分開，避免資源爭用。
- **優先級處理**：為高優先級任務分配專用隊列與工作者。
- **地理分佈**：將任務路由到特定區域的工作者，減少延遲。

## 注意事項

- **隊列管理**：確保隊列與工作者匹配，否則任務可能堆積。
- **消息代理支援**：不同消息代理（如 RabbitMQ、Redis）對路由功能的支援有所不同，RabbitMQ 提供更豐富的交換機類型。
- **配置一致性**：所有 Celery 實例（生產者與工作者）應使用相同的路由配置，避免任務丟失。

## 小結

任務路由是 Celery 實現任務分類與資源優化的重要功能，通過合理配置隊列與路由規則，您可以提升系統的可擴展性與效率。下一章節，我們將介紹「任務優先級」，探索如何確保重要任務優先處理。

> **提示**：任務路由應根據實際業務需求與系統架構設計，避免過多隊列導致管理複雜性增加。
