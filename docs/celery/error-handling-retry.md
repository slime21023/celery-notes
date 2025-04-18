# 錯誤處理與重試機制

在分散式任務處理中，任務可能因各種原因失敗（如網絡問題、資源不可用）。Celery 提供錯誤處理與重試機制，幫助您應對失敗並提高系統可靠性。本文將介紹相關功能與配置方法。

## 錯誤處理基礎

當 Celery 任務執行過程中發生異常，任務狀態會標記為 `FAILURE`，錯誤資訊會存儲在結果後端中（若已配置）。您可以通過 `AsyncResult` 物件獲取錯誤詳情。

### 示例：捕獲任務錯誤
```python
from celery.result import AsyncResult
from app import some_task  # 假設 some_task 是已定義的任務

# 發送任務
result = some_task.delay()

# 檢查結果
if result.ready():
    if result.successful():
        print("Task completed successfully:", result.get())
    else:
        print("Task failed:", result.get(propagate=False))  # 不拋出異常
else:
    print("Task is still running...")
```

### 在任務中處理異常

建議在任務內部使用 `try-except` 捕獲異常，並記錄錯誤或執行清理操作：
```python
@app.task
def risky_task(data):
    try:
        # 模擬可能失敗的操作
        result = 1 / 0  # 故意引發異常
        return result
    except Exception as e:
        print(f"Error occurred: {e}")
        raise  # 重新拋出異常，任務狀態設為 FAILURE
```

## 任務重試機制

Celery 支援自動重試失敗任務，您可以在任務裝飾器中設置重試參數，或在異常時手動觸發重試。

### 自動重試配置

在任務定義時，設置 `retry` 參數以啟用自動重試：
```python
@app.task(
    retry_backoff=True,  # 使用指數退避策略
    retry_backoff_max=700,  # 最大重試間隔（秒）
    retry_jitter=True,  # 添加隨機抖動，避免重試衝突
    max_retries=3  # 最大重試次數
)
def fetch_data(url):
    import requests
    response = requests.get(url)
    if response.status_code != 200:
        raise Exception(f"Failed to fetch {url}, status: {response.status_code}")
    return response.json()
```

- **retry_backoff**：啟用指數退避，重試間隔逐漸增加。
- **retry_backoff_max**：設置最大重試間隔，避免無限延長。
- **retry_jitter**：添加隨機抖動，防止多個任務同時重試。
- **max_retries**：限制最大重試次數，避免無限重試。

### 手動觸發重試

在任務內部捕獲異常並使用 `self.retry()` 方法手動觸發重試：
```python
@app.task(bind=True)
def process_data(self, data):
    try:
        # 模擬可能失敗的操作
        if some_condition_fails(data):
            raise Exception("Processing failed")
    except Exception as e:
        # 記錄錯誤
        print(f"Error: {e}")
        # 手動重試，最多3次
        raise self.retry(exc=e, countdown=5, max_retries=3)
```

- **bind=True**：將任務實例作為第一個參數傳入，以便調用 `self.retry()`。
- **countdown**：設置重試前的等待時間（秒）。
- **exc**：傳遞原始異常，供後續分析。

## 錯誤處理策略

- **忽略錯誤**：若任務失敗不影響整體流程，可設置 `ignore_result=True` 忽略結果。
  ```python
  @app.task(ignore_result=True)
  def non_critical_task():
      # 即使失敗也不影響主流程
      pass
  ```
- **自定義錯誤處理**：為特定異常設置不同的重試策略。
  ```python
  @app.task(bind=True, retry_backoff=True, max_retries=3)
  def custom_retry_task(self):
      try:
          risky_operation()
      except NetworkError as e:
          raise self.retry(exc=e, countdown=10)  # 網絡錯誤重試
      except ValueError:
          return None  # 值錯誤不重試，直接返回
  ```
- **死信隊列**：對於無法重試的任務，可將失敗任務發送到死信隊列（需結合 RabbitMQ 配置）進行人工處理。

## 注意事項

- **重試性能影響**：過多重試可能導致隊列擁塞，應合理設置 `max_retries`。
- **錯誤日誌**：確保記錄詳細錯誤資訊，便於調試與監控。
- **結果後端**：錯誤資訊與重試狀態依賴結果後端，確保其配置正確。

## 小結

Celery 的錯誤處理與重試機制為任務處理提供了可靠性保障，通過自動與手動重試，您可以應對大多數失敗場景。合理設計重試策略與錯誤處理邏輯，能顯著提升系統穩定性。本章節結束了「Celery 基礎」的學習，下一階段可以進一步探索 Celery 的進階功能，如定時任務與工作流程管理。

> **提示**：重試機制應謹慎使用，避免因無限重試導致資源浪費，建議結合日誌與監控工具追蹤任務失敗原因。
