# Java 线程隔离ThreadLocal

有关Java线程隔离方面的问题也是面试中必问的一个问题，今天就好好看一下ThreadLocal 实现原理，也是我们在解决多线程数据安全时比较常用的一个手段

## 实现理论

将变量保存为每个线程栈内部的变量，因为线程栈与线程栈之间内部的变量不不会相互影响的，所以就不会存在数据不安全的情况。

## 代码实现

Thread类

```java
ThreadLocal.ThreadLocalMap threadLocals = null;
```

Thread内由此Map 负责维护 Thread 内被隔离的变量

ThreadLocalMap

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

此Map中的key为 ThreadLocal对象，value为存在Thread被隔离变量的值。

ThreadLocal

当使用ThreadLocal set值的时候

```java
    public void set(T value) {
        //获取当前操作线程对象
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            //将值 放置在 当前线程局部变量 ThreadLocalMap 中
            map.set(this, value);
        else
            createMap(t, value);
    }
```

```java
    ThreadLocalMap getMap(Thread t) {
        //获取当前线程局部变量 ThreadLocalMap 
        return t.threadLocals;
    }
```

至此可以发现达到了 线程隔离的效果

当使用 ThreadLocal get值的时候

```java
    public T get() {
        //获取当前线程对象
        Thread t = Thread.currentThread();
        //获取当前线程局部变量 ThreadLocalMap 
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            //从Map中 根据 threadlocal key及this 获取value
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

## 小结

至此大概可以总结出Thread ThreadLocal ThreadLocalMap 之间的关系

Thread 中存在 ThreadLocalMap 的强引用，ThreadLocalMap 中 Entry 存在 key 为 ThreadLocal 的弱引用，以及任意类型 value 的强引用

此处就会出现 内存泄漏 的情况，当一次GC过后，ThreadLocal 会被回收掉变为null，所以ThreadLocalMap 中的key变为null,此时value理应被回收掉，但是如果线程对象没有结束的话，就会存在 Thread -> ThreadLocalMap -> Entry -> (null,value) 的引用链，导致value不会被回收，为了应对这种情况的发生，ThreadLocal 在get 和 set 操作时会将key为null的 Entry 的 value 也变为 null

```java
  private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // expunge entry at staleSlot
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            Entry e;
            int i;
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                //如果key为null，则将对应的value也变为null，同时将长度减一
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
             //省略部分代码
                }
            }
            return i;
        }
```

## 总结

1. Thread -> ThreadLocal.ThreadLocalMap -> Entry -> (weakReferencr<ThreadLocal>,T value)
2. 在 ThreadLocalMap set ，get 时 会执行 key为null 的 Entry 的 value 也变为null ，同时减少Map长度。