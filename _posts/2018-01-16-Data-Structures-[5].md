---
layout: post
title:  "数据结构[5] -- 排序算法"
categories: 数据结构和算法
tags:  数据结构 算法
author: G.Fukang
---

* content
{:toc}
常用的排序算法：插入排序、希尔排序、归并排序、堆排序



## 插入排序 `O(N^2)`

- 从第一个元素开始，该元素可以认为已经被排序

- 取出下一个元素，在已经排序的元素序列中从后向前扫描

- 如果该元素（已排序）大于新元素，将该元素移到下一位置

- 重复上面步骤，直到找到已排序的元素小于或者等于新元素的位置，将新元素插入到该位置后

- 代码片段

  ```java
  /**
   * @Auther gongfukang
   * @Date 2017/12/4 17:10
   * 插入排序
   */
  public class InsertionSort {
      private long[] a;
      private int Elems;

      public InsertionSort(int max){
          a=new long[max];
          Elems=0;
      }

      public void insertionSort(){
          int in,out;

          for(out=1;out<Elems;out++){
              long temp=a[out];
              in=out;
              while (in>0&&a[in-1]>=temp){
                  a[in]=a[in-1];
                  --in;
              }
              a[in]=temp;
          }
      }
  }
  ```

## 归并排序 `O(N*logN)`

归并排序的中心思想是归并两个有序数组，可以通过递归和非递归实现。

### 递归

将一个数组分成两半，分别排序每一半，然后用`merge()`方法将数组的两半归并成一个有序数组

- 初始化一个数组，将左右数组的数进行比较，将较小的数存入中间数组

- 将左右数组剩下的数存到中间数组，然后将中间数组复制回原来的数组

  代码片段

  ```java
  /**
   * @Auther gongfukang
   * @Date 2017/12/22 11:38
   * 归并排序（递归实现）
   */

  private void recMergeSort_2(long[] workSpace, int lowerBound, int upperBound) {
          if (lowerBound == upperBound)
              return;
          else {
              int mid = (lowerBound + upperBound) / 2;
              recMergeSort_2(workSpace, lowerBound, mid);
              recMergeSort_2(workSpace, mid + 1, upperBound);
              merge(workSpace,lowerBound,mid+1,upperBound);
          }
      }
  ```



### 非递归

将数组中的相邻元素两两配对，用`merge()`函数将他们排序，构成`n/2`组长度为2的排序好的子数组段，然后将他们排序成长度为4的子数组段，如此继续，直至整个数组排好序。

- 从归并段的长度为1开始，一次使归并段的长度变为原来的2倍

- 在每趟归并的过程中，要注意处理归并段的长度为奇数和最后一个归并段的长度和前面的不等的情况

- 代码片段

  ```java
  /**
   * @Auther gongfukang
   * @Date 2017/12/7 22:26
   * 归并排序（非递归）
   */

  private void recMergeSort_1(long[] workSpace) {
          int size=workSpace.length;
          int k=1;

          while (k< size){
              mergePass(workSpace,k,size);
              k*=2;
          }
      }

      private void mergePass(long[] workSpace,int k,int n){
          int i=0;

          while (i<n-2*k+1){
              merge(workSpace,i,i+k-1,i+2*k-1);
              i+=2*k;
          }

          if(i<n-k){
              merge(workSpace,i,i+k-1,n-1);
          }
      }
  ```

###  关于`merge()`方法

- `merge()`方法主要实现将两个有序数组归并成一个有序数组，要注意处理归并段的长度为奇数和最后一个归并段的长度和前面的不等的情况

  代码片段

  ```java
  private void merge(long[] workSpace, int lowPtr, int highPtr, int upperBound) {
          int j = 0;
          int lowerBound = lowPtr;
          int mid = highPtr - 1;
          int n = upperBound - lowerBound + 1;

          while (lowPtr <= mid && highPtr <= upperBound) {
              if (theArray[lowPtr] < theArray[highPtr])
                  workSpace[j++] = theArray[lowPtr++];
              else
                  workSpace[j++] = theArray[highPtr++];
          }
          while (lowPtr <= mid)
              workSpace[j++] = theArray[lowPtr++];

          while (highPtr <= upperBound)
              workSpace[j++] = theArray[highPtr++];

          for (j = 0; j < n; j++) {
              theArray[lowerBound + j] = workSpace[j];
          }
      }
  ```

## 堆排序 `O(N*logN)`

关于堆的定义及其实现，可以先查看[这篇博文](https://gongfukangee.github.io/2018/01/08/Data-Structures-1/)，基于堆的排序就是使用`insert()`方法插入全部数据项，然后重复用`remove()`方法移除所有数据项。

代码片段

```java
for(int j=0;j<size;j++)
    theHeap.insert(array[j]);
for(int i=0;i<size;i++)
    array[i]=theHeap.remove();
```

## Github

完整代码，我托管在[Github](https://github.com/gongfukangEE/Data-Structures-Java)上

- [插入排序](https://github.com/gongfukangEE/Data-Structures-Java/blob/master/src/Sort/Simple_Sort/InsertionSort.java)


- [归并排序-递归](https://github.com/gongfukangEE/Data-Structures-Java/blob/master/src/Sort/Advanced_Sort/MergeSort_2.java)
- [归并排序-非递归](https://github.com/gongfukangEE/Data-Structures-Java/blob/master/src/Sort/Advanced_Sort/MergeSort_1.java)
- [堆排序](https://github.com/gongfukangEE/Data-Structures-Java/tree/master/src/Sort/Advanced_Sort/HeapSort)


如果对你有帮助，请给我点个star以示肯定和鼓励。