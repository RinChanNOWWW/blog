---
title: 数组中的逆序对
date: 2020-03-24 17:42:30
tags: 
	- 算法题
	- 分治法
	- 归并排序
categories:
	- 算法题
---
# 数组中的逆序对

题目描述：
> 在数组中的两个数字，如果前面一个数字大于后面的数字，则这两个数字组成一个逆序对。输入一个数组,求出这个数组中的逆序对的总数P。并将P对1000000007取模的结果输出。 即输出P%1000000007。

输入描述：
> 题目保证输入的数组中没有的相同的数字。		
> 数据范围：			
> &nbsp;&nbsp;&nbsp;&nbsp;对于%50的数据,size<=10^4		
> &nbsp;&nbsp;&nbsp;&nbsp;对于%75的数据,size<=10^5		
> &nbsp;&nbsp;&nbsp;&nbsp;对于%100的数据,size<=2*10^5		

示例：
> 输入：	
>> 1,2,3,4,5,6,7,0	

> 输出：
>> 7

思路：分治法，类似归并排序的做法。分别计算每小组的逆序对，然后再将其归并排序得到一个大组，然后就可以算出更大组之间的逆序对了。最后得到整个数组的逆序对个数。

代码实现：

```python
def InversePairs(self, data):
    # write code here
    def mergeSort(data, left, right):
        if left == right:
            return 0
        print(left, right)
        mid = (left + right) // 2
        left_count = mergeSort(data, left, mid)
        right_count = mergeSort(data, mid + 1, right)
        two_side_count = merge(data, left, mid, right)
        return left_count + two_side_count + right_count
    
    def merge(data, left, mid, right):
        i, j, count = left, mid + 1, 0
        temp = []
        while i <= mid and j <= right:
            if data[i] <= data[j]:
                temp.append(data[i])
                i += 1
            else:
                temp.append(data[j])
                j += 1
                count += mid - i + 1
        while i <= mid:
            temp.append(data[i])
            i += 1
        while j <= right:
            temp.append(data[j])
            j += 1
        data[left: right + 1] = temp
        del temp
        return count
	
    if len(data) == 0: 
        return 0
    return mergeSort(data, 0, len(data) - 1) % 1000000007

```




