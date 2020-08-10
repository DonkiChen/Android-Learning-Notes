# ConCurrentHashMap jdk 7
_version: 7_
___

## 构造函数

`concurrencyLevel`: 并发级别, 最大是65536, 默认是16, 最大并发数量, 也会向上取整为2的n次方, 也就是分段的段数; 最大是65536的原因, 对于一个哈希值, 高16位用于存储在哪个段, 低16位用与存储段中的索引

```Java
    @SuppressWarnings("unchecked")
    public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (concurrencyLevel > MAX_SEGMENTS)
            concurrencyLevel = MAX_SEGMENTS;
        // Find power-of-two sizes best matching arguments
        int sshift = 0;
        //分段数
        int ssize = 1;
        while (ssize < concurrencyLevel) {
            ++sshift;
            ssize <<= 1;
        }
        this.segmentShift = 32 - sshift;
        this.segmentMask = ssize - 1;
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        int c = initialCapacity / ssize;
        if (c * ssize < initialCapacity)
            ++c;
        //分段中每个段哈希表的最小容量
        int cap = MIN_SEGMENT_TABLE_CAPACITY;
        while (cap < c)
            cap <<= 1;
        // create segments and segments[0]
        //这里先创建一个Segment, 可以拿来存 容量, 阈值, 负载系数
        Segment<K,V> s0 =
            new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                             (HashEntry<K,V>[])new HashEntry[cap]);
        Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
        UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
        this.segments = ss;
    }

```

## put

**key和value都不能为null!!!**, 因为会带来二义性
1. 为何key不能为null, 不清楚
2. 如果value为null: 当调用get方法的时候, 有可能是put了一个null的value, 也可能是没有put过这个key, 如果在 `HashMap`, 可以用 `contains` 去判断, 但是在多线程环境下, 你可能此刻还存在, 但是下一刻就不存在了

```Java
    @SuppressWarnings("unchecked")
    public V put(K key, V value) {
        Segment<K,V> s;
        //value不能为空
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key);
        //找出是在哪个Segment
        int j = (hash >>> segmentShift) & segmentMask;
        if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
             (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
             //如果 Segment不存在, 就新建
            s = ensureSegment(j);
        //将键值对放到对应的Segment的哈希表内
        return s.put(key, hash, value, false);
    }

    @SuppressWarnings("unchecked")
    private Segment<K,V> ensureSegment(int k) {
        final Segment<K,V>[] ss = this.segments;
        long u = (k << SSHIFT) + SBASE; // raw offset
        Segment<K,V> seg;
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
            //取第0个Segment, 用它的容量, 负载系数
            Segment<K,V> proto = ss[0]; // use segment 0 as prototype
            int cap = proto.table.length;
            float lf = proto.loadFactor;
            int threshold = (int)(cap * lf);
            HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
            if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                == null) { // recheck
                Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
                //这一块没有加锁, 是用CAS保证线程安全的
                while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                       == null) {
                    //ss[j] == null, 则用CAS设置值
                    if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                        break;
                }
            }
        }
        return seg;
    }

```

### get

```Java
    public V get(Object key) {
        Segment<K,V> s; // manually integrate access methods to reduce overhead
        HashEntry<K,V>[] tab;
        int h = hash(key);
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
            (tab = s.table) != null) {
            //Segment存在, 且用的是 Volatile 语义去取节点
            for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                     (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
                 e != null; e = e.next) {
                K k;
                if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                    //找到了节点
                    return e.value;
            }
        }
        return null;
    }
```

### remove

```Java
    public V remove(Object key) {
        int hash = hash(key);
        //找到Segment
        Segment<K,V> s = segmentForHash(hash);
        //删除
        return s == null ? null : s.remove(key, hash, null);
    }

```

## Segment

### put

