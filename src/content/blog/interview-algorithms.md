---
title: 算法面试全攻略 - 排序、搜索、动态规划
description: 全面的算法面试准备指南，涵盖排序、搜索、链表、树、动态规划、滑动窗口等核心算法
pubDate: '2025-03-01'
categories:
- Algorithm
tags:
- 排序
- 搜索
- 动态规划
- 面试
---
# 算法面试必备题目总结

> 关键词：排序、查找、链表、树、动态规划

---

## 1. 排序算法 (Sorting)

### 1.1 快速排序 (Quick Sort)

**时间复杂度**: O(n log n) 平均，O(n²) 最坏  
**空间复杂度**: O(log n) 递归栈  
**稳定性**: 不稳定
pivot始终在原位置不动
先将所有小元素集中到左边
最后一次性将pivot放到正确位置

```python
def quick_sort(arr, left, right):
    if left >= right:
        return
    
    # Partition
    pivot = arr[right]
    i = left  //i = left	0	指向下一个小元素应放位置	先交换再i+=1
    
    for j in range(left, right):
        if arr[j] <= pivot:
            
            arr[i], arr[j] = arr[j], arr[i]
            i += 1
    
    arr[i], arr[right] = arr[right], arr[i]  //不能直接和pivot交换，因为pivot是局部变量
    # pivot 是一个局部变量，存储的是 nums[right] 的值
    # 这个交换只是在 nums[i] 和变量 pivot 之间交换
    # nums[right] 数组中的元素根本没有被修改
    pivot_index = i 
    
    # 递归排序
    quick_sort(arr, left, pivot_index - 1)
    quick_sort(arr, pivot_index + 1, right)
# 使用
arr = [3, 6, 8, 10, 1, 2, 1]
quick_sort(arr, 0, len(arr) - 1)
```

**面试要点**:
- Partition 过程：选择 pivot，将小于 pivot 的放左边，大于的放右边
- 优化：三数取中选 pivot，避免最坏情况
- 原地排序，不需要额外空间

---

### 1.2 归并排序 (Merge Sort)

**时间复杂度**: O(n log n)  
**空间复杂度**: O(n)  
**稳定性**: 稳定

```python
def merge_sort(arr):
    if len(arr) <= 1:
        return arr
    
    # 分割
    mid = len(arr) // 2
    left = merge_sort(arr[:mid])
    right = merge_sort(arr[mid:])
    
    # 合并
    return merge(left, right)

def merge(left, right):
    result = []
    i = j = 0
    
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            result.append(left[i])
            i += 1
        else:
            result.append(right[j])
            j += 1
    
    result.extend(left[i:])
    result.extend(right[j:])
    return result
```

**面试要点**:
- 分治思想：分割 → 递归排序 → 合并
- 稳定排序，适合链表排序
- 需要额外 O(n) 空间

**merge list**： 找中点（快慢指针，fast初始为head.next），然后断开链表，递归排序左右两部分，合并两个有序链表
---

### 1.3 堆排序 (Heap Sort)

大顶堆：父节点的值总是大于或等于它的子节点的值
小顶堆：父节点的值总是小于或等于它的子节点的值

**时间复杂度**: O(n log n)  
**空间复杂度**: O(1)  原地排序
**稳定性**: 不稳定

```python
def heap_sort(arr):
    n = len(arr)
    
    # 建堆，从n//2-1开始，因为n//2到n-1的节点都是叶子节点,往前遍历
    for i in range(n // 2 - 1, -1, -1):   # range(start, stop, step)    >=start, <stop, step
        heapify(arr, n, i)
    
    # 排序
    for i in range(n - 1, 0, -1):
        arr[0], arr[i] = arr[i], arr[0]  # 将堆顶（最大值）与末尾交换
        heapify(arr, i, 0)   # i 是当前堆的大小, 0 是根节点索引,除了堆顶，剩下的元素依然是大顶堆,所以只需要从根节点开始下沉

def heapify(arr, n, i):
    largest = i
    left = 2 * i + 1
    right = 2 * i + 2
    
    if left < n and arr[left] > arr[largest]:
        largest = left
    if right < n and arr[right] > arr[largest]:
        largest = right
    
    if largest != i:
        arr[i], arr[largest] = arr[largest], arr[i]
        heapify(arr, n, largest)
```

