# Java Concurrency Guide

## Table of Contents
1. [Java Memory Model (JMM)](#java-memory-model)
2. [Synchronization Basics](#synchronization-basics)
3. [Atomic Operations](#atomic-operations)
4. [Locks](#locks)
5. [Thread Coordination](#thread-coordination)
6. [Concurrent Collections](#concurrent-collections)
7. [Virtual Threads](#virtual-threads)
8. [Common Patterns & Interview Questions](#common-patterns)

---

## Java Memory Model (JMM)

### What is JMM?
The Java Memory Model defines how threads interact through memory and what guarantees the JVM provides. Without JMM, multi-threaded programs would have unpredictable behavior across different processors and JVM implementations. JMM ensures consistency by defining:

1. **Visibility**: When changes made by one thread to shared variables become visible to other threads
2. **Atomicity**: Whether an operation is indivisible (cannot be interrupted mid-operation)
3. **Ordering**: What relative ordering of operations the JVM guarantees

### Core Concepts
- **Visibility**: Changes by one thread are visible to other threads
- **Atomicity**: Operation completes without interruption
- **Ordering**: Order of operations is guaranteed
- **Happens-Before Relationship**: If action A happens-before action B, A's memory effects are visible to B

### `volatile` Keyword

**What it does:** Marks a variable as shared across threads and ensures visibility. When a thread writes to a volatile variable, the JVM forces a write to main memory (not just CPU cache). When a thread reads a volatile variable, it reads from main memory.

**When to use:** Single boolean flags, simple state flags, counters that don't need atomic increment

**What it doesn't do:** Does NOT guarantee atomicity. Does NOT prevent race conditions in compound operations.

**Memory effect:** Creates a memory barrier that prevents certain optimizations and reorderings.

```java
// Ensures visibility of shared variable changes across threads
public class VolatileExample {
    private volatile boolean flag = false;
    private volatile int counter = 0;
    
    public void writer() {
        counter = 42;
        flag = true;  // All previous writes visible to other threads
    }
    
    public void reader() {
        if (flag) {
            System.out.println(counter);  // Guaranteed to see 42, not stale value
        }
    }
}
// Guarantees: visibility, NOT atomicity. Does NOT prevent race conditions.
// flag = !flag  // This is NOT atomic even with volatile

// BAD: Multiple threads can call this and still race
public volatile int count = 0;
public void increment() { count++; }  // NOT atomic! Race condition possible
```

**Interview Q:** When would you use volatile over synchronized? 
**A:** When you need visibility without mutual exclusion (e.g., simple flags). For counters or compound operations, use AtomicInteger or synchronized.

### `final` Keyword

**What it does:** Prevents reassignment and enables safe publication. Once a final field is set (in constructor), all threads are guaranteed to see the correctly initialized value without synchronization.

**Why it matters:** Solves the problem of "final field safety" - ensures that an object is fully constructed and visible to all threads before it's shared.

**Use cases:** Making objects immutable, preventing accidental reassignments, thread-safe immutable objects

**Important:** final makes the REFERENCE immutable, not the object. A final List can still have elements added/removed.

```java
// Ensures safe publication and immutability
public class FinalExample {
    private final int x;
    private final List<String> list;
    
    public FinalExample(int x, List<String> list) {
        this.x = x;
        this.list = new ArrayList<>(list);  // Defensive copy
    }
    // Once constructed, x and list references cannot change
    // Safe to share across threads without synchronization
    
    // But we can still modify contents
    public void addItem(String item) {
        list.add(item);  // This is allowed - modifying list contents
    }
    
    // To truly immutable, return unmodifiable view
    public List<String> getList() {
        return Collections.unmodifiableList(list);
    }
}
```

**Interview Q:** Why is final important for thread safety?
**A:** It guarantees safe publication - if a thread sees a reference to a final-constructed object, it will see all writes that happened in the constructor without needing explicit synchronization.

---

## Synchronization Basics

### `synchronized` Keyword

**What it does:** Provides mutual exclusion - only one thread can execute the synchronized block at a time. Also guarantees memory visibility (like volatile).

**Types of locks:**
1. **Method-level lock** — locks on `this` (instance) or class object (static)
2. **Block-level lock** — fine-grained control over what's protected
3. **Custom object lock** — use a separate lock object to avoid contention

**Advantages:** Simple, built-in, JVM-optimized (biased locking, lock coarsening)

**Disadvantages:** 
- Coarse-grained locking (hard to optimize)
- No timeout capability
- No tryLock ability
- Can cause deadlocks if not careful
- Can lead to thread starvation

```java
public class SynchronizedExample {
    private int counter = 0;
    
    // Instance method lock (locks on 'this')
    // All calls to any synchronized instance method compete for same lock
    public synchronized void increment() {
        counter++;
    }
    
    // Static method lock (locks on class object - MyClass.class)
    // Separate from instance locks
    public static synchronized void staticMethod() {
        // ...
    }
    
    // Block-level lock (fine-grained control)
    // Only protects the critical section
    public void incrementFast() {
        synchronized(this) {
            counter++;
        }
    }
    
    // Lock on different object (reduces contention)
    private final Object lock = new Object();
    public void incrementSafe() {
        synchronized(lock) {
            counter++;
        }
    }
    
    // Anti-pattern: locking on mutable object
    private List<String> list = new ArrayList<>();
    public void badLock() {
        synchronized(list) {  // BAD! list can be reassigned
            list.add("item");
        }
    }
}

// Pattern: Synchronized collections
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
List<String> syncList = Collections.synchronizedList(new ArrayList<>());
```

**Interview Q:** What happens if synchronized method calls another synchronized method on same object?
**A:** Reentrant locks - same thread can acquire the lock multiple times. Counter increases each time, decreases when exiting each level.

---

## Atomic Operations

### What are Atomics?

**Lock-free alternative** to synchronized blocks using Compare-And-Swap (CAS) at the CPU level. Instead of acquiring locks, atomics use optimistic locking - they assume no conflict and retry if conflict detected.

**Why use them:**
- Better performance under high contention (no blocking)
- No risk of deadlocks
- Simpler than locks in many cases
- More scalable

**CAS Mechanism:** Compare-And-Swap is a CPU instruction that atomically:
1. Compares current value with expected value
2. If equal, swaps to new value
3. Returns success/failure

If CAS fails, the operation retries (spin-retry pattern).

### Atomic Classes
```java
import java.util.concurrent.atomic.*;

public class AtomicExample {
    private AtomicInteger counter = new AtomicInteger(0);
    private AtomicReference<String> name = new AtomicReference<>("Alice");
    
    public void increment() {
        counter.incrementAndGet();      // Atomic: get, increment, return
        counter.addAndGet(5);           // Atomic: add 5
        counter.getAndSet(100);         // Atomic: set to 100, return old value
    }
    
    // Compare-and-swap (CAS) - the fundamental operation
    public boolean compareAndIncrement(int expected) {
        return counter.compareAndSet(expected, expected + 1);
        // If current == expected, set to expected+1 and return true
        // Otherwise return false (conflict detected)
    }
    
    // Atomic reference update
    public void updateName(String newName) {
        name.set(newName);
        String current = name.getAndSet(newName);  // Set and get old value
    }
    
    // Pattern: Retry on conflict
    public void retryPattern() {
        int current;
        do {
            current = counter.get();
        } while (!counter.compareAndSet(current, current + 1));
        // Keeps trying until CAS succeeds
    }
}

// Common operations
AtomicInteger ai = new AtomicInteger(5);
ai.getAndIncrement();      // Return 5, set to 6
ai.incrementAndGet();      // Set to 7, return 7
ai.getAndAdd(3);           // Return 7, set to 10
ai.addAndGet(3);           // Set to 13, return 13
ai.getAndSet(20);          // Return 13, set to 20
ai.compareAndSet(20, 25);  // If value is 20, set to 25, return success
```

**Performance Characteristics:**
- Very fast when no contention
- Retries under contention (can be slow if many conflicts)
- Better than locks for low contention, similar for high contention

**Interview Q:** Why does AtomicInteger use CAS instead of synchronized?
**A:** Lock-free - no thread blocking. Better scalability under low-to-moderate contention. CPU instruction support makes it efficient.

### Common Atomic Classes
- `AtomicInteger`, `AtomicLong`, `AtomicBoolean`
- `AtomicReference<T>`, `AtomicReferenceArray<T>`
- `AtomicStampedReference<V>` — solves ABA problem
- `AtomicMarkableReference<V>` — marks if value is logically deleted

---

## Locks

### ReentrantLock

**What it is:** Explicit lock from java.util.concurrent.locks. More flexible than synchronized - supports timeouts, try-acquire, and fair/unfair modes.

**Advantages over synchronized:**
- tryLock() - attempt to acquire with timeout
- Non-blocking checks (tryLock(0))
- Fair lock option (FIFO order)
- More fine-grained control
- Can check lock status

**Disadvantages:**
- More verbose (must unlock in finally)
- Easy to forget unlock (synchronized is automatic)
- Slightly more overhead

**Reentrant:** Same thread can acquire the lock multiple times (lock count increments/decrements)

```java
import java.util.concurrent.locks.*;

public class ReentrantLockExample {
    private final ReentrantLock lock = new ReentrantLock(false);  // false=unfair (faster)
    private int counter = 0;
    
    // Basic usage - MUST unlock in finally
    public void increment() {
        lock.lock();
        try {
            counter++;
        } finally {
            lock.unlock();  // Always unlock in finally
        }
    }
    
    // With timeout - useful for avoiding deadlocks
    public boolean incrementIfAvailable() throws InterruptedException {
        if (lock.tryLock(1, TimeUnit.SECONDS)) {
            try {
                counter++;
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false;  // Lock not acquired within timeout
    }
    
    // Non-blocking try
    public boolean tryNow() {
        if (lock.tryLock()) {
            try {
                counter++;
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false;  // Someone else has it
    }
    
    // Check if locked
    public int getCounterIfLocked() {
        if (lock.tryLock()) {
            try {
                return counter;
            } finally {
                lock.unlock();
            }
        }
        return -1;  // Lock held by another thread
    }
    
    // Reentrant: same thread can acquire multiple times
    public void reentrant() {
        lock.lock();
        try {
            lock.lock();  // Same thread - OK (lock count = 2)
            try {
                // critical section
            } finally {
                lock.unlock();  // count = 1
            }
        } finally {
            lock.unlock();  // count = 0
        }
    }
}
```

**Interview Q:** When would you use ReentrantLock instead of synchronized?
**A:** When you need timeout (tryLock), fairness guarantee, or to check lock status without blocking.

### ReadWriteLock

**What it is:** Allows multiple threads to read simultaneously, but exclusive access for writers. Optimizes scenarios with many readers and few writers.

**Characteristics:**
- Multiple readers OR single writer (never both)
- Reader threads don't block each other
- Writer blocks all readers and other writers
- More overhead than simple locks

**Use case:** Caches, configuration objects, leaderboards - scenarios where reads vastly outnumber writes

```java
private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
private int counter = 0;

// Multiple threads can execute this simultaneously
public int read() {
    rwLock.readLock().lock();
    try {
        return counter;  // Multiple readers can read simultaneously
    } finally {
        rwLock.readLock().unlock();
    }
}

// Exclusive access - blocks all readers and other writers
public void write(int value) {
    rwLock.writeLock().lock();
    try {
        counter = value;  // Only one writer at a time, blocks readers
    } finally {
        rwLock.writeLock().unlock();
    }
}

// Upgrade from read to write lock (NOT SAFE - deadlock risk!)
public void unsafeUpgrade() {
    rwLock.readLock().lock();
    try {
        if (counter < 0) {
            rwLock.readLock().unlock();
            rwLock.writeLock().lock();
            try {
                counter = 0;
            } finally {
                rwLock.writeLock().unlock();
            }
        }
    } finally {
        rwLock.readLock().unlock();
    }
}
```

**Interview Q:** Why can't you upgrade from read lock to write lock?
**A:** Deadlock risk - another writer might acquire write lock between releasing read and acquiring write.

---

## Thread Coordination

Thread coordination primitives allow threads to synchronize at specific points without explicit locks. Used to implement patterns like:
- Main thread waiting for worker threads to finish
- Multiple threads reaching a synchronization point
- Rate limiting concurrent access
- Multi-phase computations

### CountDownLatch

**What it is:** A one-time-use synchronization point. Threads wait until a counter reaches zero.

**Use cases:** 
- Main thread waits for N worker threads to complete
- Start a batch of threads together
- Implement join-like behavior

**Characteristics:**
- Counter initialized at creation
- countDown() decrements counter
- await() blocks until counter reaches 0
- One-time use - cannot be reset
- All waiting threads are released together

```java
import java.util.concurrent.*;

public class CountDownLatchExample {
    public static void main(String[] args) throws InterruptedException {
        int numWorkers = 3;
        CountDownLatch latch = new CountDownLatch(numWorkers);
        
        for (int i = 0; i < numWorkers; i++) {
            new Thread(() -> {
                try {
                    Thread.sleep(1000);  // Simulate work
                    System.out.println("Worker done");
                } finally {
                    latch.countDown();  // Decrement counter (must be in finally)
                }
            }).start();
        }
        
        latch.await();  // Wait until counter reaches 0
        System.out.println("All workers finished");
    }
}
```

**Interview Q:** How is CountDownLatch different from Thread.join()?
**A:** join() waits for specific threads. CountDownLatch works with any threads (don't need references) and supports multiple waiters.

### CyclicBarrier

**What it is:** A reusable synchronization point where threads wait for each other before proceeding.

**Key difference from CountDownLatch:**
- Reusable - can be used multiple times
- Bidirectional - all threads must arrive at barrier
- Can run action when all arrive
- Fixed party count

**Use cases:**
- Iterative algorithms (barrier between iterations)
- Game loops (all players wait before next frame)
- Parallel testing (all threads start at same time)

```java
public class CyclicBarrierExample {
    public static void main(String[] args) throws InterruptedException {
        int numThreads = 3;
        CyclicBarrier barrier = new CyclicBarrier(numThreads, () -> {
            System.out.println("All threads reached barrier! Proceeding...");
        });
        
        for (int i = 0; i < numThreads; i++) {
            int threadId = i;
            new Thread(() -> {
                try {
                    System.out.println("Thread " + threadId + " waiting at barrier");
                    barrier.await();  // Wait for all threads to arrive
                    System.out.println("Thread " + threadId + " proceeding");
                    
                    // Second iteration with same barrier
                    Thread.sleep(500);
                    System.out.println("Thread " + threadId + " at barrier again");
                    barrier.await();  // Reusable!
                    System.out.println("Thread " + threadId + " done");
                } catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```

**Interview Q:** What happens if a thread doesn't reach the barrier?
**A:** Other threads wait indefinitely. This is why await() can be interrupted - allows timeout detection.

### Semaphore

**What it is:** Maintains a permit count. Threads acquire permits to access a resource, release when done.

**Use cases:**
- Rate limiting (max N concurrent operations)
- Resource pooling (limited database connections)
- Throttling requests

**Characteristics:**
- Binary (1 permit) = mutex lock
- Counting (N permits) = allow N concurrent threads
- Fair/unfair ordering available
- Non-blocking tryAcquire() available

```java
public class SemaphoreExample {
    // Allow max 3 threads concurrently
    private final Semaphore semaphore = new Semaphore(3);  
    
    public void accessResource() throws InterruptedException {
        semaphore.acquire();  // Decrement permit count, block if 0
        try {
            System.out.println("Accessing resource (permits left: " + semaphore.availablePermits() + ")");
            Thread.sleep(1000);
        } finally {
            semaphore.release();  // Increment permit count
        }
    }
    
    // Non-blocking try
    public boolean tryAccess() {
        if (semaphore.tryAcquire()) {  // Don't block
            try {
                System.out.println("Got access");
                return true;
            } finally {
                semaphore.release();
            }
        }
        return false;  // No permits available
    }
    
    // With timeout
    public boolean tryAccessWithTimeout() throws InterruptedException {
        if (semaphore.tryAcquire(1, TimeUnit.SECONDS)) {
            try {
                System.out.println("Got access");
                return true;
            } finally {
                semaphore.release();
            }
        }
        return false;
    }
}
```

### Phaser

**What it is:** Flexible synchronization barrier for multi-phase computations with dynamic party count.

**Advantages over CyclicBarrier:**
- Dynamic party registration/deregistration
- More flexible for complex scenarios
- Hierarchical phasers possible

**Use cases:**
- Multi-stage iterative algorithms
- Complex batch operations
- Simulation with varying participants

```java
public class PhaserExample {
    public static void main(String[] args) {
        Phaser phaser = new Phaser(1);  // Start with 1 party (main thread)
        
        for (int i = 0; i < 3; i++) {
            phaser.register();  // Dynamically register new party
            int threadId = i;
            new Thread(() -> {
                System.out.println("Thread " + threadId + " in phase " + phaser.getPhase());
                phaser.arriveAndAwaitAdvance();  // Arrive and wait for others
                
                System.out.println("Thread " + threadId + " in phase " + phaser.getPhase());
                phaser.arriveAndDeregister();  // Signal done and leave
            }).start();
        }
        
        // Main thread waits for all
        phaser.arriveAndDeregister();
    }
}
```

---

## Concurrent Collections

**Why they matter:** Thread-safe alternatives to Collections. Avoid the need for external synchronization and provide better performance through fine-grained locking or lock-free algorithms.

**Never do this:**
```java
// WRONG - not thread-safe
List<String> list = new ArrayList<>();
for (String item : items) list.add(item);  // Race condition possible
```

**Use these instead:**

```java
import java.util.concurrent.*;

public class ConcurrentCollectionsExample {
    // Thread-safe map with segment locking (better than synchronized)
    ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
    
    // Thread-safe list for mostly-reads scenarios
    CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
    
    // Thread-safe unbounded queue (FIFO, lock-free)
    ConcurrentLinkedQueue<Integer> queue = new ConcurrentLinkedQueue<>();
    
    // BlockingQueue for producer-consumer pattern
    BlockingQueue<String> blockingQueue = new LinkedBlockingQueue<>(10);
    
    // ConcurrentHashMap usage - atomic compound operations
    public void useMap() {
        map.put("key", 100);
        
        // Atomic putIfAbsent - single atomic operation
        map.putIfAbsent("key", 200);  // Returns old value if present
        
        // Atomic compute operations
        map.computeIfPresent("key", (k, v) -> v + 1);
        map.computeIfAbsent("newKey", k -> 0);
        
        // Iteration is safe (snapshot-style)
        for (String key : map.keySet()) {
            System.out.println(key);
        }
    }
    
    // CopyOnWriteArrayList - good for read-heavy workloads
    public void useList() {
        list.add("item");  // Copies entire array on write
        list.remove(0);    // Also copies
        
        // Iteration doesn't require synchronization
        // Iterator sees consistent snapshot even during writes
        for (String item : list) {
            System.out.println(item);
        }
    }
    
    // ConcurrentLinkedQueue - unbounded, lock-free
    public void useQueue() {
        queue.offer("item");  // Always succeeds (unbounded)
        String item = queue.poll();  // Returns null if empty
    }
    
    // BlockingQueue - producer-consumer synchronization
    public void producerConsumer() throws InterruptedException {
        // Producer thread
        blockingQueue.put("item");  // Blocks if queue full (capacity 10)
        blockingQueue.offer("item", 1, TimeUnit.SECONDS);  // Timeout version
        
        // Consumer thread
        String item = blockingQueue.take();  // Blocks if queue empty
        String itemOrNull = blockingQueue.poll(1, TimeUnit.SECONDS);  // Timeout
    }
}

// Common concurrent queues and when to use them
BlockingQueue<String> linked = new LinkedBlockingQueue<>();      // Unbounded
BlockingQueue<String> array = new ArrayBlockingQueue<>(10);      // Bounded
BlockingQueue<String> priority = new PriorityBlockingQueue<>();   // Priority order
BlockingQueue<String> sync = new SynchronousQueue<>();            // Direct handoff
```

**Comparison Table:**

| Collection | Thread-Safety | Locking Strategy | Use Case |
|------------|---------------|------------------|----------|
| `ConcurrentHashMap` | Yes | Segment locking | General concurrent map |
| `CopyOnWriteArrayList` | Yes | Copy-on-write | Read-heavy, few writes |
| `ConcurrentLinkedQueue` | Yes | Lock-free | High-throughput queue |
| `LinkedBlockingQueue` | Yes | Locks | Producer-consumer, unbounded |
| `ArrayBlockingQueue` | Yes | Locks | Producer-consumer, bounded |
| `PriorityBlockingQueue` | Yes | Locks | Priority ordering |
| `SynchronousQueue` | Yes | Direct handoff | Thread pool, direct transfer |

**Interview Q:** Why is ConcurrentHashMap better than Collections.synchronizedMap()?
**A:** Segment locking - multiple threads can modify different segments concurrently. synchronizedMap() locks entire map for every operation.

**Interview Q:** When would you use CopyOnWriteArrayList?
**A:** Read-heavy workloads where writes are rare. Iterators never throw ConcurrentModificationException because they work on snapshots.

---

## Virtual Threads

**What are they?** Lightweight threads introduced in Java 19 (preview), made standard in Java 21. Managed entirely by the JVM, not the OS.

**Why they matter:** Enable massive concurrency with minimal resource overhead. Traditional threads don't scale well for thousands of concurrent operations (e.g., web servers handling 10k+ connections).

**How they work:** 
- Virtual threads run on a small pool of platform threads (carrier threads)
- JVM switches virtual threads when they block on I/O
- No actual OS context switch needed
- Thousands/millions can run on handful of OS threads

**When blocking occurs:** Virtual thread is "unmounted" from carrier thread. Carrier thread can run another virtual thread. When I/O completes, virtual thread is "mounted" back on a carrier.

```java
public class VirtualThreadExample {
    public static void main(String[] args) throws InterruptedException {
        Runnable task = () -> {
            System.out.println("Hello from " + 
                (Thread.currentThread().isVirtual() ? "Virtual" : "Platform") 
                + " Thread");
        };
        
        // Platform Thread (traditional, OS-level)
        Thread platform = new Thread(task);
        platform.start();
        platform.join();
        
        // Virtual Thread (Java 19+)
        // Two ways to create:
        Thread virtual1 = Thread.ofVirtual().start(task);
        Thread virtual2 = Thread.startVirtualThread(task);  // Java 21+
        virtual1.join();
        virtual2.join();
        
        // Structured Concurrency (Java 21+)
        // Better resource management and error handling
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            scope.fork(task);
            scope.fork(task);
            scope.join();  // Wait for all to complete
        }
        
        // Pattern: massive concurrency
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 100_000; i++) {
                executor.submit(() -> {
                    // I/O operation
                    Thread.sleep(1000);
                    System.out.println("Task done");
                });
            }
        }
    }
}
```

**Key Differences:**
| Aspect | Platform Thread | Virtual Thread |
|--------|-----------------|----------------|
| Memory per thread | ~1-2MB | ~100-200 bytes |
| Creation cost | Expensive (OS call) | Cheap (JVM object) |
| Scalability | Limited (thousands) | Millions feasible |
| OS mapping | 1:1 to OS kernel thread | Many:1, managed by JVM |
| Context switch | Expensive (OS level) | Cheap (JVM level) |
| Blocking I/O | Blocks carrier thread | Unmounts, allows other work |
| CPU binding | CPU-intensive OK | Avoid long CPU work |
| Debugging | Works with debuggers | Still improving tools |

**Best for:**
- I/O-intensive applications (web servers, HTTP clients)
- High concurrency workloads (10k+ concurrent operations)
- Microservices handling many connections

**Not ideal for:**
- CPU-intensive tasks (virtual threads still compete for CPU cores)
- Long-running computations without I/O

**Interview Q:** How do virtual threads scale better than platform threads?
**A:** Virtual threads use JVM-level scheduling. 1 million virtual threads can run on 8 carrier threads (one per CPU core). Platform threads would require 1 million OS threads.

---

## Common Patterns

### Thread-Safe Singleton

**Pattern 1: Double-Checked Locking** - Lazily initialized, but has subtle issues
```java
public class Singleton {
    private static Singleton instance;
    
    private Singleton() {}
    
    // Double-checked locking - minimize lock acquisitions
    public static Singleton getInstance() {
        if (instance == null) {  // First check (no lock, fast path)
            synchronized(Singleton.class) {
                if (instance == null) {  // Second check (with lock)
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**Why double-check?**
- First check: avoids lock after instance created (performance)
- Second check: prevents multiple instances if multiple threads pass first check simultaneously

**Problem:** Requires volatile (which it's not), or relies on final constructor semantics (Java 5+)

**Pattern 2: Bill Pugh Singleton (Preferred)** - Thread-safe, lazy, elegant
```java
public class Singleton {
    private Singleton() {}
    
    // Holder class - initialized only on first access to getInstance()
    public static class SingletonHolder {
        public static final Singleton INSTANCE = new Singleton();
    }
    
    public static Singleton getInstance() {
        return SingletonHolder.INSTANCE;  // Class loading is thread-safe
    }
}
```

**Why this works:**
- JVM guarantees class initialization is thread-safe
- SingletonHolder class is only loaded when getInstance() called
- No synchronization needed, no performance overhead

**Pattern 3: Enum Singleton (Simplest)** - Handle serialization, reflection
```java
public enum Singleton {
    INSTANCE;
    
    public void doSomething() {
        System.out.println("Doing work");
    }
}

// Usage
Singleton s = Singleton.INSTANCE;
```

**Interview Q:** Why not use eager initialization?
**A:** Wastes resources if singleton never used. Lazy initialization defers creation until needed.

### Producer-Consumer

**Pattern:** Separate threads produce data, separate threads consume. Queue decouples them.

**Advantages:**
- Different production/consumption rates possible
- Backpressure when queue fills up
- Easy to add/remove producers/consumers

```java
public class ProducerConsumer {
    private final BlockingQueue<Integer> queue = new LinkedBlockingQueue<>(10);
    private volatile boolean running = true;
    
    public void producer(int id, int count) {
        for (int i = 0; i < count; i++) {
            try {
                int item = id * 1000 + i;
                queue.put(item);  // Blocks if queue full
                System.out.println("Producer " + id + " produced: " + item);
                Thread.sleep(100);  // Simulate work
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
    
    public void consumer(int id) {
        while (running) {
            try {
                Integer item = queue.poll(1, TimeUnit.SECONDS);  // Timeout prevents hanging
                if (item != null) {
                    System.out.println("Consumer " + id + " consumed: " + item);
                    Thread.sleep(200);  // Simulate processing
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
    
    public void shutdown() {
        running = false;
    }
}
```

**Interview Q:** What happens if producer is faster than consumer?
**A:** Queue fills up. Producer's put() blocks until consumer makes space. This is backpressure - prevents memory overflow.

### Thread Pool (ExecutorService)

**Why use it:** Reuse threads instead of creating new ones for each task. More efficient for many short-lived tasks.

**Built-in pools:**
- `newFixedThreadPool(n)` - fixed N threads, unbounded queue
- `newCachedThreadPool()` - dynamic threads, reuses if available
- `newSingleThreadExecutor()` - single background thread
- `newScheduledThreadPool(n)` - for scheduled/periodic tasks
- `newVirtualThreadPerTaskExecutor()` - one virtual thread per task (Java 21+)

```java
import java.util.concurrent.*;

public class ThreadPoolExample {
    public static void main(String[] args) throws InterruptedException {
        // Create fixed thread pool with 4 threads
        ExecutorService executor = Executors.newFixedThreadPool(4);
        
        // Submit 10 tasks - they queue up, 4 run concurrently
        for (int i = 0; i < 10; i++) {
            int taskId = i;
            executor.submit(() -> {
                System.out.println("Task " + taskId + " on " + Thread.currentThread().getName());
                try {
                    Thread.sleep(1000);  // Simulate work
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }
        
        // Don't submit new tasks
        executor.shutdown();
        
        // Wait for completion (with timeout)
        if (executor.awaitTermination(30, TimeUnit.SECONDS)) {
            System.out.println("All tasks completed");
        } else {
            System.out.println("Timeout - forcing shutdown");
            executor.shutdownNow();  // Interrupt remaining tasks
        }
    }
    
    // Advanced: Future and callbacks
    public static void advancedExample() throws ExecutionException, InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(2);
        
        // Submit and get Future
        Future<Integer> future = executor.submit(() -> {
            Thread.sleep(1000);
            return 42;
        });
        
        // Wait for result with timeout
        Integer result = future.get(2, TimeUnit.SECONDS);
        System.out.println("Result: " + result);
        
        executor.shutdown();
    }
}
```

**Pattern: Submit and track**
```java
List<Future<?>> futures = new ArrayList<>();
for (int i = 0; i < 10; i++) {
    futures.add(executor.submit(() -> doWork()));
}

// Wait for all
for (Future<?> future : futures) {
    future.get();  // Blocks until done
}
```

**Interview Q:** Fixed thread pool with unbounded queue - what's the risk?
**A:** If tasks arrive faster than threads can process, queue grows unbounded = memory exhaustion. Solution: bounded queue + rejection policy.

---

## Interview Cheat Sheet

| Problem | Solution | Key Concept |
|---------|----------|------------|
| Shared counter across threads | `AtomicInteger` or `synchronized` | Atomicity |
| One thread waits for N threads | `CountDownLatch` | Thread coordination |
| All threads wait at barrier | `CyclicBarrier` or `Phaser` | Synchronization point |
| Rate limiting (N threads max) | `Semaphore` | Resource pooling |
| Thread-safe map | `ConcurrentHashMap` | Lock-free data structure |
| Many readers, few writers | `ReadWriteLock` | Optimize read throughput |
| Producer-consumer pattern | `BlockingQueue` | Thread coordination |
| Prevent ABA problem | `AtomicStampedReference` | CAS correctness |
