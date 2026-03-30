---
title: 链表算法代码实现 - 归并排序与打印工具
description: 链表归并排序和增强版链表打印函数的Python实现
pubDate: '2025-03-03'
categories:
- Algorithm
tags:
- 链表
- 代码
- Python
---
# 链表算法代码实现

本文包含两个实用的链表工具函数的 Python 实现：链表归并排序和增强版链表打印函数。

---

## 1. 链表归并排序 (Merge Sort for Linked List)

使用分治法对链表进行排序：先找到中间节点并断开链表，递归排序左右两部分，最后合并两个有序链表。

```python
class Solution:
    
    def sortList(self, head):
        if not head or not head.next:
            return head
        
        # 找到中间节点并断开链表
        mid = self.findMiddleAndSplit(head)
        left = self.sortList(head)
        right = self.sortList(mid)
        return self.merge(left, right)
    
    def findMiddleAndSplit(self, head):
        prev = None
        slow = head
        fast = head
        
        while fast and fast.next:
            prev = slow
            slow = slow.next
            fast = fast.next.next
        
        # 断开链表
        if prev:
            prev.next = None
        
        return slow
    
    def merge(self, left, right):
        dummy = ListNode(0)
        tail = dummy
        
        while left and right:
            if left.val <= right.val:
                tail.next = left
                left = left.next
            else:
                tail.next = right
                right = right.next
            tail = tail.next
        
        # 处理剩余节点
        if left:
            tail.next = left
        if right:
            tail.next = right
            
        return dummy.next
```

---

## 2. 增强版链表打印函数

一个支持自定义名称和可选显示链表长度的打印工具函数，方便调试链表问题。

```python
def print_list_advanced(head, name="链表", show_none=False):
    """增强版链表打印函数"""
    print(f"{name}: ", end="")
    
    if head is None:
        if show_none:
            print("None")
        else:
            print("")
        return
    
    values = []
    current = head
    while current:
        values.append(str(current.val))
        current = current.next
    
    print(" → ".join(values))
    
    # 可选：显示链表长度
    if show_none:
        print(f"  长度: {len(values)}")

# 使用示例
# print_list_advanced(head, "原始链表", show_none=True)
# print_list_advanced(None, "空链表", show_none=True)
```
