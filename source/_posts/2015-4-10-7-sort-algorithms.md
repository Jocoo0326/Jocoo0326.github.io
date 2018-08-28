---
layout: post
title: 7 Sort algorithms
date: 2015-04-10
categories:
- 算法
tags:
- 排序
---

7大经典排序算法
<!-- more -->

## 冒泡排序
从第一个元素开始，与右边的元素相比，将较大者移到右边，以此类推，将最大元素移至最右边，若移动的过程中未发生交换，则排序完成
``` java
// O(n^2)
  private static void bubbleSort(int[] arr) {
    boolean swapped = true;
    int j = 0, n = arr.length;
    while (swapped) {
      swapped = false;
      j++;
      for (int i = 0; i < n - j; i++) {
        if (arr[i] > arr[i + 1]) {
          swap(arr, i, i + 1);
          swapped = true;
        }
      }
    }
  }
```
## 选择排序
分成2部分，左边已排序，右边未排序，将右边最小的元素移到最左边，加入左边部分，直至右边部分empty
``` java
  // O(n^2)
  private static void selectionSort(int[] arr) {
    int n = arr.length, minIndex;
    for (int i = 0; i < n - 1; i++) {
      minIndex = i;
      for (int j = i + 1; j < n; j++) {
        if (arr[j] < arr[minIndex]) {
          minIndex = j;
        }
      }
      if (minIndex != i) {
        swap(arr, minIndex, i);
      }
    }
  }
```

## 快速排序
分治法
divide: 分成2部分，使得左边每个元素 <= pivot <= 右边每个元素
conquer: 处理左右2部分
``` java
  // O(nlogn)
  private static void quickSort(int[] arr, int left, int right) {
    if (arr == null || arr.length == 1) return;
    int l = left, r = right, pivot = arr[(l + r) / 2];
    while (l <= r) {
      while (arr[l] < pivot)
        l++;
      while (arr[r] > pivot)
        r--;
      if (l <= r) {
        swap(arr, l, r);
        l++;
        r--;
      }
    }
    if (left < r) {
      quickSort(arr, left, r);
    }
    if (l < right) {
      quickSort(arr, l, right);
    }
  }
```

## 插入排序
分成2部分，左边已排序， 右边未排序，将右边第一个元素插入到左边部分，直至右边为empty
``` java
  private static void insertionSort(int[] arr) {
    if (arr == null || arr.length == 1) return;
    int n = arr.length, newValue, j;
    for (int i = 0; i < n - 1; i++) {
      j = i + 1;
      newValue = arr[j];
      while (j > 0 && arr[j - 1] > newValue) {
        arr[j] = arr[j - 1];
        j--;
      }
      if (j != i + 1) {
        arr[j] = newValue;
      }
    }
  }
```

## 堆排序
最大堆化，每个节点都>=左右子节点；将根节点与未排序部分尾节点交换，重新堆化，直至未排序部分为empty
``` java
  // O(nlogn)
  private static void heapSort(int[] arr) {
    if (arr == null || arr.length == 1) return;
    int maxIndex = arr.length - 1;
    int n = (maxIndex - 1) >> 1; // 最大堆中，节点i的左子节点为2i + 1，右子节点为2i + 2
    for (int i = n; i >= 0; i--) {
      maxHeapify(arr, i, maxIndex);
    }
    for (int i = maxIndex; i > 0; i--) {
      swap(arr, 0, i);
      maxHeapify(arr, 0, i - 1);
    }
  }

  private static void maxHeapify(int[] arr, int n, int maxIndex) {
    int root = n, child;
    while (true) {
      child = (root << 1) + 1;
      if (root > maxIndex || child > maxIndex) return;
      if (child + 1 <= maxIndex && arr[child + 1] > arr[child]) {
        ++child;
      }
      if (arr[child] > arr[root]) {
        swap(arr, root, child);
        root = child;
      } else {
        return;
      }
    }
  }
```

## 希尔排序
做步长为数组长度一半的插入排序，步长减半，重复执行，直至步长为1
``` java
  private static void shellSort(int[] arr) {
    if (arr == null || arr.length == 1) return;
    int n = arr.length, step = n >> 1, tmp, j;
    while (step >= 1) {
      for (int i = step; i < n; i++) { // 特定步长的多个数组的插入排序可以合并
        tmp = arr[i];
        j = i;
        while (j >= step && arr[j - step] > tmp) {
          arr[j] = arr[j - step];
          j -= step;
        }
        if (j != i) {
          arr[j] = tmp;
        }
      }
      step = step >> 1;
    }
  }
```

## 归并排序
将数组分半，直至每组1or2元素，排序，然后两两合并
``` java
  private static void mergeSort(int[] arr) {
    if (arr == null || arr.length == 1) return;
    int len = arr.length;
    int[] result = new int[len];
    mergeSortRecursive(arr, result, 0, len - 1);
  }

  private static void mergeSortRecursive(int[] arr, int[] result, int start, int end) {
    if (start >= end) return;
    int len = end - start, mid = (len >> 1) + start;
    int start1 = start, end1 = mid, start2 = mid + 1, end2 = end;
    mergeSortRecursive(arr, result, start1, end1);
    mergeSortRecursive(arr, result, start2, end2);
    int k = start;
    while (start1 <= end1 && start2 <= end2) {
      result[k++] = arr[start1] < arr[start2] ? arr[start1++] : arr[start2++];
    }
    while (start1 <= end1) {
      result[k++] = arr[start1++];
    }
    while (start2 <= end2) {
      result[k++] = arr[start2++];
    }
    for (k = start; k <= end; k++) {
      arr[k] = result[k];
    }
  }
```