# ConCurrentHashMap jdk 14
_version: 14_
___

## 参考链接

因为太难了, 可以看看别人的文章

[https://juejin.im/post/6844903763103186958](https://juejin.im/post/6844903763103186958)

## 差异

1. 在 7 中, 每个Segment持有一部分哈希表, 在 14 中 哈希表是完整的

### 字段

1. `nextTable`: 下一个要使用的哈希表, 仅在扩容的时候使用
2. `baseCount`
3. `sizeCtl`: 表初始化和大小控制, < 0: 表在初始化或者在扩容, -1: 表在初始化, 其他负数情况 = -(1+....线程数量); 当表未创建时, 保存初始大小, 默认为0, 初始化后保存阈值
4. `transferIndex`
5. `cellsBusy`
6. `counterCells`

### View

### TreeBin

继承自 `ConcurrentHashMap.Node`, 只是一个代理类, 不保存节点的key, value, 只是用作维护树节点, 树根节点, 锁状态等. 是红黑树根节点的的代理类, 存在哈希表中, 即哈希表中是红黑树结构的, 存的都是TreeBin类型

**和HashMap不同**, `HashMap` 在树化后 会把红黑树根节点放到哈希表下标处, 而 `TreeBin` 只是改变`root` 指向, 不会导致链表顺序改变, 从而可以在写的同时可以使用**链表顺序**遍历

#### 锁

[参考链接](https://www.jianshu.com/p/e6d151cd1ee1)

#### 字段

1. `waiter`: 等待线程
2. `lockState`: 锁状态
3. `WRITER`: 1, 写锁
4. `WAITER`: 2, 等待写锁
5. `READER`: 4, 有线程在读

#### 锁

```Java
    final Node<K,V> find(int h, Object k) {
        if (k != null) {
            for (Node<K,V> e = first; e != null; ) {
                int s; K ek;
                //如果有写锁 或者 等待状态, 使用slow path, 使用next指针遍历; 因为红黑树的结构可能会变, 但是链表的顺序不会变
                if (((s = lockState) & (WAITER|WRITER)) != 0) {
                    if (e.hash == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                    e = e.next;
                }
                //CAS获取读锁
                else if (U.compareAndSetInt(this, LOCKSTATE, s,
                                             s + READER)) {
                    TreeNode<K,V> r, p;
                    try {
                        p = ((r = root) == null ? null :
                             r.findTreeNode(h, k, null));
                    } finally {
                        Thread w;
                        //如果原来的锁状态是READER|WAITER 并且存在等待线程, 那就唤醒它
                        if (U.getAndAddInt(this, LOCKSTATE, -READER) ==
                            (READER|WAITER) && (w = waiter) != null)
                            LockSupport.unpark(w);
                    }
                    return p;
                }
            }
        }
        return null;
    }

    private final void lockRoot() {
        //尝试获取写锁, 从无锁状态 0 -> 写锁
        if (!U.compareAndSetInt(this, LOCKSTATE, 0, WRITER))
            //获取失败
            contendedLock(); // offload to separate method
    }
    /**
     * Releases write lock for tree restructuring.
     */
    private final void unlockRoot() {
        lockState = 0;
    }
    /**
     * Possibly blocks awaiting root lock.
     */
    private final void contendedLock() {
        boolean waiting = false;
        //自旋/上锁
        for (int s;;) {
            //~WAITER = 101, 如果 lockState & ~WAITER == 0 => lockState == 010 || 000
            //如果没有读锁或者写锁
            if (((s = lockState) & ~WAITER) == 0) {
                //CAS获取写锁
                if (U.compareAndSetInt(this, LOCKSTATE, s, WRITER)) {
                    //如果设置成功了 waiter置空
                    if (waiting)
                        waiter = null;
                    return;
                }
            }
            //有读锁或者写锁, 且不在等待状态
            else if ((s & WAITER) == 0) {
                //CAS加上等待状态
                if (U.compareAndSetInt(this, LOCKSTATE, s, s | WAITER)) {
                    waiting = true;
                    //设置为当前线程
                    waiter = Thread.currentThread();
                }
            }
            else if (waiting)
                //如果是等待状态, 阻塞当前线程
                LockSupport.park(this);
        }
    }
```

## 构造函数

## 初始化

初始化, 用的 volatile + CAS

```Java
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            //未初始化时
            if ((sc = sizeCtl) < 0)
                //其他线程已经在初始化了, 让出cpu 并自旋一会
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSetInt(this, SIZECTL, sc, -1)) {
                //使用CAS把sizeCtl设置为-1, 仅有一个线程能到达初始化, 但是不能保证是否已经初始化, 有可能一个线程刚进循环, 另一个线程已经初始化结束了
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        //重新判空
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }

```

## put

```Java
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh; K fk; V fv;
            if (tab == null || (n = tab.length) == 0)
                //如果没有初始化, 先初始化
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                //如果当前下标没节点
                if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
                    //利用CAS存新节点
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                //当前正在扩容
                tab = helpTransfer(tab, f);
            else if (onlyIfAbsent // check first node without acquiring lock
                     && fh == hash
                     && ((fk = f.key) == key || (fk != null && key.equals(fk)))
                     && (fv = f.val) != null)
                return fv;
            else {
                //当前下标存在节点
                V oldVal = null;
                //锁住头节点
                synchronized (f) {
                    //如果头节点没变过
                    if (tabAt(tab, i) == f) {
                        //并且是有效的哈希值, 经过 扰动函数得到的哈希值不可能是负数, 为负数就是ConcurrentHashMap设计的一些节点
                        if (fh >= 0) {
                            binCount = 1;
                            //普通的替换值 或者存新节点
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key, value);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            //如果变成了红黑树, 去红黑树里替换或存新节点
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                        else if (f instanceof ReservationNode)
                            throw new IllegalStateException("Recursive update");
                    }
                }
                //大于树化阈值就树化
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }

    private final void addCount(long x, int check) {
        //check if <0, don't check resize, if <= 1 only check if uncontended
        //如果 check < 0 不检测扩容, 如果 <= 1, 检查是否无竞争
        CounterCell[] cs; long b, s;
        if ((cs = counterCells) != null ||
            !U.compareAndSetLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell c; long v; int m;
            boolean uncontended = true;
            if (cs == null || (m = cs.length - 1) < 0 ||
                (c = cs[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSetLong(c, CELLVALUE, v = c.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                int rs = resizeStamp(n) << RESIZE_STAMP_SHIFT;
                if (sc < 0) {
                    if (sc == rs + MAX_RESIZERS || sc == rs + 1 ||
                        (nt = nextTable) == null || transferIndex <= 0)
                        break;
                    if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                else if (U.compareAndSetInt(this, SIZECTL, sc, rs + 2))
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
```

## 扩容

### 转移

仅仅在处理单个索引(单个链表/红黑树)的时候加锁了, 而且在put的时候如果发现其他线程正在扩容, 还可以帮忙一起扩容, 并发扩容

```Java
    final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            int rs = resizeStamp(tab.length) << RESIZE_STAMP_SHIFT;
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
                if (sc == rs + MAX_RESIZERS || sc == rs + 1 ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1)) {
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }

    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            transferIndex = n;
        }
        int nextn = nextTab.length;
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            while (advance) {
                int nextIndex, nextBound;
                if (--i >= bound || finishing)
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                else if (U.compareAndSetInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                if (U.compareAndSetInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    //再检查一遍
                    i = n; // recheck before commit
                }
            }
            else if ((f = tabAt(tab, i)) == null)
                //将哈希表中空的 置为 ForwardingNode
                advance = casTabAt(tab, i, null, fwd);
            else if ((fh = f.hash) == MOVED)
                //当前节点已经被处理过了
                advance = true; // already processed
            else {
                synchronized (f) {
                    //如果没被其他线程修改成 ForwardingNode
                    if (tabAt(tab, i) == f) {
                        //分为两个链表, 放到新旧索引
                        Node<K,V> ln, hn;
                        //如果是普通链表节点
                        if (fh >= 0) {
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            //该索引转移完成, 设置成 ForwardingNode
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        //如果是红黑树节点
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            //该索引转移完成, 设置成 ForwardingNode
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else if (f instanceof ReservationNode)
                            throw new IllegalStateException("Recursive update");
                    }
                }
            }
        }
    }

```