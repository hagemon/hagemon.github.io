+++
title = '实现基于LRU的Cache'
date = 2021-02-06 22:58:25
tags = ['Algorithm']
+++

LRU需要满足：

- O(1)时间的读写
- 将最久未被使用的数据剔除
- 剔除操作在写入的时候进行

要满足这些条件，需要具备的数据结构特点：

- 哈希表：O(1)读取
- 链表：最近被使用的在表头

所以使用双向链表+哈希表的组合作为cache结构。

# Python简易实现

使用Python快速实现上述功能，包括数据结点、双向链表和Cache的实现。

## 数据结点

```python
class Node:
    def __init__(self, key, val):
        self.next = None
        self.prev = None
        self.key = key
        self.val = val
```

## 双向链表

```python
class BidirectionalList:
    def __init__(self):
        self.head = None
        self.tail = None

    # 添加结点到链表头
    def add_node(self, node):
        # 链表为空时设置头尾节点
        if not self.head:
            self.head = node
            self.tail = node
            return
        node.next = self.head
        self.head.prev = node
        self.head = node

    # 移除结点并返回
    def remove_node(self, node):
        if node.prev:
            node.prev.next = node.next
        else:
            self.head = node.next
        if node.next:
            node.next.prev = node.prev
        else:
            self.tail = node.prev
        node.prev, node.next = None, None
        return node
```

## Cache实现

```python
class Cache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.b_list = BidirectionalList()
        self.map = {}

    def set(self, key, value):
        node = Node(key=key, val=value)
				if key in self.map:
            self.b_list.add_node(self.b_list.remove_node(node))
        self.map[key] = node
        self.b_list.add_node(node)
        if len(self.map) > self.capacity:
            node = self.b_list.remove_node(self.b_list.tail)
            del self.map[node.key]

    def get(self, key):
        if key not in self.map:
            return None
        node = self.map[key]
        self.b_list.remove_node(node)
        self.b_list.add_node(node)
        return node.val
```

# Swift实现

基于LRU算法，支持范型的Cache实现

## Node

```swift
import Cocoa

final class Node<T>: NSObject {
    var key: String
    var val: T
    var next: Node?
    var prev: Node?
    
    init(key: String, value: T) {
        self.key = key
        self.val = value
        self.next = nil
        self.prev = nil
        super.init()
    }
}
```

## Bidirectional List

```swift
import Cocoa

final class BidirectionalList<T>: NSObject {
    var head: Node<T>?
    var tail: Node<T>?
    var size = 0
    
    final func addFront(node: Node<T>) {
        guard self.size > 0,
              let head = self.head
        else {
            self.head = node
            self.tail = node
            self.size = 1
            return
        }
        head.prev = node
        node.next = head
        self.head = node
        self.size += 1
    }
    
    final func remove(node: Node<T>) -> Node<T>{
        if let prev = node.prev, let next = node.next {
            prev.next = next
            next.prev = prev
        } else if let prev = node.prev {
            prev.next = nil
            self.tail = prev
        } else if let next = node.next {
            next.prev = nil
            self.head = next
        }
        node.prev = nil
        node.next = nil
        return node
    }
    
    final func hit(node: Node<T>) {
        self.addFront(node: self.remove(node: node))
    }
}
```

## Cache

```swift
import Cocoa

final class Cache<T>: NSObject {
    let capacity: Int
    var bList = BidirectionalList<T>()
    var map: [String: Node<T>] = [:]
    
    init(capacity: Int) {
        self.capacity = capacity
    }
    
    final func set(value: T, for key:String) {
        let node = Node(key: key, value: value)
        if self.map.keys.contains(key) {
            self.bList.hit(node: node)
            return
        }
        self.bList.addFront(node: node)
        self.map[key] = node
    }
    
    final func get(key:String) -> T? {
        guard let node = self.map[key] else {
            return nil
        }
        self.bList.hit(node: node)
        return node.val
    }
    
}
```

## Usage

```swift
let cache = Cache<Int>(capacity: 4)

cache.set(value: 3, for: "3")
cache.set(value: 4, for: "4")
print(cache.get(key: "4") ?? "nil")
```
参考文章：
[halfrost的文章](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Go/LRU:LFU_interview.md)