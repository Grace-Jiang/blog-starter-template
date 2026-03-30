---
title: Python装饰器深入详解 - 从闭包到类装饰器
description: 闭包、装饰器执行时机、functools.wraps、参数化装饰器、类装饰器
pubDate: '2025-02-15'
categories:
- Python
tags:
- 装饰器
- 闭包
- functools
- 面试
---
# 装饰器: decorator 原理和实现

## 核心原理

### 1. 闭包是装饰器的基础

装饰器本质上是**利用闭包来包装函数**。

```python
# 闭包示例
def outer(x):
    def inner(y):
        return x + y  # inner 可以访问 outer 的变量 x
    return inner

add_5 = outer(5)
print(add_5(3))  # 8
```

**闭包三要素**：
1. 嵌套函数
2. 内层函数引用外层函数的变量
3. 外层函数返回内层函数

### 2. 装饰器的执行时机

**关键点：装饰器在函数定义时执行，而不是调用时！**

```python
def my_decorator(func):
    print(f"装饰器执行: 装饰 {func.__name__}")
    def wrapper():
        print("wrapper 执行")
        return func()
    return wrapper

print("开始定义函数")

@my_decorator
def say_hello():
    print("Hello!")

print("函数定义完成")
print("开始调用函数")
say_hello()

# 输出顺序:
# 开始定义函数
# 装饰器执行: 装饰 say_hello
# 函数定义完成
# 开始调用函数
# wrapper 执行
# Hello!
```

### 3. @ 语法糖的本质

```python
@decorator
def func():
    pass

# 完全等价于:
def func():
    pass
func = decorator(func)
```

## 深入实现细节

### 1. 装饰器的三层结构

```python
# 带参数的装饰器 = 三层嵌套函数
def repeat(times):              # 第1层: 接收装饰器参数
    def decorator(func):        # 第2层: 接收被装饰的函数
        def wrapper(*args, **kwargs):  # 第3层: 接收函数调用参数
            for _ in range(times):
                result = func(*args, **kwargs)
            return result
        return wrapper
        return decorator

@repeat(3)  # repeat(3) 返回 decorator，decorator(say_hello) 返回 wrapper
def say_hello(name):
    print(f"Hello, {name}")

# 等价于:
# say_hello = repeat(3)(say_hello)
```

**执行流程**：
1. `repeat(3)` → 返回 `decorator`
2. `decorator(say_hello)` → 返回 `wrapper`
3. `say_hello("Alice")` → 实际调用 `wrapper("Alice")`

### 2. 为什么需要 functools.wraps

```python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        """这是 wrapper 的文档"""
        return func(*args, **kwargs)
    return wrapper

@my_decorator
def greet(name):
    """这是 greet 的文档"""
    return f"Hello, {name}"

# 问题: 原函数的元信息丢失
print(greet.__name__)    # wrapper (应该是 greet)
print(greet.__doc__)     # 这是 wrapper 的文档 (应该是 greet 的)
print(greet.__module__)  # __main__
```

**解决方案**：
```python
from functools import wraps

def my_decorator(func):
    @wraps(func)  # 复制 func 的元信息到 wrapper
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@my_decorator
def greet(name):
    """这是 greet 的文档"""
    return f"Hello, {name}"

print(greet.__name__)  # greet ✓
print(greet.__doc__)   # 这是 greet 的文档 ✓
```

**wraps 的实现原理**：
```python
# functools.wraps 简化实现
WRAPPER_ASSIGNMENTS = ('__module__', '__name__', '__qualname__', '__doc__', '__annotations__')
WRAPPER_UPDATES = ('__dict__',)

def wraps(wrapped):
    def decorator(wrapper):
        for attr in WRAPPER_ASSIGNMENTS:
            setattr(wrapper, attr, getattr(wrapped, attr))
        wrapper.__wrapped__ = wrapped
        return wrapper
    return decorator
```

### 3. 类装饰器的实现原理

**使用 `__call__` 方法**：

```python
class MyDecorator:
    def __init__(self, func):
        self.func = func
        self.count = 0
    
    def __call__(self, *args, **kwargs):
        self.count += 1
        print(f"调用次数: {self.count}")
        return self.func(*args, **kwargs)

@MyDecorator
def say_hello():
    print("Hello!")

# 等价于: say_hello = MyDecorator(say_hello)
# 调用 say_hello() 实际调用 MyDecorator.__call__()

say_hello()  # 调用次数: 1
say_hello()  # 调用次数: 2
```

