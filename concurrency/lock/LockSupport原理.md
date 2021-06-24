## LockSupport源码解读

### LockSupport.park

```java
public static void park() {
  UNSAFE.park(false, 0L);
}
```

Unsafe中的实现

```java
public native void park(boolean var1, long var2);
```

unsafe.cpp

```c++
UNSAFE_ENTRY(void, Unsafe_Park(JNIEnv *env, jobject unsafe, jboolean isAbsolute, jlong time))
  UnsafeWrapper("Unsafe_Park");
  EventThreadPark event;
	// 省略代码...
  JavaThreadParkedState jtps(thread, time != 0);
	// 关键代码，thread是native层的JavaThread对象，然后调用Parker的park方法，继续跟进去linux平台的os_linux.cpp的实现
  thread->parker()->park(isAbsolute != 0, time);
	// 省略代码...
  if (event.should_commit()) {
    oop obj = thread->current_park_blocker();
    event.set_klass((obj != NULL) ? obj->klass() : NULL);
    event.set_timeout(time);
    event.set_address((obj != NULL) ? (TYPE_ADDRESS) cast_from_oop<uintptr_t>(obj) : 0);
    event.commit();
  }
UNSAFE_END
```

os_linux.cpp

```c++
void Parker::park(bool isAbsolute, jlong time) {
  // 先原子的将_counter的值设为0，并返回_counter的原值，如果原值>0说明有通行证，直接返回
  if (Atomic::xchg(0, &_counter) > 0) return;

  Thread* thread = Thread::current();
  assert(thread->is_Java_thread(), "Must be JavaThread");
  JavaThread *jt = (JavaThread *)thread;

  // 判断线程是否已经被中断
  if (Thread::is_interrupted(thread, false)) {
    return;
  }

  // Next, demultiplex/decode time arguments
  timespec absTime;
  if (time < 0 || (isAbsolute && time == 0) ) { // don't wait at all
    return;
  }
  if (time > 0) {
    unpackTime(&absTime, isAbsolute, time);
  }


  // Enter safepoint region
  // Beware of deadlocks such as 6317397.
  // The per-thread Parker:: mutex is a classic leaf-lock.
  // In particular a thread must never block on the Threads_lock while
  // holding the Parker:: mutex.  If safepoints are pending both the
  // the ThreadBlockInVM() CTOR and DTOR may grab Threads_lock.
  ThreadBlockInVM tbivm(jt);

  // 再次判断线程是否被中断，如果没有被中断，尝试获得互斥锁，如果获取失败，直接返回
  if (Thread::is_interrupted(thread, false) || pthread_mutex_trylock(_mutex) != 0) {
    return;
  }

  // 此时互斥锁已上锁
  int status ;
  // 这里再次检查_counter的值，如果_counter > 0, 不需要等待
  if (_counter > 0)  { // no wait needed
    _counter = 0;
    status = pthread_mutex_unlock(_mutex);
    assert (status == 0, "invariant") ;
    // Paranoia to ensure our locked and lock-free paths interact
    // correctly with each other and Java-level accesses.
    OrderAccess::fence();
    return;
  }

	// 省略assert代码

  OSThreadWaitState osts(thread->osthread(), false /* not Object.wait() */);
  jt->set_suspend_equivalent();
  // cleared by handle_special_suspend_equivalent_condition() or java_suspend_self()

  assert(_cur_index == -1, "invariant");
  if (time == 0) {
    _cur_index = REL_INDEX;
    // 让线程等待_cond[_cur_index]信号，到这里线程进入等待状态
    // ？？？？？这里的_mutex并没有释放，其他线程如何获得该互斥锁
    status = pthread_cond_wait (&_cond[_cur_index], _mutex) ;
  } else {
    _cur_index = isAbsolute ? ABS_INDEX : REL_INDEX;
    // 线程进入有超时时间的等待，内部实现调用了pthread_cond_timedwait系统调用
    status = os::Linux::safe_cond_timedwait (&_cond[_cur_index], _mutex, &absTime) ;
    if (status != 0 && WorkAroundNPTLTimedWaitHang) {
      pthread_cond_destroy (&_cond[_cur_index]) ;
      pthread_cond_init    (&_cond[_cur_index], isAbsolute ? NULL : os::Linux::condAttr());
    }
  }
  _cur_index = -1;

  // 省略assert代码

  _counter = 0 ;
  status = pthread_mutex_unlock(_mutex) ;
  assert_status(status == 0, status, "invariant") ;
  // Paranoia to ensure our locked and lock-free paths interact
  // correctly with each other and Java-level accesses.
  OrderAccess::fence();

  // If externally suspended while waiting, re-suspend
  if (jt->handle_special_suspend_equivalent_condition()) {
    jt->java_suspend_self();
  }
}
```

