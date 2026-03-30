---
title: Python垃圾回收机制 - 引用计数、循环引用与分代GC
description: 引用计数、循环引用检测、分代垃圾回收、weakref弱引用
pubDate: '2025-02-13'
categories:
- Python
tags:
- GC
- 内存管理
- weakref
- 面试
---
# Python GC 面试重点

## 第1部分：面试常见问题与答案

### 面试高频问题
```python
# Q1: Python GC 机制是什么？
# 答：混合策略
# - 引用计数：主要机制，实时回收
# - 分代回收：补充机制，处理循环引用

# Q2: 什么是循环引用？如何解决？
# 答：
a.ref = b
b.ref = a
# 解决：
# 1. 弱引用 weakref
# 2. 手动断开引用
# 3. 手动 gc.collect()
# 4. 分代回收 当第 1/2 代达到阈值时，会触发标记-清除：从根对象开始标记所有可达对象，未被标记的（包括循环引用中只相互引用但不再被外部引用的对象）就会在清除阶段被回收。

# Q3: sys.getrefcount() 为什么返回 2？
# 答：因为 sys.getrefcount() 本身也会增加一个引用。

# Q4: 如何优化内存使用？
# 答：
# 1. 及时删除大对象：del large_obj
# 2. 避免循环引用，使用弱引用
# 3. 批量处理，减少对象创建
# 4. 使用生成器减少内存占用

# Q5: GC 的触发时机？
# 答：
# - 引用计数：引用归零立即触发
# - 分代回收：默认阈值 (700, 10, 10)；新对象先进入第 0 代，第 0 代回收 每10 次后触发第 1 代，第 1 代回收 每10 次后触发第 2 代。
# - 手动触发：gc.collect()
```

### 面试准备清单

✅ 能解释引用计数原理  
✅ 能说明循环引用问题  
✅ 能描述分代回收策略  
✅ 能写出弱引用代码  
✅ 知道 GC 触发时机  
✅ 了解内存优化方法  
✅ 能说明分代回收好处  
✅ 理解对象晋升机制  

#### Checklist 答案
1. **引用计数原理**：每个对象维护一个引用计数器；当计数为 0 时立即释放，sys.getrefcount() 读到的数多 1 是因为函数调用也算引用。
2. **循环引用问题**：a.ref = b，b.ref = a，引用计数永远不为 0。用 weakref、手动断开或分代 GC 的标记-清除处理。
3. **分代回收策略**：默认阈值 (700, 10, 10)；新对象先进入第 0 代，第 0 代回收 10 次后触发第 1 代，第 1 代回收 10 次后触发第 2 代。
4. **弱引用代码**：import weakref；weakref.ref(obj) 不增加引用计数，可绕过循环引用。
5. **GC 触发时机**：引用计数归零立即；分代 GC：第 0 代对象达 700 触发，0 代回收 10 次触发第 1 代，依此类推；也可 gc.collect()。
6. **内存优化方法**：及时 del obj，使用弱引用，批量处理减少临时对象，生成器逐步计算。
7. **分代回收好处**：大部分时间只回收第 0 代，降低延迟，按年龄分组减少碎片，策略加性能。
8. **对象晋升机制**：第 0 代幸存晋升第 1 代，第 1 代幸存晋升第 2 代，长期存活对象集中高代，阈值决定晋升频率。

## 第2部分：详解

### 核心机制

### 1. 引用计数（主要机制）
```python
import sys

a = "hello"                    # 引用计数 = 1
b = a                          # 引用计数 = 2
del b                          # 引用计数 = 1
print(sys.getrefcount(a))      # 2 (sys.getrefcount 也算一个引用)

# 引用计数归零，立即回收
del a                          # 立即回收内存
```

**特点**：
- ✅ 实时回收，高效
- ❌ 无法处理循环引用

