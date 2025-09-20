---
title: LRU
date: 2022-09-18 15:18:34
tags:
  - 算法
---
用线性表加哈希表实现LRU
<!-- more -->

# 线性表+哈希表>>LRU算法

## 方法：哈希表 + 双向链表

### 算法

LRU 缓存机制可以通过哈希表辅以双向链表实现，我们用一个哈希表和一个双向链表维护所有在缓存中的键值对。

双向链表按照被使用的顺序存储了这些键值对，靠近头部的键值对是最近使用的，而靠近尾部的键值对是最久未使用的。

哈希表即为普通的哈希映射（HashMap），通过缓存数据的键映射到其在双向链表中的位置。

这样以来，我们首先使用哈希表进行定位，找出缓存项在双向链表中的位置，随后将其移动到双向链表的头部，即可在 O(1)O(1)O(1) 的时间内完成 get 或者 put 操作。具体的方法如下：

```c++
int LRU::get(int key)

{

}
```

对于 get 操作，首先判断 key 是否存在：

如果 key 不存在，则返回 −1；

如果 key 存在，则 key 对应的节点是最近被使用的节点。通过哈希表定位到该节点在双向链表中的位置，并将其移动到双向链表的头部，最后返回该节点的值。



对于 put 操作，首先判断 key 是否存在：

```
void LRU::put(int key,int value)
{

}
```

如果 key 不存在，使用 key 和 value 创建一个新的节点，在双向链表的头部添加该节点，并将 key 和该节点添加进哈希表中。然后判断双向链表的节点数是否超出容量，如果超出容量，则删除双向链表的尾部节点，并删除哈希表中对应的项；

如果 key 存在，则与 get 操作类似，先通过哈希表定位，再将对应的节点的值更新为 value，并将该节点移到双向链表的头部。

上述各项操作中，访问哈希表的时间复杂度为 O(1)，在双向链表的头部添加节点、在双向链表的尾部删除节点的复杂度也为 O(1)。而将一个节点移到双向链表的头部，可以分成「删除该节点」和「在双向链表的头部添加节点」两步操作，都可以在 O(1)时间内完成。

### 小贴士

在双向链表的实现中，使用一个伪头部（dummy head）和伪尾部（dummy tail）标记界限，这样在添加节点和删除节点的时候就不需要检查相邻的节点是否存在。

### 复杂度分析

时间复杂度：对于 put 和 get 都是 O(1)。

空间复杂度：O(capacity)O(\text{capacity})O(capacity)，因为哈希表和双向链表最多存储 capacity+1\text{capacity} + 1capacity+1 个元素。



```c++
#include <iostream>
#include<list>
#include<unordered_map>
using namespace std;
using std::cout;
using std::endl;
class LRU{
public:
    LRU(int cap)
    :_capacity(cap)
    {

        cout << "LRU(int cap)" << endl;
        
    }

    int get(int key);
    void put(int key,int value);

private:
    struct cacheNode
    {
        cacheNode(int key,int v)
        :_key(key)
        ,_value(v)
        {

            cout << "cacheNode(int key,int v)" << endl;

        }
        int _key;
        int _value;

    };

    list<cacheNode> _nodes;//双向链表存，
    int _capacity;//缓存的大小
    unordered_map<int,list<cacheNode>::iterator > _cache;//无序map ，
    //存放的是 key值，和链表的迭代器

};
int LRU::get(int key)
{

    //TODO 判断key值是否在map中，如果存在，直接把他
    //更新在链表的头，并且返回他的value,不存在则返回-1；
    
    auto it = _cache.find(key);//unordered_map 的 find 函数返回值为
                            // 该key值所对应的迭代器
    if(it!=_cache.end())
    {
        _nodes.splice(_nodes.begin(),_nodes,it->second);
        //链表的splice 函数可以将 _nodes链表中的 it->second所指向的元素
        //转移到_nodes.begin()的前面
        return it->second->_value;
    }
    else
    {
        return -1;
    }

}
void LRU::put(int key,int value)
{
    //TODO 判断key 是否存在，如果存在那么，直接放在链表表头
    //如果不存在则判断链表是不是满的，如果满了删除末尾元素
    //然后在链表头插入，并且插入到map中
    
    auto it = _cache.find(key); 
    if(it!=_cache.end())
    {
        it->second->_value= value;
        _nodes.splice(_nodes.begin(),_nodes,it->second);
    }
    else
    {
        if((int)_nodes.size()==_capacity)
        {
            auto &deleteNode = _nodes.back();
            _cache.erase(deleteNode._key);//unordered_map 的 earse操作
                                        // size_type earse(const key_type&key);
            _nodes.pop_back();
        }
        _nodes.push_front(cacheNode(key,value));
        _cache.insert(std::make_pair(key,_nodes.begin()));
    }
}

void test0(){

    
    LRU lru(2);
    lru.put(1,88);
    cout << "get(1)" << lru.get(1) << endl;
    lru.put(3,99);
    lru.put(4,77);
    cout << "get(1)" << lru.get(1) << endl;

}

int main(void){
    test0();
    return 0;
}
```

