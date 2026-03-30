---
title: Python线程安全单例模式实现
description: 双重检查锁定和装饰器实现的线程安全Singleton模式
pubDate: '2025-02-17'
categories:
- Python
tags:
- 单例模式
- 线程安全
- 设计模式
---
# Python线程安全单例模式实现

以下是两种线程安全的单例模式实现方式：双重检查锁定（Double-Check Locking）和基于装饰器的实现。

```python
# from concurrent.futures import ThreadPoolExecutor, as_completed
# import requests

# def fetch(url):
#     return requests.get(url).text

# urls = ["https://baidu.com", "https://google.com"]

# with ThreadPoolExecutor(max_workers=5) as executor:
#     futures = [executor.submit(fetch, url) for url in urls]

#     for future in as_completed(futures):
#         print(future.result())
        
        
# class Singleton:
#     _instance = None
#     def __new__(cls):
#         if cls._instance is None:
#             cls._instance = super().__new__(cls)
#         return cls._instance
    
import threading
class Singleton:
    _instance = None
    _lock = threading.Lock()
    
    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance
    
a = Singleton()
b = Singleton()
print(a is b)


def singleton(cls):
    _instance = {}
    def wrapper(*args, **kwargs):
        if _instance.get(cls) is None:
            _instance[cls] = cls(*args, **kwargs)
        return _instance[cls]
    
    return wrapper

@singleton
class MyClass:
    def __init__(self, a):
        self.value = a
    
    
a = MyClass(1)
b = MyClass(2)
print(a is b)
print(a.value)
print(b.value)
```
