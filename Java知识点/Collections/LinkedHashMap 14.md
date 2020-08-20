# LinkedHashMap jdk 14
_version: 14_
___

## 如何保证顺序

在 7 中是使用了哨兵节点, before指向最新插入的节点, after指向最早插入的节点; 在14, 使用了两个指针, `head` 指向最早插入的节点(最老的节点), `tail` 指向最新插入的节点(最年轻的节点)

## HashMap定义的关于顺序的空方法

```Java
    //节点被访问到了
    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        //如果是按照访问顺序排序 并且当前节点不是最新的, 就把当前节点从链表中放到队尾
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }

    //节点被插入
    void afterNodeInsertion(boolean evict) { // possibly remove eldest
        //evict = false, 表处于创建模式, 即在如果在构造map的时候传入了一个已存在的map, 会传入false, 此时不会删除过早插入的节点
        LinkedHashMap.Entry<K,V> first;
        //如果是put的插入, 并且头节点不为空, 如果需要删除过早的节点, 就去删除, 即 lru
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }

    //节点被删除
    void afterNodeRemoval(Node<K,V> e) { // unlink
        //从双向链表中删除节点
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.before = p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a == null)
            tail = b;
        else
            a.before = b;
    }
```