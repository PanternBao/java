HashTable性能差主要是由于所有操作需要竞争同一把锁，而如果容器中有多把锁，每一把锁锁一段数据，这样在多线程访问时不同段的数据时，就不会存在锁竞争了，这样便可以有效地提高并发效率。这就是ConcurrentHashMap所采用的"**分段锁**"思想。

### JDK1.8的实现

JDK1.8的实现已经摒弃了Segment的概念，而是直接用Node数组+链表+红黑树的数据结构来实现，并发控制使用Synchronized和CAS来操作，整个看起来就像是优化过且线程安全的HashMap，虽然在JDK1.8中还能看到Segment的数据结构，但是已经简化了属性，只是为了兼容旧版本



```java
private final Node<K,V>[] initTable() {
  Node<K,V>[] tab; int sc;
  while ((tab = table) == null || tab.length == 0) {
    if ((sc = sizeCtl) < 0)
      Thread.yield(); // lost initialization race; just spin
    // CAS操作SIZECTL为-1，表示初始化状态
    else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
      try {
        if ((tab = table) == null || tab.length == 0) {
          int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
          Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
          table = tab = nt;
          // 保留为 3 / 4
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



```java
public V put(K key, V value) {
  return putVal(key, value, false);
}

/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
  if (key == null || value == null) throw new NullPointerException();
  // 0. 计算 key 的 hash 值
  int hash = spread(key.hashCode());
  int binCount = 0;
  for (Node<K,V>[] tab = table;;) {
    Node<K,V> f; int n, i, fh;
    // 1. 初始化 table
    if (tab == null || (n = tab.length) == 0)
      tab = initTable();
    // 2. 若当前数组索引无值，直接创建
    else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
      // CAS 在索引 i 处创建新的节点，当索引 i 为 null 时，即能创建成功，结束循环,否则继续自旋
      if (casTabAt(tab, i, null,
                   new Node<K,V>(hash, key, value, null)))
        break;                   // no lock when adding to empty bin
    }
    // 3. 若当前桶为转移节点，表明该桶的点正在扩容，一直等待扩容完成
    else if ((fh = f.hash) == MOVED)
      tab = helpTransfer(tab, f);
    // 4. 当前索引位置有值
    else {
      V oldVal = null;
      // 锁定当前槽点,保证只会有一个线程能对槽点进行修改
      synchronized (f) {
        // 再次判断 i 位置数据有无被修改
        if (tabAt(tab, i) == f) {
          if (fh >= 0) {
            // 新增节点
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
        }
      }
      if (binCount != 0) {
        // 链表数量太长，需要转化为红黑树
        if (binCount >= TREEIFY_THRESHOLD)
          treeifyBin(tab, i);
        if (oldVal != null)
          return oldVal;
        break;
      }
    }
  }
  // 5. check 容器是否需要扩容，如果需要去扩容，调用 transfer 方法扩容
  // 如果已经在扩容中了，check有无完成
  addCount(1L, binCount);
  return null;
}
```



```java
private final void addCount(long x, int check) {
  // 增加size的数量
  CounterCell[] as; long b, s;
  if ((as = counterCells) != null ||
      !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
    CounterCell a; long v; int m;
    boolean uncontended = true;
    if (as == null || (m = as.length - 1) < 0 ||
        (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
        !(uncontended =
          U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
      fullAddCount(x, uncontended);
      return;
    }
    if (check <= 1)
      return;
    s = sumCount();
  }
  // 准备扩容
  if (check >= 0) {
    Node<K,V>[] tab, nt; int n, sc;
    // 当前节点数量是否超过阈值
    while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
           (n = tab.length) < MAXIMUM_CAPACITY) {
      // 与table容量n大小有关的扩容标记，而n是2的幂次，可得知返回值rs对于不同容量大小的table值必然不相同
      // rs 低 16 位是扩容的stamp 1xxxxxx
      int rs = resizeStamp(n);
      // sc < 0 说明有其他线程正在扩容
      if (sc < 0) {
        // sc >>> RESIZE_STAMP_SHIFT： 是否是对同一n的扩容
        // sc == rs + 1 ｜｜ sc == rs + MAX_RESIZERS：扩容线程数达到最大值，sizectl 低 16 位达到0xffff，加一
        // https://bugs.java.com/bugdatabase/view_bug.do?bug_id=JDK-8214427
        if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
            sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
            transferIndex <= 0)
          break;
        // 每有一个线程扩容，sizectl + 1
        if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
          transfer(tab, nt);
      }
      else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                   (rs << RESIZE_STAMP_SHIFT) + 2))
        transfer(tab, null);
      s = sumCount();
    }
  }
}
```





```java
static final int resizeStamp(int n) {
  // RESIZE_STAMP_BITS 16
  return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```





扩容流程

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
  int n = tab.length, stride;
  // 多线程扩容，每核处理的量小于16，则强制赋值16
  if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
    stride = MIN_TRANSFER_STRIDE; // subdivide range
  
  // 第一个线程需要初始化 nextTable
  if (nextTab == null) {
    try {
      @SuppressWarnings("unchecked")
      Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
      nextTab = nt;
    } catch (Throwable ex) {      // try to cope with OOME
      sizeCtl = Integer.MAX_VALUE;
      return;
    }
    nextTable = nextTab;
    // 从原先的数组最后开始转移
    transferIndex = n;
  }
  int nextn = nextTab.length;
  // 扩容时的特殊节点，标明此节点正在进行迁移，扩容期间的元素查找要调用其find()方法在nextTable中查找元素。
  ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
  boolean advance = true;
  // 所有桶是否都已迁移完成。
  boolean finishing = false;
  // 死循环,i 表示下标，bound 表示当前线程可以处理的当前桶区间最小下标
  for (int i = 0, bound = 0;;) {
    Node<K,V> f; int fh;
    // 此循环的作用是确定当前线程要迁移的桶的范围或通过更新i的值确定当前范围内下一个要处理的节点。
    while (advance) {
      int nextIndex, nextBound;
      // --i 将 index 往前推
      if (--i >= bound || finishing)
        advance = false;
      else if ((nextIndex = transferIndex) <= 0) {
        i = -1;
        advance = false;
      }
      // cas 争抢旧table的某一块区域，索引范围 nextBound - nextIndex
      else if (U.compareAndSwapInt
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
      if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
        if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
          return;
        finishing = advance = true;
        //  为什么需要做最后一次检查
        i = n; // recheck before commit
      }
    }
    else if ((f = tabAt(tab, i)) == null)
      advance = casTabAt(tab, i, null, fwd);
    // 如果当前这个桶已经有别的线程在做迁移了（实际上是做完了迁移，也就是最后一次检查），就不需要本线程再做了，此时将advance设置为true，进入下一次循环即可
    else if ((fh = f.hash) == MOVED)
      advance = true; // already processed
    else {
      synchronized (f) {
        if (tabAt(tab, i) == f) {
          Node<K,V> ln, hn;
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
            setTabAt(tab, i, fwd);
            advance = true;
          }
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
            setTabAt(tab, i, fwd);
            advance = true;
          }
        }
      }
    }
  }
}
```


