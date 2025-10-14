---
title: Mymap Myset
date: 2022-10-15 15:51
tags:
  - cpp
---
红黑树模拟实现map和set

<!-- more -->

## 一、map和set模板

set用value标识元素(value就是key，类型为T)，并且每个value必须唯一 。



```c++
template < class Key>//set
```



在map中，键值key通常用于排序和惟一地标识元素，而值value中存储与此键值key关联的内容。键值key和值value的类型可能不同，并且在map的内部，key与value通过成员类型value_type绑定在一起，为其取别名称为pair：

```c++
typedef pair<const Key, T> value_type;
template < class Key, class T>//map
```

 用[红黑树](https://so.csdn.net/so/search?q=红黑树&spm=1001.2101.3001.7020)同时封装出set和map时，set传给value的是一个value，map传给value的是一个pair，set和map传给红黑树的value决定了这棵树里面存的节点值类型。上层容器不同，底层红黑树的Key和T也不同。

![](D:\Blog\host_source\image-20230730170956974.png)

在上层容器set中，K和T都代表Key，底层红黑树节点当中存储K和T都是一样的；map中，K代表键值Key，T代表由Key和Value构成的键值对，底层红黑树中只能存储T。所以红黑树为了满足同时支持set和map，节点当中存储T

这就要对红黑树进行改动。

## 二、红黑树节点定义

### 1.红黑树节点定义由类模板

```c++
template<class K,class V>
```

修改为

```c++
template<class T>
```

那么节点定义修改为:

```c++
//红黑树节点定义
template<class T>
struct RBTreeNode
{
	RBTreeNode<T>* _left;//节点的左孩子
	RBTreeNode<T>* _right;//节点的右孩子
	RBTreeNode<T>* _parent;//节点的父亲
 
	T _data;//节点的值，_data里面存的是K就传K，存的是pair就传pair
	Colour _col;//节点颜色
 
	RBTreeNode(const T& x)
		:_left(nullptr)
		, _right(nullptr)
		, _parent(nullptr)
		, _data(x)
		, _col(RED)
	{}
};
```

由于红黑树不知道上层传的是K还是pair，这是由上层传递的模板参数T决定的，上层是封装我的map和set

### 2.仿函数

#### （1）节点比较大小时存在的问题

红黑树插入节点时，需要比较节点的大小，kv需要改成_data:

```c++
	//插入
	pair<Node*, bool> Insert(const T& data)
	{
		if (_root == nullptr)
		{
			_root = new Node(data);
			_root->_col = BLACK;
			return make_pair(_root, true);
		}
 
		//1.先看树中，kv是否存在
		Node* parent = nullptr;
		Node* cur = _root;
		while (cur)
		{
			if (cur->_data < data)
			{
				//kv比当前节点值大，向右走
				parent = cur;
				cur = cur->_right;
			}
			else if (cur->_data > data)
			{
				//kv比当前节点值小，向左走
				parent = cur;
				cur = cur->_left;
			}
			else
			{
				//kv和当前节点值相等，已存在，插入失败
				return make_pair(cur, false);
			}
		}
 
		//2.走到这里，说明kv在树中不存在，需要插入kv，并且cur已经为空，parent已经是叶子节点了
		Node* newNode = new Node(kv);
		newNode->_col = RED;
		if (parent->_data < data)
		{
			//kv比parent值大，插入到parent的右边
			parent->_right = newNode;
			newNode->_parent = parent;
		}
		else
		{
			//kv比parent值小，插入到parent的左边
			parent->_left = newNode;
			newNode->_parent = parent;
		}
		cur = newNode;
		
        //如果父亲存在，且父亲颜色为红就要处理
		while (parent && parent->_col == RED)
		{
			//情况一和情况二、三的区别关键看叔叔
			Node* grandfather = parent->_parent;//当父亲是红色时，根据规则（2）根节点一定是黑色，祖父一定存在
			if (parent == grandfather->_left)//父亲是祖父的左子树
			{
				Node* uncle = grandfather->_right;
				//情况一：叔叔存在且为红
				if (uncle->_col == RED)
				{
					parent->_col = uncle->_col = BLACK;
					grandfather->_col = RED;
 
					//继续向上调整
					cur = grandfather;
					parent = cur->_parent;
				}
				else//情况二+情况三：叔叔不存在或叔叔存在且为黑
				{
					//情况二：单旋
					if (cur == parent->_left)
					{
						RotateR(grandfather);
						parent->_col = BLACK;
						grandfather->_col = RED;
					}
					else//情况三：双旋
					{
						RotateL(parent);
						RotateR(grandfather);
						cur->_col = BLACK;
						grandfather->_col = RED;
					}
					break;//插入结束
				}
			}
			else//父亲是祖父的右子树
			{
				Node* uncle = grandfather->_left;
				//情况一：叔叔存在且为红
				if (uncle && uncle->_col == RED)
				{
					parent->_col = uncle->_col = BLACK;
					grandfather->_col = RED;
 
					//继续往上调整
					cur = grandfather;
					parent = grandfather->_parent;
				}
				else//情况二+情况三：叔叔不存在或叔叔存在且为黑
				{
					//情况二：单旋
					if (cur == parent->_right)
					{
						RotateL(grandfather);
						parent->_col = BLACK;
						grandfather->_col = RED;
					}
					else//情况三：双旋
					{
						RotateR(parent);
						RotateL(grandfather);
						cur->_col = BLACK;
						grandfather->_col = RED;
					}
					break;//插入结束
				}
			}
 
		}
		_root->_col = BLACK;
 
		return make_pair(newNode, true);
	}
```

但是以上代码在插入新节和查找节点时，当和当前节点比较大小时，Key可以比较，但是pair比较不了，也就是set可以比较，但是map比较不了。这就需要写一个仿函数，如果是map就取_data里面的first也就是Key进行比较，通过泛型解决红黑树里面存的是什么。所以上层容器map需要向底层的红黑树提供仿函数来获取T里面的Key，这样无论上层容器是set还是map，都可以用统一的方式进行比较了。

#### (2) 仿函数

仿函数让一个类的使用看上去像个函数。仿函数是在类中实现了一个operator( )，是一个类的对象，这个类就有了类似函数的行为，所以这个类就是一个仿函数类，目的是为了让函数拥有类的性质。

这个类的对象即仿函数，可以当作一般函数去用，只不过仿函数的功能是在一个类中的运算符operator()中实现的，使用的时候把函数作为参进行传递即可。

set有set的仿函数，map有map的仿函数，尽管set的仿函数看起来没有什么作用，但是，必须要把它传给底层红黑树，这样红黑树就能根据仿函数分别获取set的key和map的first。

①：set的仿函数

```c++
namespace delia
{
	template<class K>
	class set
	{
		//仿函数，获取set的key
		struct SetKeyOfT
		{
			const K& operator()(const K& key)
			{
				return key;
			}
		};
    
    public:
		bool insert(const K& k)
		{
			_t.Insert(k);
			return true;
		}
 
	private:
		RBTree<K, K,SetKeyOfT> _t;
	};
}
```

②map的仿函数

```c++
namespace delia
{
	template<class K,class V>
	class map
	{
		//仿函数，获取map的first
		struct MapKeyOfT
		{
			const K& operator()(const pair<const K, V>& kv)
			{
				return kv.first;
			}
		};
 
    public:
        //插入
		bool insert(const pair<const K, V>& kv)
		{
			_t.Insert(kv);
			return true;
		}
	private:
		RBTree<K, pair<const K, V>, MapKeyOfT> _t;
	};
}
```

有了仿函数红黑树的类在实现时，就要在模板参数中增加KeyOfT仿函数。

#### （3）修改红黑树定义

```c++
template<class K, class T, class KeyOfT>
class RBTree
{
	typedef RBTreeNode<T> Node;
	
private:
	Node* _root;
};
```

#### （4）修改红黑树插入

```c++
	//插入
	pair<Node*, bool> Insert(const pair<K, V>& kv)
	{
		if (_root == nullptr)
		{
			_root = new Node(kv);
			_root->_col = BLACK;
			return make_pair(_root, true);
		}
 
		KeyOfT kot;
 
		//1.先看树中，kv是否存在
		Node* parent = nullptr;
		Node* cur = _root;
		while (cur)
		{
			if (kot(cur->_data) < kot(data))
			{
				//kv比当前节点值大，向右走
				parent = cur;
				cur = cur->_right;
			}
			else if (kot(cur->_data) > kot(data))
			{
				//kv比当前节点值小，向左走
				parent = cur;
				cur = cur->_left;
			}
			else
			{
				//kv和当前节点值相等，已存在，插入失败
				return make_pair(cur, false);
			}
		}
 
		//2.走到这里，说明kv在树中不存在，需要插入kv，并且cur已经为空，parent已经是叶子节点了
		Node* newNode = new Node(kv);
		newNode->_col = RED;
		if (kot(parent->_data) < kot(data))
		{
			//kv比parent值大，插入到parent的右边
			parent->_right = newNode;
			newNode->_parent = parent;
		}
		else
		{
			//kv比parent值小，插入到parent的左边
			parent->_left = newNode;
			newNode->_parent = parent;
		}
		cur = newNode;
 
		//如果父亲存在，且父亲颜色为红就要处理
		while (parent && parent->_col == RED)
		{
			//情况一和情况二、三的区别关键看叔叔
			Node* grandfather = parent->_parent;//当父亲是红色时，根据规则（2）根节点一定是黑色，祖父一定存在
			if (parent == grandfather->_left)//父亲是祖父的左子树
			{
				Node* uncle = grandfather->_right;
				//情况一：叔叔存在且为红
				if (uncle->_col == RED)
				{
					parent->_col = uncle->_col = BLACK;
					grandfather->_col = RED;
 
					//继续向上调整
					cur = grandfather;
					parent = cur->_parent;
				}
				else//情况二+情况三：叔叔不存在或叔叔存在且为黑
				{
					//情况二：单旋
					if (cur == parent->_left)
					{
						RotateR(grandfather);
						parent->_col = BLACK;
						grandfather->_col = RED;
					}
					else//情况三：双旋
					{
						RotateL(parent);
						RotateR(grandfather);
						cur->_col = BLACK;
						grandfather->_col = RED;
					}
					break;//插入结束
				}
			}
			else//父亲是祖父的右子树
			{
				Node* uncle = grandfather->_left;
				//情况一：叔叔存在且为红
				if (uncle && uncle->_col == RED)
				{
					parent->_col = uncle->_col = BLACK;
					grandfather->_col = RED;
 
					//继续往上调整
					cur = grandfather;
					parent = grandfather->_parent;
				}
				else//情况二+情况三：叔叔不存在或叔叔存在且为黑
				{
					//情况二：单旋
					if (cur == parent->_right)
					{
						RotateL(grandfather);
						parent->_col = BLACK;
						grandfather->_col = RED;
					}
					else//情况三：双旋
					{
						RotateR(parent);
						RotateL(grandfather);
						cur->_col = BLACK;
						grandfather->_col = RED;
					}
					break;//插入结束
				}
			}
 
		}
		_root->_col = BLACK;
 
		return make_pair(newNode, true);
	}
 
	void RotateR(Node* parent)
	{
		Node* subL = parent->_left;
		Node* subLR = nullptr;
 
		if (subL)
		{
			subLR = subL->_right;
		}
		//1.左子树的右子树变我的左子树
		parent->_left = subLR;
 
		if (subLR)
		{
			subLR->_parent = parent;
		}
 
		//左子树变父亲
		subL->_right = parent;
		Node* parentParent = parent->_parent;
		parent->_parent = subL;
 
 
		if (parent == _root)//parent是根
		{
			_root = subL;
			_root->_parent = nullptr;
		}
		else//parent不是根，是子树
		{
			if (parentParent->_left == parent)
			{
				//parent是自己父亲的左子树,将subL作为parent父亲的左孩子
				parentParent->_left = subL;
			}
			else
			{
				//parent是自己父亲的右子树,将subL作为parent父亲的右孩子
				parentParent->_right = subL;
			}
 
			//subL的父亲就是parent的父亲
			subL->_parent = parentParent;
		}
	}
 
	void RotateL(Node* parent)
	{
		Node* subR = parent->_right;
		Node* subRL = nullptr;
 
		if (subR)
		{
			subRL = subR->_left;
		}
 
		//1.右子树的左子树变我的右子树
		parent->_right = subRL;
 
		if (subRL)
		{
			subRL->_parent = parent;
		}
 
		//2.右子树变父亲
		subR->_left = parent;
		Node* parentParent = parent->_parent;
		parent->_parent = subR;
 
		if (parent == _root)//parent是根
		{
			_root = parent;
			_root->_parent = nullptr;
		}
		else//parent不是根，是子树
		{
			if (parentParent->_left == parent)
			{
				//parent是自己父亲的左子树,将subR作为parent父亲的左孩子
				parentParent->_left = subR;
			}
			else
			{
				//parent是自己父亲的右子树,将subR作为parent父亲的右孩子
				parentParent->_right = subR;
			}
 
			//subR的父亲就是parent的父亲
			subR->_parent = parentParent;
		}
	}
```

#### （5）修改红黑树查找

```c++
	//查找
	Node* Find(const K& key)
	{
		KeyOfT kot;
		Node* cur = _root;
		while (cur)
		{
			if (kot(cur->_data) < key)
			{
				cur = cur->_right;
			}
			else if (kot(cur->_data) > key)
			{
				cur = cur->_left;
			}
			else
			{
				return cur;
			}
		}
		return nullptr;//空树，直接返回
	}
```

## 三、红黑树迭代器

map和set的迭代器的实现其实本质上是红黑树迭代器的实现，迭代器的实现需要定义模板类型、模板类型引用、模板类型指针。 

### 1.红黑树中迭代器重命名

 在红黑树中重命名模板类型、模板类型引用、模板类型指针，定义为public，外部就能使用iterator了：

```c++
template<class K, class T, class KeyOfT>
class RBTree
{
	typedef RBTreeNode<T> Node;
 
public:
	typedef __TreeIterator<T, T&, T*> iterator;//模板类型、模板类型引用、模板类型指针
    
    //红黑树函数...
    
private:
	Node* _root;
};
```

### 2.正向迭代器定义

红黑树的迭代器的本质是对节点指针进行封装，所以迭代器中只有封装红黑树节点指针这一个成员变量 。正向迭代器：

```
template<class T,class Ref,class ptr>
struct __TreeIterator
{
	typedef RBTreeNode<T> Node;
	typedef __TreeIterator<T, Ref, ptr> Self;
      
	Node* _node;//成员变量
	
};
```

### 3.迭代器构造

用节点指针构造正向迭代器：

```
	//构造函数
	__TreeIterator(Node* node)
		:_node(node)
	{}
```

###   4.正向迭代器重载*

Ref对正向迭代器解引用，返回节点数据引用

```
	//* 解引用，返回节点数据
	Ref Operator*()
	{
		return _node->_data;
	}
```

### 5.正向迭代器重载->

Ptr对正向迭代器使用->，返回节点数据指针：

```
	//-> 返回节点数据地址
	Ptr Operator->()
	{
		return &_node->_data;
	}
```

### 6.正向迭代器重载==

判断节点是否相同

```
	//判断两个迭代器是否相同
	bool operator==(const Self& s)
	{
		return _node == s._node;//判断节点是否相同
	}
```

### 7.正向迭代器重载！=

判断节点是否不同

```
	//判断两个迭代器是否不同
	bool operator!=(const Self& s)
	{
		return _node != s._node;//判断节点是否不同
	}
```

### 8.正向迭代器++

①当节点的右子树不为空时，++就要走到右子树的最左节点

 ②当节点的右子树为空时，++就要走到节点的父亲

```c++
	//红黑树迭代器的++也就是红黑树的++
	Self operator++()
	{
		//1.右子树不为空
		if (_node->_right)
		{
			//下一个访问的是右树的中序第一个节点（即右子树最左节点）。
			Node* left = _node->_right;
 
			//找最左节点
			while (left->_left)
			{
				left = left->_left;
			}
			_node = left;
		}
		else//2.右子树为空，下一个访问的就是当前节点的父亲
		{
			Node* cur = _node;
			Node* parent = cur->_parent;
			while (parent && cur == parent->_right)
			{
				cur = cur->_parent;
				parent = parent->_parent;
			}
			_node = parent;
		}
 
		return *this;
	}
};
```

### 9.正向迭代器--

 ①当节点的左子树不为空时，++就要走到左子树的最右节点

 ②当节点的左子树为空时，++就要走到节点的父亲

```c++
	//红黑树迭代器的--也就是红黑树的--
	Self operator--()
	{
		//1.左子树不为空
		if (_node->_left)
		{
			//下一个访问的是左树的中序左后节点（即做子树最右节点）。
			Node* right = _node->_left;
 
			//找最右节点
			while (right->_right)
			{
				right = right->_right;
			}
			_node = right;
		}
		else//2.左子树为空，下一个访问的就是当前节点的父亲
		{
			Node* cur = _node;
			Node* parent = cur->_parent;
			while (parent && cur == parent->_left)
			{
				cur = cur->_parent;
				parent = parent->_parent;
			}
			_node = parent;
		}
 
		return *this;
	}
```

### 10.红黑树中实现迭代器

实现begin( )找最左节点，end( )最后一个节点的下一个位置

```c++
template<class K, class T, class KeyOfT>
class RBTree
{
	typedef RBTreeNode<T> Node;
 
public:
	typedef __TreeIterator<T, T&, T*> iterator;//模板类型、模板类型引用、模板类型指针
    
    //找最左节点
	iterator begin()
	{
		Node* left = _root;
		while (left && left->_left)
		{
			left = left->_left;
		}
 
		return iterator(left)//返回最左节点的正向迭代器
	}
 
	//结束
	iterator end()
	{
		return iterator(nullptr);
	}
    
private:
	Node* _root;
};
```

## 四、set模拟实现

调用红黑树对应接口实现set，插入和查找函数返回值当中的节点指针改为迭代器:

```c++
#pragma once
#include "RBTree.h"
namespace delia
{
	template<class K>
	class set
	{
		//仿函数，获取set的key
		struct SetKeyOfT
		{
			const K& operator()(const K& key)
			{
				return key;
			}
		};
	public:
		typedef typename RBTree<K, K, SetKeyOfT>::iterator iterator;
		
		//迭代器开始
		iterator begin()
		{
			return _t.begin();
		}
 
		//迭代器结束
		iterator end()
		{
			return _t.end();
		}
 
		//插入函数
		pair<iterator,bool> insert(const K& key)
		{
			
			return _t.Insert(key);
		}
 
		//查找
		iterator find(const K& key)
		{
			return _t.find(key);
		}
	private:
		RBTree<K, K, SetKeyOfT> _t;
	};
}
```

## 五、map模拟实现

调用红黑树对应接口实现map，插入和查找函数返回值当中的节点指针改为迭代器，增加operator[ ]的重载:

```c++
#pragma once
#include "RBTree.h"
namespace delia
{
	template<class K, class V>
	class map
	{
		//仿函数，获取map的first
		struct MapKeyOfT
		{
			const K& operator()(const pair<const K, V>& kv)
			{
				return kv.first;
			}
		};
	public:
		typedef typename RBTree<K, K, MapKeyOfT>::iterator iterator;
 
		//迭代器开始
		iterator begin()
		{
			return _t.begin();
		}
 
		//迭代器结束
		iterator end()
		{
			return _t.end();
		}
 
		//插入
		pair<iterator, bool> insert(const pair<const K, V>& kv)
		{
			return _t.Insert(kv);
		}
 
		//重载operator[]
		V& operator[](const K& key)
		{
			pair<iterator, bool> ret = insert(make_pair(key, V()));
			iterator it = ret.first;
			return it->second;
		}
 
		//查找
		iterator find(const K& key)
		{
			return _t.find(key);
		}
 
	private:
		RBTree<K, pair<const K, V>, MapKeyOfT> _t;
	};
}
```

## 六、红黑树完整代码段

```c++
#pragma once
#include<iostream>
using namespace std;
 
 
//节点颜色
enum Colour
{
	RED,
	BLACK,
};
 
//红黑树节点定义
template<class T>
struct RBTreeNode
{
	RBTreeNode<T>* _left;//节点的左孩子
	RBTreeNode<T>* _right;//节点的右孩子
	RBTreeNode<T>* _parent;//节点的父亲
 
	T _data;//节点的值
	Colour _col;//节点颜色
 
	RBTreeNode(const T& x)
		:_left(nullptr)
		, _right(nullptr)
		, _parent(nullptr)
		, _data(x)
		, _col(RED)
	{}
};
 
 
template<class T,class Ref,class ptr>
struct __TreeIterator
{
	typedef RBTreeNode<T> Node;
	typedef __TreeIterator<T, Ref, ptr> Self;
 
	Node* _node;
 
	//构造函数
	__TreeIterator(Node* node)
		:_node(node)
	{}
	
	//* 解引用，返回节点数据
	Ref operator*()
	{
		return _node->_data;
	}
 
	//-> 返回节点数据地址
	//Ptr operator->()
	//{
	//	return &_node->_data;
	//}
 
	//判断两个迭代器是否相同
	bool operator==(const Self& s)
	{
		return _node == s._node;
	}
 
	//判断两个迭代器是否不同
	bool operator!=(const Self& s)
	{
		return _node != s._node;
	}
 
	//红黑树迭代器的++也就是红黑树的++
	Self operator++()
	{
		//1.右子树不为空
		if (_node->_right)
		{
			//下一个访问的是右树的中序第一个节点（即右子树最左节点）。
			Node* left = _node->_right;
 
			//找最左节点
			while (left->_left)
			{
				left = left->_left;
			}
			_node = left;
		}
		else//2.右子树为空，下一个访问的就是当前节点的父亲
		{
			Node* cur = _node;
			Node* parent = cur->_parent;
			while (parent && cur == parent->_right)
			{
				cur = cur->_parent;
				parent = parent->_parent;
			}
			_node = parent;
		}
 
		return *this;
	}
 
	//红黑树迭代器的--也就是红黑树的--
	Self operator--()
	{
		//1.左子树不为空
		if (_node->_left)
		{
			//下一个访问的是左树的中序左后节点（即做子树最右节点）。
			Node* right = _node->_left;
 
			//找最右节点
			while (right->_right)
			{
				right = right->_right;
			}
			_node = right;
		}
		else//2.左子树为空，下一个访问的就是当前节点的父亲
		{
			Node* cur = _node;
			Node* parent = cur->_parent;
			while (parent && cur == parent->_left)
			{
				cur = cur->_parent;
				parent = parent->_parent;
			}
			_node = parent;
		}
 
		return *this;
	}
 
 
};
 
//插入节点颜色是红色好，还是黑色好，红色
//因为插入红色节点，可能破坏规则3，影响不大
//插入黑色节点，一定破坏规则4 ，并且影响其他路径，影响很大
 
template<class K, class T, class KeyOfT>
class RBTree
{
	typedef RBTreeNode<T> Node;
public:
	typedef __TreeIterator<T, T&, T*> iterator;//模板类型、模板类型引用、模板类型指针
 
	//构造函数
	RBTree()
		:_root(nullpte)
	{}
 
	//析构
	~RBTree()
	{
		_Destroy(_root);
		_root = nullptr;
	}
 
	void _Destroy(Node* root)
	{
		if (root == nullptr)
		{
			return;
		}
		_Destroy(root->_left);
		_Destroy(root->_right);
		delete root;
	}
 
	//找最左节点
	iterator begin()
	{
		Node* left = _root;
		while (left && left->_left)
		{
			left = left->_left;
		}
 
		return iterator(left);//返回最左节点的正向迭代器
	}
 
	//结束
	iterator end()
	{
		return iterator(nullptr);
	}
 
	//构造函数
	RBTree()
		:_root(nullptr)
	{}
 
	void Destroy(Node* root)
	{
		if (root == nullptr)
		{
			return;
		}
 
		Destroy(root->_left);
		Destroy(root->_right);
	}
	~RBTree()
	{
		Destroy(_root);
		_root = nullptr;
	}
 
	//插入
	pair<Node*, bool> Insert(const T& data)
	{
		if (_root == nullptr)
		{
			_root = new Node(data);
			_root->_col = BLACK;
			return make_pair(_root, true);
		}
 
		KeyOfT kot;
 
		//1.先看树中，kv是否存在
		Node* parent = nullptr;
		Node* cur = _root;
		while (cur)
		{
			if (kot(cur->_data) < kot(data))
			{
				//kv比当前节点值大，向右走
				parent = cur;
				cur = cur->_right;
			}
			else if (kot(cur->_data) > kot(data))
			{
				//kv比当前节点值小，向左走
				parent = cur;
				cur = cur->_left;
			}
			else
			{
				//kv和当前节点值相等，已存在，插入失败
				return make_pair(cur, false);
			}
		}
 
		//2.走到这里，说明kv在树中不存在，需要插入kv，并且cur已经为空，parent已经是叶子节点了
		Node* newNode = new Node(data);
		newNode->_col = RED;
		if (kot(parent->_data) < kot(data))
		{
			//kv比parent值大，插入到parent的右边
			parent->_right = newNode;
			newNode->_parent = parent;
		}
		else
		{
			//kv比parent值小，插入到parent的左边
			parent->_left = newNode;
			newNode->_parent = parent;
		}
		cur = newNode;
 
		//如果父亲存在，且父亲颜色为红就要处理
		while (parent && parent->_col == RED)
		{
			//情况一和情况二、三的区别关键看叔叔
			Node* grandfather = parent->_parent;//当父亲是红色时，根据规则（2）根节点一定是黑色，祖父一定存在
			if (parent == grandfather->_left)//父亲是祖父的左子树
			{
				Node* uncle = grandfather->_right;
				//情况一：叔叔存在且为红
				if (uncle->_col == RED)
				{
					parent->_col = uncle->_col = BLACK;
					grandfather->_col = RED;
 
					//继续向上调整
					cur = grandfather;
					parent = cur->_parent;
				}
				else//情况二+情况三：叔叔不存在或叔叔存在且为黑
				{
					//情况二：单旋
					if (cur == parent->_left)
					{
						RotateR(grandfather);
						parent->_col = BLACK;
						grandfather->_col = RED;
					}
					else//情况三：双旋
					{
						RotateL(parent);
						RotateR(grandfather);
						cur->_col = BLACK;
						grandfather->_col = RED;
					}
					break;//插入结束
				}
			}
			else//父亲是祖父的右子树
			{
				Node* uncle = grandfather->_left;
				//情况一：叔叔存在且为红
				if (uncle && uncle->_col == RED)
				{
					parent->_col = uncle->_col = BLACK;
					grandfather->_col = RED;
 
					//继续往上调整
					cur = grandfather;
					parent = grandfather->_parent;
				}
				else//情况二+情况三：叔叔不存在或叔叔存在且为黑
				{
					//情况二：单旋
					if (cur == parent->_right)
					{
						RotateL(grandfather);
						parent->_col = BLACK;
						grandfather->_col = RED;
					}
					else//情况三：双旋
					{
						RotateR(parent);
						RotateL(grandfather);
						cur->_col = BLACK;
						grandfather->_col = RED;
					}
					break;//插入结束
				}
			}
 
		}
		_root->_col = BLACK;
 
		return make_pair(newNode, true);
	}
 
	void RotateR(Node* parent)
	{
		Node* subL = parent->_left;
		Node* subLR = nullptr;
 
		if (subL)
		{
			subLR = subL->_right;
		}
		//1.左子树的右子树变我的左子树
		parent->_left = subLR;
 
		if (subLR)
		{
			subLR->_parent = parent;
		}
 
		//左子树变父亲
		subL->_right = parent;
		Node* parentParent = parent->_parent;
		parent->_parent = subL;
 
 
		if (parent == _root)//parent是根
		{
			_root = subL;
			_root->_parent = nullptr;
		}
		else//parent不是根，是子树
		{
			if (parentParent->_left == parent)
			{
				//parent是自己父亲的左子树,将subL作为parent父亲的左孩子
				parentParent->_left = subL;
			}
			else
			{
				//parent是自己父亲的右子树,将subL作为parent父亲的右孩子
				parentParent->_right = subL;
			}
 
			//subL的父亲就是parent的父亲
			subL->_parent = parentParent;
		}
	}
 
	void RotateL(Node* parent)
	{
		Node* subR = parent->_right;
		Node* subRL = nullptr;
 
		if (subR)
		{
			subRL = subR->_left;
		}
 
		//1.右子树的左子树变我的右子树
		parent->_right = subRL;
 
		if (subRL)
		{
			subRL->_parent = parent;
		}
 
		//2.右子树变父亲
		subR->_left = parent;
		Node* parentParent = parent->_parent;
		parent->_parent = subR;
 
		if (parent == _root)//parent是根
		{
			_root = parent;
			_root->_parent = nullptr;
		}
		else//parent不是根，是子树
		{
			if (parentParent->_left == parent)
			{
				//parent是自己父亲的左子树,将subR作为parent父亲的左孩子
				parentParent->_left = subR;
			}
			else
			{
				//parent是自己父亲的右子树,将subR作为parent父亲的右孩子
				parentParent->_right = subR;
			}
 
			//subR的父亲就是parent的父亲
			subR->_parent = parentParent;
		}
	}
 
	//查找
	Node* Find(const K& key)
	{
		KeyOfT kot;
		Node* cur = _root;
		while (cur)
		{
			if (kot(cur->_data) < key)
			{
				cur = cur->_right;
			}
			else if (kot(cur->_data) > key)
			{
				cur = cur->_left;
			}
			else
			{
				return cur;
			}
		}
		return nullptr;//空树，直接返回
	}
 
	bool _CheckBalance(Node* root, int blackNum, int count)
	{
		if (root == nullptr)
		{
			if (count != blackNum)
			{
				cout << "黑色节点数量不相等" << endl;
				return false;
			}
			return true;
		}
 
		if (root->_col == RED && root->_parent->_col == RED)
		{
			cout << "存在连续红色节点" << endl;
			return false;
		}
 
		if (root->_col == BLACK)
		{
			count++;
		}
 
		return _CheckBalance(root->_left, blackNum, count)
			&& _CheckBalance(root->_right, blackNum, count);
	}
 
	//检查是否平衡
	bool CheckBalance()
	{
		if (_root == nullptr)
		{
			return true;
		}
 
		if (_root->_col == RED)
		{
			cout << "根节点为红色" << endl;
			return false;
		}
 
		//找最左路径做黑色节点数量参考值
		int blackNum = 0;
		Node* left = _root;
		while (left)
		{
			if (left->_col == BLACK)
			{
				blackNum++;
			}
			left = left->_left;
		}
 
		int count = 0;
		return _CheckBalance(_root, blackNum, count);
	}
 
 
	//遍历
	void _InOrder(Node* root)
	{
		if (root == nullptr)
		{
			return;
		}
 
		_InOrder(root->_left);
		cout << root->_kv.first << ":" << root->_kv.second << endl;
		_InOrder(root->_right);
	}
 
	void InOrder()
	{
		_InOrder(_root);
		cout << endl;
	}
private:
	Node* _root;
};
```

## 七、验证代码

```c++
#pragma once
#include "RBTree.h"
#include <vector>
#include <stdlib.h>
#include <time.h>
#include "Map.h"
#include "Set.h"
 
int main()
{
	delia::map<int, int> m;
	m.insert(make_pair(1, 1));
	m.insert(make_pair(3, 3));
	m.insert(make_pair(0, 0));
	m.insert(make_pair(9, 9));
 
 
	delia::set<int> s;
	s.insert(1);
	s.insert(5);
	s.insert(2);
	s.insert(1);
	s.insert(13);
	s.insert(0);
	s.insert(15);
	s.insert(18);
 
 
	delia::set<int>::iterator sit = s.begin();
	while (sit != s.end())
	{
		cout << *sit << " ";
		++sit;
	}
	cout << endl;
 
 
	return 0;
}
```

