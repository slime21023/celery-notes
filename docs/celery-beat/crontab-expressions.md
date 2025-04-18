# Crontab 表達式

Crontab 表達式是 Celery Beat 中用於定義複雜週期性任務時間表的一種語法，類似於 Linux 系統中的 `cron` 工具。本文將介紹 Crontab 表達式的基本格式與用法。

## 什麼是 Crontab 表達式？

Crontab 表達式是一種用於指定任務執行時間的字符串格式，通過定義分鐘、小時、日期、月份和星期等字段，實現靈活的任務調度。

- **作用**：定義精確的任務執行時間表，支援複雜週期性調度。
- **應用場景**：需要按特定時間或週期執行任務，如每天特定時間、每週特定日期等。

## Crontab 表達式格式

Crontab 表達式由五個字段組成，按順序分別表示：
```
* * * * *
分鐘 小時 日期 月份 星期
```

- **分鐘**：0-59，表示每小時的第幾分鐘。
- **小時**：0-23，表示每天的第幾小時。
- **日期**：1-31，表示每月的第幾天。
- **月份**：1-12，表示每年的第幾個月。
- **星期**：0-6（0 表示星期日），表示每週的第幾天。

每個字段支援以下特殊字符：
- **`*`**：表示任意值（即每一個）。
- **`,`**：列舉多個值，如 `1,3,5`。
- **`-`**：表示範圍，如 `1-5`。
- **`/`**：表示步長，如 `*/5`（每 5 個單位執行一次）。

## 在 Celery Beat 中使用 Crontab

Celery Beat 提供 `crontab` 函數來解析 Crontab 表達式，定義任務調度時間表。

### 示例：基本 Crontab 表達式
```python
from celery.schedules import crontab

beat_schedule = {
    'run-every-minute': {
        'task': 'app.tasks.my_task',
        'schedule': crontab(),  # 每分鐘執行，等同於 * * * * *
    },
    'run-daily-at-3am': {
        'task': 'app.tasks.daily_task',
        'schedule': crontab(hour=3, minute=0),  # 每天凌晨 3 點執行
    },
    'run-every-hour-on-weekdays': {
        'task': 'app.tasks.hourly_task',
        'schedule': crontab(minute=0, day_of_week='1-5'),  # 工作日每小時執行
    },
}
```

### 示例：複雜 Crontab 表達式
```python
beat_schedule = {
    'run-every-5-minutes': {
        'task': 'app.tasks.frequent_task',
        'schedule': crontab(minute='*/5'),  # 每 5 分鐘執行
    },
    'run-on-specific-days': {
        'task': 'app.tasks.special_task',
        'schedule': crontab(hour=9, minute=0, day_of_month='1,15'),  # 每月 1 號與 15 號上午 9 點執行
    },
}
```

## 應用場景

- **每日報告**：每天凌晨生成業務報告。
  ```python
  beat_schedule = {
      'daily-report': {
          'task': 'app.tasks.generate_report',
          'schedule': crontab(hour=0, minute=0),
      },
  }
  ```
- **工作日提醒**：工作日上午 9 點發送提醒。
  ```python
  beat_schedule = {
      'weekday-reminder': {
          'task': 'app.tasks.send_reminder',
          'schedule': crontab(hour=9, minute=0, day_of_week='1-5'),
      },
  }
  ```
- **定期檢查**：每 15 分鐘檢查系統狀態。
  ```python
  beat_schedule = {
      'system-check': {
          'task': 'app.tasks.check_system',
          'schedule': crontab(minute='*/15'),
      },
  }
  ```

## 注意事項

- **星期與日期衝突**：當同時指定 `day_of_month` 和 `day_of_week` 時，Celery Beat 會將其視為「或」關係，即滿足任一條件即執行。
- **時間精確度**：Crontab 表達式最小單位為分鐘，無法實現秒級調度。
- **時區設置**：確保 Celery 應用的時區配置正確（`timezone`），以避免調度時間偏差。

## 小結

Crontab 表達式是 Celery Beat 中定義複雜週期性任務時間表的強大工具，通過靈活組合各個時間字段，您可以滿足多樣化的調度需求。下一章節，我們將介紹「動態調度」，探索如何在運行時添加或修改任務調度。

> **提示**：使用 Crontab 表達式時，建議先在線上工具（如 crontab.guru）測試表達式，確保符合預期。