### LockSupport.unpark

```java
public static void unpark(Thread thread) {
  if (thread != null)
    UNSAFE.unpark(thread);
}
```

Unsafe中的实现

```java
public native void unpark(Object thread);
```

unsafe.cpp

```c++
UNSAFE_ENTRY(void, Unsafe_Unpark(JNIEnv *env, jobject unsafe, jobject jthread))
  UnsafeWrapper("Unsafe_Unpark");
  Parker* p = NULL;
  if (jthread != NULL) {
    oop java_thread = JNIHandles::resolve_non_null(jthread);
    if (java_thread != NULL) {
      jlong lp = java_lang_Thread::park_event(java_thread);
      if (lp != 0) {
        // This cast is OK even though the jlong might have been read
        // non-atomically on 32bit systems, since there, one word will
        // always be zero anyway and the value set is always the same
        p = (Parker*)addr_from_java(lp);
      } else {
        // Grab lock if apparently null or using older version of library
        MutexLocker mu(Threads_lock);
        java_thread = JNIHandles::resolve_non_null(jthread);
        if (java_thread != NULL) {
          JavaThread* thr = java_lang_Thread::thread(java_thread);
          if (thr != NULL) {
            p = thr->parker();
            if (p != NULL) { // Bind to Java thread for next time.
              java_lang_Thread::set_park_event(java_thread, addr_to_java(p));
            }
          }
        }
      }
    }
  }
  if (p != NULL) {
#ifndef USDT2
    HS_DTRACE_PROBE1(hotspot, thread__unpark, p);
#else /* USDT2 */
    HOTSPOT_THREAD_UNPARK(
                          (uintptr_t) p);
#endif /* USDT2 */
    p->unpark();
  }
UNSAFE_END
```



```java
void Parker::unpark() {
  int s, status ;
  status = pthread_mutex_lock(_mutex);
  assert (status == 0, "invariant") ;
  s = _counter;
  _counter = 1;
  if (s < 1) {
    // thread might be parked
    if (_cur_index != -1) {
      // thread is definitely parked
      if (WorkAroundNPTLTimedWaitHang) {
        status = pthread_cond_signal (&_cond[_cur_index]);
        assert (status == 0, "invariant");
        status = pthread_mutex_unlock(_mutex);
        assert (status == 0, "invariant");
      } else {
        status = pthread_mutex_unlock(_mutex);
        assert (status == 0, "invariant");
        status = pthread_cond_signal (&_cond[_cur_index]);
        assert (status == 0, "invariant");
      }
    } else {
      pthread_mutex_unlock(_mutex);
      assert (status == 0, "invariant") ;
    }
  } else {
    pthread_mutex_unlock(_mutex);
    assert (status == 0, "invariant") ;
  }
}
```

总结：

通过一个用户空间原子变量控制并发的park，



## Linux的timedwait实现

### pthread_mutex_trylock

先查看pthread_mutex_t的数据结构：