**面试要点**:
- 大顶堆：父节点 ≥ 子节点
- 原地排序，不需要额外空间
- 不稳定，但时间复杂度稳定

---

### 1.4 两个无序数组合并排序

**思路**: 分别排序，再归并合并  
**时间复杂度**: O(m log m + n log n)  
**空间复杂度**: O(m + n)

```python
def merge_two_unsorted(arr1, arr2):
    arr1.sort()
    arr2.sort()
    
    result = []
    i = j = 0
    
    while i < len(arr1) and j < len(arr2):
        if arr1[i] <= arr2[j]:
            result.append(arr1[i])
            i += 1
        else:
            result.append(arr2[j])
            j += 1
    
    result.extend(arr1[i:])
    result.extend(arr2[j:])
    return result
```

**面试要点**:
- 先各自排序，再利用归并合并两个有序数组，比直接合并后排序更高效
- 如果其中一个数组已经有序，只需排另一个，再归并
- 如果数据量巨大内存放不下，用外部排序：分块排序 + 多路归并（最小堆）

---

## 2. 查找算法 (Search)

### 2.1 二分查找 (Binary Search)

**时间复杂度**: O(log n)  
**空间复杂度**: O(1)

```python
def binary_search(arr, target):
    left, right = 0, len(arr) - 1
    
    while left <= right:
        mid = left + (right - left) // 2
        
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    
    return -1
```

**面试要点**:
- 边界条件：`left <= right` vs `left < right`
区间约定       条件                  缩减方式
[left, right]  left <= right         left = mid+1, right = mid-1
[left, right)  left < right          left = mid+1, right = mid

- 防溢出：`mid = left + (right - left) // 2`
- 变种：查找第一个/最后一个等于目标的位置

**变种 1: 查找第一个等于目标的位置**
```python
def binary_search_first(arr, target):
    left, right = 0, len(arr) - 1
    result = -1
    
    while left <= right:
        mid = left + (right - left) // 2
        
        if arr[mid] == target:
            result = mid
            right = mid - 1  # 继续向左找  如果是查找最后一个等于目标的位置，这里应该是 left = mid + 1
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    
    return result
```

**变种 2: 旋转数组中的查找**
```python
def search_rotated(arr, target):
    left, right = 0, len(arr) - 1
    
    while left <= right:
        mid = left + (right - left) // 2
        
        if arr[mid] == target:
            return mid
        
        # 判断哪一半是有序的
        if arr[left] <= arr[mid]:  # 左半有序
            if arr[left] <= target < arr[mid]:
                right = mid - 1
            else:
                left = mid + 1
        else:  # 右半有序
            if arr[mid] < target <= arr[right]:
                left = mid + 1
            else:
                right = mid - 1
    
    return -1
```

---

## 3. 链表 (Linked List) - 5 大万能模板

> LeetCode 90% 链表题都在这几个套路里

---

### 3.1 模板 1: Dummy + Tail（构造新链表）

**适用场景**: 合并链表、过滤节点、分区链表、构造新链表

**核心思想**: 永远在 tail 后面插入

```python
def template_dummy_tail():
    dummy = ListNode(0)
    tail = dummy
    
    while condition:
        tail.next = node
        tail = tail.next
    
    return dummy.next
```

#### 例题 1: 合并两个有序链表
```python
def merge_two_lists(list1, list2):
    dummy = ListNode()
    tail = dummy
    
    while list1 and list2:
        if list1.val < list2.val:
            tail.next = list1
            list1 = list1.next
        else:
            tail.next = list2
            list2 = list2.next
        
        tail = tail.next

    tail.next = list1 or list2
    
    return dummy.next
```

