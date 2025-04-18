# Chord：並行後回調

Chord 是 Celery Canvas 中的一種工具，用於在並行執行一組任務後，執行一個回調任務來處理所有結果。本文將介紹 Chord 的用法與應用場景。

## 什麼是 Chord？

Chord 是一種結合 Group 與回調任務的工作流工具。它首先並行執行一組任務（類似 Group），然後在所有任務完成後，將結果傳遞給指定的回調任務進行處理。

- **作用**：並行執行任務後進行結果匯總或後續處理。
- **應用場景**：批量處理後需要匯總結果的場景，如分散式計算後統計。

## Chord 基本用法

使用 `chord` 函數創建工作流，指定並行任務組與回調任務：

### 示例：基本 Chord
```python
from celery import chord, group
from app import process_chunk, aggregate_results  # 假設這些是已定義的任務

# 定義並行任務組
data_chunks = [1, 2, 3, 4, 5]
parallel_tasks = group(process_chunk.s(chunk) for chunk in data_chunks)

# 定義 Chord 工作流：並行任務完成後執行回調
workflow = chord(parallel_tasks)(aggregate_results.s())

# 執行工作流
result = workflow.delay()
print("Chord started, ID:", result.id)

# 等待結果（回調任務的結果）
print("Final result:", result.get())
```

- **並行任務**：`process_chunk` 任務並行處理每個數據塊。
- **回調任務**：`aggregate_results` 接收所有並行任務的結果（作為列表）進行處理。
- **結果**：最終結果是回調任務的返回值。

## 應用場景

- **分散式計算**：將大數據集分塊並行處理，然後匯總結果。
  ```python
  workflow = chord(group(compute_chunk.s(chunk) for chunk in chunks))(sum_results.s())
  workflow.delay()
  ```
- **批量更新後通知**：並行更新多個記錄，完成後發送總結通知。
- **多任務結果處理**：並行執行多個 API 調用，然後分析所有響應。

## 注意事項

- **回調依賴**：回調任務僅在所有並行任務成功完成後執行，若有任務失敗，需妥善處理。
- **結果後端**：Chord 依賴結果後端存儲中間結果，確保配置正確。
- **效能考量**：並行任務數量與回調任務的處理時間會影響整體執行時間。

## 小結

Chord 是 Celery Canvas 中用於並行任務後執行回調的工具，特別適合需要匯總結果的場景。通過 Chord，您可以實現高效的分散式處理與結果整合。下一章節，我們將介紹「Map & Starmap」，探索如何對數據集應用任務。

> **提示**：使用 Chord 時，確保回調任務能處理並行任務的所有結果，並考慮異常情況下的錯誤處理。
