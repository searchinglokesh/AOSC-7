- On Multiprocessor synchronization is fundamentally dependent on hardware support
- Example : Basic operation of locking resource for exclusive use by setting locked flag
	- Read the flag
	- If the flag is 0(Unlock),lock the resource by setting flag to 1
	- Return True if lock was obtained else False
On Multiprocessor two threads on different processor may simultaneously attempt to carry out this sequence of operations.
![[Pasted image 20241111233720.png]]

### Atomic Test and Set
- Acts on single bit in memory.
- Tests  the bit, sets it to one, and returns its old value.
- At the completion of the operation the value of bit is one(locked).
- Returns value indicating whether it was already set to one before the operation.
- GUARANTEED TO BE ATOMIC
- Atomic even with respect to Interrupts
- Suited for simple locks
	- If test-and-set returns one, calling thread owns the resource. 
	- IF False - Locked by another resource
	- Unlocking the resource by setting bit to zero.

- Examples are Branch and bit set and set interlocked, LDSTUB

### Software Architecture
- From software perspective :
	 - Master-Slave
	 - Functionally Asymmetric
	 - Symmetric
	 
- Master - Slave : One processor plays the role of master processor, rest are slaves.
	- Master processor may be only one allowed to do I/O and receive device interrupts
	- Rest are slaves.
	- Master processor runs kernel code.
	- Slaves run user level code.

- Functionally asymmetric multiprocessors - Run on different subsystems on different processors.
	- Ex : One processor may run the networking layer.
	- Another manages I/O.
	- More suitable for general purpose I/O.

- Symmetric Multiprocessing: More popular approach . In  SMP system all CPUs are equal ,share single copy of kernel text and data, and compete for system resources such as devices and memory. 
	- Each CPU may run kernel code.
	- User process may be scheduled on any processor.

### Multiprocessor Synchronization Issues
- Traditional synchronization model 
	- Thread retains exclusive use of the kernel until it is ready to leave the kernel or block on resource
- No Longer valid for multiprocessor, each processor could be executing kernel code at the same time.
- Need to protect data which was previously not needed to be protected.
- Example : IPC Resource Table.
- Locking Primitives must be changed as well
	- Two threads can concurrently examine locked flag for the same resource.
	- Race Condition
- Blocking Interrupts
	- Thread can only block on processor which it is running.
	- Not possible to interrupts across all processors
	- One solution is global ipl managed in software.
### Lost Wakeup Problem

- Traditional sleep/wakeup mechanism can fail due to race condition.
- If thread is about to sleep on locked resources, resource may be unlocked and all waiters woken up before the thread has chance to sleep . This results in thread missing wakeup and get struck.

### Thundering Herd Problem

- When resource is unlocked, waking up all waiting threads can lead to unnecessary overhead, as multiple threads may try to acquire the resources again.
- More likely to happen in multiprocessor system , where the woken threads may be schedule on different processor.
![[Pasted image 20241112094050.png]]
### Semaphores
- Early synchronization used in UNIX multiprocessor implementation.
- Two Basic Operations
	- P()  -  Acquire
	- V()  -  Release
	- Guaranteed to be atomic
- Used to provide Mutual Exclusion, Event Waiting and control of Countable Resources.
![[Pasted image 20241112094238.png]]
![[Pasted image 20241112094143.png]]
### Drawbacks of Semaphores
* Relatively expensive due to need for atomic access and blocking/unblocking mechanism.
* Can hide information about whether thread actually had to block, which can be problematic in some situations.
* Semaphores can also lead to performance issue called " Semaphore Convoys ", where unnecessary context switch occurs due to the way semaphores allocate resources.

### Mutual Exclusion
- Semaphore is initialized to 1 and associated with shared resource.
- Threads acquire resources by calling P() on the semaphore, which decrements the value.
- The first P() call sets the value to 0, causing subsequent p() calls to block.
- When thread is done using resource it calls the V() to release the semaphore, incrementing the value and waking up one blocked thread.
![[Pasted image 20241112094313.png]]
### Event Waiting
- Semaphore is initialized to 0 to represent an event.
- Threads wait for the event by calling P() on the semaphore, which blocks them.
- When event occurs , a V() call is made, which wakes up one blocked thread.
- Each woken thread should also call V() to allow other waiting calls to be scheduled
![[Pasted image 20241112094343.png]]
### Controlling Countable Resources
- A semaphore is initialized to the number of available resource instances.
- Thread acquire an instance by calling P() which blocks if no instances are available.
- Threads release instance by calling V(), which increments the semaphore value.
- Semaphore value represents the number of available instances, and negative value represents the number of pending requests.
![[Pasted image 20241112094354.png]]