```Java
    //Segment代码
    final V put(K key, int hash, V value, boolean onlyIfAbsent) {
        //尝试获取锁, 如果没拿到就去找key对应的节点, 如果找到了, 返回null, 如果没找到, 返回要插入的新节点
        HashEntry<K,V> node = tryLock() ? null :
            scanAndLockForPut(key, hash, value);
        V oldValue;
        try {
            HashEntry<K,V>[] tab = table;
            int index = (tab.length - 1) & hash;
            HashEntry<K,V> first = entryAt(tab, index);
            for (HashEntry<K,V> e = first;;) {
                if (e != null) {
                    K k;
                    if ((k = e.key) == key ||
                        (e.hash == hash && key.equals(k))) {
                        //找到了节点
                        oldValue = e.value;
                        if (!onlyIfAbsent) {
                            e.value = value;
                            ++modCount;
                        }
                        break;
                    }
                    e = e.next;
                }
                else {
                    //没找到对应节点, 准备插入
                    if (node != null)
                        //node不为空 说明之前尝试获取锁失败, 调用过scanAndLockForPut, 返回了新节点, 从队头插入
                        node.setNext(first);
                    else
                        node = new HashEntry<K,V>(hash, key, value, first);
                    int c = count + 1;
                    if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                        //如果当前容量大于阈值, 并且小于最大容量
                        rehash(node);
                    else
                        //容量没超过阈值 或者 达到了可设置的最大容量, 设置队头节点
                        setEntryAt(tab, index, node);
                    ++modCount;
                    count = c;
                    oldValue = null;
                    break;
                }
            }
        } finally {
            unlock();
        }
        return oldValue;
    }

    //Segment方法 在尝试获取锁失败时, 也不闲着, 如果找到了返回null, 否则返回新节点, 这个函数返回后当前线程肯定持有锁
    private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
        //队头节点
        HashEntry<K,V> first = entryForHash(this, hash);
        HashEntry<K,V> e = first;
        HashEntry<K,V> node = null;
        //-1: 还在寻找key对应的节点; 0: 找到了节点或者已经创建了新节点
        int retries = -1; // negative while locating node
        while (!tryLock()) {
            //没有获取到锁的时候才会找, 如果获取到了就退出循环, 继续走put流程
            HashEntry<K,V> f; // to recheck first below
            if (retries < 0) {
                //如果还在寻找节点
                if (e == null) {
                    //如果在链表里没找到节点
                    if (node == null) // speculatively create node
                        //如果也没有新创建过节点, 就创建一个, 这个节点就是之后要插入队头的节点
                        node = new HashEntry<K,V>(hash, key, value, null);
                    retries = 0;
                }
                else if (key.equals(e.key))
                    //如果找到了节点, 将返回null
                    retries = 0;
                else
                    //继续去下一个节点
                    e = e.next;
            }
            else if (++retries > MAX_SCAN_RETRIES) {
                //超过最大自旋次数, 阻塞
                lock();
                break;
            }
            else if ((retries & 1) == 0 &&
                     (f = entryForHash(this, hash)) != first) {
                //如果对头节点变了, 重新开始寻找
                e = first = f; // re-traverse if entry changed
                retries = -1;
            }
        }
        return node;
    }
```


### 扩容

```Java
        @SuppressWarnings("unchecked")
        private void rehash(HashEntry<K,V> node) {
            HashEntry<K,V>[] oldTable = table;
            int oldCapacity = oldTable.length;
            int newCapacity = oldCapacity << 1;
            threshold = (int)(newCapacity * loadFactor);
            HashEntry<K,V>[] newTable =
                (HashEntry<K,V>[]) new HashEntry[newCapacity];
            int sizeMask = newCapacity - 1;
            for (int i = 0; i < oldCapacity ; i++) {
                HashEntry<K,V> e = oldTable[i];
                if (e != null) {
                    HashEntry<K,V> next = e.next;
                    int idx = e.hash & sizeMask;
                    if (next == null)   //  Single node on list
                        //当前索引只有一个元素
                        newTable[idx] = e;
                    else { // Reuse consecutive sequence at same slot
                        //当前索引有多个元素
                        HashEntry<K,V> lastRun = e;
                        int lastIdx = idx;
                        //直到哪个节点之后 索引都不变了
                        //lastRun 指向的是最后一个索引不变的后一个, 即从lastRun开始后面的索引都不变了
                        //比如有5个节点, 新索引分别是 0 1 0 0 0, 那么指向的是1后面的那个0节点
                        for (HashEntry<K,V> last = next;
                             last != null;
                             last = last.next) {
                            int k = last.hash & sizeMask;
                            if (k != lastIdx) {
                                lastIdx = k;
                                lastRun = last;
                            }
                        }
                        //索引不变的那个节点开始 一整个链表都挪过去
                        newTable[lastIdx] = lastRun;
                        // Clone remaining nodes
                        // 循环直到索引开始不变的那个节点
                        for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                            V v = p.value;
                            int h = p.hash;
                            int k = h & sizeMask;
                            HashEntry<K,V> n = newTable[k];
                            newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                        }
                    }
                }
            }
            //扩容之后加入新节点
            int nodeIndex = node.hash & sizeMask; // add the new node
            node.setNext(newTable[nodeIndex]);
            newTable[nodeIndex] = node;
            table = newTable;
        }
```


