[Last time](https://geidav.wordpress.com/2016/03/23/test-and-set-spinlocks/) we saw how a spinlock can be implemented using an atomic test-and-set (`TAS`) operation on a single shared synchronization variable. Today, I want to talk about another spinlock variant called *Ticket Lock*. The Ticket Lock is very similar to `TAS`-based spinlocks in terms of scalability, but supports *first-in-first-out* (*FIFO*) fairness.

As the name suggests, the Ticket Lock employs the same concept as e.g. hair dressers do to serve their customers in the order of arrival. On arrival customers draw a ticket from a ticket dispenser which hands out tickets with increasing numbers. A screen displays the ticket number served next. The customer holding the ticket with the number currently displayed on the screen is served next.

# Implementation
The C++11 implementation below uses two `std::atomic_size_t` variables as counters for the ticket number currently served (`ServingTicketNo`) and the ticket number handed out to the next arriving thread (`NextTicketNo`). The implementation is optimized for x86 CPUs. The `PAUSE` instruction (called from `CpuRelax()`) is used when spin-waiting and both counters are cache line padded to prevent *false sharing*. Read the previous article on [`TAS`-based locks](https://geidav.wordpress.com/2016/03/23/test-and-set-spinlocks/) for more information.

```cpp
class TicketSpinLock
{
public:
    ALWAYS_INLINE void Enter()
    {
        const auto myTicketNo = NextTicketNo.fetch_add(1, std::memory_order_relaxed);

        while (ServingTicketNo.load(std::memory_order_acquire) != myTicketNo)
            CpuRelax();
    }

    ALWAYS_INLINE void Leave()
    {
        // We can get around a more expensive read-modify-write operation
        // (std::atomic_size_t::fetch_add()), because no one can modify
        // ServingTicketNo while we're in the critical section.
        const auto newNo = ServingTicketNo.load(std::memory_order_relaxed)+1;
        ServingTicketNo.store(newNo, std::memory_order_release);
    }

private:
    alignas(CACHELINE_SIZE) std::atomic_size_t ServingTicketNo = {0};
    alignas(CACHELINE_SIZE) std::atomic_size_t NextTicketNo = {0};
};

static_assert(sizeof(TicketSpinLock) == 2*CACHELINE_SIZE, "");
```

Overflow is pretty much impossible when 64-bit counters are used. But what happens when the counters are of smaller bit width and eventually overflow? It turns out that overflow is safe, as long as the number of threads using the lock is less than or equal to the value range representable by the counter's underlying integer type (e.g. 256 for 8-bit counters). Let's consider a 3-bit integer, which can represent values from 0 to 7. Overflow is safe as long as the there are never more than 8 threads competing for the lock, because the condition `ServingTicketNo != myTicketNo` is guaranteed to be always only false for the next thread in line. If there were 9 or more threads, the `NextTicketNo` counter could reach the same value `ServingTicketNo` has and accordingly two threads could enter the critical section (CS) at the same time. The figure below illustrates the case where 8 threads are competing for the lock. Just one more competing thread could cause multiple threads entering the CS at the same time.

```
   ServingTicketNo   NextTicketNo
          |             |
          V             V
0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 ...
          \_____________/
          Max. 8 threads

No second thread may grab another NextTicketNo=5!
```
The above means that we can crunch down our implementation in terms of memory footprint by removing the padding and using 8- or 16-bit counters, depending on the expected maximum number of threads. Removing the padding comes at the cost of potential *false sharing* issues.

# Proportional back-off
Under high contention the *Test-And-Test-And-Set Lock* backs-off exponentially. The reduced amount of memory bus traffic improves performance. What about using the same backing-off strategy with Ticket Locks? It turns out that this is a very bad idea, because the FIFO order causes the back-off delays to accumulate. For example, the thread with `myTicketNo == ServingTicketNo+3` must always wait as least as long as the thread with `myTicketNo == ServingTicketNo+2` and the thread with `myTicketNo == ServingTicketNo+2` must always wait as least as long as the thread with `myTicketNo == ServingTicketNo+1`.

Is there something else we can do? It turns out we can thanks to the FIFO order in which threads are granted access to the CS. Every thread can calculate how many other threads are going to be granted access to the CS before it-self as `numBeforeMe = myTicketNo-ServingTicketNo`. With this knowledge and under the assumption that every thread holds the lock approximately for the same duration, we can back-off for the number of threads in line before us times some constant `BACKOFF_BASE`. `BACKOFF_BASE` is the expected average time that every thread spends inside the CS. This technique is called *proportional back-off*.

```cpp
ALWAYS_INLINE void PropBoTicketSpinLock::Enter()
{
    const auto myTicketNo = NextTicketNo.fetch_add(1, std::memory_order_relaxed);

    while (true)
    {
        const auto servingTicketNo = ServingTicketNo.load(std::memory_order_acquire);
        if (servingTicketNo == myTicketNo)
            break;

        const size_t numBeforeMe = myTicketNo-servingTicketNo;
        const size_t waitIters = BACKOFF_BASE*numBeforeMe;

        for (size_t i=0; i<waitIters; i++)
            CpuRelax();
    }
}
```

# Wrap up
How does the Ticket Lock compare against `TAS`-based locks and in which situations which of the two locks variants is preferably used? The Ticket Lock has the following advantages over `TAS`-based locks:

- The Ticket Lock is fair, because the threads are granted access to the CS in FIFO order. This prevents the same thread from reacquiring the lock multiple times in a row, which - at least in theory - could starve other threads.
- In contrast to all `TAS`-based locks the Ticket Lock avoids the *Thundering Herd* problem, because waiting for the lock and acquiring it doesn't require any read-modify-write or store operation.
- In contrast to the *Test-And-Set Lock* only a single atomic load operation is repeatedly executed while waiting for the lock to become available. Though, this problem is solved by the TTAS Lock.

The biggest disadvantage of the Ticket Lock is that the fairness property backfires once there are more threads competing for the lock than there are CPU cores in the system. The problem is that in that case the thread which can enter the CS next might be sleeping. This means that all other threads must wait, because of the strict fairness guarantee. This property is sometimes referred to as *preemption intolerance*.

Furthermore, the Ticket Lock doesn't solve the scalability issue of `TAS`-based spinlocks. Both spinlock variants don't scale well, because the number of cache line invalidations triggered when acquiring/releasing the lock is `O(#threads)`. There are scalable lock implementations where all threads spin on different memory locations. These spinlock variants only trigger `O(1)` many cache line invalidations. Next time we'll look into scalable spinlock variants.