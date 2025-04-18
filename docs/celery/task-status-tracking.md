# 任務狀態追蹤

Celery 提供任務狀態追蹤功能，允許應用程式監控任務的執行進度與結果。本文將介紹如何使用 Celery 追蹤任務狀態，幫助您管理異步任務。

## 任務狀態概述

Celery 中的任務在執行過程中會經歷多種狀態，這些狀態存儲在結果後端中，可供查詢：

- **PENDING**：任務已發送但尚未開始執行。
- **STARTED**：任務已開始執行（需設置 `task_track_started=True`）。
- **SUCCESS**：任務成功完成。
- **FAILURE**：任務執行失敗。
- **RETRY**：任務正在重試（通常因錯誤觸發）。
- **REVOKED**：任務被撤銷或終止。

## 配置狀態追蹤

要啟用任務狀態追蹤，需確保結果後端已正確配置，並啟用相關選項：

```python
from celery import Celery

app = Celery('tasks',
             broker='amqp://guest:guest@localhost:5672//',
             backend='redis://localhost:6379/0')

app.conf.update(
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='UTC',
    task_track_started=True,  # 追蹤任務開始狀態
    task_store_errors_even_if_ignored=True,  # 存儲錯誤資訊
)
```

## 查詢任務狀態

使用 `AsyncResult` 物件查詢任務狀態與結果：

### 示例：追蹤任務狀態
```python
from celery.result import AsyncResult
from app import add  # 假設 add 是已定義的任務

# 發送任務
result = add.delay(2, 3)

# 查詢狀態
print("Task ID:", result.id)
print("Task Status:", result.status)

# 檢查是否完成
if result.ready():
    print("Task is done!")
    if result.successful():
        print("Result:", result.get())
    else:
        print("Task failed:", result.get(propagate=False))
else:
    print("Task is still running...")
```

## 自定義狀態更新

在長時間運行的任務中，可手動更新狀態以提供進度反饋：

### 示例：更新任務進度
```python
@app.task(bind=True)
def long_running_task(self):
    for i in range(10):
        # 更新任務狀態與進度
        self.update_state(state='PROGRESS', meta={'current': i, 'total': 10})
        # 模擬耗時操作
        import time
        time.sleep(1)
    return "Task completed!"

# 查詢進度
result = long_running_task.delay()
while not result.ready():
    meta = result.info or {}
    if result.status == 'PROGRESS':
        print(f"Progress: {meta.get('current', 0)}/{meta.get('total', 0)}")
    import time
    time.sleep(0.5)
```

## 任務撤銷

若需中止任務，可使用 `revoke` 方法：
```python
result = long_running_task.delay()
# 撤銷任務
app.control.revoke(result.id, terminate=True)
print("Task revoked")
```

## 小結

任務狀態追蹤是 Celery 管理異步任務的重要功能，通過查詢狀態與自定義進度更新，您可以有效監控任務執行過程。下一章節，我們將介紹「錯誤處理與重試機制」，探索如何應對任務失敗的情況。

> **提示**：任務狀態追蹤依賴結果後端，確保後端配置正確且效能足夠。