### remove

```Java
    final V remove(Object key, int hash, Object value) {
        if (!tryLock())
            scanAndLock(key, hash);
        //拿到锁了以后
        V oldValue = null;
        try {
            HashEntry<K,V>[] tab = table;
            int index = (tab.length - 1) & hash;
            HashEntry<K,V> e = entryAt(tab, index);
            HashEntry<K,V> pred = null;
            while (e != null) {
                K k;
                HashEntry<K,V> next = e.next;
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    //寻找节点并删除
                    V v = e.value;
                    if (value == null || value == v || value.equals(v)) {
                        if (pred == null)
                            setEntryAt(tab, index, next);
                        else
                            pred.setNext(next);
                        ++modCount;
                        --count;
                        oldValue = v;
                    }
                    break;
                }
                pred = e;
                e = next;
            }
        } finally {
            unlock();
        }
        return oldValue;
    }

    private void scanAndLock(Object key, int hash) {
        // similar to but simpler than scanAndLockForPut
        HashEntry<K,V> first = entryForHash(this, hash);
        HashEntry<K,V> e = first;
        int retries = -1;
        while (!tryLock()) {
            HashEntry<K,V> f;
            if (retries < 0) {
                if (e == null || key.equals(e.key))
                    retries = 0;
                else
                    e = e.next;
            }
            else if (++retries > MAX_SCAN_RETRIES) {
                lock();
                break;
            }
            else if ((retries & 1) == 0 &&
                     (f = entryForHash(this, hash)) != first) {
                e = first = f;
                retries = -1;
            }
        }
    }

```

## size

因为变成了分段, 所以 `size()` 需要计算;一开始不获取锁, 如果上一次和这一次modCount总数不一致说明计算期间有改动, 重新计算, 一定次数后如果还是不一致, 就获取所有Segment的锁, 再计算

```Java
    public int size() {
        // Try a few times to get accurate count. On failure due to
        // continuous async changes in table, resort to locking.
        final Segment<K,V>[] segments = this.segments;
        int size;
        boolean overflow; // true if size overflows 32 bits
        long sum;         // sum of modCounts
        long last = 0L;   // previous sum
        int retries = -1; // first iteration isn't retry
        try {
            for (;;) {
                if (retries++ == RETRIES_BEFORE_LOCK) {
                    //计算一定次数后操作数还改变了, 说明一直有其他线程在改动, 创建所有的Segment并加锁
                    for (int j = 0; j < segments.length; ++j)
                        ensureSegment(j).lock(); // force creation
                }
                sum = 0L;
                size = 0;
                overflow = false;
                //计算所有的数量
                for (int j = 0; j < segments.length; ++j) {
                    Segment<K,V> seg = segmentAt(segments, j);
                    if (seg != null) {
                        //保存改动数
                        sum += seg.modCount;
                        int c = seg.count;
                        if (c < 0 || (size += c) < 0)
                            overflow = true;
                    }
                }
                //如果这一次和上一次操作数没有变, 说明计算成功, 否则继续计算
                if (sum == last)
                    break;
                last = sum;
            }
        } finally {
            if (retries > RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    segmentAt(segments, j).unlock();
            }
        }
        return overflow ? Integer.MAX_VALUE : size;
    }
```