```c++
# pthreadtypes.h
typedef union
{
  struct __pthread_mutex_s __data;
  char __size[__SIZEOF_PTHREAD_MUTEX_T];
  long int __align;
} pthread_mutex_t;

struct __pthread_mutex_s
{
  int __lock; // 锁变量, 传给系统调用futex,用作用户空间的锁变量
  unsigned int __count; // 可重入的计数
  int __owner; // 被哪个线程占有了
  int __kind; // 是否进程间共享,等等...

  // 省略部分条件编译语句，留下主要的属性
  unsigned int __nusers;
  int __spins;
  __pthread_list_t __list;
}

```

代码在glibc的`pthread_mutex_trylock.c`文件中，

```c++
int __pthread_mutex_trylock (pthread_mutex_t *mutex)
{
  int oldval;
  pid_t id = THREAD_GETMEM (THREAD_SELF, tid);

  /* See concurrency notes regarding mutex type which is loaded from __kind
     in struct __pthread_mutex_s in sysdeps/nptl/bits/thread-shared-types.h.  */
  switch (__builtin_expect (PTHREAD_MUTEX_TYPE_ELISION (mutex),
			    PTHREAD_MUTEX_TIMED_NP))
  {
    // 省略其他类型
    case PTHREAD_MUTEX_TIMED_NP:
    	FORCE_ELISION (mutex, goto elision);
    case PTHREAD_MUTEX_ADAPTIVE_NP:
    case PTHREAD_MUTEX_ERRORCHECK_NP:
      if (lll_trylock (mutex->__data.__lock) != 0)
				break;

      /* 记录持有锁的任务的 pid */
      mutex->__data.__owner = id;
      ++mutex->__data.__nusers;

      return 0;
    ...
  }
}

lowlevellock.h,cas操作，将ptr的值 0 -》 1，成功返回0，失败返回-1
/* Try to acquire the lock at PTR, without blocking.
   Evaluates to zero on success.  */
#define lll_trylock(ptr)   \
  ({   \
     int *__iptr = (int *)(ptr);   \
     *__iptr == 0   \
       && atomic_compare_and_exchange_bool_acq (__iptr, 1, 0) == 0 ? 0 : -1;   \
   })
```

### `pthread_mutex_unlock`

```c++
int __pthread_mutex_unlock (pthread_mutex_t *mutex)
{
  return __pthread_mutex_unlock_usercnt (mutex, 1);
}
weak_alias (__pthread_mutex_unlock, pthread_mutex_unlock)
  
int attribute_hidden __pthread_mutex_unlock_usercnt (pthread_mutex_t *mutex, int decr)
{
  /* See concurrency notes regarding mutex type which is loaded from __kind
     in struct __pthread_mutex_s in sysdeps/nptl/bits/thread-shared-types.h.  */
  int type = PTHREAD_MUTEX_TYPE_ELISION (mutex);
  if (__builtin_expect (type
			& ~(PTHREAD_MUTEX_KIND_MASK_NP
			    |PTHREAD_MUTEX_ELISION_FLAGS_NP), 0))
    return __pthread_mutex_unlock_full (mutex, decr);

  if (__builtin_expect (type, PTHREAD_MUTEX_TIMED_NP)
      == PTHREAD_MUTEX_TIMED_NP)
    {
    normal:
    	// 重置锁的所有者
      mutex->__data.__owner = 0;
      if (decr)
				--mutex->__data.__nusers;

      // 解锁，锁变量有三个值，trylock 成功会置为 1，0表示锁空闲，2 表示有其他线程等待锁，需要唤醒
      lll_unlock (mutex->__data.__lock, PTHREAD_MUTEX_PSHARED (mutex));

      LIBC_PROBE (mutex_release, 1, mutex);

      return 0;
    }
  
  // 省略其他类型锁的代码
}

/* Release the lock at PTR.  */
#define lll_unlock(ptr, flags)   \
  ({   \
     int *__iptr = (int *)(ptr);   \
     if (atomic_exchange_rel (__iptr, 0) == 2)   \
       lll_wake (__iptr, (flags));   \
     (void)0;   \
   })
#endif
```

