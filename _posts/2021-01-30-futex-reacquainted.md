---
layout: blog
title: Futex Reacquainted
category: tech
---

To be fair, the futex (Fast Userspace muTEX) syscall has a rather misleading name, in the sense that it simply isn't a mutex. In the original 2002 paper (Fuss, Futexes and Furwocks: Fast Userlevel Locking in Linux), it was invented specifically to implement a fast mutex, but capable of much more. I prefer to think of futex as a minimal useful subset that can fulfil most high level synchronization constructs.

## Minimal Useful Subset

Before futex was born, people in need of a proper mutex (waiter actually blocks instead of spinning) have to use heavy kernel objects like file lock ([fcntl](https://man7.org/linux/man-pages/man2/fcntl.2.html)) or [System V semaphore](https://man7.org/linux/man-pages/man7/sysvipc.7.html). Since system calls are toxic to performance, kernel developers then seeked to compress kernel involvement in userspace synchronization by inventing futex.

So essentially this design process is a code refactor that encapsulates the thread blocking functionality into a new lightweight syscall. And such blocking can be easily elided when the lock isn't contended.

Here by "lightweight" I mean the resource allocated for a futex should be minimal, so that thousands of futexes could live happily within a commodity setup. This requirement implicitly forbids the use of file descriptor as futex handle.

To meet all these standards, futex provides an interface like this (minimal form):

```c
int futex_wake(int *addr);
int futex_wait(int *addr, int val);
```

Without doubt this is a piece of elegance. The idea of using userspace address as an unique identifier isn't rare in nowaday system code (see boost's [thread local pointer](https://github.com/boostorg/thread/blob/409c98f8b745e72bc326e93bfaf8a353d94a69b0/include/boost/thread/tss.hpp)). What's brilliant is, futex further incorporates the CAS-like (compare and swap) semantics into this address -- the thread is blocked when `*addr` is indeed `val`. This turns out to be essential to enforce the atomicity of user\~kernel transition:

```rust
static int a = NO_SIGNAL;
// properly wait for a signal
let cached_a = atomic_load(&a);
while (cached_a != HAS_SIGNAL) {
  futex_wait(&a, cached_a);
  cached_a = atomic_load(&a);
}
// incorrect version
if (atomic_load(&a) != HAS_SIGNAL) {
  futex_wait_racy(&a);
}
```

In the incorrect version, this thread can block indefinitely when the signal has already arrived at `a`. The conditional loop also defends against spurious wakeups, which further propagates itself to the use of condition variable.

## Technical Details

<div class="ascii-art">
       ...
    ----------
        |          tid1       tid2       tid3
    +--------+      |          |          |
    |spinlock|  ~~~~~~~~~  ---------  ~~~~~~~~~
    |   key  +->| addr1 |->| addr2 |->| addr1 |->...
    +--------+  ~~~~~~~~~  ---------  ~~~~~~~~~
        | 
    ----------
</div>

Kernel uses a fixed-size, seperate chaining, central hash table to store mappings between user provided address and the waiting threads (wait queue). A futex won't occupy any resource until some threads start blocking on it. What puzzles me a little is why they didn't bother to maintain private wait queues for individual futexes, guess the benefit of fewer key comparisons isn't worth the complexity.

Unsurprisingly, this table becomes a performance bottleneck as computing nodes scale out, and poses challenge to system predictability. In a [talk](https://www.youtube.com/watch?v=-8c47dHuGIY) by SUSE engineer, three major issues were identified:

- Not optimized for NUMA access
- Hash table collisions
- Hash bucket spinlock contention

When problems are defined, solutions aren't that far either. Below is a list of the major optimizations made to mitigate these issues over the years:

1. `FUTEX_PRIVATE_FLAG`: fast path for single-process use. (2007 [patch](https://lwn.net/Articles/229668/): new PRIVATE futexes)
<br>
<br>When a futex is declared private within current process, which is true for most cases, there is no need for kernel to generate and maintain a global UID.
<br>
<br>This new command managed to avoid: (a) Taking the mmap_sem semaphore, conflicting with other subsystems. (b) Modifying a ref_count on mm or an inode, still conflicting with mm or fs.
<br>
<br>Benchmark shows 20% less instructions, 4x throughput under 4 threads stress test, 16x under 16 threads.

2. Lockless `get_futex_key()` (a [series](https://lore.kernel.org/patchwork/cover/645514/) of patches)

3. Make hash table size adaptive (2014 [commit](https://github.com/torvalds/linux/commit/a52b89ebb6d4499be38780db8d176c5d3a6fbc17): Increase hash table size for better performance)
<br>
<br>After this patch, global hash table is sized to `256 * cpu_num` instead of `256`.

4. Lockless wakeups (2015 [commit](https://github.com/torvalds/linux/commit/1d0dcb3ad9d336e6d6ee020a750a7f8d907e28de): Implement lockless wakeups)
<br>
<br>Internally acknowledge one or more tasks are to be awoken, then do the actual wakeups outside the critical section.

5. Use MCS lock as bucket spinlock (32-bit qspinlock is [merged](https://github.com/torvalds/linux/commit/a33fda35e3a7655fb7df756ed67822afb5ed5e8d) in 2015)
<br>
<br>Unlike ticket spinlock, one acquires MCS lock by spinning on a private word part of a larger linked queue, which avoids lots of cache line bouncing and remote socket access.

6. Use private hash table for each process ([rfc](https://lore.kernel.org/lkml/49C4D5A0.5020106@cosmosbay.com/t/))

## Romance Between Futex & Condition Variable

I was going to take a closer look at how pthread(NPTL) implements condition variable with futexes, and unveil some of the mysteries that prompted me to write this post in the first place.

To look "closer", I skimmed through the commits of NPTL's condvar from 2002 to 2016. Even though during this period the code skeleton is relatively stable, the minor details kept changing. Plus the commit messages being chaotic, it's almost impossible to reason those changes.

This leaves me no choice but to read the latest implementation from 2016, which is simplified to this pseudocode:

```rust
/** cv's data members **
 * lock: internal mutex
 * wait_seq: sequence number for waiter
 * wakeup_seq: sequence number for wakeup signaled
 * woken_seq: sequence number for woken thread
 * broadcast_seq: sequence number for broadcast signaled
 * mutex_ref: reference to user mutex
 */
fn cond_wait(cv, mutex):
  lock(cv.lock);
  unlock(mutex);
  cv.wait_seq ++;
  cv.futex ++;
  cv.mutex_ref = mutex;
  let wakeup_seq_before = cv.wakeup_seq;
  let broadcast_seq_before = cv.broadcast_seq;
  loop {
    let futexval = cv.futex;
    unlock(cv.lock);
    let ret = FUTEX_WAIT(cv.futex, futexval);
    lock(cv.lock);
    if broadcast_seq_before != cv.broadcast_seq {
      break;
    }
    if cv.wakeup_seq != wakeup_seq_before && cv.wakeup_seq != cv.woken_seq {
      cv.woken_seq ++;
      break;
    }
    if ret == TIMEDOUT {
      cv.wakeup_seq ++;
      cv.woken_seq ++;
      break;
    }
  }
  unlock(cv.lock);
  lock(mutex);

fn cond_signal(cv):
  lock(cv.lock);
  if cv.wait_seq > cv.wakeup_seq {
    cv.wakeup_seq ++;
    cv.futex ++;
    if FUTEX_WAKE_OP(cv.futex, 1, cv.lock, 1, FUTEX_OP_CLEAR_WAKE_IF_GT_ONE).is_err() {
      FUTEX_WAKE(cv.futex, 1);
    }
  }
  unlock(cv.lock);

fn cond_broadcast(cv):
  lock(cv.lock);
  if cv.wait_seq > cv.wakeup_seq {
    cv.wakeup_seq = cv.wait_seq;
    cv.woken_seq = cv.wait_seq;
    cv.futex = cv.wait_seq * 2;
    let futexval = cv.futex;
    cv.broadcast_seq ++;
    unlock(lock);
    if FUTEX_CMP_REQUEUE(cv.futex, 1, ALL, cv.mutex_ref, futexval).is_err() {
      FUTEX_WAKE(cv.futex, ALL);
    }
  } else {
    unlock(cv.lock);
  }
```

What interests me the most is how condvar interplays with futexes. Unlike what I expected, condvar owns two futexes: the first is `cv.lock`, used to synchronize internal modifications; the second is `cv.futex`, used to park threads. Besides them, condvar also interacts with the user provided mutex, which has one futex inside.

It is important for condvar to manage its own lock, because `cond_signal` can be called without lock, stated by [POSIX](https://pubs.opengroup.org/onlinepubs/009696699/functions/pthread_cond_broadcast.html):

>The pthread_cond_broadcast() or pthread_cond_signal() functions may be called by a thread whether or not it currently owns the mutex that threads calling pthread_cond_wait() or pthread_cond_timedwait() have associated with the condition variable during their waits; however, if predictable scheduling behavior is required, then that mutex shall be locked by the thread calling pthread_cond_broadcast() or pthread_cond_signal().

And a condvar can even be paired with multiple mutexes (not at the same time), quote [POSIX](https://pubs.opengroup.org/onlinepubs/007908775/xsh/pthread_cond_wait.html) again:

>The effect of using more than one mutex for concurrent pthread_cond_wait() or pthread_cond_timedwait() operations on the same condition variable is undefined; that is, a condition variable becomes bound to a unique mutex when a thread waits on the condition variable, and this (dynamic) binding ends when the wait returns.

It's also important to notice that multiple options are available to send out wakeups:

`cond_signal` first attempts to call `FUTEX_WAKE_OP` before falling back to traditional `FUTEX_WAKE`. This particular command was introduced to kernel in [2005](https://github.com/torvalds/linux/commit/4732efbeb997189d9f9b04708dc26bf8613ed721) specifically for optimizing `cond_signal`, which has the capability to modify and conditionally wakes up a second futex. The optimization targets at avoiding "hurry up and wait" situation, where "this waiter wakes up and after a few instructions it attempts to acquire the cv internal lock, but that lock is still held by the thread calling pthread_cond_signal". It works by moving the whole unlock procedure into kernel space:

```rust
// fn cond_signal [userspace]                       // fn cond_wait [userspace]
FUTEX_WAKE_OP(cv.futex, 1, cv.lock, 1);             FUTEX_WAIT(cv.futex);
  // [kernel]                                       /* blocked */
  let (key1, key2) = (key(cv.futex), key(cv.lock)); /*         */
  spin_lock(min(key1, key2));                       /*         */
  spin_lock(max(key1, key2));                       /*         */
  let ret = OP_UNLOCK(cv.lock);                     /*         */
  wake(key1, 1);  // ------------------------------>
                                                    lock(cv.lock);  // already unlocked
  if ret {
    wake(key2, 1);  // ---------------------------->  in case the lock is contended
  }
```

One thing I haven't figured out though is why calling `FUTEX_WAKE(1)` without lock is racy, which seems like a straightforward fix to this issue.

Similarly, `cond_broadcast` tries to use `FUTEX_CMP_REQUEUE`, which is the kernel's implementation of "wait morphing". The wait queue of `cv.futex` is moved directly to user's mutex, so that when the first woken thread finishes its work (by releasing mutex), rest of the threads can be woken up in sequence. In this way, we won't encounter "thundering herd" problem where multiple threads wake up at the same time competing for the same mutex.

```rust
// fn cond_wait [userspace]
pthread_mutex_unlock_usercnt(mutex, 0);
FUTEX_WAIT(cv.futex);
// blocked //                         // fn cond_broadcast [userspace]
                                      FUTEX_CMP_REQUEUE(cv.futex, cv.mutex_ref);
// not the first in line <------------+-----------> // the first in line, woken
/* queued to mutex */                               lock(mutex);
FUTEX_WAIT(mutex);                                  /* do things with mutex */
/* blocked */                                       unlock(mutex);
/*         */                                         // examine if mutex is contended
/*         */                                         FUTEX_WAKE(mutex);  // -- (1)
// <--------------------------------------------------+
lock(cv.lock);
/* modify internals */
unlock(cv.lock);
pthread_mutex_cond_lock(mutex);  // --- (2)
```

Notice that instead of normal mutex functions, condvar uses `pthread_mutex_unlock_usercnt` and `pthread_mutex_cond_lock` to avoid changing mutex's user count. When there are multiple waiters, mutex is guaranteed to stay in contended state, so that (1) could happen, and (2) won't park the woken thread again.



<!-- 
When
In this version (around 2002), signal calls are outside the lock.

```rust
/*
 * lock: internal mutex
 * wakeups: number of wakeups signaled
 * sleepers: number of waiters
 */
fn cond_wait(cv, mutex):
  lock(cv.lock);
  unlock(mutex);
  cv.sleepers ++;
  loop {
    if cv.wakeups > 0 {
      cv.wakeups --;
      break;
    }
    wakeups = cv.wakeups;
    unlock(cv.lock);
    FUTEX_WAIT(cv.wakeups, wakeups);
    lock(cv.lock);
  }
  if --cv.sleepers == 0 {
    cv.wakeups = 0;
  }
  unlock(cv.lock);
  lock(mutex);

fn cond_signal(cv):
  lock(cv.lock);
  if cv.sleepers > 0 {
    cv.wakeups ++;
    if cv.wakeups == 0 {
      // overflow
      cv.wakeups = 0x8000;
    }
    unlock(cv.lock);
    FUTEX_WAKE(cv.wakeups, 1);
  } else {
    unlock(cv.lock);
  }

fn cond_broadcast(cv):
  lock(cv.lock);
  if cv.sleepers > 0 {
    cv.wakeups |= 0x8000;
    unlock(cv.lock);
    FUTEX_WAKE(cv.wakeups, ALL);
  } else {
    unlock(cv.lock);
  }
```

While in the second version (before 2004), there is one more sequence number and those signal calls are moved inside the lock:

```rust
/*
 * lock: internal mutex
 * wait_seq: sequence number for waiter
 * wakeup_seq: sequence number for wakeup signaled
 * woken_seq: sequence number for woken threads
 */
fn cond_wait(cv, mutex):
  lock(cv.lock);
  unlock(mutex);
  cv.wait_seq ++;
  let wakeup_seq_before = cv.wakeup_seq;
  loop {
    unlock(cv.lock);
    let ret = FUTEX_WAIT(cv.wakeup_seq, wakeup_seq_before);
    lock(cv.lock);
    if cv.woken_seq >= wakeup_seq_before && cv.woken_seq < cv.wakeup_seq {
      break;
    }
    if ret == TIMEDOUT {
      cv.wakeup_seq ++;
      break;
    }
  }
  cv.woken_seq ++;
  unlock(cv.lock);
  lock(mutex);

fn cond_signal(cv):
  lock(cv.lock);
  if cv.wait_seq > cv.wakeup_seq {
    cv.wakeup_seq ++;
    FUTEX_WAKE(cv.wakeup_seq, 1);
  }
  unlock(cv.lock);

fn cond_broadcast(cv):
  lock(cv.lock);
  if cv.wait_seq > cv.wakeup_seq {
    cv.wakeup_seq = cv.wait_seq;
    FUTEX_WAKE(cv.wakeup_seq, ALL);
  }
  unlock(cv.lock);
```

Sadly, I was unable to find detailed commit messages for those changes. Therefore this section is suspended until I find some more time to verify them with TLA+. After that, I'll have to checkout the latest [implementation](https://github.com/bminor/glibc/commit/ed19993b5b0d05d62cc883571519a67dae481a14) came out at 2016, and an interesting [bug report](https://sourceware.org/bugzilla/show_bug.cgi?id=25847) with [TLA+ attempts](https://probablydance.com/2020/10/31/using-tla-in-the-real-world-to-understand-a-glibc-bug/).

**- Q: Why `wait` with mutex held?**

This is the same "compare-and-block" pattern discussed before. We have to make sure certain condition isn't already true before going to sleep, so the lock release and sleep must appear as one atomic operation.

**- Q: Why `signal` with mutex held?**

Quote POSIX [pthread_cond_broadcast](https://pubs.opengroup.org/onlinepubs/009696699/functions/pthread_cond_broadcast.html):

>The pthread_cond_broadcast() or pthread_cond_signal() functions may be called by a thread whether or not it currently owns the mutex that threads calling pthread_cond_wait() or pthread_cond_timedwait() have associated with the condition variable during their waits; however, if predictable scheduling behavior is required, then that mutex shall be locked by the thread calling pthread_cond_broadcast() or pthread_cond_signal().

Extra locking is good for reasoning, it can make sure the signal is sent in the same order as condition mutations.

Performance-wise, it might be slightly better to signal without lock, depending on the condvar implementation.

**- Q: How to reduce premature wakeups?**

It is deemed premature to wake up a waiter when it can't acquire the resource needed to progress. Such wakeups produce unnecessary context switches and cache bouncing. But sadly, it's quite difficult to truly avoid them without compromising correctness.

With condition variable, the first case we are faced with is called "hurry up and wait": one thread signals with lock held, and the waiter will block until lock is freed. Here the lock isn't user mutex, but the lock used to protect condition variable's internals.

In case you don't know yet, "thundering herd" refers to the situation where multiple threads woken up (by a broadcast event) competing for mutex and the majority fall asleep again, causing surges of cache race and syscall. With condition variable,  we have a even more elaborated issue "hurry up and wait", which refers to one or more threads woken by `signal` or `broadcast` competing for the lock held by signaler.

"Wait morphing" is a technique specifically used to tackle this issue. It means directly moving wait queue between different synchronization constructs without actually waking them. Futex provides a `FUTEX_REQUEUE` opcode back in [2003](https://lwn.net/Articles/32746/) to provide this capability. But its fate isn't quite foreseen at that time.

In 2004, `FUTEX_CMP_REQUEUE` was introduced as a replacement because they forgot the "compare-and-block" semantics for the second address, which isn't needed until we call contidion variable's wakeup without lock held (doesn't own `addr2` anymore).

Further on, in 2005, `FUTEX_WAKE_OP` (2005 [github](https://github.com/torvalds/linux/commit/4732efbeb997189d9f9b04708dc26bf8613ed721))

- Q: How often does spurious wakeup happen with condition variable?

For simplicity, we don't consider the spurious wakeups caused by race condition in userland.

Ultimately, spurious wakeup is caused by OS. In Linux, after the lockless wakeup [patch](https://github.com/torvalds/linux/commit/1d0dcb3ad9d336e6d6ee020a750a7f8d907e28de), which essentially 

Going upwards, some efforts have been made to hide spurious wakeups at condition variable layer.

https://github.com/torvalds/linux/commit/d58e6576b0deec6f0b9ff8450fe282da18c50883

- Q: What's the deal with `std::condition_variable_any`?

**- Q: Can we pair multiple mutexes with one condition variable?**

Quote POSIX [pthread_cond_wait](https://pubs.opengroup.org/onlinepubs/007908775/xsh/pthread_cond_wait.html):

>The effect of using more than one mutex for concurrent pthread_cond_wait() or pthread_cond_timedwait() operations on the same condition variable is undefined; that is, a condition variable becomes bound to a unique mutex when a thread waits on the condition variable, and this (dynamic) binding ends when the wait returns.

At a certain point in time, when `REQUEUE` is still used in `condv::broadcast` to moves wait threads from cv's futex to mutex's futex, the mutex reference is dynamically bound when calling `condv::wait`. So it appears mixing mutexes would be catastrophic. But right now,

**- Q: Can we pair multiple condition variables with one mutex?**

Sure, why not. -->

## What's More

We've seen how futex is driven to its limit by pthread, multiple syscall commands are brought to kernel for that purpose. But in 2016, in order to satisfy a stronger ordering requirement, a [redesign](https://github.com/bminor/glibc/commit/ed19993b5b0d05d62cc883571519a67dae481a14) from the ground up was proposed and merged into the library, abandoning these eye-catching optimizations from decades before. You are welcome to revisit their debates on this matter.

Futex *blocks* threads, which is powerful yet dangerous. That's why there are much more complexities behind their innocent looks. To go deeper, I found these kernel writeups to be most informative:

[PI-futex](https://www.kernel.org/doc/Documentation/pi-futex.txt) to avoid priority inversion, which replies on [RT-mutex](https://www.kernel.org/doc/Documentation/locking/rt-mutex.txt) ([design](https://www.kernel.org/doc/Documentation/locking/rt-mutex-design.txt)) internally.

[Robust futex](https://www.kernel.org/doc/Documentation/robust-futexes.txt) to handle thread cancellation.

## See Also

Fuss, Futexes and Furwocks: Fast Userlevel Locking in Linux - Hubertus Franke & Rusty Russell, 2002 ([pdf](http://kernel.org/doc/ols/2002/ols2002-pages-479-495.pdf))

Futexes are tricky - Ulrich Drepper, 2005, revised 2011 ([pdf](http://www.akkadia.org/drepper/futex.pdf))

Requeue-PI: Making Glibc Condvars PI-Aware - Darren Hart & Dinakar Guniguntalay, 2009 ([pdf](http://lwn.net/images/conf/rtlws11/papers/proc/p10.pdf))

[A futex overview and update](https://lwn.net/Articles/360699/) - Darren Hart, 2009

Priority Inheritance on Condition Variables - Tommaso Cucinotta, 2013

Futex Scaling in the Linux Kernel - Davidlohr Bueso, 2014 ([youtube](https://www.youtube.com/watch?v=-8c47dHuGIY), [slides](https://www.slideshare.net/davidlohr/futex-scaling-for-multicore-systems))

An Overview of Kernel Lock Improvements - Davidlohr Bueso & Scott Norton, 2014 ([slides.pdf](http://events.linuxfoundation.org/sites/events/files/slides/linuxcon-2014-locking-final.pdf))

[In pursuit of faster futexes](https://lwn.net/Articles/685769/) - Neil Brown, 2016

The path of the private futex - Sebastian A. Siewior, 2016 ([slides.pdf](https://linutronix.de/PDF/2016_futex_Siewior.pdf))

<!-- [An analysis of Towelroot and the futex vulnerability](https://appdome.github.io/2017/11/23/towelroot.html) - Dany Zatuchna -->

[Futex Cheat Sheet](http://locklessinc.com/articles/futex_cheat_sheet/) - Lockless Inc.

[Mutexes and Condition Variables using Futexes](http://locklessinc.com/articles/mutex_cv_futex/) - Lockless Inc.

[Condition variable with futex](https://www.remlab.net/op/futex-condvar.shtml) - Rémi Denis-Courmont, 2016

<!---
> Joe Duffy's "Concurrent Programming On Windows" mentions this (P311-312, P598). This bit is interesting:

> Note that in all of the above examples, threads must be resilient to something called spurious wake-ups - code that uses condition variables should remain correct and lively even in cases where it is awoken prematurely, that is, before the condition being sought has been established. This is not because the implementation will actually do such things (although some implementations on other platforms like Java and Pthreads are known to do so), nor because code will wake threads intentionally when it's unnecessary, but rather due to the fact that there is no guarantee around when a thread that has been awakened will become scheduled. Condition variables are not fair. It's possible - and even likely - that another thread will acquire the associated lock and make the condition false again before the awakened thread has a chance to reacquire the lock and return to the critical region.

Side note: this won't be a problem if mutex is tightly coupled to condition variable.

Another anomaly is wake up one becoming wake all. As per Windows [document](https://devblogs.microsoft.com/oldnewthing/20180201-00/?p=97946), due to the limited bits used to track signal count, large amount of continous signal can be aggregated to one single wake all.

Finally, it's even possible for a wakeup to happend without any signal involved. This is caused by process interrupt. [Solaris reference](https://docs.oracle.com/cd/E19455-01/806-5257/gen-24356/index.html)

> In the Solaris implementation of condition variables, a spurious wakeup may occur without the condition being signaled if the process is signaled; the wait system call aborts and returns EINTR.

>The pthread_cond_wait() function in Linux is implemented using the futex system call. Each blocking system call on Linux returns abruptly with EINTR when the process receives a signal. ... pthread_cond_wait() can't restart the waiting because it may miss a real wakeup in the little time it was outside the futex system call. This race condition can only be avoided by the caller checking for an invariant. A POSIX signal will therefore generate a spurious wakeup

Just think of it... like any code, thread scheduler may experience temporary blackout due to something abnormal happening in underlying hardware / software. Of course, care should be taken for this to happen as rare as possible, but since there's no such thing as 100% robust software it is reasonable to assume this can happen and take care on the graceful recovery in case if scheduler detects this (eg by observing missing heartbeats).

Now, how could scheduler recover, taking into account that during blackout it could miss some signals intended to notify waiting threads? If scheduler does nothing, mentioned "unlucky" threads will just hang, waiting forever - to avoid this, scheduler would simply send a signal to all the waiting threads.

This makes it necessary to establish a "contract" that waiting thread can be notified without a reason. To be precise, there would be a reason - scheduler blackout - but since thread is designed (for a good reason) to be oblivious to scheduler internal implementation details, this reason is likely better to present as "spurious".

misc:

[C++11 concurrency: condition variables](http://codexpert.ro/blog/2013/03/01/cpp11-concurrency-condition-variables/): notified_all_at_thread_exit 

from [quora](https://www.quora.com/How-different-is-a-futex-from-mutex-conceptually-and-also-implementation-wise):

Understanding futexes is not simple at all and the man pages unfortunately are not updated. I had a look at futex after towelroot by geohot exploited a vulnerability in the futex subsystem to gain root privileges in some android phones. The answer by Bharat Vasudevan is outdated. Plus, the way rusty explains how to design mutexes (this was way back in 2002) is not efficient, since it doesn't take into account the issue of live locking (See Drepper's paper included in the answer for more details about this)

It is as Behdad Esfahbod says, it is a basic synchronization primitive which is used to construct complicated synchronization constructs like mutex, condition variables, semaphores etc. The main difference is that, it is a user space address (always 32 bit). In the non-contended case, locking becomes as simple as cmpxchg() on the user space address. In the contended case, we still require kernel help for waiting. But it is not as simple as that, for eg) futex can be used for inter-process synchronization as well, so what if one process acquires the futex in the non-contended case and then terminates? In the non-contended case, there is no kernel involved, then who frees the lock?. How does futexes provide support against priority inversion? As you can see, it quickly gets complicated. The paper from 2002 is outdated as hell, but it provides a background on how futexes work. Read a latter paper from Ulrich Drepper Page(futex.pdf) on akkadia.org, which tells how to construct mutexes from futexes. Read [Page](https://www.kernel.org/doc/Documentation/locking/rt-mutex-design.txt) on kernel.org to understand the design on real time mutexes, which has priority inheritance support to prevent priority inversion (futexes, most of the time are backed by rt-mutex in the kernel). Using rt-mutex causes a particular problem in glibc's condition variable implementation, read more about the problem and the proposed solution in this paper Page on(p10.pdf) lwn.net . Read about robust futexes here: [Page](https://www.kernel.org/doc/Documentation/robust-futexes.txt) on kernel.org. Also, read the kernel source code for mutex and rt-mutex to gain a better understanding. Then, you can understand the state-of-the-art futex implementation in the linux kernel.

mutex tech [note](http://www.smxrtos.com/articles/techppr/mutex.htm)

Implementing Condition Variables with Semaphores. By Andrew D. Birrell, 2004.
-->
