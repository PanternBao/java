jdk 1.8 引入

AtomicLong vs LongAdder



设计思想上：

LongAdder 采用"分段"的方式降低 CAS 失败的频次

用空间来换时间

AtomicLong 中有个内部变 value 保存着实际的`long`值，所有的操作都是针对该变量进行。也就是说，高并发环境下，value 变量其实是一个热点数据，也就是N个线程竞争一个热点。

LongAdder 的基本思路就是分散热点。LongAdder 有一个全局变量 volatile long base 值，当并发不高的情况下都是通过CAS 来直接操作 base 值，如果 CAS 失败，则针对 LongAdder 中的 Cell[] 数组中的 Cell 进行 CAS 操作，减少失败的概率。

例如当前类中 base = 10，有三个线程进行`CAS`原子性的 +1 操作，线程一执行成功，此时base=11，线程二、线程三执行失败后开始针对于 Cell[] 数组中的 Cell 元素进行 +1 操作，同样也是`CAS`操作，此时数组 index=1 和 index=2 中 Cell 的 value 都被设置为了1。执行完成后，统计累加数据：sum = 11 + 1 + 1 = 13，利用 LongAdder 进行累加的操作就执行完了。

add 方法

```java
public void add(long x) {
  Cell[] as; long b, v; int m; Cell a;
  // 在并发比较低的情况下，对 base 进行操作，和 AtomicLong 效果差不多
  if ((as = cells) != null || !casBase(b = base, b + x)) {
    boolean uncontended = true;
    if (as == null || (m = as.length - 1) < 0 ||
        (a = as[getProbe() & m]) == null ||
        !(uncontended = a.cas(v = a.value, v + x)))
      longAccumulate(x, null, uncontended);
  }
}
```

Cell 是 `java.util.concurrent.atomic` 下 `Striped64` 的一个内部类。

longAccumulate初始化cell 数组

```java
final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
  int h;
  if ((h = getProbe()) == 0) {
    ThreadLocalRandom.current(); // force initialization
    h = getProbe();
    wasUncontended = true;
  }
  boolean collide = false;                // True if last slot nonempty
  for (;;) {
    Cell[] as; Cell a; int n; long v;
    // 1. cell 数组已初始化
    if ((as = cells) != null && (n = as.length) > 0) {
      // 1.1 通过 thread hash 索引，cell 为 null
      if ((a = as[(n - 1) & h]) == null) {
        if (cellsBusy == 0) {
          // 乐观创建
          Cell r = new Cell(x);
          if (cellsBusy == 0 && casCellsBusy()) {
            boolean created = false;
            try {
              // Recheck under lock
              Cell[] rs; int m, j;
              if ((rs = cells) != null &&
                  (m = rs.length) > 0 &&
                  rs[j = (m - 1) & h] == null) {
                rs[j] = r;
                created = true;
              }
            } finally {
              cellsBusy = 0;
            }
            if (created)
              break;
            // Slot is now non-empty
            continue;
          }
        }
        collide = false;
      }
      // 1.2 CAS already known to fail
      else if (!wasUncontended)
        // Continue after rehash
        wasUncontended = true;
      // 1.3 CAS cell
      else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                   fn.applyAsLong(v, x))))
        break;
      // 1.4 当cell达到核心数或者已经被扩容，停止扩容
      else if (n >= NCPU || cells != as)
        collide = false;
      // 两次机会都不能cas cell 成功就扩容
      else if (!collide)
        collide = true;
      // 1.5 扩容
      else if (cellsBusy == 0 && casCellsBusy()) {
        try {
          if (cells == as) {      // Expand table unless stale
            Cell[] rs = new Cell[n << 1];
            for (int i = 0; i < n; ++i)
              rs[i] = as[i];
            cells = rs;
          }
        } finally {
          cellsBusy = 0;
        }
        collide = false;
        // Retry with expanded table
        continue;
      }
      // rehash
      h = advanceProbe(h);
    }
    // 2. cell 数组未初始化，通过cellsBusy的cas操作上锁，0和1标识位
    else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
      boolean init = false;
      try {
        // 这里为什么需要二次判断？
        // cells == as && casCellsBusy() 两个判断之间存在间隙
        if (cells == as) {
          // 初始cell数组长度为2
          Cell[] rs = new Cell[2];
          rs[h & 1] = new Cell(x);
          cells = rs;
          init = true;
        }
      } finally {
        cellsBusy = 0;
      }
      if (init)
        break;
    }
    // casbase
    else if (casBase(v = base, ((fn == null) ? v + x :
                                fn.applyAsLong(v, x))))
      break;                          // Fall back on using base
  }
}
```

sum 方法就是将base和每个cell的value叠加

```java
public long sum() {
    Cell[] as = cells; Cell a;
    long sum = base;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

