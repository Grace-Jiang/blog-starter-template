---
title: 链表五大模板 - 解决90%的链表问题
description: 5个通用链表模板：哨兵节点、快慢指针、迭代反转、合并模式和递归思维
pubDate: '2025-03-02'
categories:
- Algorithm
tags:
- 链表
- 模板
- 面试
---
# 链表 5 大万能模板 - 90% 题目通用

> 刷 100 道链表题总结出的核心模板

---

## 模板 1: 哨兵节点 (Dummy Node) - 处理头节点变化

**适用场景**: 删除节点、合并链表、插入节点、头节点可能改变的所有情况

**核心思想**: 创建虚拟头节点，统一处理头节点和其他节点

### 模板代码

```python
def solution(head):
    # 1. 创建哨兵节点
    dummy = ListNode(0)
    dummy.next = head
    
    # 2. 从 dummy 开始操作
    curr = dummy
    
    # 3. 遍历处理
    while curr and curr.next:
        # 你的逻辑
        curr = curr.next
    
    # 4. 返回 dummy.next（新的头节点）
    return dummy.next
```

### 典型题目

#### 1. 删除链表节点
```python
def delete_node(head, val):
    dummy = ListNode(0)
    dummy.next = head
    curr = dummy
    
    while curr.next:
        if curr.next.val == val:
            curr.next = curr.next.next  # 删除
        else:
            curr = curr.next
    
    return dummy.next
```

#### 2. 合并两个有序链表
```python
def merge_two_lists(l1, l2):
    dummy = ListNode(0)
    curr = dummy
    
    while l1 and l2:
        if l1.val <= l2.val:
            curr.next = l1
            l1 = l1.next
        else:
            curr.next = l2
            l2 = l2.next
        curr = curr.next
    
    curr.next = l1 if l1 else l2
    return dummy.next
```

#### 3. 删除倒数第 N 个节点
```python
def remove_nth_from_end(head, n):
    dummy = ListNode(0, head)
    fast = slow = dummy
    
    # fast 先走 n+1 步
    for _ in range(n + 1):
        fast = fast.next
    
    # 同时移动
    while fast:
        fast = fast.next
        slow = slow.next
    
    # 删除
    slow.next = slow.next.next
    return dummy.next
```

**关键点**:
- ✅ 避免特殊处理头节点
- ✅ 简化边界条件
- ✅ 统一删除/插入逻辑

---

## 模板 2: 快慢指针 (Two Pointers) - 找中点、检测环

**适用场景**: 找中点、检测环、找倒数第 K 个、判断回文

**核心思想**: fast 走 2 步，slow 走 1 步

### 模板代码

```python
def two_pointers(head):
    # 1. 初始化快慢指针
    slow = fast = head
    
    # 2. 快指针走 2 步，慢指针走 1 步
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        
        # 3. 在这里判断条件
        if slow == fast:  # 例如：检测环
            return True
    
    # 4. 循环结束，slow 在中点
    return slow
```

### 典型题目

#### 1. 找链表中点
```python
def find_middle(head):
    slow = fast = head
    
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
    
    return slow  # slow 就是中点
```

#### 2. 检测环
```python
def has_cycle(head):
    if not head or not head.next:
        return False
    
    slow = fast = head
    
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        
        if slow == fast:
            return True
    
    return False
```

#### 3. 找环的起点
```python
def detect_cycle(head):
    slow = fast = head
    
    # 第一次相遇
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow == fast:
            break
    
    if not fast or not fast.next:
        return None
    
    # 找起点：slow 回到 head，同步走
    slow = head
    while slow != fast:
        slow = slow.next
        fast = fast.next
    
    return slow
```

#### 4. 判断回文链表
```python
def is_palindrome(head):
    # 1. 找中点
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
    
    # 2. 反转后半部分
    prev = None
    while slow:
        next_temp = slow.next
        slow.next = prev
        prev = slow
        slow = next_temp
    
    # 3. 比较前后两部分
    left, right = head, prev
    while right:
        if left.val != right.val:
            return False
        left = left.next
        right = right.next
    
    return True
```

**关键点**:
- ✅ fast 走完时，slow 在中点
- ✅ 相遇点到起点 = 头到起点（环问题）
- ✅ 找中点后可以分割链表

---

## 模板 3: 三指针反转 (Reverse) - 反转链表

**适用场景**: 反转整个链表、反转部分链表、K 个一组反转

**核心思想**: prev, curr, next 三个指针协作

### 模板代码

```python
def reverse(head):
    # 1. 初始化三指针
    prev = None
    curr = head
    
    # 2. 遍历反转
    while curr:
        next_temp = curr.next  # 保存下一个
        curr.next = prev       # 反转指针
        prev = curr            # prev 前进
        curr = next_temp       # curr 前进
    
    # 3. prev 是新头节点
    return prev
```

### 典型题目

#### 1. 反转整个链表
```python
def reverse_list(head):
    prev = None
    curr = head
    
    while curr:
        next_temp = curr.next
        curr.next = prev
        prev = curr
        curr = next_temp
    
    return prev
```

