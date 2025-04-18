# 持久化調度器

持久化調度器是 Celery Beat 的一種進階功能，允許您將任務調度信息存儲在持久化存儲（如數據庫）中，而不是依賴臨時文件或記憶體，從而確保調度信息在服務重啟後不會丟失。本文將介紹持久化調度器的配置與應用場景。

## 什麼是持久化調度器？

持久化調度器是指 Celery Beat 使用持久化存儲（如數據庫）來保存任務調度時間表與相關配置。預設情況下，Celery Beat 使用 SQLite 文件作為調度存儲，但這在分佈式環境或高可用性場景中可能不足以滿足需求。持久化調度器解決了這一問題，提供可靠的調度信息管理。

- **作用**：將任務調度信息持久化存儲，避免服務重啟或故障導致調度丟失。
- **應用場景**：分佈式系統、多節點部署、需要高可用性的任務調度。

## 配置持久化調度器

Celery Beat 支援多種持久化調度器實現，其中最常用的是基於數據庫的調度器，特別是通過 `django-celery-beat` 擴展實現的 `DatabaseScheduler`。以下是配置步驟。

### 步驟 1：安裝必要套件

安裝 Celery Beat 與持久化調度相關的套件：
```bash
pip install celery[beat] django-celery-beat
```

- **django-celery-beat**：提供基於 Django 模型的持久化調度器。

### 步驟 2：配置 Django 項目

將 `django_celery_beat` 添加到 Django 項目的 `INSTALLED_APPS` 中：
```python
# settings.py
INSTALLED_APPS = [
    ...
    'django_celery_beat',
]
```

指定 Celery Beat 使用數據庫調度器：
```python
# settings.py
CELERY_BEAT_SCHEDULER = 'django_celery_beat.schedulers:DatabaseScheduler'
```

配置 Celery 應用與數據庫連接（假設使用 MySQL 或 PostgreSQL）：
```python
# settings.py
CELERY_BROKER_URL = 'redis://localhost:6379/0'
CELERY_RESULT_BACKEND = 'redis://localhost:6379/1'
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'UTC'

# 數據庫配置
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',  # 或 postgresql
        'NAME': 'celery_beat_db',
        'USER': 'user',
        'PASSWORD': 'password',
        'HOST': 'localhost',
        'PORT': '3306',
    }
}
```

### 步驟 3：運行數據庫遷移

執行數據庫遷移，創建必要的調度表：
```bash
python manage.py migrate
```

這將創建 `django_celery_beat` 所需的數據表，用於存儲任務調度信息。

### 步驟 4：啟動 Celery Beat 與工作者

啟動 Celery Beat，使用數據庫作為持久化調度器：
```bash
celery -A your_app beat --loglevel=info
```

同時啟動 Celery 工作者以執行任務：
```bash
celery -A your_app worker --loglevel=info
```

### 步驟 5：管理調度信息

使用 Django Admin 介面或程式碼管理調度信息。啟用 Django Admin：
```python
# urls.py
from django.contrib import admin
from django.urls import path

urlpatterns = [
    path('admin/', admin.site.urls),
]
```

在 Admin 介面中，您可以手動添加、修改或刪除定時任務與時間表。

或者通過程式碼管理：
```python
from django_celery_beat.models import PeriodicTask, CrontabSchedule
import json

# 創建時間表（每天凌晨執行）
schedule, created = CrontabSchedule.objects.get_or_create(
    hour=0,
    minute=0,
)

# 創建定時任務
PeriodicTask.objects.create(
    name='Daily Persistent Task',
    task='app.tasks.my_persistent_task',
    crontab=schedule,
    args=json.dumps(['param1', 'param2']),
)
```

## 其他持久化調度器選項

除了 `django-celery-beat`，Celery Beat 還支援其他持久化調度器實現：
- **ShelveScheduler**：使用 Python 的 `shelve` 模組存儲調度信息到文件，適合簡單場景。
  ```bash
  celery -A app beat --scheduler=celery.beat:ShelveScheduler --schedule=/path/to/schedule.db
  ```
- **RedisScheduler**：使用 Redis 作為調度存儲，適合分佈式環境（需額外套件支援）。
- **自定義調度器**：您可以實現自己的調度器類，滿足特定需求。

## 應用場景

- **分佈式部署**：在多節點環境中使用數據庫作為統一調度存儲，避免調度衝突。
  ```python
  CELERY_BEAT_SCHEDULER = 'django_celery_beat.schedulers:DatabaseScheduler'
  ```
- **高可用性**：確保調度信息在服務重啟或故障後仍可恢復。
- **管理介面**：通過 Django Admin 提供友好的調度管理介面，方便非技術人員操作。

## 注意事項

- **數據庫效能**：持久化調度器依賴數據庫，頻繁的調度查詢可能影響效能，建議使用高效數據庫（如 PostgreSQL）並設置索引。
- **單一調度器實例**：避免多個 Celery Beat 實例同時運行相同的調度器，可能導致任務重複執行。可以使用鎖機制（如 `django-celery-beat` 的內建鎖）。
- **備份與恢復**：定期備份調度數據庫，避免數據丟失導致調度中斷。

## 小結

持久化調度器是 Celery Beat 在分佈式與高可用性場景中的重要功能，通過將調度信息存儲在數據庫或其他持久化存儲中，您可以確保調度的可靠性與可管理性。本章節結束了「Celery Beat」的學習，您已掌握定時任務配置、Crontab 表達式、動態調度與持久化調度器的核心技術。

> **提示**：選擇持久化調度器時，根據系統規模與需求選擇合適的存儲後端，避免過度複雜化配置。
