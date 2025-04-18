# Group：並行執行

Group 是 Celery Canvas 中的一種工具，用於並行執行一組任務，適合需要同時處理多個獨立任務的場景。本文將介紹 Group 的用法與應用場景。

## 什麼是 Group？

Group 允許您將多個任務組成一個組，這些任務將並行執行，互不依賴。Group 會等待所有任務完成後返回結果集合。

- **作用**：並行執行多個任務，提升處理效率。
- **應用場景**：批量處理獨立數據或執行無依賴關係的任務。

## Group 基本用法

使用 `group` 函數創建並行任務組：

### 示例：基本 Group
```python
from celery import group
from app import process_user_data  # 假設這是已定義的任務

# 定義 Group 工作流
user_ids = [1, 2, 3, 4, 5]
workflow = group(process_user_data.s(user_id) for user_id in user_ids)

# 執行工作流
result = workflow.delay()
print("Group started, ID:", result.id)

# 等待結果
print("Results:", result.get())
```

- **並行執行**：每個 `process_user_data` 任務獨立執行，互不影響。
- **結果集合**：`result.get()` 返回所有任務結果的列表。

## 應用場景

- **批量用戶處理**：同時處理多個用戶的數據，如發送批量郵件。
  ```python
  workflow = group(send_email.s(email) for email in email_list)
  workflow.delay()
  ```
- **並行計算**：對多個數據塊進行獨立計算，提升處理速度。
- **多服務調用**：同時向多個外部 API 發送請求，收集結果。

## 注意事項

- **資源限制**：並行任務可能導致資源爭用，需根據系統能力調整 Group 大小。
- **錯誤處理**：若某個任務失敗，Group 仍會等待其他任務完成，但整體結果可能包含異常。建議檢查結果：
  ```python
  results = result.get(propagate=False)
  for res in results:
      if isinstance(res, Exception):
          print("Task failed:", res)
  ```
- **工作者數量**：確保有足夠的工作者處理並行任務，否則任務仍會排隊。

## 小結

Group 是 Celery Canvas 中用於並行執行任務的工具，特別適合批量處理獨立任務的場景。通過合理使用 Group，您可以顯著提升任務處理效率。下一章節，我們將介紹「Chord」，探索如何在並行任務後執行回調。

> **提示**：使用 Group 時，需關注系統資源使用情況，避免過多並行任務導致效能下降。
