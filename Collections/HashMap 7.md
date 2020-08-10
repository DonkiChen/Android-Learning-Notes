# HashMap jdk 7
_version: 7_
___

## 构造函数

`initialCapacity`: 初始容量

`loadFactor`: 负载系数, 数量达到扩容阈值(容量*负载系数)时 进行扩容

## Entry

```Java
    static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        //下一个Entry, 产生了哈希碰撞的解决方法, 变成了链表
        Entry<K,V> next;
        int hash;
    }
```

## put
```Java
    public V put(K key, V value) {
        //如果表没有初始化, 则进行初始化
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
        if (key == null)
            //当key为null时, 
            return putForNullKey(value);
        //这里对key的hashCode进行了一个处理, 避免坏的哈希实现方式导致碰撞增加, 称为扰动函数
        int hash = hash(key);
        //找到对应的索引(利用与运算 进行取余操作)
        int i = indexFor(hash, table.length);
        //寻找当前索引有没有Entry存在
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            //当前key已经在map里了
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

        modCount++;
        //添加
        addEntry(hash, key, value, i);
        return null;
    }

    void addEntry(int hash, K key, V value, int bucketIndex) {
        //容量大于阈值且发生了碰撞的情况下
        if ((size >= threshold) && (null != table[bucketIndex])) {
            //扩容
            resize(2 * table.length);
            //重新得到hash
            hash = (null != key) ? hash(key) : 0;
            //取新下标
            bucketIndex = indexFor(hash, table.length);
        }

        createEntry(hash, key, value, bucketIndex);
    }

    void createEntry(int hash, K key, V value, int bucketIndex) {
        //取当前索引的Entry, 可能为null, 如果e!=null, 就发生了碰撞, Entry里有个next指针, 用来解决碰撞
        Entry<K,V> e = table[bucketIndex];
        //从队头插入 e 被传入构造函数里, 新构造出来的Entry.next指向它
        table[bucketIndex] = new Entry<>(hash, key, value, e);
        size++;
    }

```

### 当key为null时

可以存入, 默认hashCode = 0

```Java
    private V putForNullKey(V value) {
        //寻找key.hash == 0 且 key == null 的Entry, 即有冲突的情况下
        for (Entry<K,V> e = table[0]; e != null; e = e.next) {
            if (e.key == null) {
                //更新值
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
        modCount++;
        addEntry(0, null, value, 0);
        return null;
    }
```

### table的初始化
容量变为2的n次方的原因是在 `indexFor` 函数里, 即根据hashCode进行取索引的时候可以直接用位运算来得到, 提高效率

```Java

    static int indexFor(int h, int length) {
        // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
        return h & (length-1);
    }


    private void inflateTable(int toSize) {
        // Find a power of 2 >= toSize
        // 找一个大于等于 toSize 的 2的n次方
        int capacity = roundUpToPowerOf2(toSize);

        //扩容阈值
        threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
        table = new Entry[capacity];
        initHashSeedAsNeeded(capacity);
    }
```

### 扩容

扩容成当前容量的两倍(如果当前容量达到最大值2^30时, 先把阈值改成Integer.MAX_VALUE), 然后new出新表, 把旧表中的数据转移到新表中

```Java

    void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }

        //新的table
        Entry[] newTable = new Entry[newCapacity];
        //转移旧的数据到新的表
        transfer(newTable, initHashSeedAsNeeded(newCapacity));
        table = newTable;
        //计算新阈值
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
    }

    /**
     * Transfers all entries from current table to newTable.
     */
    void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        for (Entry<K,V> e : table) {
            while(null != e) {
                Entry<K,V> next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                //新表中的索引
                int i = indexFor(e.hash, newCapacity);
                //从队头插入
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            }
        }
    }

```

## get

注意的点: 因为要找到Entry需要根据当前的hashCode, 如果key.hashCode发生了改变, 那么就找不到相应Entry了