Let me explain the concept of semaphore convoys, which is a performance issue in multiprocessor systems. I'll break this down step by step.

### Convoys
The Problem:
Semaphores, while preventing unnecessary wake-ups, can lead to a performance issue called "convoys" due to their strict first-come-first-served ownership transfer semantics.

1. Initial State (Step a):
```
- Thread T2: Currently holds the semaphore (in critical region R1)
- Thread T3: Waiting to acquire the semaphore
- Thread T1: Running on another processor
- Thread T4: Waiting in scheduler queue
```

2. Middle State (Step b):
```
- T2 exits critical region and releases semaphore
- T3 is awakened and gets ownership of semaphore
- T3 moves to scheduler queue (important: hasn't started running yet)
```

3. Final State (Step c):
```
- T1 needs to enter critical region
- Must block because T3 "owns" semaphore
- T1 releases its processor (P1)
- T4 gets scheduled on P1
- Result: T3 has semaphore but isn't running
- T1 is blocked unnecessarily
```

![[Pasted image 20241112094601.png]]
The Core Issue:
The problem arises because:
1. Semaphore ownership transfers immediately upon wake-up
2. The new owner (T3) might not run immediately
3. Other threads (T1) must block even though no thread is actually in the critical region

Contrast with Mutex Locks:
With a mutex lock instead of a semaphore:
- T2 would release the lock and wake up T3
- T3 wouldn't own the lock until it actually runs
- T1 could acquire the lock immediately
- Saves unnecessary context switches

Key Takeaways:
1. Semaphore semantics enforce strict ordering (first-come-first-served)
2. This can lead to unnecessary blocking and context switches
3. Mutex locks can be more efficient as they don't transfer ownership until the thread actually runs
4. Modern kernels tend to prefer simpler, lower-level primitives over complex monolithic abstractions

### Spin Locks:
1. Basic Concept:
- A spin lock is the simplest locking primitive
- Threads "busy-wait" (spin) in a loop until the resource becomes available
- Uses a scalar variable: 0 = available, 1 = locked

2. Implementation:
```c
void spinlock(spinlock_t *s) {
    while (test_and_set(s) != 0)  // if already locked
        while (*s != 0)           // wait until unlocked
            ;  // busy wait
}

void spin_unlock(spinlock_t *s) {
    *s = 0;
}
```

3. Key Characteristics:
- Very fast when there's no contention (single instruction)
- Must be held for extremely short durations
- Cannot be held during blocking operations
- Ties up CPU while waiting (busy-waiting)
- Best suited for multiprocessor systems
- Ideal for briefly accessed data structures
![[Pasted image 20241112094703.png]]
### Condition Variables:
1. Basic Concept:
- More complex mechanism associated with a predicate
- Allows threads to block and wake up based on condition changes
- Uses a mutex (usually spin lock) for protection

2. Implementation:
```c
struct condition {
    proc *next;           // doubly linked list
    proc *prev;
    spinlock_t listlock;  // protects the list
};

void wait(condition *c, spinlock_t *s) {
    spin_lock(&c->listlock);
    // add self to linked list
    spin_unlock(&c->listlock);
    spin_unlock(s);
    swtch();  // context switch
    spin_lock(s);
}

void do_signal(condition *c) {
    spin_lock(&c->listlock);
    // wake one thread if any waiting
    spin_unlock(&c->listlock);
}

void do_broadcast(condition *c) {
    spin_lock(&c->listlock);
    // wake all waiting threads
    spin_unlock(&c->listlock);
}
```

3. Important Features:
- Uses two separate mutexes:
  - listlock: protects the blocked threads list
  - External mutex: protects the condition's data
- Provides two wake-up mechanisms:
  - do_signal(): wakes one thread
  - do_broadcast(): wakes all waiting threads
- Prevents lost wakeup problem through atomic operations
- Requires careful ordering of locks to prevent deadlocks

4. Use Cases:
- Server threads waiting for client requests
- Multiple threads waiting for I/O completion
- Any scenario where threads need to wait for a condition to become true

The key difference between spin locks and condition variables is their purpose:
- Spin locks: For very brief mutual exclusion
- Condition variables: For thread synchronization based on conditions/events

