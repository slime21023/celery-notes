# Map & Starmap：數據集映射

Map 和 Starmap 是 Celery Canvas 中用於對數據集應用任務的工具，類似於 Python 的 `map` 函數，支援對多個數據項並行執行任務。本文將介紹 Map 和 Starmap 的用法與應用場景。

## 什麼是 Map 和 Starmap？

- **Map**：將單一任務應用於數據集中的每個元素，等同於對每個元素調用 `task.delay()`，並行執行。
- **Starmap**：類似 Map，但用於需要多個參數的任務，將數據集中的每個元素解包為參數傳入任務。

- **作用**：對數據集並行應用任務，提升處理效率。
- **應用場景**：批量處理數據集，如對多個文件或記錄執行相同操作。

## Map 基本用法

使用 `map` 方法對數據集應用任務：

### 示例：基本 Map
```python
from app import process_item  # 假設這是已定義的任務

# 定義數據集
items = [1, 2, 3, 4, 5]

# 使用 Map 對每個元素應用任務
result = process_item.map(items).delay()
print("Map started, ID:", result.id)

# 等待結果
print("Results:", result.get())
```

- **並行執行**：每個元素獨立應用 `process_item` 任務。
- **結果**：返回結果列表，與輸入順序一致。

## Starmap 基本用法

使用 `starmap` 方法對數據集應用多參數任務：

### 示例：基本 Starmap
```python
from app import process_pair  # 假設這是接受兩個參數的任務

# 定義數據集（每個元素是參數元組）
pairs = [(1, 10), (2, 20), (3, 30)]

# 使用 Starmap 對每個元組解包並應用任務
result = process_pair.starmap(pairs).delay()
print("Starmap started, ID:", result.id)

# 等待結果
print("Results:", result.get())
```

- **解包參數**：每個元組 `(x, y)` 解包為 `process_pair(x, y)`。
- **結果**：返回結果列表，與輸入順序一致。

## 應用場景

- **批量數據處理**：對多個數據項執行相同操作，如圖片縮放或文本處理。
  ```python
  images = ["img1.jpg", "img2.jpg", "img3.jpg"]
  result = resize_image.map(images).delay()
  ```
- **多參數批量操作**：對成對數據執行操作，如計算多組數據的距離。
  ```python
  coordinates = [(0, 0, 1, 1), (1, 1, 2, 2)]
  result = calculate_distance.starmap(coordinates).delay()
  ```

## 注意事項

- **資源限制**：Map 和 Starmap 會並行執行任務，可能導致資源爭用，需根據系統能力調整數據集大小。
- **錯誤處理**：若某個任務失敗，整體結果可能包含異常，建議檢查結果：
  ```python
  results = result.get(propagate=False)
  for res in results:
      if isinstance(res, Exception):
          print("Task failed:", res)
  ```
- **工作者數量**：確保有足夠的工作者處理並行任務，否則任務仍會排隊執行。

## 小結

Map 和 Starmap 是 Celery Canvas 中用於對數據集並行應用任務的工具，特別適合批量處理場景。通過這些工具，您可以高效處理大規模數據，提升系統吞吐量。本章節結束了「工作流（Canvas）」的學習，下一章節將介紹「任務路由」，探索如何將任務分發到特定隊列或工作者。

> **提示**：使用 Map 和 Starmap 時，需關注系統資源使用情況，避免過多並行任務導致效能下降。