#### 例题 2: Partition List
```python
def partition(head, x):
    dummy = ListNode(0)
    tail = dummy
    
    while head:
        if head.val < x:
            tail.next = head
            tail = tail.next
        head = head.next
    
    tail.next = None
    return dummy.next
```

---

### 3.2 模板 2: 快慢指针（Fast & Slow Pointer）

**适用场景**: 找链表中点、判断环、找倒数第 k 个、删除倒数第 k 个

**核心思想**: fast 速度是 slow 的 2 倍

```python
def template_fast_slow():
    slow = head
    fast = head
    
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
```

#### 例题 1: 找链表中点
```python
def middle_of_list(head):
    slow = fast = head
    
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
    
    return slow
```

#### 例题 2: 判断是否有环
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

#### 例题 3: 删除倒数第 N 个节点
```python
def remove_nth_from_end(head, n):
    dummy = ListNode(0, head)
    
    slow = dummy
    fast = dummy
    
    for _ in range(n):
        fast = fast.next
    
    while fast.next:
        slow = slow.next
        fast = fast.next
    
    slow.next = slow.next.next
    return dummy.next
```

---

### 3.3 模板 3: 反转链表（Reverse List）

**适用场景**: 反转链表、区间反转、K组反转、判断回文

**核心思想**: prev ← curr → next 的指针变换

```python
def template_reverse():
    prev = None
    curr = head
    
    while curr:
        nxt = curr.next
        curr.next = prev
        prev = curr
        curr = nxt
    
    return prev
```

#### 例题 1: 反转整个链表
```python
def reverse_list(head):
    prev = None
    curr = head
    
    while curr:
        nxt = curr.next   # 存 next
        curr.next = prev  # 断 curr
        prev = curr       # 移 prev
        curr = nxt        # 移 curr
    
    return prev
```

#### 例题 2: 反转区间 [m,n]
```python
def reverse_between(head, left, right):
    dummy = ListNode(0)
    dummy.next = head
    pre = dummy
    
    for _ in range(left - 1):
        pre = pre.next
    
    curr = pre.next
    for _ in range(right - left):
        nxt = curr.next
        curr.next = nxt.next
        nxt.next = pre.next
        pre.next = nxt
    
    return dummy.next
```

#### 例题 3: K 个一组反转
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
        curr = pre.next
        for _ in range(k - 1):
            nxt = curr.next
            curr.next = nxt.next
            nxt.next = pre.next
            pre.next = nxt
        
        pre = curr
    
    return dummy.next
```

---

### 3.4 模板 4: 双链表拆分（Two Dummy）

**适用场景**: 分区链表、奇偶链表、按条件拆分

**核心思想**: list → small list + big list，最后拼起来

```python
def template_two_dummy():
    small_dummy = ListNode()
    big_dummy = ListNode()
    
    small = small_dummy
    big = big_dummy
    
    while head:
        if condition:
            small.next = head
            small = small.next
        else:
            big.next = head
            big = big.next
        head = head.next
    
    big.next = None
    small.next = big_dummy.next
    return small_dummy.next
```

#### 例题 1: Partition List
```python
def partition(head, x):
    small_dummy = ListNode()
    big_dummy = ListNode()
    
    small = small_dummy
    big = big_dummy
    
    while head:
        if head.val < x:
            small.next = head
            small = small.next
        else:
            big.next = head
            big = big.next
        head = head.next
    
    big.next = None
    small.next = big_dummy.next
    return small_dummy.next
```

#### 例题 2: 奇偶链表
```python
def odd_even_list(head):
    if not head or not head.next:
        return head
    
    odd_dummy = ListNode()
    even_dummy = ListNode()
    
    odd = odd_dummy
    even = even_dummy
    
    index = 1
    while head:
        if index % 2 == 1:
            odd.next = head
            odd = odd.next
        else:
            even.next = head
            even = even.next
        head = head.next
        index += 1
    
    even.next = None
    odd.next = even_dummy.next
    return odd_dummy.next