### Implementation
1. Condition Variables Implementation Details:
- Uses two separate mutexes:
  1. `listlock`: Protects the doubly linked list of blocked threads
  2. A separate mutex that protects the tested data itself (passed as argument to wait())
  3. Potentially a third mutex for scheduler queues
- The predicate is not part of the condition variable
- Locking order must be strict (predicate lock before listlock) to avoid deadlocks

2. Sleep Queue Implementation Options:
- Option 1: Queue of blocked threads as part of condition structure
- Option 2: Global set of sleep queues (traditional UNIX approach)
   - Replaces listlock with mutex protecting appropriate sleep queue

3. Event Handling:
Two completion notification approaches:
- `do_signal()`: Wakes up just one thread
- `do_broadcast()`: Wakes up all waiting threads

Example use cases:
- Server application: One thread per request → `do_signal()` sufficient
- Page faults: Multiple threads waiting for same page → `do_broadcast()` better

4. Events as Higher-Level Abstraction:
Combines:
- Done flag
- Protecting spin lock
- Condition variable
Key operations:
```
awaitDone()  // blocks until event occurs
setDone()    // marks event as done and wakes threads
testDone()   // optional non-blocking test
reset()      // marks event as not done
```

5. Read-Write Lock Design:
Basic operations:
```
lockShared()      // shared/read lock
lockExclusive()   // exclusive/write lock
unlockShared()
unlockExclusive()
```

Optional operations:
```
trylockShared()    // non-blocking version
trylockExclusive() // non-blocking version
upgrade()          // shared → exclusive
downgrade()        // exclusive → shared
```

Key Design Considerations:
- Lock release strategies:
  - Last reader releases → wake single waiting writer
  - Writer releases → wake all waiting readers (or single writer if no readers)
- Writer starvation prevention:
  - `lockShared()` blocks if writers are waiting
  - Results in alternating access between writers and reader batches
- Upgrade considerations:
  - Must avoid deadlocks
  - Options:
    1. Give preference to upgrade requests
    2. Release shared lock before blocking
    3. Fail upgrade if another upgrade pending

```c
struct rwlock {
    int nActive;           // Number of active readers (or -1 if writer active)
    int nPendingReads;     // Number of readers waiting
    int nPendingWrites;    // Number of writers waiting
    spinlock_t sl;         // Spin lock for protecting the structure
    condition canRead;     // Condition for readers to wait on
    condition canWrite;    // Condition for writers to wait on
};
```

Key Operations:

1. `lockShared()` (Reader Lock):
```c
void lockShared(struct rwlock *r) {
    spin_lock(&r->sl);
    r->nPendingReads++;
    
    // Block if writers are waiting (prevent writer starvation)
    if (r->nPendingWrites > 0)
        wait(&r->canRead, &r->sl);
        
    // Block if writer is active
    while (r->nActive < 0)
        wait(&r->canRead, &r->sl);
        
    r->nActive++;
    r->nPendingReads--;
    spin_unlock(&r->sl);
}
```

2. `lockExclusive()` (Writer Lock):
```c
void lockExclusive(struct rwlock *r) {
    spin_lock(&r->sl);
    r->nPendingWrites++;
    
    // Wait while any readers or writers are active
    while (r->nActive)
        wait(&r->canWrite, &r->sl);
        
    r->nPendingWrites--;
    r->nActive = -1;  // Indicates writer active
    spin_unlock(&r->sl);
}
```

3. Unlock Operations:

`unlockShared()`:
- Decrements active readers count
- If last reader, signals one waiting writer

`unlockExclusive()`:
- Sets nActive to 0
- If readers waiting: wakes all readers
- Otherwise: wakes one writer

4. Grade Change Operations:

`downgrade()` (Exclusive → Shared):
- Changes nActive from -1 to 1
- Wakes all waiting readers

`upgrade()` (Shared → Exclusive):
- If sole reader: directly converts to writer
- Otherwise: 
  - Releases shared lock
  - Waits for other readers
  - Acquires exclusive lock

Key Design Considerations:

1. Writer Starvation Prevention:
- Readers block if writers are waiting
- Ensures alternation between readers and writers under contention

2. Wake-up Strategy:
- Writer release: Prefers waking readers over writers
- Last reader release: Wakes one writer
- Prevents unnecessary wake-ups

3. Deadlock Prevention:
- Careful handling in upgrade operation
- Releases shared lock if can't upgrade immediately

4. Fairness:
- Under heavy load, alternates between:
  - Individual writers
  - Batches of readers