### 2. 分代回收（补充机制）
```python
import gc

# 查看分代配置
print(gc.get_threshold())     # (700, 10, 10)
# 第 0 代：分配 700 个对象触发
# 第 1 代：第 0 代回收 每10 次触发  
# 第 2 代：第 1 代回收 每10 次触发

# 手动触发 GC
gc.collect()
```

**特点**：
- ✅ 解决循环引用问题
- ✅ 分代优化性能
- ❌ 有一定延迟

## 循环引用问题

### 什么是循环引用
```python
class Node:
    def __init__(self):
        self.ref = None

# 创建循环引用
a = Node()
b = Node()
a.ref = b      # a → b
b.ref = a      # b → a，形成循环引用

del a, b       # 引用计数不为 0，无法回收
gc.collect()   # 需要分代回收处理
```

### 解决方法
```python
import weakref

# 方法1：使用弱引用
a.ref = weakref.ref(b)

# 方法2：手动断开
a.ref = None
b.ref = None

# 方法3：手动触发 GC
gc.collect()
```

## 分代回收原理

### 分代策略
- **第 0 代**：新对象，回收频率最高
- **第 1 代**：存活对象，回收频率中等
- **第 2 代**：老对象，回收频率最低

### 标记-清除算法
```python
# 1. 标记阶段：从根对象开始，标记所有可达对象
# 2. 清除阶段：回收未标记的对象（包括循环引用）

# 触发条件：某代对象数量达到阈值
当第 1/2 代达到阈值时，会触发标记-清除：从根对象开始标记所有可达对象，未被标记的（包括循环引用中只相互引用但不再被外部引用的对象）就会在清除阶段被回收。
gc.collect()  # 手动触发
```

```python
# 阈值：(700, 10, 10)
gen0_threshold = 700      # 第 0 代：700 个对象
gen1_threshold = 10       # 第 0 代回收 每10 次触发第 1 代
gen2_threshold = 10       # 第 1 代回收 每10 次触发第 2 代

def python_gc_loop():
    """伪代码：把 GC 逻辑按代分层执行"""
    stats = gc.get_stats()

    def need_collect_gen0():
        gen0_count, _, _ = gc.get_count()
        return gen0_count >= 700

    def need_collect_gen1():
        return stats[0]['collections'] % 10 ==0 and stats[0]['collections'] > 0

    def need_collect_gen2():
        return stats[1]['collections'] % 10 == 0 and stats[1]['collections'] > 0

    if need_collect_gen2():
        collect_gen0()
        collect_gen1()
        collect_gen2()
        update_counters(['gen0', 'gen1', 'gen2'])
        return "gen0+1+2"
    if need_collect_gen1():
        collect_gen0()
        collect_gen1()
        update_counters(['gen0', 'gen1'])
        return "gen0+1"
    if need_collect_gen0():
        collect_gen0()
        update_counters(['gen0'])
        return "gen0"
    perform_regular_work()

def update_counters(collected_generations):
    """伪代码：GC 执行后更新计数器"""
    stats = gc.get_stats()
    for gen in collected_generations:
        if gen == 'gen0':
            stats[0]['collections'] += 1
        elif gen == 'gen1':
            stats[1]['collections'] += 1
        elif gen == 'gen2':
            stats[2]['collections'] += 1
```

## 实用代码

### 检查内存泄漏
```python
import gc

# 检查不可达对象
unreachable = gc.collect()
print(f"发现 {unreachable} 个不可达对象")

# 查看各代对象数
print(gc.get_count())  # (count0, count1, count2)
```

### GC 配置
```python
import gc

# 修改阈值
gc.set_threshold(1000, 15, 15)

# 启用/禁用
gc.disable()  # 禁用自动 GC
gc.enable()   # 启用自动 GC
```

### 弱引用使用
```python
import weakref

class Parent:
    def __init__(self):
        self._children = []
    
    def add_child(self, child):
        # 使用弱引用避免循环引用
        self._children.append(weakref.ref(child))
```

## 分代回收的好处