```

---

### 3.5 模板 5: 递归链表

**适用场景**: merge、reverse、delete

**核心思想**: 当前节点 + 子问题

```python
def template_recursive():
    if not head:
        return None
    
    # 处理当前节点
    result = solve_subproblem(head.next)
    return result
```

#### 例题 1: 递归合并两个有序链表
```python
def merge_two_lists(l1, l2):
    if not l1:
        return l2
    if not l2:
        return l1
    
    if l1.val < l2.val:
        l1.next = merge_two_lists(l1.next, l2)
        return l1
    else:
        l2.next = merge_two_lists(l1, l2.next)
        return l2
```

#### 例题 2: 递归反转链表
```python
def reverse_list_recursive(head):
    if not head or not head.next:
        return head
    
    new_head = reverse_list_recursive(head.next)
    head.next.next = head
    head.next = None
    
    return new_head
```

#### 例题 3: 两两交换节点
```python
def swap_pairs(head):
    if not head or not head.next:
        return head
    
    new_head = head.next
    head.next = swap_pairs(new_head.next)
    new_head.next = head
    
    return new_head
```

---

### 3.6 链表题核心技巧总结

#### 三种基本操作
1. **改 next 指针**
2. **找位置**
3. **拼接**

#### 四个常用变量
```python
prev    # 前一个节点
curr    # 当前节点
next    # 下一个节点
dummy   # 哨兵节点
```

#### 面试官最爱考的 8 道题
1. **Reverse Linked List** - 反转链表
2. **Merge Two Sorted Lists** - 合并两个有序链表
3. **Remove Nth Node From End** - 删除倒数第N个
4. **Linked List Cycle** - 判断环
5. **Middle of Linked List** - 找中点
6. **Reverse Linked List II** - 反转区间
7. **Palindrome Linked List** - 判断回文
8. **Merge K Sorted Lists** - 合并K个有序链表

---

### 3.7 模板组合使用

#### 排序链表（归并排序）
**组合**: 快慢指针（找中点） + 递归（分治） + Dummy+Tail（合并）

```python
def sort_list(head):
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

#### 重排链表
**组合**: 快慢指针（找中点） + 反转链表 + 双链表（合并）

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
        nxt = curr.next
        curr.next = prev
        prev = curr
        curr = nxt
    
    # 3. 合并两部分
    first, second = head, prev
    while second:
        tmp1, tmp2 = first.next, second.next
        first.next = second
        second.next = tmp1
        first, second = tmp1, tmp2
```

#### 判断回文链表
**组合**: 快慢指针（找中点） + 反转链表 + 比较

```python
def is_palindrome(head):
    if not head or not head.next:
        return True
    
    # 1. 找中点
    slow, fast = head, head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
    
    # 2. 反转后半部分
    prev = None
    curr = slow
    while curr:
        nxt = curr.next
        curr.next = prev
        prev = curr
        curr = nxt
    
    # 3. 比较前后两部分
    left, right = head, prev
    while right:
        if left.val != right.val:
            return False
        left = left.next
        right = right.next
    
    return True
```

---

## 4. 树 (Tree)

### 4.1 二叉树遍历

**前序遍历 (Pre-order)**: 根 → 左 → 右

```python
# 递归
def preorder_recursive(root):
    if not root:
        return []
    return [root.val] + preorder_recursive(root.left) + preorder_recursive(root.right)

# 迭代
def preorder_iterative(root):
    if not root:
        return []
    
    result = []
    stack = [root]
    
    while stack:
        node = stack.pop()
        result.append(node.val)
        
        if node.right:
            stack.append(node.right)
        if node.left:
            stack.append(node.left)
    
    return result
