# HashMap

底层采用哈希表+链表+红黑树来实现。
一般来说主要关注点在以下几个方面：

HashMap的`put()`方法源码分析：

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //首次put初始化哈希表数组
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //计算索引，使用n-1来与上哈希值可以保证索引不超过数组长度
        //如果没有冲突直接实例化一个节点
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        //发生冲突
        else {
            Node<K,V> e; K k;
            //key相同，用新的值更新哈希表内的旧值
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //如果是红黑树，插入
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            //是链表
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        //链表长度大于等于红黑树化阈值，触发链表转换为红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        //如果哈希表当前长度大于阈值，resize
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }

```

HashMap默认的初始长度为：`static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16`
哈希表的最大长度为：`static final int MAXIMUM_CAPACITY = 1 << 30;`
哈希表的负载因子默认为：`static final float DEFAULT_LOAD_FACTOR = 0.75f;`
链表转化为红黑树长度阈值为：`static final int TREEIFY_THRESHOLD = 8;`
链表转化为红黑树哈希表的最新长度为：`static final int MIN_TREEIFY_CAPACITY = 64;`
红黑树转化为链表的长度阈值为：`static final int UNTREEIFY_THRESHOLD = 6;`


# SparseArray

稀疏数组，与HashMap不同的是SparseArray的key和value是两个数组分开存储的，并且插入和查找操作使用的是二分查找。

```java

public void put(int key, E value) {

        //二分查找
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        if (i >= 0) {
            mValues[i] = value;
        } else {
            i = ~i;

            if (i < mSize && mValues[i] == DELETED) {
                mKeys[i] = key;
                mValues[i] = value;
                return;
            }

            if (mGarbage && mSize >= mKeys.length) {
                gc();

                // Search again because indices may have changed.
                i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
            }

            mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
            mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
            mSize++;
        }
    }
```

# ConcurrentHashMap

ConcurrentHashMap与HashMap的不同之处在于HashMap不是线程安全的。ConcurrentHashMap底层也是采用一个数组实现哈希表。简单看下`put`方法：
```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            //tabAt()内部采用volatile机制取出对应索引的值
            //未冲突的情况
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                //使用cas赋值
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            //待研究
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            //冲突的情况
            else {
                V oldVal = null;
                //synchronized当前索引的元素
                synchronized (f) {
                    //链表
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
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
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        //红黑树
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                        //待研究
                        else if (f instanceof ReservationNode)
                            throw new IllegalStateException("Recursive update");
                    }
                }
                //链表转换为红黑树的阈值与HashMap相同。
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
    
    
    @SuppressWarnings("unchecked")
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }

    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }

    static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
    }
```