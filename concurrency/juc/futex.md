引入背景



实现，为什么能解决这个问题

### futex_wait

如何确保挂起进程时，val的值是没有被其他进程修改过的？

Linux 5.1 version

代码在kernel/futex.c中

```c++
 * CPU 0                               CPU 1
 * val = *futex;
 * sys_futex(WAIT, futex, val);
 *   futex_wait(futex, val);
 *   uval = *futex;
 *                                     *futex = newval;
 *                                     sys_futex(WAKE, futex);
 *                                       futex_wake(futex);
 *                                       if (queue_empty())
 *                                         return;
 *   if (uval == val)
 *      lock(hash_bucket(futex));
 *      queue();
 *     unlock(hash_bucket(futex));
 *     schedule();

 * CPU 0                                 CPU 1
 * val = *futex;
 * sys_futex(WAIT, futex, val);
 *   futex_wait(futex, val);
 *
 *   waiters++; (a)
 *   smp_mb(); (A) <-- paired with -.
 *                                  |
 *   lock(hash_bucket(futex));      |
 *                                  |
 *   uval = *futex;                 |
 *                                  |        *futex = newval;
 *                                  |        sys_futex(WAKE, futex);
 *                                  |          futex_wake(futex);
 *                                  |
 *                                  `--------> smp_mb(); (B)
 *   if (uval == val)
 *     queue();
 *     unlock(hash_bucket(futex));
 *     schedule();                         if (waiters)
 *                                           lock(hash_bucket(futex));
 *   else                                    wake_waiters(futex);
 *     waiters--; (b)                        unlock(hash_bucket(futex));
```





```c
static int futex_wait(u32 __user *uaddr, unsigned int flags, u32 val,
		      ktime_t *abs_time, u32 bitset)
{
	struct hrtimer_sleeper timeout, *to = NULL;
	struct restart_block *restart;
	struct futex_hash_bucket *hb;
	struct futex_q q = futex_q_init;
	int ret;

	if (!bitset)
		return -EINVAL;
	q.bitset = bitset;

	if (abs_time) {
		to = &timeout;

		hrtimer_init_on_stack(&to->timer, (flags & FLAGS_CLOCKRT) ?
				      CLOCK_REALTIME : CLOCK_MONOTONIC,
				      HRTIMER_MODE_ABS);
		hrtimer_init_sleeper(to, current);
		hrtimer_set_expires_range_ns(&to->timer, *abs_time,
					     current->timer_slack_ns);
	}

retry:
	/*
	 * Prepare to wait on uaddr. On success, holds hb lock and increments
	 * q.key refs.
	 */
	ret = futex_wait_setup(uaddr, val, flags, &q, &hb);
	if (ret)
		goto out;

	/* queue_me and wait for wakeup, timeout, or a signal. */
	futex_wait_queue_me(hb, &q, to);

	/* If we were woken (and unqueued), we succeeded, whatever. */
	ret = 0;
	/* unqueue_me() drops q.key ref */
	if (!unqueue_me(&q))
		goto out;
	ret = -ETIMEDOUT;
	if (to && !to->task)
		goto out;

	/*
	 * We expect signal_pending(current), but we might be the
	 * victim of a spurious wakeup as well.
	 */
	if (!signal_pending(current))
		goto retry;

	ret = -ERESTARTSYS;
	if (!abs_time)
		goto out;

	restart = &current->restart_block;
	restart->fn = futex_wait_restart;
	restart->futex.uaddr = uaddr;
	restart->futex.val = val;
	restart->futex.time = *abs_time;
	restart->futex.bitset = bitset;
	restart->futex.flags = flags | FLAGS_HAS_TIMEOUT;

	ret = -ERESTART_RESTARTBLOCK;

out:
	if (to) {
		hrtimer_cancel(&to->timer);
		destroy_hrtimer_on_stack(&to->timer);
	}
	return ret;
}
```

```c
static int futex_wait_setup(u32 __user *uaddr, u32 val, unsigned int flags,
			   struct futex_q *q, struct futex_hash_bucket **hb)
{
	u32 uval;
	int ret;
  
retry:
  // 根据用户空间变量的地址计算出一个key，存储在 &q->key 中
	ret = get_futex_key(uaddr, flags & FLAGS_SHARED, &q->key, FUTEX_READ);
	if (unlikely(ret != 0))
		return ret;

retry_private:
  /*
   * 1. hash &q->key 得到 futex_hash_bucket
   * 2. futex_hash_bucket.waiters + 1
   * 3. futex_hash_bucket 加自旋锁
   */
	*hb = queue_lock(q);

  // 原子获取 uaddr 值，赋给 uval
	ret = get_futex_value_locked(&uval, uaddr);

	if (ret) {
		queue_unlock(*hb);

		ret = get_user(uval, uaddr);
		if (ret)
			goto out;

		if (!(flags & FLAGS_SHARED))
			goto retry_private;

		put_futex_key(&q->key);
		goto retry;
	}

  // 用户空间的值如果被修改
	if (uval != val) {
		queue_unlock(*hb);
		ret = -EWOULDBLOCK;
	}

out:
	if (ret)
		put_futex_key(&q->key);
	return ret;
}


/* The key must be already stored in q->key. */
static inline struct futex_hash_bucket *queue_lock(struct futex_q *q)
	__acquires(&hb->lock)
{
	struct futex_hash_bucket *hb;

	hb = hash_futex(&q->key);

	hb_waiters_inc(hb); /* implies smp_mb(); (A) */

	q->lock_ptr = &hb->lock;

	spin_lock(&hb->lock);
	return hb;
}

struct futex_hash_bucket {
	atomic_t waiters;
	spinlock_t lock; // 自旋锁
	struct plist_head chain;
} ____cacheline_aligned_in_smp;

/*
 * futex_hash_bucket 链表
 */
static struct {
	struct futex_hash_bucket *queues;
	unsigned long            hashsize;
} __futex_data;
#define futex_queues   (__futex_data.queues)
#define futex_hashsize (__futex_data.hashsize)
```



```c++
/*
 * Wake up waiters matching bitset queued on this futex (uaddr).
 */
static int futex_wake(u32 __user *uaddr, unsigned int flags, int nr_wake, u32 bitset)
{
	struct futex_hash_bucket *hb;
	struct futex_q *this, *next;
	union futex_key key = FUTEX_KEY_INIT;
	int ret;
  // 需要唤醒的任务链表
	DEFINE_WAKE_Q(wake_q);

	if (!bitset)
		return -EINVAL;
	// 计算key
	ret = get_futex_key(uaddr, flags & FLAGS_SHARED, &key, FUTEX_READ);
	if (unlikely(ret != 0))
		goto out;
	// hash(key) 得到 futex_hash_bucket
	hb = hash_futex(&key);

	/* Make sure we really have tasks to wakeup */
	if (!hb_waiters_pending(hb))
		goto out_put_key;

	spin_lock(&hb->lock);
	// 遍历等待列表，唤醒
	plist_for_each_entry_safe(this, next, &hb->chain, list) {
		if (match_futex (&this->key, &key)) {
			if (this->pi_state || this->rt_waiter) {
				ret = -EINVAL;
				break;
			}

			/* Check if one of the bits is set in both bitsets */
			if (!(this->bitset & bitset))
				continue;

			mark_wake_futex(&wake_q, this);
			if (++ret >= nr_wake)
				break;
		}
	}

	spin_unlock(&hb->lock);
	wake_up_q(&wake_q);
out_put_key:
	put_futex_key(&key);
out:
	return ret;
}
```