```

**中序遍历 (In-order)**: 左 → 根 → 右

```python
# 递归
def inorder_recursive(root):
    if not root:
        return []
    return inorder_recursive(root.left) + [root.val] + inorder_recursive(root.right)

# 迭代
def inorder_iterative(root):
    result = []
    stack = []
    curr = root
    
    while curr or stack:
        while curr:
            stack.append(curr)
            curr = curr.left
        
        curr = stack.pop()
        result.append(curr.val)
        curr = curr.right
    
    return result
```

**后序遍历 (Post-order)**: 左 → 右 → 根

```python
# 递归
def postorder_recursive(root):
    if not root:
        return []
    return postorder_recursive(root.left) + postorder_recursive(root.right) + [root.val]

# 迭代
def postorder_iterative(root):
    if not root:
        return []
    
    result = []
    stack = [root]
    
    while stack:
        node = stack.pop()
        result.append(node.val)
        
        if node.left:
            stack.append(node.left)
        if node.right:
            stack.append(node.right)
    
    return result[::-1]  # 反转
```

**层序遍历 (Level-order)**: BFS

```python
from collections import deque

def level_order(root):
    if not root:
        return []
    
    result = []
    queue = deque([root])
    
    while queue:
        level_size = len(queue)
        level = []
        
        for _ in range(level_size):
            node = queue.popleft()
            level.append(node.val)
            
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)
        
        result.append(level)
    
    return result
```

**面试要点**:
- 递归：简洁但有栈溢出风险
- 迭代：使用栈模拟递归
- 层序：使用队列 BFS

---

### 4.2 二叉搜索树 (BST)

**查找**:
```python
def search_bst(root, val):
    if not root or root.val == val:
        return root
    
    if val < root.val:
        return search_bst(root.left, val)
    else:
        return search_bst(root.right, val)
```

**插入**:
```python
def insert_bst(root, val):
    if not root:
        return TreeNode(val)
    
    if val < root.val:
        root.left = insert_bst(root.left, val)
    else:
        root.right = insert_bst(root.right, val)
    
    return root
```

**验证 BST**:
```python
def is_valid_bst(root):
    def validate(node, low=float('-inf'), high=float('inf')):
        if not node:
            return True
        
        if node.val <= low or node.val >= high:
            return False
        
        return (validate(node.left, low, node.val) and
                validate(node.right, node.val, high))
    
    return validate(root)
```

**面试要点**:
- BST 性质：左子树 < 根 < 右子树
- 中序遍历 BST 得到有序序列
- 验证时注意传递上下界

---

### 4.3 最近公共祖先 (LCA)

**二叉树 LCA**:
```python
def lowest_common_ancestor(root, p, q):
    if not root or root == p or root == q:
        return root
    
    left = lowest_common_ancestor(root.left, p, q)
    right = lowest_common_ancestor(root.right, p, q)
    
    if left and right:
        return root
    
    return left if left else right
```

**BST LCA**:
```python
def lowest_common_ancestor_bst(root, p, q):
    while root:
        if p.val < root.val and q.val < root.val:
            root = root.left
        elif p.val > root.val and q.val > root.val:
            root = root.right
        else:
            return root
```

**面试要点**:
- 递归思路：在左右子树中查找
- BST 优化：利用有序性质
- 边界：节点本身是祖先

---

### 4.4 路径和问题

**路径总和**:
```python
def has_path_sum(root, target_sum):
    if not root:
        return False
    
    if not root.left and not root.right:
        return root.val == target_sum
    
    return (has_path_sum(root.left, target_sum - root.val) or
            has_path_sum(root.right, target_sum - root.val))
```

**所有路径和**:
```python
def path_sum(root, target_sum):
    def dfs(node, current_sum, path, result):
        if not node:
            return
        
        path.append(node.val)
        current_sum += node.val
        
        if not node.left and not node.right and current_sum == target_sum:
            result.append(path[:])
        
        dfs(node.left, current_sum, path, result)
        dfs(node.right, current_sum, path, result)
        
        path.pop()
    
    result = []
    dfs(root, 0, [], result)
    return result
