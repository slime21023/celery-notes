# 任務定義與執行

Celery 的核心功能是處理異步任務，本文將介紹如何使用 Celery 定義任務並在後台執行，幫助您快速實現應用程式的異步處理。

## Celery 環境設置

在使用 Celery 之前，需完成基本環境設置。以下假設您已安裝 RabbitMQ 作為消息代理。

### 步驟 1：安裝 Celery
使用 pip 安裝 Celery：
```bash
pip install celery
```

### 步驟 2：安裝消息代理
若使用 RabbitMQ，確保已安裝並啟動服務。若使用 Redis，可安裝：
```bash
pip install redis
```

## 定義 Celery 任務

Celery 任務是使用 `@app.task` 裝飾器定義的 Python 函數。以下是基本步驟與示例：

### 步驟 1：創建 Celery 實例
創建一個名為 `app.py` 的文件，初始化 Celery 應用：
```python
from celery import Celery

# 初始化 Celery，指定消息代理（這裡使用 RabbitMQ）
app = Celery('tasks', broker='amqp://guest:guest@localhost:5672//')

# 配置 Celery（可選）
app.conf.update(
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='UTC',
)
```

### 步驟 2：定義任務
在同一文件或另一文件中定義任務：
```python
@app.task
def add(x, y):
    return x + y

@app.task
def send_email(to, subject, body):
    # 模擬發送郵件
    print(f"Sending email to {to}: {subject}")
    return f"Email sent to {to}"
```

## 執行 Celery 任務

任務定義後，可通過以下方式執行：

### 方式 1：異步執行
使用 `delay()` 方法將任務發送到隊列，立即返回而不等待結果：
```python
# 調用任務（異步）
result = add.delay(4, 6)
print("Task sent, ID:", result.id)
```

### 方式 2：同步執行（測試用）
使用 `apply()` 方法同步執行任務（僅用於調試，不推薦在生產環境使用）：
```python
result = add.apply(args=(4, 6))
print("Result:", result.get())
```

## 啟動 Celery 工作者

任務發送後，需啟動工作者從隊列中取出並執行任務：
```bash
celery -A app worker --loglevel=info
```
- `-A app`：指定 Celery 應用模組（這裡是 `app.py`）。
- `--loglevel=info`：設置日誌等級，方便調試。

## 應用場景示例

- **郵件發送**：將耗時的郵件發送操作交由 Celery 處理，避免阻塞主應用。
  ```python
  send_email.delay("user@example.com", "Welcome", "Welcome to our platform!")
  ```
- **數據處理**：處理大規模數據匯總或圖片縮放。
  ```python
  @app.task
  def process_data(data):
      # 模擬耗時處理
      return len(data)
  ```

## 小結

通過定義與執行 Celery 任務，您可以輕鬆實現應用程式的異步處理，提升響應速度與用戶體驗。下一章節，我們將介紹「結果後端（Result Backend）」，學習如何存儲與查詢任務執行結果。

> **提示**：任務應設計為輕量且可重入，避免在任務中執行過於複雜或不可預測的操作。