### Reference Count
1. Purpose of Reference Counts:
- Protects objects themselves (beyond just protecting data inside objects)
- Prevents invalid access to deallocated objects
- Ensures objects remain valid as long as threads have pointers to them

2. How Reference Counting Works:
- When object is first allocated → count set to 1 (first pointer)
- Each new pointer to object → count increments
- When thread releases reference → count decrements 
- When count reaches zero → object can be safely deallocated

3. Example Given: File System vnodes
- vnodes hold information about active files
- When user opens file → gets file descriptor (reference to vnode)
- Multiple users can have references to same vnode
- Only when last user closes file → vnode can be deallocated

4. Benefits:
- Useful in both uniprocessor and multiprocessor systems
- Especially important in multiprocessors to prevent one thread from deallocating while another is accessing
- Prevents memory corruption from accessing deallocated objects

5. Key Implementation Details:
- Thread gains a "reference" when it gets a pointer
- Thread must explicitly release reference when no longer needed
- Kernel tracks count of active references
- Deallocation only happens when no valid references remain

### DEADLOCK AVOIDANCE

1. The Deadlock Problem:
- Occurs when threads need multiple locks
- Example scenario:
  ```
  Thread T1: Has R1, wants R2
  Thread T2: Has R2, wants R1
  Result: Both threads blocked forever
  ```
![[Pasted image 20241112095034.png]]
2. Two Main Deadlock Avoidance Techniques:

A. Hierarchical Locking:
- Imposes strict ordering on locks
- All threads must acquire locks in same order
- Example order: "predicate lock before list lock"
- Prevents deadlock if strictly followed

B. Stochastic Locking:
- Used when hierarchical order must be violated
- Uses `try_lock()` instead of regular `lock()`
- Implementation:
```c
int try_lock(spinlock_t *s) {
    if (test_and_set(s) != 0)  // already locked
        return FAILURE;
    else
        return SUCCESS;
}
```

3. Real-World Example: Buffer Cache Implementation
- Structure:
  - LRU list of disk block buffers
  - List has a spin lock for queue headers/pointers
  - Each buffer has its own spin lock

- Two Access Patterns That Conflict:
  1. Getting specific buffer:
     - Lock buffer first
     - Lock list second
  
  2. Getting any free buffer:
     - Lock list first
     - Lock head buffer second
     - This violates normal order!

4. Solution Using Stochastic Locking:
- When getting any free buffer:
  1. Lock the list
  2. Use `try_lock()` on buffers
  3. Keep trying until finding unlocked buffer
  4. Prevents deadlock by avoiding blocking

This approach provides a practical solution when strict hierarchical locking isn't feasible, trading potential retries for deadlock prevention.



1. Recursive Locks:
- Definition: A lock that can be re-acquired by the thread that already owns it
- Benefits:
  - Enables modular code design
  - Avoids need to pass lock state through function calls
  - Prevents single-process deadlocks
- Cost: Additional overhead for:
  - Storing owner ID
  - Checking ownership on lock attempts
- Example: BSD file system's ufs_write()
  - Handles both file and directory writes
  - Directory vnode comes pre-locked
  - Needs recursive lock to prevent self-deadlock

2. Blocking vs Spinning:
- Trade-offs:
  - Spinning wastes CPU but avoids context switch overhead
  - Blocking frees CPU but has context switch costs
- Considerations:
  - Must spin if already holding a simple mutex
  - Blocking is expensive (context switches + queue manipulation)
  - Some resources need both short-term and long-term locking

3. Solutions for Lock Type Selection:
A. Hint-Based Approach:
- Lock stores hint about whether to spin or block
- Hint set by lock owner
- Can be advisory or mandatory

B. Adaptive Locks (Solaris 2.x):
- Checks if lock owner is active
- Spins if owner is running
- Blocks if owner is blocked

4. What to Lock:
- Data: Traditional data protection
- Predicates: Used with condition variables
- Invariants: True except when lock held
- Operations: Restricts code execution to one processor

5. Granularity and Duration:
6. 
Granularity Spectrum:
- Coarse-grained: Single lock for whole subsystem
  - Pro: Simple
  - Con: Potential bottleneck
- Fine-grained: Lock per data item
  - Pro: Maximum concurrency
  - Con: Memory overhead, deadlock risk

Duration Considerations:
- Principle: Hold locks as briefly as possible
- Trade-off: Sometimes better to hold lock longer to avoid repeated lock/unlock
- Decision factors:
  - Duration of intermediate work
  - Contention patterns
  - System requirements

