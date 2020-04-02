---
title: 4月1日腾讯 TEG 后台开发一轮面试凉经
date: 2020-04-02 15:07:29
tags:
	- 腾讯面试
	- C++
categories:
	- 面经
---

# 4月1日腾讯 TEG 后台开发一轮面试凉经

## 面试过程

- 一开始介绍自己写的几个项目（其实就是大作业）

- 基础知识

  - TCP 的三次握手和四次挥手

  - DNS 的原理

  - C++ vector 的原理和实现

  - C++ 的多态

    这里主要问的是虚函数的实现机制，也就是**虚函数表 + 虚表指针**，然后问了虚函数表是属于谁（类还是实例）。

  - C++ 构造函数为什么不能是虚函数

    因为调用虚函数需要虚表指针，但是虚表指针需要对象实例化之后才分配内存。

  - 然后问了一个代码的运行效果

  ```cpp
  #include <iostream>
  using namespace std;
  
  class A {
  public:
      virtual void func() {
          cout << "Hello" << endl;
      }
  };
  
  int main() {
      A *p = nullptr;
      p->func();
      return 0;
  }
  ```

  一开始我说会运行失败，因为 p 没有分配内存。然后面试官让我下来自己跑跑看。我还以为会有什么意想不到的结果，之后我自己跑了下看看发现发生了段错误 `segmentation fault`，那不是和我想的一样吗，可能我们都误解了对方的意思。。。

- 算法题

  - LRU cache 的实现，要求时间复杂度为 O(1)

    这是 LeetCode 上一道很经典的算法题。我一开始想用哈希表来做，也想过用优先队列，但是都没办法实现 `get()` 和 `set()` 都是 O(1)。然后最后也没有想出来。后来在网上查，方法是**双向链表 + 哈希表**（面试官还提示了我可能不止用一种数据结构 23333）。

    LeetCode 上对这个问题的分析：

    > 哈希表查找快，但是数据无固定顺序；链表有顺序之分，插入删除快，但是查找慢。所以结合一下，形成一种新的数据结构：哈希链表。
    >
    > 来源：https://leetcode-cn.com/problems/lru-cache/solution/lru-ce-lue-xiang-jie-he-shi-xian-by-labuladong/

    现在用 C++ 实现一下。

    ```cpp
    #include <list>
    #include <unordered_map>
    
    using namespace std;
    
    class LRUCache {
    private:
        // 容量
        int capacity;
        // STL 中的双向链表
        // 元素为 Key-Value 键值对
        list<pair<int, int> > cache;
        // STL 中的哈希表
        // Value 为双向链表上某一元素的位置，用迭代器保存
        unordered_map<int, list<pair<int, int> >::iterator> map;
    public:
        LRUCache(int c) {
            this->capacity = c;
        }
    
        int get(int key) {
            auto it = map.find(key);
            if (it == map.end())
                return -1;
            pair<int, int> kv = *map[key];
            cache.erase(map[key]);
            cache.push_front(kv);
            map[key] = cache.begin();
            return kv.second;
        }
    
        int put(int key, int value) {
            auto it = map.find(key);
            if (it == map.end()) {
                if (cache.size() == capacity) {
                    auto last = cache.back();
                    int last_key = last.first;
                    map.erase(last_key);
                    cache.pop_back();
                } 
                cache.push_front(make_pair(key, value));
                map[key] = cache.begin();
            } else {
                cache.erase(map[key]);
                cache.push_front(make_pair(key, value));
                map[key] = cache.begin();
            }
        }
    
    };
    ```
    
    - 判断字符串是否有重复字符（C++ 实现）
  
  我用了 STL 里的 map 来实现。时间复杂度算上 map 的原理是 O(nlogn)，然后面试官就问我 map 的实现原理。我说可能是哈希表或者是二叉树，然后他告诉我 hash_map（C++ 标准库中叫 unordered_map） 才是哈希表，map 的原理是红黑树。
  
- 最后面试官介绍了下他们部门

  腾讯的 TEG，很好的一个技术向的部门，可惜我去不了。。

## 总结

​		面试官挺好的，一直很有耐心的在听我描述，然后也一直在和我一起讨论问题，引导我做一些更多的尝试。面试的过程挺轻松，我一开始还挺紧张，后来就越来放下心来回答问题。最后也滔滔不绝地给我介绍了他们部门。结果最后给我来一个当头棒喝。。就是他们部门的后台开发只有深圳。。。。然后我说我去不了深圳，然后面试官确认了一下我不会去深圳，结束面试过后就把流程变灰了。。。。可谓是秒凉。。。

​		有一种努力全白费的感觉，就当是吸取面试经验了。。

