# 動態調度

動態調度是 Celery Beat 的一種進階功能，允許您在運行時添加、修改或刪除任務調度，而無需修改配置文件或重啟服務。本文將介紹動態調度的實現方法與應用場景。

## 什麼是動態調度？

動態調度是指通過程式碼或 API 在 Celery Beat 運行時更新任務調度時間表，無需依賴靜態配置文件。這使得任務調度更加靈活，能夠根據業務需求即時調整。

- **作用**：運行時管理任務調度，適應動態業務需求。
- **應用場景**：用戶自定義任務執行時間、臨時任務安排、根據事件觸發調度。

## 實現動態調度

Celery Beat 支援通過持久化調度器（如數據庫）實現動態調度，常用的方法是使用 `django-celery-beat` 擴展，它提供了基於 Django 模型的調度管理功能。

### 步驟 1：安裝 django-celery-beat

首先安裝 `django-celery-beat`，並將其整合到 Django 項目中：
```bash
pip install django-celery-beat
```

在 Django 設定中添加應用：
```python
INSTALLED_APPS = [
    ...
    'django_celery_beat',
]
```

### 步驟 2：配置 Celery Beat 調度器

在 Django 設定中指定 Celery Beat 使用數據庫調度器：
```python
# settings.py
CELERY_BEAT_SCHEDULER = 'django_celery_beat.schedulers:DatabaseScheduler'
```

運行數據庫遷移以創建調度表：
```bash
python manage.py migrate
```

### 步驟 3：動態添加與修改調度

使用 Django 模型 API 動態管理任務調度：
```python
from django_celery_beat.models import PeriodicTask, CrontabSchedule

# 創建一個 Crontab 時間表（例如每天凌晨執行）
schedule, created = CrontabSchedule.objects.get_or_create(
    hour=0,
    minute=0,
)

# 創建或更新一個定時任務
task, created = PeriodicTask.objects.get_or_create(
    name='Dynamic Daily Task',
    defaults={
        'task': 'app.tasks.my_dynamic_task',  # 任務路徑
        'crontab': schedule,
        'args': json.dumps([arg1, arg2]),  # 任務參數，可選
    },
)

# 修改現有任務的時間表
task.crontab.hour = 1  # 修改為凌晨 1 點執行
task.crontab.save()
task.save()

# 刪除任務
task.delete()
```

- **CrontabSchedule**：定義任務執行時間表。
- **PeriodicTask**：定義定時任務，關聯時間表與任務路徑。

### 步驟 4：啟動 Celery Beat

使用數據庫調度器啟動 Celery Beat：
```bash
celery -A app beat --loglevel=info
```

確保 Celery 工作者也在運行：
```bash
celery -A app worker --loglevel=info
```

## 應用場景

- **用戶自定義調度**：允許用戶通過介面設定任務執行時間。
  ```python
  def schedule_user_task(user_id, task_time):
      schedule, _ = CrontabSchedule.objects.get_or_create(
          hour=task_time.hour,
          minute=task_time.minute,
      )
      PeriodicTask.objects.create(
          name=f'User Task {user_id}',
          task='app.tasks.user_task',
          crontab=schedule,
          args=json.dumps([user_id]),
      )
  ```
- **臨時任務安排**：根據事件臨時添加一次性任務。
- **調度調整**：根據業務高峰期動態調整任務頻率。

## 注意事項

- **數據庫依賴**：動態調度依賴持久化調度器（如 `django-celery-beat`），需確保數據庫可用。
- **任務衝突**：避免創建名稱或功能重複的任務，建議