```Java
    public V get(Object key) {
        if (key == null)
            return getForNullKey();
        Entry<K,V> entry = getEntry(key);

        return null == entry ? null : entry.getValue();
    }

    private V getForNullKey() {
        if (size == 0) {
            return null;
        }
        for (Entry<K,V> e = table[0]; e != null; e = e.next) {
            //找到key == null 的那个 Entry
            if (e.key == null)
                return e.value;
        }
        return null;
    }

    final Entry<K,V> getEntry(Object key) {
        if (size == 0) {
            return null;
        }

        int hash = (key == null) ? 0 : hash(key);
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            //找到 hash 相等 && (e.key == key || key.equals(k)) 的 Entry
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))
                return e;
        }
        return null;
    }

```

## remove

```Java
    public V remove(Object key) {
        Entry<K,V> e = removeEntryForKey(key);
        return (e == null ? null : e.value);
    }

    final Entry<K,V> removeEntryForKey(Object key) {
        if (size == 0) {
            return null;
        }
        int hash = (key == null) ? 0 : hash(key);
        int i = indexFor(hash, table.length);
        Entry<K,V> prev = table[i];
        Entry<K,V> e = prev;

        while (e != null) {
            Entry<K,V> next = e.next;
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k)))) {
                modCount++;
                size--;
                if (prev == e)
                    //如果是从队头删除的, 后面的补上来
                    table[i] = next;
                else
                    //从中间删除
                    prev.next = next;
                e.recordRemoval(this);
                return e;
            }
            prev = e;
            e = next;
        }

        return e;
    }
```

## HashIterator

### `HashMap.keySet().iterator()`, `HashMap.values().iterator()`, `HashMap.entrySet().iterator()` 各自的迭代器

他们三个的迭代器的父类都是 `HashIterator`, 没有实现自己的remove方法, 在调用 `Iterator.next()` -> `HashIterator.nextEntry()` 的时候都能拿到当前的 Entry, 然后再调用外部 `HashMap.removeEntryForKey`

#### 为什么调用 `HashMap.keySet().remove()`, `HashMap.values().remove()`, `HashMap.entrySet().remove()` 能同步修改`HashMap`中的值

* `HashMap.keySet().remove()` 直接调用了 `HashMap.removeEntryForKey()` 传入了 key
* `HashMap.values().remove()` 继承自 `AbstractCollection` 其中 `AbstractCollection.remove` 是调用迭代器去删除
* `HashMap.entrySet().remove()` 直接调用了 `HashMap.removeMapping()` 传入了 Entry

### 构造函数
```Java
    HashIterator() {
        expectedModCount = modCount;
        if (size > 0) { // advance to first entry
            Entry[] t = table;
            //找到第一个非空的Entry
            while (index < t.length && (next = t[index++]) == null)
                ;
        }
    }
```
### next
```Java
    final Entry<K,V> nextEntry() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        Entry<K,V> e = next;
        if (e == null)
            throw new NoSuchElementException();
        //寻找下一个非空的Entry
        if ((next = e.next) == null) {
            Entry[] t = table;
            while (index < t.length && (next = t[index++]) == null)
                ;
        }
        current = e;
        return e;
    }
```

### remove
```Java
    public void remove() {
        if (current == null)
            throw new IllegalStateException();
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        Object k = current.key;
        current = null;
        //调用了外部 HashMap.remove(k)
        HashMap.this.removeEntryForKey(k);
        expectedModCount = modCount;
    }
```

## 多线程情况下可能会发生的成环

扩容过程中: 如果在索引i 有一个节点1, 现在有两个线程T1, T2, e1, e2 现在都指向旧表中的队头元素, T2先被阻塞, T1先执行, 语句1, e1的next指针指向新表中的队头元素; 语句2, 新表中的队头元素改为节点1(这时候新表与旧表对应索引的队头元素都是节点1); 如果这时候T1被阻塞, T2开始执行, e2指向的是节点1, 执行语句1, 指向新表的队头元素, 因为两个队头元素都是节点1, 即节点1.next 指向了自己, 就构成了一个环

```Java
    void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        for (Entry<K,V> e : table) {
            while(null != e) {
                //...
                //从队头插入
                e.next = newTable[i];//1
                newTable[i] = e;//2
                e = next;//3
            }
        }
    }
```