#### 2. 反转区间 [left, right]
```python
def reverse_between(head, left, right):
    dummy = ListNode(0)
    dummy.next = head
    pre = dummy
    
    # 1. 找到 left 前一个节点
    for _ in range(left - 1):
        pre = pre.next
    
    # 2. 反转 [left, right]
    curr = pre.next
    for _ in range(right - left):
        next_temp = curr.next
        curr.next = next_temp.next
        next_temp.next = pre.next
        pre.next = next_temp
    
    return dummy.next
```

#### 3. K 个一组反转
```python
def reverse_k_group(head, k):
    dummy = ListNode(0)
    dummy.next = head
    pre = dummy
    
    while True:
        # 检查剩余节点是否够 k 个
        tail = pre
        for _ in range(k):
            tail = tail.next
            if not tail:
                return dummy.next
        
        # 反转 k 个节点
        prev = None
        curr = pre.next
        for _ in range(k):
            next_temp = curr.next
            curr.next = prev
            prev = curr
            curr = next_temp
        
        # 连接
        head_node = pre.next
        pre.next = prev
        head_node.next = curr
        pre = head_node
    
    return dummy.next
```

**关键点**:
- ✅ 三指针：prev, curr, next
- ✅ 反转指针：`curr.next = prev`
- ✅ 前进：`prev = curr, curr = next`

---

## 模板 4: 递归 (Recursion) - 从后往前处理

**适用场景**: 反转链表、合并链表、删除节点、树形结构

**核心思想**: 先递归到底，再从后往前处理

### 模板代码

```python
def recursive(head):
    # 1. 递归终止条件
    if not head or not head.next:
        return head
    
    # 2. 递归处理子问题
    new_head = recursive(head.next)
    
    # 3. 处理当前节点
    # 你的逻辑
    
    # 4. 返回结果
    return new_head
```

### 典型题目

#### 1. 递归反转链表
```python
def reverse_list_recursive(head):
    # 终止条件
    if not head or not head.next:
        return head
    
    # 递归反转后面的链表
    new_head = reverse_list_recursive(head.next)
    
    # 处理当前节点
    head.next.next = head
    head.next = None
    
    return new_head
```

#### 2. 两两交换节点
```python
def swap_pairs(head):
    # 终止条件
    if not head or not head.next:
        return head
    
    # 递归处理后面的节点
    new_head = head.next
    head.next = swap_pairs(new_head.next)
    new_head.next = head
    
    return new_head
```

#### 3. 合并 K 个有序链表（分治）
```python
def merge_k_lists(lists):
    if not lists:
        return None
    if len(lists) == 1:
        return lists[0]
    
    # 分治
    mid = len(lists) // 2
    left = merge_k_lists(lists[:mid])
    right = merge_k_lists(lists[mid:])
    
    # 合并
    return merge_two_lists(left, right)
```

**关键点**:
- ✅ 明确递归终止条件
- ✅ 递归处理子问题
- ✅ 处理当前节点与子问题的关系

---

## 模板 5: 双链表遍历 (Two Lists) - 处理两个链表

**适用场景**: 合并链表、相交链表、相加链表

**核心思想**: 同时遍历两个链表，处理不同长度

### 模板代码

```python
def two_lists(l1, l2):
    # 1. 初始化指针
    p1, p2 = l1, l2
    
    # 2. 同时遍历
    while p1 or p2:
        # 处理 p1
        if p1:
            # 你的逻辑
            p1 = p1.next
        
        # 处理 p2
        if p2:
            # 你的逻辑
            p2 = p2.next
    
    return result
```

### 典型题目

#### 1. 相加两数（链表表示）
```python
def add_two_numbers(l1, l2):
    dummy = ListNode(0)
    curr = dummy
    carry = 0
    
    while l1 or l2 or carry:
        val1 = l1.val if l1 else 0
        val2 = l2.val if l2 else 0
        
        total = val1 + val2 + carry
        carry = total // 10
        curr.next = ListNode(total % 10)
        
        curr = curr.next
        if l1: l1 = l1.next
        if l2: l2 = l2.next
    
    return dummy.next
```

#### 2. 相交链表
```python
def get_intersection_node(headA, headB):
    if not headA or not headB:
        return None
    
    p1, p2 = headA, headB
    
    # 两个指针走完两个链表
    while p1 != p2:
        p1 = p1.next if p1 else headB
        p2 = p2.next if p2 else headA
    
    return p1  # 相交点或 None
```

#### 3. 合并 K 个有序链表（优先队列）
```python
import heapq

def merge_k_lists_heap(lists):
    heap = []
    
    # 初始化堆
    for i, node in enumerate(lists):
        if node:
            heapq.heappush(heap, (node.val, i, node))
    
    dummy = ListNode(0)
    curr = dummy
    
    while heap:
        val, i, node = heapq.heappop(heap)
        curr.next = node
        curr = curr.next
        
        if node.next:
            heapq.heappush(heap, (node.next.val, i, node.next))
    
    return dummy.next
```

**关键点**:
- ✅ 处理不同长度：`while p1 or p2`
- ✅ 处理进位/余数
- ✅ 双指针技巧：走完自己走对方