**带参数的类装饰器**：

```python
class Repeat:
    def __init__(self, times):
        self.times = times
    
    def __call__(self, func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(self.times):
                result = func(*args, **kwargs)
            return result
        return wrapper

@Repeat(3)
def say_hello():
    print("Hello!")

# 等价于: say_hello = Repeat(3)(say_hello)
```

## 常见面试题实现

### 1. 手写计时装饰器

```python
import time
from functools import wraps

def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(f"{func.__name__} 耗时: {end - start:.4f}秒")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(1)
    return "完成"

slow_function()
```

### 2. 手写缓存装饰器（LRU 简化版）

```python
from functools import wraps

def memo(func):
    cache = {}
    
    @wraps(func)
    def wrapper(*args):
        if args not in cache:
            cache[args] = func(*args)
        return cache[args]
    return wrapper

@memo
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

print(fibonacci(100))  # 很快返回结果
```

**Python 内置的 lru_cache**：
```python
from functools import lru_cache

@lru_cache(maxsize=128)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)
```

### 3. 手写重试装饰器

```python
from functools import wraps
import time

def retry(max_attempts=3, delay=1, exceptions=(Exception,)):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == max_attempts - 1:
                        raise
                    print(f"第 {attempt + 1} 次尝试失败: {e}")
                    time.sleep(delay)
        return wrapper
    return decorator

@retry(max_attempts=3, delay=2, exceptions=(ConnectionError,))
def unstable_api():
    import random
    if random.random() < 0.7:
        raise ConnectionError("连接失败")
    return "成功"
```

### 4. 手写权限检查装饰器

```python
from functools import wraps

def require_permission(permission):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            user = kwargs.get('user')
            if not user:
                raise PermissionError("需要登录")
            if permission not in user.get('permissions', []):
                raise PermissionError(f"需要权限: {permission}")
            return func(*args, **kwargs)
        return wrapper
    return decorator

@require_permission('admin')
def delete_user(user_id, user=None):
    return f"删除用户 {user_id}"

# 使用
admin_user = {'permissions': ['admin', 'read']}
delete_user(123, user=admin_user)  # 成功

normal_user = {'permissions': ['read']}
# delete_user(123, user=normal_user)  # PermissionError
```

### 5. 手写单例装饰器

```python
from functools import wraps

def singleton(cls):
    instances = {}
    
    @wraps(cls)
    def wrapper(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return wrapper

@singleton
class Database:
    def __init__(self):
        print("初始化数据库连接")
        self.connection = "db_connection"

db1 = Database()  # 初始化数据库连接
db2 = Database()  # 不会再次初始化
print(db1 is db2)  # True
```

## 装饰器的高级用法

### 1. 装饰器工厂模式

```python
def decorator_factory(arg1, arg2):
    print(f"工厂参数: {arg1}, {arg2}")
    
    def decorator(func):
        print(f"装饰函数: {func.__name__}")
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            print(f"执行 wrapper")
            return func(*args, **kwargs)
        return wrapper
    return decorator

@decorator_factory("hello", "world")
def my_func():
    print("my_func 执行")

# 输出顺序:
# 工厂参数: hello, world
# 装饰函数: my_func
# (调用 my_func() 时才输出下面的)
# 执行 wrapper
# my_func 执行
```

### 2. 可选参数的装饰器

```python
from functools import wraps

def smart_decorator(func=None, *, prefix="[LOG]"):
    def decorator(f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            print(f"{prefix} 调用 {f.__name__}")
            return f(*args, **kwargs)
        return wrapper
    
    if func is None:
        # 带参数调用: @smart_decorator(prefix=">>>")
        return decorator
    else:
        # 不带参数调用: @smart_decorator
        return decorator(func)

# 用法1: 不带参数
@smart_decorator
def func1():
    pass

# 用法2: 带参数
@smart_decorator(prefix=">>>")
def func2():
    pass

func1()  # [LOG] 调用 func1
func2()  # >>> 调用 func2
```

### 3. 装饰器叠加顺序

