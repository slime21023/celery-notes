# 結果後端（Result Backend）

Celery 支援將任務執行結果存儲在結果後端（Result Backend），以便應用程式查詢任務狀態與結果。本文將介紹結果後端的配置與使用方法。

## 什麼是結果後端？

結果後端是 Celery 用於存儲任務執行結果的組件，允許應用程式在任務完成後獲取返回值或檢查執行狀態。常見的結果後端包括 Redis、MongoDB、SQLAlchemy 等。

- **作用**：存儲任務結果與狀態（如成功、失敗）。
- **應用場景**：需要異步任務結果的場景，如用戶提交任務後查詢處理進度。

## 配置結果後端

以下以 Redis 作為結果後端為例，介紹配置步驟。假設您已安裝 Redis 伺服器。

### 步驟 1：安裝 Redis 客戶端
```bash
pip install redis
```

### 步驟 2：配置 Celery 應用
在 Celery 應用初始化時指定結果後端：
```python
from celery import Celery

# 初始化 Celery，指定消息代理與結果後端
app = Celery('tasks',
             broker='amqp://guest:guest@localhost:5672//',
             backend='redis://localhost:6379/0')

# 配置 Celery
app.conf.update(
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='UTC',
    task_track_started=True,  # 追蹤任務開始狀態
    task_store_errors_even_if_ignored=True,  # 即使任務被忽略也存儲錯誤
)
```

## 查詢任務結果

任務發送後，可使用 `AsyncResult` 物件查詢結果與狀態：

### 示例：發送任務與查詢結果
```python
from celery.result import AsyncResult
from app import add  # 假設 add 是已定義的任務

# 發送任務
result = add.delay(5, 3)

# 查詢任務狀態
print("Task ID:", result.id)
print("Task Status:", result.status)  # PENDING, STARTED, SUCCESS, FAILURE 等

# 等待結果（可選）
if result.ready():  # 檢查任務是否完成
    if result.successful():  # 檢查任務是否成功
        print("Result:", result.get())  # 獲取結果
    else:
        print("Task failed:", result.get(propagate=False))  # 獲取錯誤資訊
else:
    print("Task not finished yet")
```

## 結果後端選項

- **Redis**：輕量高效，適合大多數場景。
  - 配置：`backend='redis://localhost:6379/0'`
- **MongoDB**：適合需要持久化結果的場景。
  - 配置：`backend='mongodb://localhost:27017/'`，需安裝 `pymongo`。
- **SQLAlchemy**：適合與現有關係型資料庫整合。
  - 配置：`backend='sqlalchemy+sqlite:///celery.db'`，需安裝對應驅動。

## 注意事項

- **結果過期**：設置結果過期時間，避免後端存儲過多數據。
  ```python
  app.conf.update(result_expires=3600)  # 結果保留1小時
  ```
- **性能影響**：啟用結果後端會增加寫入開銷，若無需結果，可禁用。
- **安全性**：確保後端存儲（如 Redis）設置適當的訪問控制，避免數據洩露。

## 小結

結果後端是 Celery 實現任務結果存儲與查詢的關鍵組件，通過合理配置，您可以滿足不同場景的需求。下一章節，我們將介紹「任務狀態追蹤」，深入探索如何監控任務執行過程。

> **提示**：選擇結果後端時，需平衡性能與持久化需求，Redis 是大多數場景的首選。