```

**面试要点**:
- DFS + 回溯
- 叶子节点判断
- 路径记录与恢复

---

## 5. 动态规划 (Dynamic Programming)

### 5.1 斐波那契数列

**时间复杂度**: O(n)  
**空间复杂度**: O(1) 优化版

```python
# 递归 (指数时间)
def fib_recursive(n):
    if n <= 1:
        return n
    return fib_recursive(n - 1) + fib_recursive(n - 2)

# DP (线性时间)
def fib_dp(n):
    if n <= 1:
        return n
    
    dp = [0] * (n + 1)
    dp[1] = 1
    
    for i in range(2, n + 1):
        dp[i] = dp[i - 1] + dp[i - 2]
    
    return dp[n]

# 空间优化
def fib_optimized(n):
    if n <= 1:
        return n
    
    prev, curr = 0, 1
    
    for _ in range(2, n + 1):
        prev, curr = curr, prev + curr
    
    return curr
```

**面试要点**:
- 状态定义：`dp[i]` = 第 i 个斐波那契数
- 状态转移：`dp[i] = dp[i-1] + dp[i-2]`
- 空间优化：只需保存前两个状态

---

### 5.2 爬楼梯

**时间复杂度**: O(n)  
**空间复杂度**: O(1)

```python
def climb_stairs(n):
    if n <= 2:
        return n
    
    prev, curr = 1, 2
    
    for _ in range(3, n + 1):
        prev, curr = curr, prev + curr
    
    return curr
```

**面试要点**:
- 本质是斐波那契数列
- 状态转移：到第 n 阶 = 到第 n-1 阶 + 到第 n-2 阶

---

### 5.3 0-1 背包问题

**时间复杂度**: O(n × W)  
**空间复杂度**: O(W) 优化版

```python
def knapsack(weights, values, capacity):
    n = len(weights)
    dp = [[0] * (capacity + 1) for _ in range(n + 1)]
    
    for i in range(1, n + 1):
        for w in range(capacity + 1):
            if weights[i - 1] <= w:
                dp[i][w] = max(
                    dp[i - 1][w],  # 不选
                    dp[i - 1][w - weights[i - 1]] + values[i - 1]  # 选
                )
            else:
                dp[i][w] = dp[i - 1][w]
    
    return dp[n][capacity]

# 空间优化
def knapsack_optimized(weights, values, capacity):
    dp = [0] * (capacity + 1)
    
    for i in range(len(weights)):
        for w in range(capacity, weights[i] - 1, -1):
            dp[w] = max(dp[w], dp[w - weights[i]] + values[i])
    
    return dp[capacity]
```

**面试要点**:
- 状态定义：`dp[i][w]` = 前 i 个物品，容量 w 的最大价值
- 状态转移：选或不选当前物品
- 空间优化：逆序遍历容量

---

### 5.4 最长公共子序列 (LCS)

**时间复杂度**: O(m × n)  
**空间复杂度**: O(m × n)

```python
def longest_common_subsequence(text1, text2):
    m, n = len(text1), len(text2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if text1[i - 1] == text2[j - 1]:
                dp[i][j] = dp[i - 1][j - 1] + 1
            else:
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1])
    
    return dp[m][n]
```

**面试要点**:
- 状态定义：`dp[i][j]` = text1[0:i] 和 text2[0:j] 的 LCS 长度
- 状态转移：
  - 相等：`dp[i][j] = dp[i-1][j-1] + 1`
  - 不等：`dp[i][j] = max(dp[i-1][j], dp[i][j-1])`

---

### 5.5 最长递增子序列 (LIS)

**时间复杂度**: O(n²) DP，O(n log n) 二分  
**空间复杂度**: O(n)

```python
# DP 解法
def length_of_lis_dp(nums):
    if not nums:
        return 0
    
    n = len(nums)
    dp = [1] * n
    
    for i in range(1, n):
        for j in range(i):
            if nums[i] > nums[j]:
                dp[i] = max(dp[i], dp[j] + 1)
    
    return max(dp)

