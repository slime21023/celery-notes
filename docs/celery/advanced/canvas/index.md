# 工作流（Canvas）概述

Celery 的 Canvas 模組提供了一組工具，用於編排複雜的任務工作流，允許您定義任務之間的依賴關係與執行順序。本文將介紹 Canvas 的基本概念與功能，後續文章將深入各個工具的具體用法。

## 什麼是 Canvas？

Canvas 是 Celery 提供的一組高級原語（Primitives），用於組合與編排任務，實現複雜的工作流。它允許您將多個任務組織成有結構的執行流程，滿足業務邏輯中的順序、並行或條件執行需求。

- **作用**：任務編排與流程控制。
- **核心工具**：
  - **Chain**：按順序執行一系列任務。
  - **Group**：並行執行一組任務。
  - **Chord**：在並行任務完成後執行回調任務。
  - **Map & Starmap**：對數據集應用任務，類似於 Python 的 `map` 函數。

## 為什麼需要 Canvas？

在實際應用中，單一任務往往無法滿足複雜需求。例如：
- 處理訂單時，需先驗證數據、再更新庫存、最後發送通知（順序執行）。
- 批量處理多個用戶數據時，需並行處理以提升效率（並行執行）。
- 在多個子任務完成後，需匯總結果進行後續處理（並行後回調）。

Canvas 工具通過結構化方式解決這些問題，讓任務執行更加靈活與可控。

## Canvas 基本用法

Canvas 工具通過 Celery 提供的函數或方法組合任務，生成一個可執行的流程。以下是簡單示例：

### 示例：組合任務
```python
from celery import chain, group
from app import task1, task2, task3  # 假設這些是已定義的任務

# 定義順序執行的工作流
workflow = chain(task1.s(), task2.s(), task3.s())
result = workflow.delay()

# 定義並行執行的工作流
parallel_workflow = group(task1.s(), task2.s(), task3.s())
result = parallel_workflow.delay()
```

- **`.s()`**：生成任務的簽名（Signature），用於 Canvas 組合，不直接執行。
- **`.delay()`**：啟動工作流，將其發送到隊列執行。

## 小結

Canvas 是 Celery 實現複雜任務工作流的關鍵工具，通過 Chain、Group、Chord 等原語，您可以靈活編排任務執行順序與關係。接下來的文章將逐一介紹這些工具的具體用法與應用場景。

> **提示**：使用 Canvas 時，確保任務之間的依賴關係清晰，避免過於複雜的嵌套結構導致調試困難。
