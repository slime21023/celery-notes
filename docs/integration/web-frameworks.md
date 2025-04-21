# Celery 與 Web 框架整合

在實際項目中，Celery 通常需要與 Web 框架（如 Django、Flask 或 FastAPI）整合，以便在用戶請求時觸發異步任務，例如發送電子郵件、處理長時間運行的計算或執行後台數據處理。本章節將介紹如何將 Celery 與常見的 Web 框架整合，並提供最佳實踐建議。

## Celery 與 Django 整合

Django 是 Python 生態中最受歡迎的 Web 框架之一，與 Celery 的整合非常自然。以下是整合的基本步驟。

### 安裝與配置

首先，確保您已安裝必要的套件：

```bash
pip install celery django
```

在 Django 項目中，創建一個 Celery 配置文件（例如 `your_project/celery.py`）：

```python
from celery import Celery
import os

# 設定 Django 的環境變數
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'your_project.settings')

# 創建 Celery 應用
app = Celery('tasks', broker='amqp://localhost')

# 從 Django 設定中載入 Celery 配置
app.config_from_object('django.conf:settings', namespace='CELERY')

# 自動發現任務
app.autodiscover_tasks()
```

在 Django 的 `settings.py` 中，添加 Celery 配置：

```python
# Celery 配置
CELERY_BROKER_URL = 'amqp://localhost'  # 使用 RabbitMQ 作為消息代理
CELERY_RESULT_BACKEND = 'rpc://'  # 結果儲存後端
CELERY_ACCEPT_CONTENT = ['json']  # 只接受 JSON 格式
CELERY_TASK_SERIALIZER = 'json'  # 任務序列化格式
CELERY_RESULT_SERIALIZER = 'json'  # 結果序列化格式
```

### 定義任務

在 Django 應用中創建任務（例如 `your_app/tasks.py`）：

```python
from celery import shared_task

@shared_task
def send_email(to_email, subject, body):
    # 模擬發送電子郵件
    print(f"發送郵件到 {to_email}，主題：{subject}")
    return True
```

### 在 View 中調用任務

在 Django 的視圖中，您可以異步調用任務：

```python
from django.http import HttpResponse
from your_app.tasks import send_email

def send_email_view(request):
    send_email.delay('user@example.com', '歡迎', '歡迎使用我們的服務！')
    return HttpResponse("郵件已發送（異步處理）")
```

### 最佳實踐
- **使用 `shared_task`**：使用 `@shared_task` 裝飾器而非 `@app.task`，以避免直接依賴特定的 Celery 應用實例。
- **設定結果後端**：如果需要追蹤任務結果，確保設定一個結果後端（如 Redis 或資料庫）。
- **避免阻塞請求**：確保在 Web 請求中只調用 `.delay()` 或 `.apply_async()`，避免直接執行任務。

## Celery 與 Flask 整合

Flask 是一個輕量級的 Web 框架，與 Celery 的整合同樣簡單，但需要額外處理 Flask 的應用上下文。

### 安裝與配置

安裝必要的套件：

```bash
pip install celery flask
```

創建 Celery 應用（例如 `app/celery_app.py`）：

```python
from celery import Celery

def make_celery(app):
    celery = Celery(
        app.import_name,
        backend=app.config['CELERY_RESULT_BACKEND'],
        broker=app.config['CELERY_BROKER_URL']
    )
    celery.conf.update(app.config)
    
    class ContextTask(celery.Task):
        def __call__(self, *args, **kwargs):
            with app.app_context():
                return self.run(*args, **kwargs)
    
    celery.Task = ContextTask
    return celery
```

在 Flask 應用中初始化 Celery（例如 `app/__init__.py`）：

```python
from flask import Flask
from .celery_app import make_celery

app = Flask(__name__)
app.config.update(
    CELERY_BROKER_URL='amqp://localhost',
    CELERY_RESULT_BACKEND='rpc://'
)

celery = make_celery(app)
```

### 定義任務

在 `app/tasks.py` 中定義任務：

```python
from . import celery

@celery.task()
def process_data(data):
    # 模擬處理數據
    print(f"處理數據：{data}")
    return f"已處理：{data}"
```

### 在路由中調用任務

在 Flask 路由中異步調用任務：

```python
from flask import jsonify
from .tasks import process_data

@app.route('/process/<data>')
def process(data):
    task = process_data.delay(data)
    return jsonify({'task_id': task.id, 'status': '任務已提交'})
```

### 最佳實踐
- **處理應用上下文**：Flask 的任務需要應用上下文，使用上述 `ContextTask` 類確保任務執行時能夠訪問 Flask 的配置和資料庫。
- **結果追蹤**：若需要結果，可以通過 `task.get()` 或 API 查詢任務狀態。

## Celery 與 FastAPI 整合

FastAPI 是一個現代的非同步 Web 框架，與 Celery 的整合適合需要高性能的應用。

### 安裝與配置

安裝必要的套件：

```bash
pip install celery fastapi uvicorn
```

創建 Celery 應用（例如 `celery_app.py`）：

```python
from celery import Celery

celery = Celery('tasks', broker='amqp://localhost', backend='rpc://')
celery.conf.update(
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
)
```

### 定義任務

在 `tasks.py` 中定義任務：

```python
from celery_app import celery

@celery.task
def long_running_task(data):
    # 模擬長時間任務
    import time
    time.sleep(5)
    return f"處理完成：{data}"
```

### 在 FastAPI 中調用任務

在 FastAPI 應用中（例如 `main.py`）：

```python
from fastapi import FastAPI, BackgroundTasks
from tasks import long_running_task

app = FastAPI()

@app.post("/process/{data}")
async def process_data(data: str, background_tasks: BackgroundTasks):
    task = long_running_task.delay(data)
    return {"task_id": task.id, "message": "任務已提交"}
```

### 最佳實踐
- **非同步調用**：FastAPI 是非同步框架，但 Celery 的 `.delay()` 是同步的，確保不會阻塞主線程。
- **背景任務**：可以使用 FastAPI 的 `BackgroundTasks` 進一步管理異步行為。

## 總結與注意事項

- **選擇合適的框架**：根據您的項目需求選擇框架，Django 適合大型項目，Flask 適合輕量級應用，FastAPI 適合高性能非同步應用。
- **避免阻塞**：無論使用哪個框架，確保 Web 請求不直接執行任務，而是使用 `.delay()` 或 `.apply_async()`。
- **監控任務狀態**：在 Web 應用中提供任務狀態查詢功能，提升用戶體驗。
- **錯誤處理**：在任務中加入錯誤處理，避免因任務失敗影響用戶體驗。

通過以上方法，您可以輕鬆將 Celery 整合到 Web 框架中，實現高效的異步任務處理。
