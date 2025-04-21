# 複雜業務場景實現

在實際項目中，分散式任務處理往往需要應對複雜的業務需求，例如多步驟工作流、任務依賴管理、動態任務調度等。本章節將介紹如何使用 Celery、Kombu 和 RabbitMQ 實現這些複雜業務場景，並提供具體的代碼示例和最佳實踐。

## 多步驟工作流實現

許多業務場景需要多個任務按順序或並行執行，形成一個完整的工作流。Celery 提供了 `chain`、`group` 和 `chord` 等工具來實現這類需求。

### 使用 `chain` 實現順序任務

當任務需要按特定順序執行時，可以使用 `chain` 將多個任務串聯起來：

```python
from celery import chain

# 定義一個工作流：先處理數據，然後發送通知
workflow = chain(process_data.s(data), send_notification.s())
result = workflow.apply_async()
```

#### 最佳實踐
- 確保每個任務的輸出格式與下一個任務的輸入格式兼容。
- 使用 `link_error` 設定錯誤回調，處理工作流中斷的情況：

```python
workflow = chain(process_data.s(data), send_notification.s()).on_error(handle_error.s())
```

### 使用 `group` 實現並行任務

當多個任務可以並行執行時，使用 `group` 來提高效率：

```python
from celery import group

# 並行處理多個數據集
tasks = group(process_data.s(data) for data in datasets)
result = tasks.apply_async()
```

#### 最佳實踐
- 確保並行任務之間無依賴關係，避免競爭條件。
- 監控並行任務的資源使用，避免過載。

### 使用 `chord` 實現聚合任務

當一組任務完成後需要執行一個聚合任務時，可以使用 `chord`：

```python
from celery import chord

# 一組任務完成後執行匯總
workflow = chord(
    [process_data.s(data) for data in datasets],
    aggregate_results.s()
)
result = workflow.apply_async()
```

#### 最佳實踐
- 確保聚合任務能處理所有子任務的結果，包括部分失敗的情況。
- 設定合理的超時時間，避免因單個任務延遲導致整個工作流卡住。

## 任務依賴管理

在複雜業務中，任務之間往往存在依賴關係，例如任務 B 必須在任務 A 完成後才能執行。

### 動態任務依賴

使用 Celery 的結果後端追蹤任務狀態，實現動態依賴：

```python
from celery import Celery

app = Celery('tasks', broker='amqp://localhost', backend='rpc://')

@app.task
def task_a():
    # 執行任務 A
    return "Task A completed"

@app.task
def task_b(dependency_result):
    # 依賴任務 A 的結果
    print(f"Task B received: {dependency_result}")
    return "Task B completed"

# 調用時連接任務
result_a = task_a.delay()
result_a.get()  # 等待任務 A 完成
task_b.delay(result_a.get())
```

#### 最佳實踐
- 使用結果後端儲存任務狀態，避免直接阻塞等待結果。
- 為關鍵依賴任務設定重試機制，確保可靠性。

### 使用 Canvas 設計複雜依賴

Celery 的 Canvas 模組允許設計更複雜的任務依賴圖：

```python
from celery.canvas import chain, group

# 設計一個複雜工作流
workflow = chain(
    task_a.s(),
    group(task_b1.s(), task_b2.s()),
    task_c.s()
)
result = workflow.apply_async()
```

#### 最佳實踐
- 將複雜工作流拆分成小模塊，方便測試和調試。
- 使用日誌記錄每個任務的執行狀態，方便排查問題。

## 動態任務調度

業務需求可能要求根據條件動態調度任務，例如根據用戶行為觸發不同任務。

### 條件觸發任務

根據業務邏輯動態選擇任務：

```python
@app.task
def route_task(user_data):
    if user_data['type'] == 'premium':
        premium_task.delay(user_data)
    else:
        standard_task.delay(user_data)
    return "Task routed"
```

#### 最佳實踐
- 將路由邏輯與具體任務分離，保持代碼清晰。
- 記錄路由決策，方便分析業務邏輯是否正確。

### 基於時間的調度

使用 Celery Beat 實現基於時間的任務調度：

```bash
pip install celery[redis] django-celery-beat
```

設定定時任務（假設使用 Django）：

```python
from celery.schedules import crontab
from django_celery_beat.models import PeriodicTask, CrontabSchedule

# 創建一個每小時執行的任務
schedule, _ = CrontabSchedule.objects.get_or_create(
    minute='0',
    hour='*',
    day_of_week='*',
    day_of_month='*',
    month_of_year='*',
)
PeriodicTask.objects.create(
    crontab=schedule,
    name='hourly_task',
    task='your_app.tasks.hourly_task',
)
```

#### 最佳實踐
- 避免定時任務過於頻繁，防止資源爭用。
- 為定時任務設定監控，確保按時執行。

## 分布式鎖與資源競爭

在分散式環境中，多個 Worker 可能同時處理相同資源，需使用鎖機制避免競爭。

### 使用 Redis 實現分布式鎖

使用 Redis 作為鎖後端，確保任務間資源安全：

```python
from redis import Redis
from time import sleep

redis = Redis(host='localhost', port=6379, db=0)

@app.task
def critical_task(resource_id):
    lock_key = f"lock:{resource_id}"
    # 嘗試獲取鎖
    if redis.setnx(lock_key, 1):
        try:
            redis.expire(lock_key, 30)  # 設定鎖過期時間
            # 執行關鍵操作
            sleep(5)  # 模擬長時間操作
            return "Task completed"
        finally:
            redis.delete(lock_key)  # 釋放鎖
    else:
        return "Resource locked, task skipped"
```

#### 最佳實踐
- 設定鎖的過期時間，避免死鎖。
- 使用 `finally` 確保鎖釋放，即使任務失敗。

## 總結與注意事項

- **工作流設計**：使用 `chain`、`group` 和 `chord` 實現多步驟業務邏輯，確保清晰和可維護。
- **依賴管理**：動態處理任務依賴，結合結果後端和 Canvas 模組應對複雜場景。
- **動態調度**：根據業務條件和時間動態調度任務，提升靈活性。
- **資源安全**：使用分布式鎖避免競爭，確保數據一致性。

通過以上方法，您可以應對複雜業務場景，將 Celery 和 RabbitMQ 的能力應用到實際項目中。