### 1. 性能优化
- **减少扫描范围**：大部分时间只扫描第 0 代（几百个对象）
- **基于经验法则**：90% 对象在第 0 代死亡，只扫描最有可能回收的对象
- **性能提升**：相比不分代可提升 100+ 倍

### 2. 降低延迟
- **减少停顿时间**：第 0 代回收只需 1ms，第 2 代需要 100ms
- **用户体验**：大部分 GC 操作很快，不影响程序响应

### 3. 内存效率
- **减少内存碎片**：按年龄分组，内存布局更整齐
- **针对性策略**：不同代用不同回收策略

### 4. 实际效果
```
不分代：每次扫描 100,000 个对象 → 慢
分代：大部分时间扫描 700 个对象 → 快
性能提升：142.9 倍
```

## 并发与 GIL

### GIL 概念
- **GIL（Global Interpreter Lock）**：CPython 实现中保护对象模型的全局互斥锁，任意时刻只有一个线程可以执行 Python 字节码，避免引用计数等内部结构竞争。
- 在执行纯 Python 代码前必须获得 GIL；I/O、C 扩展释放 GIL 后其他线程才能运行。

### 与 Garbage Collection 的协同
- GIL 确保 GC 在回收过程中不会遭遇跨线程的数据竞争，这是引用计数或分代回收修改对象图时的安全保障。
- GC 触发会周期性释放 GIL，允许其他线程插入执行；高频 GC 的第 0 代回收因此对并发影响小，而第 2 代回收由于扫描量大，需要更长时间独占 GIL，因此高代 GC 尽量少触发。

### 并发编程建议
1. **CPU 密集型**：GIL 限制多线程效果，推荐用 `multiprocessing` 或把算法交给释放 GIL 的 C 扩展（如 NumPy、Cython）。
2. **I/O 密集型**：I/O 操作（网络、文件）会自动释放 GIL；可用 `threading` 达到并发效果。
3. **手动释放**：在 C 扩展中使用 `Py_BEGIN_ALLOW_THREADS`/`Py_END_ALLOW_THREADS` 包裹耗时操作，避免长时间握着 GIL。
4. **避免频繁高代 GC**：通过控制对象创建与启用 `gc.set_threshold` 减少第 2 代回收，降低在持有 GIL 状态下的停顿。

### 并发复习要点
- 分代 GC 的标记-清除和引用计数运行在持有 GIL 的上下文中。
- GIL 只在 CPython 中存在，其他解释器（PyPy、Jython）使用不同内存模型。
- 复述 CPU Vs I/O 的区别以及 GIL 的释放策略，有助于面试区分多线程与多进程。

## 记忆要点

### GC 两大机制
1. **引用计数**：实时、高效、有循环引用问题
2. **分代回收**：延迟、解决循环引用、分代优化

### 循环引用三要素
- 对象相互引用
- 引用计数永不为 0
- 需要分代回收处理

### 分代回收四好处
1. **性能提升**：大部分时间只扫描第 0 代
2. **降低延迟**：减少单次 GC 停顿时间
3. **内存效率**：按年龄分组减少碎片
4. **针对性策略**：不同代用不同回收方法

### 优化内存四方法
1. 及时删除
2. 弱引用
3. 批量处理
4. 生成器


---

## 核心总结

### **GC 机制一句话**
引用计数实时收，循环引用分代补，性能优化靠分代，弱引用解循环。

### **分代回收核心**
- **第 0 代**：新对象，高频回收（700 个触发）
- **第 1 代**：幸存对象，中频回收（10 次 0 代触发）
- **第 2 代**：老对象，低频回收（10 次 1 代触发）

### **性能提升关键**
大部分时间只扫描第 0 代，性能提升 100+ 倍。

### **面试必答**
1. **两大机制**：引用计数 + 分代回收
2. **循环引用**：问题 + 弱引用解决
3. **分代好处**：性能 + 延迟 + 内存 + 策略