`pthread_mutex_lock`

```c++
int __pthread_mutex_lock (pthread_mutex_t *mutex)
{
  /* See concurrency notes regarding mutex type which is loaded from __kind
     in struct __pthread_mutex_s in sysdeps/nptl/bits/thread-shared-types.h.  */
  unsigned int type = PTHREAD_MUTEX_TYPE_ELISION (mutex);

  LIBC_PROBE (mutex_entry, 1, mutex);

  if (__builtin_expect (type & ~(PTHREAD_MUTEX_KIND_MASK_NP
				 | PTHREAD_MUTEX_ELISION_FLAGS_NP), 0))
    return __pthread_mutex_lock_full (mutex);

  if (__glibc_likely (type == PTHREAD_MUTEX_TIMED_NP))
    {
      FORCE_ELISION (mutex, goto elision);
    simple:
      /* Normal mutex.  */
      LLL_MUTEX_LOCK (mutex);
      assert (mutex->__data.__owner == 0);
    }
  
   // 省略其他类型锁的代码
}

# define LLL_MUTEX_LOCK(mutex) \
  lll_lock ((mutex)->__data.__lock, PTHREAD_MUTEX_PSHARED (mutex))

#define __lll_lock(futex, private)                                      \
  ((void)                                                               \
   ({                                                                   \
     int *__futex = (futex);                                            \
     // 这里判断之前当前锁能不能被独占获取，存在锁冲突的话调用futex系统调用阻塞线程
     if (__glibc_unlikely                                               \
         (atomic_compare_and_exchange_bool_acq (__futex, 1, 0)))        \
       {                                                                \
         if (__builtin_constant_p (private) && (private) == LLL_PRIVATE) \
           __lll_lock_wait_private (__futex);                           \
         else                                                           \
           __lll_lock_wait (__futex, private);                          \
       }                                                                \
   }))
#define lll_lock(futex, private)	\
  __lll_lock (&(futex), private)
     
     
void
__lll_lock_wait_private (int *futex)
{
  // 判断 futex 是否为2，即是否有锁冲突
  if (atomic_load_relaxed (futex) == 2)
    goto futex;
	
  // 将 futex 设置为2
  while (atomic_exchange_acquire (futex, 2) != 0)
    {
    futex:
      LIBC_PROBE (lll_lock_wait_private, 1, futex);
      lll_futex_wait (futex, 2, LLL_PRIVATE); /* Wait if *futex == 2.  */
    }
}
```

### `pthread_cond_wait`

```c++
static __always_inline int
__pthread_cond_wait_common (pthread_cond_t *cond, pthread_mutex_t *mutex,
    clockid_t clockid,
    const struct timespec *abstime)
{
  
  	// 首先释放mutex锁
    err = __pthread_mutex_unlock_usercnt (mutex, 0);
    if (__glibc_unlikely (err != 0))
      {
        __condvar_cancel_waiting (cond, seq, g, private);
        __condvar_confirm_wakeup (cond, private);
        return err;
      }
  
}
```

```c++
typedef union
{
  struct __pthread_cond_s __data;
  char __size[__SIZEOF_PTHREAD_COND_T];
  __extension__ long long int __align;
} pthread_cond_t;

struct __pthread_cond_s
{
  __extension__ union
  {
    __extension__ unsigned long long int __wseq;
    struct
    {
      unsigned int __low;
      unsigned int __high;
    } __wseq32;
  };
  __extension__ union
  {
    __extension__ unsigned long long int __g1_start;
    struct
    {
      unsigned int __low;
      unsigned int __high;
    } __g1_start32;
  };
  unsigned int __g_refs[2] __LOCK_ALIGNMENT;
  unsigned int __g_size[2];
  unsigned int __g1_orig_size;
  unsigned int __wrefs;
  unsigned int __g_signals[2];
};
```



为什么cond 需要mutex 锁？