# 二分优化
def length_of_lis_binary(nums):
    if not nums:
        return 0
    
    tails = []
    
    for num in nums:
        left, right = 0, len(tails)
        
        while left < right:
            mid = left + (right - left) // 2
            if tails[mid] < num:
                left = mid + 1
            else:
                right = mid
        
        if left == len(tails):
            tails.append(num)
        else:
            tails[left] = num
    
    return len(tails)
```

**面试要点**:
- DP：`dp[i]` = 以 nums[i] 结尾的 LIS 长度
- 二分优化：维护一个递增数组 tails
- tails[i] = 长度为 i+1 的 LIS 的最小尾元素

---

### 5.6 股票买卖问题

**买卖一次**:
```python
def max_profit(prices):
    if not prices:
        return 0
    
    min_price = float('inf')
    max_profit = 0
    
    for price in prices:
        min_price = min(min_price, price)
        max_profit = max(max_profit, price - min_price)
    
    return max_profit
```

**买卖多次**:
```python
def max_profit_multiple(prices):
    profit = 0
    
    for i in range(1, len(prices)):
        if prices[i] > prices[i - 1]:
            profit += prices[i] - prices[i - 1]
    
    return profit
```

**买卖 k 次**:
```python
def max_profit_k(k, prices):
    if not prices or k == 0:
        return 0
    
    n = len(prices)
    if k >= n // 2:
        return max_profit_multiple(prices)
    
    # dp[i][j][0] = 第 i 天，完成 j 次交易，不持有股票
    # dp[i][j][1] = 第 i 天，完成 j 次交易，持有股票
    dp = [[[0, 0] for _ in range(k + 1)] for _ in range(n)]
    
    for j in range(k + 1):
        dp[0][j][1] = -prices[0]
    
    for i in range(1, n):
        for j in range(k + 1):
            dp[i][j][0] = max(dp[i - 1][j][0], dp[i - 1][j][1] + prices[i])
            if j > 0:
                dp[i][j][1] = max(dp[i - 1][j][1], dp[i - 1][j - 1][0] - prices[i])
    
    return dp[n - 1][k][0]
```

**面试要点**:
- 一次交易：贪心，记录最小价格
- 多次交易：累加所有上涨区间
- k 次交易：DP，状态包含交易次数和持有状态

---

## 面试技巧总结

### 1. 解题步骤
1. **理解题意**：确认输入输出，边界条件
2. **举例分析**：用具体例子推导
3. **说明思路**：先说暴力解，再优化
4. **分析复杂度**：时间和空间
5. **编写代码**：边写边解释
6. **测试验证**：考虑边界情况

### 2. 时间复杂度对比

| 算法 | 最好 | 平均 | 最坏 | 空间 |
|------|------|------|------|------|
| 快速排序 | O(n log n) | O(n log n) | O(n²) | O(log n) |
| 归并排序 | O(n log n) | O(n log n) | O(n log n) | O(n) |
| 堆排序 | O(n log n) | O(n log n) | O(n log n) | O(1) |
| 二分查找 | O(1) | O(log n) | O(log n) | O(1) |

### 3. 常见优化技巧
- **双指针**：链表、数组问题
- **哈希表**：快速查找
- **单调栈/队列**：维护单调性
- **滑动窗口**：子数组/子串问题
- **前缀和**：区间求和
- **状态压缩**：DP 空间优化

### 4. 边界情况检查
- 空输入：`[]`, `None`
- 单元素：`[1]`
- 重复元素：`[1, 1, 1]`
- 极端值：最大/最小值
- 特殊情况：负数、零

---

**文档版本**: 1.0  
**最后更新**: 2026-03-05  
**适用场景**: 技术面试算法题准备
