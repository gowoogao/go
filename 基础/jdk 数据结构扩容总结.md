下面的总结全部基于 jdk1.8.

| 集合 | 是否允许为 null | 默认初始化大小 | 是否有扩容机制 | 扩容因子 |
| :------: | :------: | :------: | :------: | :------: |
| ArrayList | Y | 10 | Y| — |
| LinkedList | Y | N | N | — |
| ArrayDeque | N | 16 | Y | — |
| PriorityQueue | N | 11 | Y | — |
| HashMap | K,V 都允许为 null  | 16 | Y | 0.75 |
| CopyOnWriteArrayList | Y  | 0 | N | — |
| Hashtable | K,V 都不允许为 null  | 11 | Y | 0.75 |
| TreeMap | K 不允许为 null，V 允许为 null  | — | N | — |
| ConcurrentHashMap | K,V 都不允许为 null  | 16 | Y | 0.75 |
| IdentityHashMap | K,V 都允许为 null  | 32 | Y | 2/3 |

PS：
- 队列元素都不允许为 null
- 当 `HashMap` 哈希表长度大于 64，并且链表长度大于 8 时，才会从链表转树
- 当 `HashMap` 树键值对个数小于 6 时会从树转回链表
- `IdentityHashMap` key 与 value 使用同一个数组，默认初始化容量为 32 是因为用一半的容量存储 key，另一半容量存储 value
- `HashSet` 与 `TreeSet` 见 `HashMap` 与 `TreeMap` 


### 一、扩容机制

**ArrayList**

扩容为原来的 1.5 倍。

``` java
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        // 数组容量被扩大为原来的 1.5 倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        // MAX_ARRAY_SIZE = 2<sup>31</sup>-1-8
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```

**ArrayDeque**

扩容为原来的 2 倍。

``` java
    private void doubleCapacity() {
        // 当尾节点指向头节点的时候才继续执行
        assert head == tail;
        int p = head;
        int n = elements.length;
        // 头结点右边元素数量
        int r = n - p; // number of elements to the right of p
        // 将数组容量扩大 2 倍
        int newCapacity = n << 1;
        if (newCapacity < 0)
            throw new IllegalStateException("Sorry, deque too big");
        Object[] a = new Object[newCapacity];
        // 将原数组中 head 右边的数据拷贝到新数组中
        System.arraycopy(elements, p, a, 0, r);
        //将原数组中 head 左边的元素拷贝到新数组中
        System.arraycopy(elements, 0, a, r, p);
        elements = a;
        // 重置头尾节点
        head = 0;
        tail = n;
    }
```

**PriorityQueue**

如果容量大于 64 扩容为 2 倍 + 2，小于 64 扩容为原来容量的 1.5 倍。

``` java
    private void grow(int minCapacity) {
        int oldCapacity = queue.length;
        // Double size if small; else grow by 50%
        // 当容量小于 64 时容量为原来的两倍 + 2，如果大于等于 64 时扩容为原来的 1.5 倍
        int newCapacity = oldCapacity + ((oldCapacity < 64) ?
                                         (oldCapacity + 2) :
                                         (oldCapacity >> 1));
        // overflow-conscious code
        // 当元素数量非常多时进行单独处理
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // 元素复制
        queue = Arrays.copyOf(queue, newCapacity);
    }
```

**HashMap**

哈希表存在的情况下容量与扩容阈值都扩容为原来的 2 倍。

``` java
        if (oldCap > 0) {
            // 如果哈希表容量已达最大值，不进行扩容，并把阈值置为 0x7fffffff，防止再次调用扩容函数
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // 新容量为原来数组大小的两倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                // 把新的扩容阈值也扩大为两倍
                newThr = oldThr << 1; // double threshold
        }
```

**Hashtable**

容量扩容为原来的 2 倍 + 1，扩容阈值为 `newCapacity * loadFactor`。

``` java
    protected void rehash() {
        // 获取老哈希表容量
        int oldCapacity = table.length;
        Entry<?,?>[] oldMap = table;

        // overflow-conscious code
        // 新哈希表容量为原容量的 2 倍 + 1，与 HashMap 不同
        int newCapacity = (oldCapacity << 1) + 1;
        // 容量很大特殊处理
        if (newCapacity - MAX_ARRAY_SIZE > 0) {
            if (oldCapacity == MAX_ARRAY_SIZE)
                // Keep running with MAX_ARRAY_SIZE buckets
                return;
            newCapacity = MAX_ARRAY_SIZE;
        }
        // 初始化新哈希表
        Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];

        modCount++;
        // 重置扩容阈值，与 HashMap 不同，HashMap 直接把 threshold 也扩大为原来的两倍
        threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
        // 重置哈希表数组
        table = newMap;

        // 从底向上进行 rehash
        for (int i = oldCapacity ; i-- > 0 ;) {
            // 获取旧哈希表对应桶位置上的链表
            for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
                // 链表
                Entry<K,V> e = old;
                // 重置继续遍历
                old = old.next;

                // 获取在新哈希表中的桶位置，逐个进行 rehash
                int index = (e.hash & 0x7FFFFFFF) % newCapacity;
                // 头插法 rehash
                e.next = (Entry<K,V>)newMap[index];
                // 自己做头节点
                newMap[index] = e;
            }
        }
    }
```

**ConcurrentHashMap**

待更新...

**IdentityHashMap**

容量扩容为原来的 2 倍，阈值扩容为新容量的 2/3。

``` java
    private void resize(int newCapacity) {
        // assert (newCapacity & -newCapacity) == newCapacity; // power of 2
        int newLength = newCapacity * 2;

        Object[] oldTable = table;
        int oldLength = oldTable.length;
        // 判断不能无限扩容
        if (oldLength == 2*MAXIMUM_CAPACITY) { // can't expand any further
            if (threshold == MAXIMUM_CAPACITY-1)
                throw new IllegalStateException("Capacity exhausted.");
            threshold = MAXIMUM_CAPACITY-1;  // Gigantic map!
            return;
        }
        if (oldLength >= newLength)
            return;

        /**
         * 如果初始容量为 32，则 len 为 64，threshold = 2/3 * 32
         * 扩容后新的容量为 64，则 len 是 128，threshold = 128 / 3
         */
        Object[] newTable = new Object[newLength];
        // 设置阈值为新容量的 2/3， newLength = 2 * newCapacity，因此只要算 1/3 即可
        threshold = newLength / 3;

        for (int j = 0; j < oldLength; j += 2) {
            Object key = oldTable[j];
            if (key != null) {
                Object value = oldTable[j+1];
                // help GC
                oldTable[j] = null;
                oldTable[j+1] = null;
                // 根据 key 计算桶位置
                int i = hash(key, newLength);
                // 从当前桶位置向后遍历，找出空位置，存放当前键值对
                while (newTable[i] != null)
                    i = nextKeyIndex(i, newLength);
                // rehash 赋值
                newTable[i] = key;
                newTable[i + 1] = value;
            }
        }
        // 重置 table
        table = newTable;
    }
```