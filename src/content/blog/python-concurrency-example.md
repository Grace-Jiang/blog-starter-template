---
title: Python并发编程实战示例
description: 项目中的并发编程实例：Task类与ThreadPoolExecutor
pubDate: '2025-02-12'
categories:
- Python
tags:
- 并发
- ThreadPoolExecutor
- 实战
---
# 收集项目里用到的多线程，多进程 协程例子
# Multi thread example
```
    def _collect_resolution_tasks(self):
        task_group1 = self.__build_compute_hostname_resolution_task()
        task_group2 = self.__build_compute_hostname_reverse_resolution_task()
        task_group3 = self.__build_ocp_domain_name_resolution_task()
        self.resolution_tasks = task_group1 + task_group2 + task_group3
        self.resolution_tasks_mapping = {
            ValidationType.COMPUTE_HOSTNAME_RESOLUTION_VALIDATION: task_group1,
            ValidationType.COMPUTE_HOSTNAME_REVERSE_RESOLUTION_VALIDATION: task_group2,
            ValidationType.OCP_DNAME_RESOLUTION_VALIDATION: task_group3
        }
```

```python
import concurrent.futures
from typing import List

MAX_CONCURRENCY = 8


class Task:
    """
    Task class to abstract any func-based task
    """
    def __init__(self, task_id, func, *args, **kwargs):
        self.task_id = task_id
        self._func = func
        self._args = args
        self._kwargs = kwargs
        self._result = None

    def run(self):
        """
        run the task
        """
        return self._func(*self._args, **self._kwargs)

    @property
    def result(self):
        """
        task result: func return, if func no return, return None
        """
        return self._result

    @result.setter
    def result(self, value):
        self._result = value


class MultiThreadsJob:
    """
    MultiThreadsJob class exec task group concurrently,
    a job consists of a list of tasks
    """
    def __init__(self, task_group: List[Task], workers=MAX_CONCURRENCY):
        self.tasks = task_group
        self.workers = workers

    def exec(self):
        """
        tasks exec entry
        """
        with concurrent.futures.ThreadPoolExecutor(max_workers=self.workers) as executor:
            futures = [executor.submit(task.run) for task in self.tasks]
            concurrent.futures.wait(futures)
            for index, task in enumerate(self.tasks):
                task.result = futures[index].result()       
```