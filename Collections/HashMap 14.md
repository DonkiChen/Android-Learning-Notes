# HashMap jdk 14
_version: 14_
___

## 构造函数

`initialCapacity`: 初始容量, 赋值给了 `threshold`, 在这里调用 `tableSizeFor` 调整为>=该数的2的n次幂了; 如果没有设置, threshold为0, 在初始化的时候会使用默认初始容量16
`loadFactor`: 负载因子

## Node

## TreeNode

继承树: `Map.Entry` -> `HashMap.Node` -> `LinkedHashMap.Entry` -> `HashMap.TreeNode`

`HashMap.Node`: 就是原来jdk7 中的 `HashMap.Entry`, 字段有: `hash`, `key`, `value`, `next`
`LinkedHashMap.Entry`: 因为要维护插入/访问顺序, 所以是双向链表, 新增字段: `after`, `before`
`HashMap.TreeNode`: 因为变成了红黑树, 新增字段: `parent`, `left`, `right`, `red`; 并且新增了 `prev`, 同时又是一个双向链表

## put

```Java
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        //p是哈希表中当前hashCode对应下标的值
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            //如果当前哈希表没有进行初始化, 则先调用resize()进行初始化
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            //如果当前下标空的, 直接放入一个Node
            tab[i] = newNode(hash, key, value, null);
        else {
            //如果不为空
            //e 是当前key对应的Node, 可能为空
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                //队头元素就是key
                e = p;
            else if (p instanceof TreeNode)
                //如果发生了冲突, 且冲突次数过多已经变成了红黑树, 将节点添加到红黑树中(key不存在时)或者返回(key存在时)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                //发生了冲突, 但是冲突次数不够多, 还是一个链表结构
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        //如果到队尾了还没有找到key, 则构造Node, 放到队尾
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            //如果链表数量达到8个, 就将数据结构转为红黑树(如果表大小小于64 先扩容)
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        //当前key存在
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                //如果当前key存在, 更新value
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

### 树化
```Java
    final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            //如果表大小小于64 先扩容
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            do {
                //根据Node生成TreeNode
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            //经过上面变成了由TreeNode构成的链表, hd = head 头指针, tl = tail 尾指针
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }
    
    //HashMap.TreeNode中的代码
    final void treeify(Node<K,V>[] tab) {
        TreeNode<K,V> root = null;
        //变成红黑树
        for (TreeNode<K,V> x = this, next; x != null; x = next) {
            next = (TreeNode<K,V>)x.next;
            x.left = x.right = null;
            if (root == null) {
                x.parent = null;
                x.red = false;
                root = x;
            }
            else {
                K k = x.key;
                int h = x.hash;
                Class<?> kc = null;
                for (TreeNode<K,V> p = root;;) {
                    int dir, ph;
                    K pk = p.key;
                    if ((ph = p.hash) > h)
                        dir = -1;
                    else if (ph < h)
                        dir = 1;
                    else if ((kc == null &&
                              (kc = comparableClassFor(k)) == null) ||
                             (dir = compareComparables(kc, k, pk)) == 0)
                        dir = tieBreakOrder(k, pk);
                    TreeNode<K,V> xp = p;
                    if ((p = (dir <= 0) ? p.left : p.right) == null) {
                        x.parent = xp;
                        if (dir <= 0)
                            xp.left = x;
                        else
                            xp.right = x;
                        root = balanceInsertion(root, x);
                        break;
                    }
                }
            }
        }
        //将根节点放到表的第一个位置
        moveRootToFront(tab, root);
    }
    //将根节点放到表的第一个位置
    static <K,V> void moveRootToFront(Node<K,V>[] tab, TreeNode<K,V> root) {
        int n;
        //双向链表将某个节点移动到头节点
        if (root != null && tab != null && (n = tab.length) > 0) {
            int index = (n - 1) & root.hash;
            TreeNode<K,V> first = (TreeNode<K,V>)tab[index];
            if (root != first) {
                Node<K,V> rn;
                tab[index] = root;
                TreeNode<K,V> rp = root.prev;
                if ((rn = root.next) != null)
                    //将root.next.prev指向root.prev
                    ((TreeNode<K,V>)rn).prev = rp;
                if (rp != null)
                    //将root.prev.next指向root.next
                    rp.next = rn;
                if (first != null)
                    //将first.prev指向root
                    first.prev = root;
                //将root.next指向first
                root.next = first;
                //将root.prev指向null
                root.prev = null;
            }
            assert checkInvariants(root);
        }
    }

