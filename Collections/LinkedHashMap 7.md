# LinkedHashMap jdk 7
_version: 7_
___

有顺序的HashMap

## 构造函数

`accessOrder`: true: 按照访问顺序排序; false: 按照插入顺序排序

## LinkedHashMap.Entry

`before`: 前一个节点的指针

`after`: 后一个节点的指针

重写的父类空方法: `recordAccess()`, `recordRemoval()`

## 哨兵节点

`init()` 方法new了一个哨兵节点 `head`, 它的before指针指向最晚插入的节点, after指针指向最早插入的节点, 构成了一个双向循环链表

## put

只重写了 `createEntry`

```Java
    void createEntry(int hash, K key, V value, int bucketIndex) {
        HashMap.Entry<K,V> old = table[bucketIndex];
        Entry<K,V> e = new Entry<>(hash, key, value, old);
        table[bucketIndex] = e;
        //主要是加了这一句, 其他与HashMap无差别, 将新节点插入到双向链表环中
        e.addBefore(header);
        size++;
    }

    //Entry的代码
    private void addBefore(Entry<K,V> existingEntry) {
        //this是新new出来的节点, 即新插入的节点
        //existingEntry 是哨兵节点

        //新节点的after指向哨兵节点
        after  = existingEntry;
        //新节点的before指向最近插入的节点
        before = existingEntry.before;
        //上一个插入的节点的after指向自身
        before.after = this;
        //哨兵节点的before指向自身
        after.before = this;
    }
```

## remove

因为父类删除的时候有调用 `HashMap.Entry.recordRemoval`, 重写方法,

```Java
    //LinkedHashMap.Entry
    void recordRemoval(HashMap<K,V> m) {
        remove();
    }

    //双向链表删除节点
    private void remove() {
        before.after = after;
        after.before = before;
    }

```

## accessOrder

### put

在调用 `put` , `get` 的时候, 会调用 `Entry.recordAccess`, 如果构造函数传入 `accessOrder = true`, 会将当前节点先从链表中删除, 然后再加到链表前

```Java
    //Entry
    void recordAccess(HashMap<K,V> m) {
        LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
        if (lm.accessOrder) {
            lm.modCount++;
            remove();
            addBefore(lm.header);
        }
    }
```

### lru

在新加节点的时候会判断需不需要删除最早加入的节点, 通过重写 `removeEldestEntry` 来完成这个判断逻辑

```Java
    void addEntry(int hash, K key, V value, int bucketIndex) {
        super.addEntry(hash, key, value, bucketIndex);

        // Remove eldest entry if instructed
        Entry<K,V> eldest = header.after;
        if (removeEldestEntry(eldest)) {
            removeEntryForKey(eldest.key);
        }
    }
```

## LinkedHashIterator

```Java
    private abstract class LinkedHashIterator<T> implements Iterator<T> {
        // 一开始 指针指向了最早插入的节点
        Entry<K,V> nextEntry    = header.after;
        Entry<K,V> lastReturned = null;

        /**
         * The modCount value that the iterator believes that the backing
         * List should have.  If this expectation is violated, the iterator
         * has detected concurrent modification.
         */
        int expectedModCount = modCount;

        public boolean hasNext() {
            return nextEntry != header;
        }

        public void remove() {
            if (lastReturned == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();

            LinkedHashMap.this.remove(lastReturned.key);
            lastReturned = null;
            expectedModCount = modCount;
        }

        Entry<K,V> nextEntry() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (nextEntry == header)
                throw new NoSuchElementException();

            Entry<K,V> e = lastReturned = nextEntry;
            //下一个比较新的节点
            nextEntry = e.after;
            return e;
        }
    }
```