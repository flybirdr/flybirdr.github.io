`Thread`类内部有个字段:
```java
class Thread extends Runnable {

...
    /*
    * ThreadLocal values pertaining to this thread. This map is maintained
    * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals;

...
}
```

`ThreadLocal.ThreadLocalMap`的`get()`和`set()`方法如下：
```java
private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.refersTo(key))
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }
```
```java
private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.refersTo(key)) {
                    e.value = value;
                    return;
                }

                if (e.refersTo(null)) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }

```
可以看到`get()`和`set()`都是以`ThreadLocal`为Key进行操作的，内部会使用哈希表来存储`ThreadLocal`

再看下`ThreadLocal`的`get()`和`set()`

```java
private T get(Thread t) {
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            if (map == ThreadLocalMap.NOT_SUPPORTED) {
                return initialValue();
            } else {
                ThreadLocalMap.Entry e = map.getEntry(this);
                if (e != null) {
                    @SuppressWarnings("unchecked")
                    T result = (T) e.value;
                    return result;
                }
            }
        }
        return setInitialValue(t);
    }
```

```java
private T setInitialValue(Thread t) {
        T value = initialValue();
        ThreadLocalMap map = getMap(t);
        assert map != ThreadLocalMap.NOT_SUPPORTED;
        if (map != null) {
            map.set(this, value);
        } else {
            createMap(t, value);
        }
        if (this instanceof TerminatingThreadLocal<?> ttl) {
            TerminatingThreadLocal.register(ttl);
        }
        return value;
    }
```