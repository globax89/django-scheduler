Scheduler
=========
_This project is still in alpha stage. Try at your own risk._

Task Scheduler for Django
-------------------------

Scheduler is a simple Django app for scheduling tasks at a specific time or repeating in a pattern based on the rules defined in **Recurrence Rule _(RRULE)_** syntax as per the [RFC2445](https://tools.ietf.org/html/rfc2445) standard.

# Special Thanks
* [Django](https://github.com/django/django/) - ORM and app system.
* [Celery](https://github.com/celery/celery/) - Task management.
* [Dateutil](https://github.com/dateutil/dateutil/) - Interpreting _RRULE_ syntax.

# Requirements
* [Django](https://github.com/django/django/) 1.8+
* [Celery](https://github.com/celery/celery/) 4.0+

# Usage
* Add the `celery.task` or `shared_task` decorator and specify `scheduler.tasks.RepeatTask` as the base.
```python
from celery import shared_task
from scheduler.tasks import RepeatTask

@shared_task(base=RepeatTask)
def say_hello(fruit):
    print(f'Hello, I ate {fruit} today.')
```
* Make sure atleast one Celery worker is running. Restart your workers so that the changes are reflected in the workers.
* Use `TaskScheduler` to schedule the task. You can specify the starting time, the ending time and the recurrence rule for the task. Arguments and keyword arguments should be specified separately. Task description is mandatory. The task scheduler returns a `task_id` which can be used to cancel the task.
```python
from datetime import timedelta

from django.utils import timezone
from scheduler.scheduler import TaskScheduler

now = timezone.now() + timedelta(hours=3)
tomorrow = now + timedelta(hours=7)

task_id = TaskScheduler.schedule(say_hello,
    description='Speaks out which fruit you ate today',
    args=('Apple', )
    kwargs=None,
    rrule_string='RRULE:FREQ=HOURLY;INTERVAL=2', # Run every 2 hours...
    trigger_at=now, # ...starting 3 hours from now...
    until=tomorrow  # ...till 10 hours from now..
)
print(f'Task ID: {task_id}')  # Task ID: UUID('53f81020-1b77-45cd-8f0d-2c5a8acee6a8')
```
* Suppose, you run the above snippet at `2017-03-25 01:30:00`, the task will execute at the following times,
```
[2017-03-25 04:30:00] Hello, I ate Apple today.
[2017-03-25 06:30:00] Hello, I ate Apple today.
[2017-03-25 08:30:00] Hello, I ate Apple today.
[2017-03-25 10:30:00] Hello, I ate Apple today.
```
* To run this task immediately, set `trigger_at` as `None`.
* To run this task infinitely, set `until` as `None`.
* To cancel a schedule, use `TaskScheduler.cancel` method and pass the `task_id` that was generated by the `TaskScheduler`.
```python
TaskScheduler.cancel(task_id)
```

# API
### Schedule a task
#### `TaskScheduler.schedule(func, description, args=None, kwargs=None, rrule_string=None, trigger_at=None, until=None)`
| Parameter | Type | Description |
| --- | --- | --- |
| `func` | `function` | **Required.** Task function. |
| `description` | `str` | **Required.** Description of the task. **This may get deprecated in future.** |
| `args` | `list` or `tuple` | **Optional.** **Default:** `None`. Arguments to be passed to `func`. **All values should be JSON serializable.** |
| `kwargs` | `dict` | **Optional.** **Default:** `None`. Keyword arguments to be passed to `func`.  **All values should be JSON serializable.** |
| `rrule_string` | `str` | **Optional.** **Default:** `None`. Recurrence rule. `func` will be executed as per the pattern defined by this rule. If not specified, `func` will be executed only once. |
| `trigger_at` | `datetime` with `tzinfo` | **Optional.** **Default:** `None`. Sets the datetime for the first run of the task. If not specified, it defaults to `datetime.now()`. |
| `until` | `datetime` with `tzinfo` | **Optional.** **Default:** `None`. Sets the end time for the task. The task will never execute after this datetime. If not specified, `task` will run as long as it is defined in the recurrence rule. |

> **NOTE:** It is important that all the `datetime` instances passed should have timezone information, i. e., they should be [timezone aware](https://docs.djangoproject.com/en/1.10/topics/i18n/timezones/#naive-and-aware-datetime-objects). Use Django's [`timezone.now()`](https://docs.djangoproject.com/en/1.10/ref/utils/#django.utils.timezone.now) utility to get a timezone aware instance with `USE_TZ=True` in settings.

##### Returns
| Type | Description |
| --- | --- |
| `UUID` | Scheduled Task ID. |

### Cancel scheduled task
#### `TaskSchedule.cancel(task_id)`
| Parameter | Type | Description |
| --- | --- | --- |
| `scheduled_task_id` | `UUID` | **Required.** Scheduled Task ID. |

#### Returns
| Type | Description |
| --- | --- |
| `ScheduledTask` | Scheduled task model instance. **This may get deprecated in future.** |

# Limitations
* Currently, there is no support for `DTSTART` and `UNTIL` rules. Use `trigger_at` and `until` parameters. Even if you specify `DTSTART` and `UNTIL` rules, they will get overidden.