---

## 模板组合使用

### 1. 排序链表（归并排序）
**组合**: 快慢指针（找中点） + 递归（分治） + 双链表（合并）

```python
def sort_list(head):
    # 终止条件
    if not head or not head.next:
        return head
    
    # 快慢指针找中点
    slow, fast = head, head.next
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
    
    # 分割
    mid = slow.next
    slow.next = None
    
    # 递归排序
    left = sort_list(head)
    right = sort_list(mid)
    
    # 合并
    return merge_two_lists(left, right)
```

### 2. 重排链表
**组合**: 快慢指针（找中点） + 三指针（反转） + 双链表（合并）

```python
def reorder_list(head):
    if not head or not head.next:
        return
    
    # 1. 找中点
    slow, fast = head, head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
    
    # 2. 反转后半部分
    prev, curr = None, slow.next
    slow.next = None
    while curr:
        next_temp = curr.next
        curr.next = prev
        prev = curr
        curr = next_temp
    
    # 3. 合并两部分
    first, second = head, prev
    while second:
        tmp1, tmp2 = first.next, second.next
        first.next = second
        second.next = tmp1
        first, second = tmp1, tmp2
```

---

## 5 大模板总结表

| 模板 | 核心结构 | 适用场景 | 关键技巧 |
|------|---------|---------|---------|
| **哨兵节点** | `dummy = ListNode(0)` | 删除、插入、头节点变化 | 统一处理头节点 |
| **快慢指针** | `slow, fast = head, head` | 找中点、检测环、倒数第K | fast 走2步，slow走1步 |
| **三指针反转** | `prev, curr, next` | 反转链表、K组反转 | `curr.next = prev` |
| **递归** | `recursive(head.next)` | 从后往前、分治 | 终止条件 + 子问题 |
| **双链表** | `p1, p2 = l1, l2` | 合并、相交、相加 | 同时遍历 + 处理长度差 |

---

## 刷题策略

### 第一阶段：掌握 5 大模板（10 题）
1. 删除节点 → 哨兵节点
2. 反转链表 → 三指针
3. 找中点 → 快慢指针
4. 检测环 → 快慢指针
5. 合并两个链表 → 哨兵节点 + 双链表
6. 删除倒数第N个 → 哨兵节点 + 快慢指针
7. 回文链表 → 快慢指针 + 反转
8. 递归反转 → 递归
9. 相交链表 → 双链表
10. 两数相加 → 双链表

### 第二阶段：组合使用（10 题）
11. 排序链表 → 快慢指针 + 递归 + 合并
12. 重排链表 → 快慢指针 + 反转 + 合并
13. K组反转 → 哨兵节点 + 反转
14. 反转区间 → 哨兵节点 + 反转
15. 环的起点 → 快慢指针
16. 两两交换 → 递归
17. 奇偶链表 → 双指针
18. 分隔链表 → 哨兵节点
19. 合并K个链表 → 递归 + 合并
20. 复制带随机指针 → 哈希表 + 遍历

### 第三阶段：综合应用（10+ 题）
- LRU 缓存：双向链表 + 哈希表
- 扁平化多级链表：递归
- 链表随机节点：蓄水池抽样
- 删除重复节点：哨兵节点
- 旋转链表：快慢指针 + 连接

---

## 调试技巧

### 1. 打印链表
```python
def print_list(head):
    vals = []
    while head:
        vals.append(str(head.val))
        head = head.next
    print(" -> ".join(vals))
```

### 2. 检查环
```python
def check_cycle(head):
    visited = set()
    while head:
        if head in visited:
            return True
        visited.add(head)
        head = head.next
    return False
```

### 3. 边界测试
```python
# 空链表
test_cases = [
    None,           # 空
    [1],            # 单节点
    [1, 2],         # 两节点
    [1, 2, 3],      # 多节点
    [1, 1, 1],      # 重复
]
```

---

## 常见错误

### ❌ 错误 1: 忘记哨兵节点
```python
# 错误：删除头节点需要特殊处理
def delete(head, val):
    if head.val == val:
        return head.next  # 特殊处理
    # ...

# 正确：使用哨兵节点
def delete(head, val):
    dummy = ListNode(0)
    dummy.next = head
    # 统一处理
```

### ❌ 错误 2: 快慢指针初始化错误
```python
# 错误：可能导致空指针
slow, fast = head, head

# 正确：检查空链表
if not head or not head.next:
    return head
slow, fast = head, head
```

### ❌ 错误 3: 反转后忘记断开
```python
# 错误：形成环
curr.next = prev

# 正确：最后断开
curr.next = prev
# 最后要 head.next = None
```

---

**总结**: 这 5 个模板覆盖了 90% 的链表题，关键是：
1. **哨兵节点**：处理头节点
2. **快慢指针**：找中点、检测环
3. **三指针**：反转链表
4. **递归**：从后往前
5. **双链表**：同时遍历

掌握这 5 个模板，链表题基本无敌！

---

**文档版本**: 1.0  
**最后更新**: 2026-03-06  
**适用场景**: LeetCode 链表题刷题