```python
def decorator_a(func):
    print("A: 装饰")
    @wraps(func)
    def wrapper(*args, **kwargs):
        print("A: 前")
        result = func(*args, **kwargs)
        print("A: 后")
        return result
    return wrapper

def decorator_b(func):
    print("B: 装饰")
    @wraps(func)
    def wrapper(*args, **kwargs):
        print("B: 前")
        result = func(*args, **kwargs)
        print("B: 后")
        return result
    return wrapper

@decorator_a
@decorator_b
def my_func():
    print("函数执行")

# 装饰时输出:
# B: 装饰
# A: 装饰

# 调用 my_func() 输出:
# A: 前
# B: 前
# 函数执行
# B: 后
# A: 后

# 等价于: my_func = decorator_a(decorator_b(my_func))
```

### 4. 保留装饰器链

```python
from functools import wraps

def my_decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@my_decorator
def original():
    """原始函数"""
    pass

# 访问原始函数
print(original.__wrapped__)  # <function original at 0x...>
print(original.__wrapped__.__doc__)  # 原始函数
```

## 内置装饰器深入

### 1. @property 实现原理

```python
class Property:
    def __init__(self, fget=None, fset=None, fdel=None):
        self.fget = fget
        self.fset = fset
        self.fdel = fdel
    
    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        if self.fget is None:
            raise AttributeError("unreadable attribute")
        return self.fget(obj)
    
    def __set__(self, obj, value):
        if self.fset is None:
            raise AttributeError("can't set attribute")
        self.fset(obj, value)
    
    def setter(self, fset):
        return type(self)(self.fget, fset, self.fdel)

class Person:
    def __init__(self, name):
        self._name = name
    
    @Property
    def name(self):
        return self._name
    
    @name.setter
    def name(self, value):
        self._name = value
```

### 2. @staticmethod 和 @classmethod 原理

```python
class StaticMethod:
    def __init__(self, func):
        self.func = func
    
    def __get__(self, obj, objtype=None):
        return self.func

class ClassMethod:
    def __init__(self, func):
        self.func = func
    
    def __get__(self, obj, objtype=None):
        if objtype is None:
            objtype = type(obj)
        def wrapper(*args, **kwargs):
            return self.func(objtype, *args, **kwargs)
        return wrapper

class MyClass:
    @StaticMethod
    def static_method():
        print("静态方法")
    
    @ClassMethod
    def class_method(cls):
        print(f"类方法: {cls.__name__}")
```

## 面试高频问题

### Q1: 装饰器和闭包的区别？

**答**：
- **闭包**：内层函数引用外层函数的变量，是一种函数特性
- **装饰器**：利用闭包来包装函数，是一种设计模式
- 装饰器是闭包的应用，但闭包不一定是装饰器

### Q2: 为什么装饰器需要返回函数？

**答**：因为装饰器的目的是**替换原函数**。`@decorator` 等价于 `func = decorator(func)`，所以 decorator 必须返回一个可调用对象来替换原函数。

### Q3: 多个装饰器的执行顺序？

**答**：
- **装饰时**：从下到上（靠近函数的先执行）
- **调用时**：从上到下（最外层的先执行）

```python
@a
@b
@c
def func():
    pass

# 等价于: func = a(b(c(func)))
# 装饰顺序: c → b → a
# 调用顺序: a → b → c → func
```

### Q4: 类装饰器和函数装饰器的区别？

**答**：
- **函数装饰器**：使用闭包，简单直接
- **类装饰器**：使用 `__call__`，可以保存状态，更面向对象

```python
# 函数装饰器
def func_decorator(func):
    def wrapper():
        return func()
    return wrapper

# 类装饰器
class ClassDecorator:
    def __init__(self, func):
        self.func = func
    
    def __call__(self):
        return self.func()
```

### Q5: 如何给类的所有方法添加装饰器？

**答**：使用元类或类装饰器

```python
def method_decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f"调用 {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

def decorate_all_methods(decorator):
    def class_decorator(cls):
        for name, method in cls.__dict__.items():
            if callable(method) and not name.startswith('_'):
                setattr(cls, name, decorator(method))
        return cls
    return class_decorator

@decorate_all_methods(method_decorator)
class MyClass:
    def method1(self):
        print("method1")
    
    def method2(self):
        print("method2")

obj = MyClass()
obj.method1()  # 调用 method1 \n method1
obj.method2()  # 调用 method2 \n method2
```

