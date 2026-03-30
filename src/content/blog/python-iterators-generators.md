---
title: Python迭代器与生成器详解
description: Python迭代器协议、__iter__/__next__、yield、生成器表达式
pubDate: '2025-02-10'
categories:
- Python
tags:
- 迭代器
- 生成器
- yield
- 面试
---
# 迭代器与生成器: __iter__, __next__, yield

## 迭代器 (Iterators)

### 什么是迭代器？
迭代器是一个实现了迭代器协议的对象，包含 `__iter__()` 和 `__next__()` 方法。

### 迭代器协议
- `__iter__()`: 返回迭代器对象本身
- `__next__()`: 返回下一个值，如果没有更多元素则抛出 `StopIteration` 异常

### 示例：自定义迭代器
```python
class Counter:
    def __init__(self, start, end):
        self.current = start
        self.end = end
    
    def __iter__(self):
        return self
    
    def __next__(self):
        if self.current >= self.end:
            raise StopIteration
        self.current += 1
        return self.current - 1

# 使用
counter = Counter(1, 5)
for num in counter:
    print(num)  # 输出: 1, 2, 3, 4
```

## 生成器 (Generators)

### 什么是生成器？
生成器是一种特殊的迭代器，使用 `yield` 关键字来产生值。生成器函数在调用时不会立即执行，而是返回一个生成器对象。

### yield 关键字
- `yield` 暂停函数执行并返回一个值
- 下次调用 `__next__()` 时，函数从上次暂停的地方继续执行
- 函数结束时自动抛出 `StopIteration`

### 示例：生成器函数
```python
def countdown(n):
    while n > 0:
        yield n
        n -= 1

# 使用
gen = countdown(5)
print(next(gen))  # 5
print(next(gen))  # 4

for num in countdown(3):
    print(num)  # 输出: 3, 2, 1
```

### 生成器表达式
```python
# 列表推导式
squares_list = [x**2 for x in range(10)]

# 生成器表达式（使用圆括号）
squares_gen = (x**2 for x in range(10))

print(next(squares_gen))  # 0
print(next(squares_gen))  # 1
```

## 迭代器 vs 生成器

| 特性 | 迭代器 | 生成器 |
|------|--------|--------|
| 定义方式 | 类，实现 `__iter__` 和 `__next__` | 函数，使用 `yield` |
| 代码复杂度 | 较复杂 | 简洁 |
| 内存效率 | 高效（惰性求值） | 高效（惰性求值） |
| 状态管理 | 手动管理 | 自动管理 |

## 实际应用场景

### 1. 处理大文件
```python
def read_large_file(file_path):
    with open(file_path, 'r') as file:
        for line in file:
            yield line.strip()

# 逐行处理，不会一次性加载整个文件到内存
for line in read_large_file('large_file.txt'):
    process(line)
```

### 2. 无限序列
```python
def fibonacci():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

# 生成斐波那契数列
fib = fibonacci()
for _ in range(10):
    print(next(fib))
```

### 3. 管道处理
```python
def numbers():
    for i in range(10):
        yield i

def square(nums):
    for num in nums:
        yield num ** 2

def filter_even(nums):
    for num in nums:
        if num % 2 == 0:
            yield num

# 链式处理
pipeline = filter_even(square(numbers()))
print(list(pipeline))  # [0, 4, 16, 36, 64]
```

## 高级特性

### send() 方法
```python
def echo():
    while True:
        value = yield
        if value is not None:
            print(f"Received: {value}")

gen = echo()
next(gen)  # 启动生成器
gen.send("Hello")  # 发送值到生成器
gen.send("World")
```

### yield from
```python
def sub_generator():
    yield 1
    yield 2

def main_generator():
    yield from sub_generator()
    yield 3

print(list(main_generator()))  # [1, 2, 3]
```

## 性能优势

```python
import sys

# 列表：占用更多内存
list_comp = [x**2 for x in range(1000000)]
print(sys.getsizeof(list_comp))  # ~8MB

# 生成器：占用很少内存
gen_exp = (x**2 for x in range(1000000))
print(sys.getsizeof(gen_exp))  # ~128 bytes
```

# __new__ 与 __init__ 的区别

## 基本概念

### __new__ 方法
- **作用**: 创建对象实例，返回一个新的对象
- **调用时机**: 在 __init__ 之前调用
- **返回值**: 必须返回一个实例

### __init__ 方法
- **作用**: 初始化对象属性，设置对象状态
- **调用时机**: 在 __new__ 返回实例后调用
- **返回值**: 总是返回 None

## 执行顺序

```python
class MyClass:
    def __new__(cls, *args, **kwargs):
        print("__new__ 被调用")
        return super().__new__(cls)
    
    def __init__(self, value):
        print("__init__ 被调用")
        self.value = value

obj = MyClass(42)
# 输出:
# __new__ 被调用
# __init__ 被调用
```

## 主要区别

| 特性 | __new__ | __init__ |
|------|---------|----------|
| **目的** | 创建对象 | 初始化对象 |
| **调用时机** | 对象创建时 | 对象创建后 |
| **参数** | cls + 其他参数 | self + 其他参数 |
| **返回值** | 必须返回实例 | 总是返回 None |

## 常见应用

### 1. 单例模式
```python
class Singleton:
    _instance = None
    
    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

s1 = Singleton()
s2 = Singleton()
print(s1 is s2)  # True
```

### 2. 控制实例创建
```python
class PositiveNumber:
    def __new__(cls, value):
        if value < 0:
            raise ValueError("Value must be positive")
        return super().__new__(cls)

# PositiveNumber(-5)  # 会抛出 ValueError
```

## 总结

- **__new__**: 负责对象的**创建**
- **__init__**: 负责对象的**初始化**
- 大多数情况下只需要重写 `__init__`
- 需要控制实例创建时才重写 `__new__`
