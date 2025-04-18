# 定時任務配置

Celery Beat 是用於定時任務調度的擴展工具，本文將介紹如何安裝與配置 Celery Beat，並定義基本的定時任務。

## 安裝 Celery Beat

Celery Beat 是一個獨立的 Python 包，需單獨安裝。使用以下命令安裝：
```bash
pip install celery[redis] celery[beat]
```

- **celery[redis]**：確保安裝 Redis 支援，作為消息代理或結果後端。
- **celery[beat]**：安裝 Celery Beat 擴展。

## 配置 Celery Beat

Celery Beat 與 Celery 應用整合使用，需在 Celery 應用中啟用 Beat 調度功能。

### 步驟 1：創建 Celery 應用

確保您已有一個 Celery 應用，配置好消息代理與結果後端：
```python
from celery import Celery

app = Celery('tasks',
             broker='redis://localhost:6379/0',
             backend='redis://localhost:6379/1')

app.conf.update(
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='UTC',
)
```

### 步驟 2：配置 Beat 調度

創建一個 Beat 配置文件（例如 `beatconfig.py`），定義任務調度時間表：
```python
from celery.schedules import crontab

beat_schedule = {
    'run-every-30-seconds': {
        'task': 'app.tasks.my_task',  # 指定任務路徑
        'schedule': 30.0,  # 每 30 秒執行一次
        'args': (arg1, arg2),  # 任務參數，可選
    },
    'run-daily-at-midnight': {
        'task': 'app.tasks.daily_report',
        'schedule': crontab(hour=0, minute=0),  # 每天凌晨執行
    },
}
```

- **beat_schedule**：定義任務調度時間表，每個任務包含任務路徑、執行時間表與可選參數。
- **schedule**：可以是浮點數（秒數間隔）或 Crontab 表達式。

### 步驟 3：啟動 Celery Beat

使用 `celery beat` 命令啟動調度器，確保 Celery 工作者也在運行：
```bash
# 啟動 Celery Beat，指定 Beat 配置文件
celery -A app beat --loglevel=info --schedule=/path/to/beat_schedule.db
```

- **-A app**：指定 Celery 應用模組。
- **--schedule**：指定調度信息存儲文件（預設為 SQLite 數據庫）。

同時，啟動 Celery 工作者以執行任務：
```bash
celery -A app worker --loglevel=info
```

## 定義定時任務

以下是一個簡單的定時任務示例：
```python
from celery import Celery

app = Celery('tasks', broker='redis://localhost:6379/0')

@app.task
def my_task(arg1, arg2):
    print(f"Task executed with args: {arg1}, {arg2}")
    return arg1 + arg2
```

在 Beat 配置文件中添加該任務的調度：
```python
beat_schedule = {
    'run-my-task-every-minute': {
        'task': 'app.tasks.my_task',
        'schedule': 60.0,  # 每 60 秒執行一次
        'args': (10, 20),
    },
}
```

## 應用場景

- **定期數據備份**：每天凌晨執行數據備份任務。
  ```python
  beat_schedule = {
      'backup-data-daily': {
          'task': 'app.tasks.backup_data',
          'schedule': crontab(hour=0, minute=0),
      },
  }
  ```
- **定時通知**：每小時向用戶發送提醒。
- **數據同步**：每 30 分鐘從外部 API 同步數據。

## 注意事項

- **時間同步**：確保伺服器時間正確，否則定時任務可能偏離預期。
- **工作者可用性**：確保有足夠的工作者執行定時任務，避免任務堆積。
- **調度重疊**：避免任務執行時間過長導致下一次調度時仍未完成，可設置任務超時。

## 小結

通過安裝與配置 Celery Beat，您可以輕鬆實現定時任務的自動化執行，為業務自動化提供強大支持。下一章節，我們將介紹「Crontab 表達式」，探索如何定義更複雜的週期性任務時間表。

> **提示**：初次使用 Celery Beat 時，建議從簡單的間隔調度開始，確保配置正確後再引入複雜時間表。