### Q6: 装饰器如何传递被装饰函数的参数？

**答**：使用 `*args, **kwargs`

```python
def my_decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):  # 接受任意参数
        print(f"参数: {args}, {kwargs}")
        return func(*args, **kwargs)  # 传递给原函数
    return wrapper

@my_decorator
def add(a, b, c=0):
    return a + b + c

add(1, 2, c=3)  # 参数: (1, 2), {'c': 3}
```

## 实战场景

### 1. Web 框架路由装饰器（Flask 风格）

```python
class Flask:
    def __init__(self):
        self.routes = {}
    
    def route(self, path):
        def decorator(func):
            self.routes[path] = func
            return func
        return decorator
    
    def dispatch(self, path):
        handler = self.routes.get(path)
        if handler:
            return handler()
        return "404 Not Found"

app = Flask()

@app.route('/')
def index():
    return "Home Page"

@app.route('/about')
def about():
    return "About Page"

print(app.dispatch('/'))      # Home Page
print(app.dispatch('/about')) # About Page
```

### 2. 异步装饰器

```python
import asyncio
from functools import wraps

def async_timer(func):
    @wraps(func)
    async def wrapper(*args, **kwargs):
        start = asyncio.get_event_loop().time()
        result = await func(*args, **kwargs)
        end = asyncio.get_event_loop().time()
        print(f"{func.__name__} 耗时: {end - start:.4f}秒")
        return result
    return wrapper

@async_timer
async def fetch_data():
    await asyncio.sleep(1)
    return "数据"

asyncio.run(fetch_data())
```

### 3. 参数验证装饰器

```python
from functools import wraps

def validate_types(**type_checks):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # 验证参数类型
            for arg_name, expected_type in type_checks.items():
                if arg_name in kwargs:
                    value = kwargs[arg_name]
                    if not isinstance(value, expected_type):
                        raise TypeError(
                            f"{arg_name} 应该是 {expected_type.__name__}, "
                            f"但得到 {type(value).__name__}"
                        )
            return func(*args, **kwargs)
        return wrapper
    return decorator

@validate_types(name=str, age=int)
def create_user(name, age):
    return f"用户: {name}, 年龄: {age}"

create_user(name="Alice", age=25)  # 成功
# create_user(name="Bob", age="30")  # TypeError
```

## 最佳实践总结

1. **始终使用 `@wraps`**：保留原函数元信息
2. **使用 `*args, **kwargs`**：支持任意参数
3. **返回原函数的返回值**：除非有特殊需求
4. **装饰器命名清晰**：一看就知道功能
5. **避免过度嵌套**：不要超过 3 层装饰器
6. **考虑性能影响**：装饰器会增加函数调用开销
7. **文档化装饰器**：说明装饰器的作用和参数
8. **测试装饰器**：确保装饰器不会破坏原函数功能

## 常见陷阱

### 1. 忘记返回函数

```python
# 错误
def bad_decorator(func):
    def wrapper():
        return func()
    # 忘记 return wrapper

@bad_decorator
def my_func():
    pass

# my_func 变成了 None！
```

### 2. 装饰器参数和函数参数混淆

```python
# 错误：想要带参数的装饰器，但写成了普通装饰器
def repeat(times):  # 这是装饰器参数
    def wrapper(*args):  # 这应该是 func 的参数
        pass
    return wrapper

# 正确：三层结构
def repeat(times):
    def decorator(func):
        def wrapper(*args):
            pass
        return wrapper
    return decorator
```

### 3. 在循环中使用装饰器

```python
# 错误：闭包陷阱
def create_multipliers():
    multipliers = []
    for i in range(3):
        def multiplier(x):
            return x * i  # i 是引用，不是值
        multipliers.append(multiplier)
    return multipliers

funcs = create_multipliers()
print([f(2) for f in funcs])  # [4, 4, 4] 而不是 [0, 2, 4]

# 正确：使用默认参数
def create_multipliers():
    multipliers = []
    for i in range(3):
        def multiplier(x, i=i):  # i=i 捕获当前值
            return x * i
        multipliers.append(multiplier)
    return multipliers
```