### Case Studies
1. Basic Locks (SVR4.2/MP):
- These are simple mutex (mutual exclusion) locks
- Key characteristics:
  - Non-recursive (can't be locked multiple times by same process)
  - For short-term resource locking only
  - Cannot be held during blocking operations
- Operations:
  ```c
  pl_t LOCK(lock_t *lockp, pl_t new_ipl);    // Acquire lock
  UNLOCK(lock_t *lockp, pl_t old_ipl);        // Release lock
  ```
- Important: When locking, it raises the interrupt priority level (IPL) and returns the old level, which must be restored during unlock

2. Read-Write Locks:
- Allows multiple readers but only one writer
- Characteristics:
  - Non-recursive
  - Short-term only
  - No blocking operations allowed
- Operations:
  ```c
  RW_RDLOCK()    // For readers
  RW_WRLOCK()    // For writers
  RW_UNLOCK()    // Release lock
  ```
- Also includes non-blocking versions (TRY variants)

3. Sleep Locks:
- Designed for long-term resource locking
- Can be held across blocking operations
- Operations:
  ```c
  SLEEP_LOCK()         // Normal lock
  SLEEP_LOCK_SIG()     // Interruptible by signals
  SLEEP_UNLOCK()       // Release lock
  ```
- When blocking, you can specify a priority level for when the process wakes up

4. Synchronization Variables:
- Similar to condition variables
- Must be used with a basic lock to protect the predicate
- Operations:
  ```c
  SV_WAIT()       // Wait for condition
  SV_WAIT_SIG()   // Wait (interruptible by signals)
  SV_SIGNAL()     // Wake one waiter
  SV_BROADCAST()  // Wake all waiters
  ```

Digital UNIX's approach is different, offering two main types of locks:

1. Simple Locks:
- Basic spin locks using atomic test-and-set
- Must be initialized before use
- Cannot be held across blocking operations

2. Complex Locks:
- More sophisticated reader-writer locks
- Features:
  - Shared/exclusive access
  - Optional blocking
  - Optional recursive locking
  - Can be upgraded/downgraded (shared ↔ exclusive)

![[Pasted image 20241112095055.png]]

Digital UNIX also retains BSD-style sleep/wakeup mechanisms but improves them for multiprocessor safety using assert_wait() and thread_block() primitives to prevent lost wakeups.

### Other Implementations

1. NCR's SVR4 Implementation:
- Introduced Advisory Processor Locks (APLs)
Key features of APLs:
- Recursive locks with hints for contending threads
- Hints specify whether threads should spin or sleep
- Hints can be advisory or mandatory
- Lock owners can change hints dynamically
- Automatically released and reacquired during context switches
- Can't be held during sleep operations
- Also offered non-recursive spin locks and read-write APLs

2. Intel Multiprocessor Consortium's Modifications:
Made significant changes to APLs:
- Added interrupt priority level parameter to lock acquisition
- Priority raising around lock holding
- Busy-waiting occurs at original (lower) priority
- Lock release restores original priority

3. Primitive Hierarchy:
	Lowest level:
	- Atomic arithmetic and logical operations
	- Used for reference counting and bit manipulation
	- Return original variable values
	
	Middle level:
	- Simple spin locks
	- Not auto-released on context switches
	- Used for quick operations (queue management)
	
	Highest level:
	- Resource locks
	- Long-term reader-writer locks
	- Can span blocking operations

4. Solaris 2.x Approach:
- Uses adaptive locks (switches between spinning and sleeping)
- Implements turnstiles for priority inheritance
- Provides high-level synchronization objects:
  - Semaphores
  - Reader-writer locks
  - Condition variables
- Unique feature: Handles interrupts using kernel threads
  - Allows interrupt handlers to use standard synchronization primitives
  - Interrupt handlers can block if needed

5. Other Notable Implementations:
- IBM/370 and AT&T 3B20A: Primarily semaphore-based
- Ultrix: Uses blocking exclusive locks
- Amdahl's UTS: Based on conditions
- DG/UX: Uses indivisible event counters for sequenced locks

Common Threads Across Implementations:
1. All use spin locks for short-term synchronization
2. Most retain sleep/wakeup mechanisms (with modifications) for compatibility
3. Main differences lie in higher-level abstractions
4. Choice of primitives often influenced by porting considerations
5. Mach-based systems have more freedom in primitive selection due to less legacy code
