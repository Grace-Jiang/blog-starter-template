---
title: 堆排序Python实现
description: 堆排序算法的Python实现
pubDate: '2025-02-18'
categories:
- Python
tags:
- 排序
- 堆排序
- Python
---
# 堆排序Python实现

堆排序（Heap Sort）是一种基于二叉堆数据结构的排序算法，时间复杂度为 O(n log n)。

```python
def heapify(arr, n, i):
    largest = i
    left = 2*i + 1
    right = 2*i + 2

    if left < n and arr[left] > arr[largest]:
        largest = left

    if right < n and arr[right] > arr[largest]:
        largest = right

    if largest != i:
        arr[i], arr[largest] = arr[largest], arr[i]
        heapify(arr, n, largest)

def heap_sort(arr):
    n = len(arr)
    # Build max heap
    i = n//2-1
    while i>=0:
        heapify(arr, n, i)
        i -= 1
    
    # Extract elements from heap one by one
    i = n-1
    while i > 0:
        arr[0], arr[i] = arr[i], arr[0]
        heapify(arr, i, 0)
        i -= 1
    
    return arr

print(heap_sort([1,2,3,4,5,6,7]))
```
