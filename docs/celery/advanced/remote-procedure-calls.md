# 遠程調用（RPC）

遠程調用（Remote Procedure Call, RPC）是 Celery 的一種進階功能，允許您以類似同步調用的方式處理異步任務結果，簡化異步任務與結果獲取的交互。本文將介紹 RPC 的配置與應用場景。

## 什麼是遠程調用（RPC）？

RPC 是一種設計模式，允許應用程式以類似本地函數調用的方式執行遠程任務，並直接獲取結果。Celery 通過結果後端與特定配置實現 RPC 風格的調用，隱藏異步任務的複雜性。

- **作用**：以同步風格調用異步任務，簡化結果處理。
- **應用場景**：需要快速獲取任務結果，且任務執行時間較短的場景。

## RPC 配置

Celery 的 RPC 功能依賴結果後端（如 Redis），並需設置特定參數以啟用同步風格調用。

### 步驟 1：配置結果後端

確保 Celery 應用配置了結果後端：
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
    task_track_started=True,
    task_ignore_result=False,
)
```

- **backend**：指定結果後端（如 Redis），用於存儲任務結果。
- **task_ignore_result=False**：確保任務結果被存儲。

### 步驟 2：使用 RPC 後端

Celery 提供了一種特殊的結果後端 `rpc://`，專為 RPC 風格調用設計。它會自動處理結果的同步返回：
```python
app = Celery('tasks',
             broker='amqp://guest:guest@localhost:5672//',
             backend='rpc://')

app.conf.update(
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='UTC',
)
```

- **backend='rpc://'**：使用 RPC 後端，任務結果會通過臨時隊列直接返回給調用者。

### 步驟 3：調用任務並獲取結果

使用 `apply_async` 或 `delay` 調用任務，並通過 `get()` 方法同步獲取結果：
```python
from app import add  # 假設這是一個簡單的加法任務

# 調用任務
result = add.delay(2, 3)

# 同步獲取結果（類似 RPC）
print("Result:", result.get(timeout=10))  # 等待最多 10 秒
```

- **result.get(timeout=10)**：設置超時時間，防止無限等待。

## 應用場景

- **簡單計算服務**：快速執行簡單計算並返回結果，如數學運算或數據處理。
  ```python
  result = compute_sum.delay(10, 20)
  print("Sum:", result.get(timeout=5))
  ```
- **即時查詢**：需要立即返回結果的查詢任務，如查詢用戶資料。
- **同步風格 API**：在需要同步 API 行為的場景中，隱藏異步任務的複雜性。

## 注意事項

- **執行時間**：RPC 適合執行時間較短的任務，長時間任務可能導致調用者超時。
- **結果後端**：RPC 依賴結果後端或臨時隊列，確保配置正確。
- **超時設置**：使用 `get(timeout=...)` 時設置合理超時，避免調用者長期阻塞。
- **效能影響**：頻繁使用同步調用可能降低系統的異步優勢，建議僅在必要時使用。

## 小結

遠程調用（RPC）是 Celery 提供的一種簡化異步任務結果處理的功能，通過以同步風格調用任務，您可以降低程式設計複雜性，特別適合需要快速響應的場景。本章節結束了「Celery 進階功能」的學習，您已掌握了工作流編排、任務路由、優先級、並發控制與 RPC 等進階技術。

> **提示**：RPC 雖然方便，但應避免在長時間任務中使用，以免影響系統效能與響應性。
