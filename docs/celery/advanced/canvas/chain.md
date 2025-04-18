# Chain：順序執行

Chain 是 Celery Canvas 中的一種工具，用於按順序執行一系列任務，前一個任務的結果可作為後一個任務的輸入。本文將介紹 Chain 的用法與應用場景。

## 什麼是 Chain？

Chain 允許您將多個任務串聯成一個順序執行的工作流，確保任務按指定順序完成。每個任務的輸出可以傳遞給下一個任務，形成管道式處理。

- **作用**：按順序執行任務，支援結果傳遞。
- **應用場景**：業務流程中需要嚴格順序的場景，如數據處理管道。

## Chain 基本用法

使用 `chain` 函數或 `|` 操作符組合任務：

### 示例：基本 Chain
```python
from celery import chain
from app import validate_data, process_data, save_result  # 假設這些是已定義的任務

# 定義 Chain 工作流
workflow = chain(validate_data.s(), process_data.s(), save_result.s())

# 執行工作流
result = workflow.delay()
print("Workflow started, ID:", result.id)
```

- **`.s()`**：生成任務簽名，用於組合。
- **順序**：任務按定義順序執行，`validate_data` 完成後執行 `process_data`，最後執行 `save_result`。

### 示例：使用 `|` 操作符
```python
# 使用 | 操作符定義 Chain
workflow = validate_data.s() | process_data.s() | save_result.s()
result = workflow.delay()
```

## 結果傳遞

Chain 支援將前一任務的結果作為後一任務的參數：

### 示例：結果傳遞
```python
@app.task
def step1():
    return 42

@app.task
def step2(data):
    return data * 2

@app.task
def step3(data):
    return f"Final result: {data}"

# 定義 Chain，結果自動傳遞
workflow = chain(step1.s(), step2.s(), step3.s())
result = workflow.delay()

# 等待結果
print("Result:", result.get())
```

## 應用場景

- **訂單處理**：驗證訂單數據 → 更新庫存 → 發送通知。
  ```python
  workflow = chain(validate_order.s(), update_inventory.s(), send_notification.s())
  workflow.delay(order_data)
  ```
- **數據處理管道**：讀取數據 → 清洗數據 → 存儲結果。
- **多步計算**：逐步處理複雜計算任務，每步依賴前一步結果。

## 注意事項

- **錯誤處理**：若 Chain 中某個任務失敗，後續任務將不會執行。建議在任務內捕獲異常或使用重試機制。
- **結果後端**：確保配置結果後端以存儲中間結果，否則無法獲取完整工作流結果。
- **執行時間**：Chain 總執行時間是各任務執行時間之和，適合非緊急任務。

## 小結

Chain 是 Celery Canvas 中用於順序執行任務的簡單而強大的工具，特別適合需要嚴格順序的業務流程。下一章節，我們將介紹「Group」，探索如何並行執行多個任務。

> **提示**：使用 Chain 時，確保每個任務的輸入輸出格式一致，以便順利傳遞結果。