```

### 扩容&初始化

在这个版本中 扩容和初始化都在一个resize里
* 如果哈希表没有进行初始化, 则初始化成 threshold 大小, 该版本已经没有 capacity 这个字段了, threshold 表在未初始化时存的是初始容量, 初始化后才会变成阈值
* 如果哈希表进行初始化了, 则进行扩容, 生成2倍大小的新数组

扩容时, 如果拆分出来的链表长度 <= 去树化的阈值(6), 就会被替换成链表形式; 树化 和 去树化一个8, 一个6, 是为了避免频繁树化 和 去树化, 影响性能

```Java
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            //已经初始化过
            if (oldCap >= MAXIMUM_CAPACITY) {
                //如果当前容量大于等于最大容量, 则不扩容, 直接修改阈值为Integer.MAX_VALUE
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                //如果新容量小于最大容量, 且旧容量大于默认初始容量, 则阈值 * 2
                //新阈值 = 新容量 * 负载因子 = 旧容量 * 2 * 负载因子 = 旧阈值 * 2
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            //未初始化的话, 且初始容量指定了, 则初始化为指定初始容量
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            //如果未指定初始容量, 则用默认的
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            //旧表容量小于初始默认容量时, 或者此次时初始化时, 设置新阈值
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    //清除旧标对应下标
                    oldTab[j] = null;
                    if (e.next == null)
                        //如果只有一个节点, 直接赋值
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        //发生了冲突, 已经转变成树了
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        //这一段代码就是把一个单向链表根据条件(该放在旧索引还是新索引) 分成了两个单向链表
                        //发生了冲突, 还是链表结构
                        //索引值较小(旧索引)的头指针, 尾指针
                        Node<K,V> loHead = null, loTail = null;
                        //索引值较大(新索引)的头指针, 尾指针
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                //还是放在原来的下标
                                //例如oldCap = 4 = 100(二进制), 那么在旧表中 3(0011), 7(0111), 11(1011) 都是在下标为3的位置, 可以看出7需要放在新下标, 根据这个算法可以快速得出哪些需要放新下标
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                //放在新下标
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        //经过上面代码 变成了2个链表, 一个是将放在旧索引, 一个将放在新索引
                        if (loTail != null) {
                            //清除原来的引用关系, 避免引用到另一个下标的值
                            loTail.next = null;
                            //放到旧索引
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            //清除原来的引用关系, 避免引用到另一个下标的值
                            hiTail.next = null;
                            //放到新索引
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
    //TreeNode中的方法, 扩容的时候 将一个双向链表(其实同时也是红黑树)分成两个双向链表, 如果两个链表过长( >= 树化的阈值), 则会转为红黑树, 如果过短( <= 去树化的阈值), 则会把节点替换为 `HashMap.Node` 即单向链表的节点
    final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
        TreeNode<K,V> b = this;
        // Relink into lo and hi lists, preserving order
        TreeNode<K,V> loHead = null, loTail = null;
        TreeNode<K,V> hiHead = null, hiTail = null;
        int lc = 0, hc = 0;
        // 下面代码就是将一个双向链表根据条件(是否需要放在新下标) 分为两个新双向链表
        for (TreeNode<K,V> e = b, next; e != null; e = next) {
            next = (TreeNode<K,V>)e.next;
            //直接置空, 不用在最后清除了
            e.next = null;
            if ((e.hash & bit) == 0) {
                if ((e.prev = loTail) == null)
                    loHead = e;
                else
                    loTail.next = e;
                loTail = e;
                ++lc;
            }
            else {
                if ((e.prev = hiTail) == null)
                    hiHead = e;
                else
                    hiTail.next = e;
                hiTail = e;
                ++hc;
            }
        }
        if (loHead != null) {
            if (lc <= UNTREEIFY_THRESHOLD)
                //小于等于去树化阈值 6, 变为单向链表
                tab[index] = loHead.untreeify(map);
            else {
                //大于树化阈值, 变为红黑树
                tab[index] = loHead;
                if (hiHead != null) // (else is already treeified)
                    loHead.treeify(tab);
            }
        }
        if (hiHead != null) {
            if (hc <= UNTREEIFY_THRESHOLD)
                //小于等于去树化阈值 6, 变为单向链表
                tab[index + bit] = hiHead.untreeify(map);
            else {
                //大于树化阈值, 变为红黑树
                tab[index + bit] = hiHead;
                if (loHead != null)
                    hiHead.treeify(tab);
            }
        }
    }
```

## get

```Java
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                //检查第一个是不是要查找的key
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    //去红黑树查找
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    //在单向链表里查找
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

## remove

删除节点, 如果删除后红黑树高度过低, 会变成链表

```Java
    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
            //寻找节点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    //当前位置已经变成了红黑树, 从树中删除对应节点
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)
                    //要找的是第一个节点
                    tab[index] = node.next;
                else
                    //要找的是中间的节点
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }

    final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab,
                          boolean movable) {
        int n;
        //...
        TreeNode<K,V> first = (TreeNode<K,V>)tab[index], root = first, rl;
        //...

        if (root.parent != null)
            root = root.root();
        if (root == null
            || (movable
                && (root.right == null
                    || (rl = root.left) == null
                    || rl.left == null))) {
            //树不够高的时候 去树化
            //根节点为空 或者 根节点没有左子树 或者根节点没有右子树, 或者根节点的左子树没有左子树
            tab[index] = first.untreeify(map);  // too small
            return;
        }
        //...
    }
```

## <= 7 和 > 7 的区别

1. <= 7 冲突解决始终用的是单链表, > 7 超过阈值会变成红黑树
2. 在扩容转移节点时, <= 7 是重新计算下标, > 7 是在当前下标直接加上旧的容量, 快速得出新下标
3. 扩容转移节点链表时, <= 7 是从队头插入(即变成了倒序, 头插法), 把旧表中的节点的next指针指向新表的头节点, > 7 把当前链表分成了2个链表, 保持了原来相对的顺序
4. 扰动函数 > 7 实现更加简单
5. 扩容条件 <= 7:容量大于阈值且发生了碰撞的情况下, > 7:先插入, 插入后如果size超过阈值, 进行扩容