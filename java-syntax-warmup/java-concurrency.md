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

### Core Concepts
- **Visibility**: Changes by one thread are visible to other threads
- **Atomicity**: Operation completes without interruption
- **Ordering**: Order of operations is guaranteed

### `volatile` Keyword
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
```

### `final` Keyword
```java
// Ensures safe publication and immutability
public class FinalExample {
    private final int x;
    private final List<String> list;
    
    public FinalExample(int x, List<String> list) {
        this.x = x;
        this.list = new ArrayList<>(list);
    }
    // Once constructed, x and list references cannot change
    // Safe to share across threads without synchronization
}
```

---

## Synchronization Basics

### `synchronized` Keyword
```java
public class SynchronizedExample {
    private int counter = 0;
    
    // Instance method lock (locks on 'this')
    public synchronized void increment() {
        counter++;
    }
    
    // Static method lock (locks on class object)
    public static synchronized void staticMethod() {
        // ...
    }
    
    // Block-level lock (fine-grained control)
    public void incrementFast() {
        synchronized(this) {
            counter++;
        }
    }
    
    // Lock on different object
    private final Object lock = new Object();
    public void incrementSafe() {
        synchronized(lock) {
            counter++;
        }
    }
}
// Guarantees: mutual exclusion, visibility of memory changes
// Limitations: coarse-grained locking, no timeout, can lead to deadlocks
```

---

## Atomic Operations

### Atomic Classes
```java
import java.util.concurrent.atomic.*;

public class AtomicExample {
    private AtomicInteger counter = new AtomicInteger(0);
    private AtomicReference<String> name = new AtomicReference<>("Alice");
    
    public void increment() {
        counter.incrementAndGet();  // Atomic operation
        counter.addAndGet(5);
        counter.getAndSet(100);
    }
    
    // Compare-and-swap (CAS)
    public boolean compareAndIncrement(int expected) {
        return counter.compareAndSet(expected, expected + 1);
    }
    
    // Atomic reference update
    public void updateName(String newName) {
        name.set(newName);
        String current = name.getAndSet(newName);
    }
}
// Lock-free, higher performance than synchronized for high contention
// Uses hardware-level CAS (Compare-And-Swap) instruction
```

### Common Atomic Classes
- `AtomicInteger`, `AtomicLong`, `AtomicBoolean`
- `AtomicReference<T>`, `AtomicReferenceArray<T>`
- `AtomicStampedReference<V>` — solves ABA problem
- `AtomicMarkableReference<V>` — marks if value is logically deleted

---

## Locks

### ReentrantLock
```java
import java.util.concurrent.locks.*;

public class ReentrantLockExample {
    private final ReentrantLock lock = new ReentrantLock();
    private int counter = 0;
    
    public void increment() {
        lock.lock();
        try {
            counter++;
        } finally {
            lock.unlock();  // Always unlock in finally
        }
    }
    
    // With timeout
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
}
```

### ReadWriteLock
```java
private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
private int counter = 0;

public int read() {
    rwLock.readLock().lock();
    try {
        return counter;  // Multiple readers can read simultaneously
    } finally {
        rwLock.readLock().unlock();
    }
}

public void write(int value) {
    rwLock.writeLock().lock();
    try {
        counter = value;  // Only one writer at a time, blocks readers
    } finally {
        rwLock.writeLock().unlock();
    }
}
// Advantages: multiple concurrent readers, exclusive writer
// Use when: many reads, few writes
```

---

## Thread Coordination

### CountDownLatch
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
                    latch.countDown();  // Decrement counter
                }
            }).start();
        }
        
        latch.await();  // Wait until counter reaches 0
        System.out.println("All workers finished");
    }
}
// One-time use. Cannot be reset.
```

### CyclicBarrier
```java
public class CyclicBarrierExample {
    public static void main(String[] args) throws InterruptedException {
        int numThreads = 3;
        CyclicBarrier barrier = new CyclicBarrier(numThreads, () -> {
            System.out.println("All threads reached barrier!");
        });
        
        for (int i = 0; i < numThreads; i++) {
            new Thread(() -> {
                try {
                    System.out.println("Thread waiting at barrier");
                    barrier.await();  // Wait for all threads
                    System.out.println("Thread proceeding");
                } catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
// Reusable. Multiple barriers can be used sequentially.
```

### Semaphore
```java
public class SemaphoreExample {
    private final Semaphore semaphore = new Semaphore(3);  // Allow 3 threads
    
    public void accessResource() throws InterruptedException {
        semaphore.acquire();  // Decrement permit count
        try {
            System.out.println("Accessing resource");
            Thread.sleep(1000);
        } finally {
            semaphore.release();  // Increment permit count
        }
    }
    
    // Non-blocking acquire
    public boolean tryAccess() {
        if (semaphore.tryAcquire()) {
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
// Use: rate limiting, resource pooling
```

### Phaser
```java
public class PhaserExample {
    public static void main(String[] args) {
        Phaser phaser = new Phaser(3);  // 3 parties to register
        
        for (int i = 0; i < 3; i++) {
            new Thread(() -> {
                System.out.println("Phase 1: " + Thread.currentThread().getName());
                phaser.arriveAndAwaitAdvance();  // Wait for all to reach phase 2
                
                System.out.println("Phase 2: " + Thread.currentThread().getName());
                phaser.arriveAndDeregister();  // Signal done and remove from phaser
            }).start();
        }
    }
}
// Like CyclicBarrier but more flexible with dynamic thread count
```

---

## Concurrent Collections

```java
import java.util.concurrent.*;

public class ConcurrentCollectionsExample {
    // Thread-safe map, no global lock
    ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
    
    // Thread-safe list
    CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
    
    // Thread-safe queue (FIFO)
    ConcurrentLinkedQueue<Integer> queue = new ConcurrentLinkedQueue<>();
    
    // BlockingQueue for producer-consumer
    BlockingQueue<String> blockingQueue = new LinkedBlockingQueue<>(10);
    
    public void useMap() {
        map.put("key", 100);
        map.putIfAbsent("key", 200);  // Atomic conditional put
        map.computeIfPresent("key", (k, v) -> v + 1);
    }
    
    public void producerConsumer() throws InterruptedException {
        // Producer
        blockingQueue.put("item");  // Blocks if queue full
        
        // Consumer
        String item = blockingQueue.take();  // Blocks if queue empty
        String itemOrNull = blockingQueue.poll(1, TimeUnit.SECONDS);
    }
}
```

---

## Virtual Threads

```java
public class VirtualThreadExample {
    public static void main(String[] args) throws InterruptedException {
        Runnable task = () -> {
            System.out.println("Hello from " + 
                (Thread.currentThread().isVirtual() ? "Virtual" : "Platform") 
                + " Thread");
        };
        
        // Platform Thread
        Thread platform = new Thread(task);
        platform.start();
        platform.join();
        
        // Virtual Thread (Java 19+)
        Thread virtual = Thread.ofVirtual().start(task);
        virtual.join();
        
        // Structured Concurrency (Java 21+)
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            scope.fork(task);
            scope.fork(task);
            scope.join();
        }
    }
}
```

**Key Differences:**
| Aspect | Platform Thread | Virtual Thread |
|--------|-----------------|----------------|
| Memory | ~1MB per thread | ~1KB per thread |
| Creation | Expensive | Cheap |
| Scalability | Limited (thousands) | Millions possible |
| OS mapping | 1:1 to OS thread | Many:1, managed by JVM |
| Context switch | Expensive | Cheap (JVM level) |

---

## Common Patterns

### Thread-Safe Singleton
```java
public class Singleton {
    private static Singleton instance;
    
    private Singleton() {}
    
    // Double-checked locking
    public static Singleton getInstance() {
        if (instance == null) {  // First check (no lock)
            synchronized(Singleton.class) {
                if (instance == null) {  // Second check (with lock)
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
    
    // Better: eager initialization
    public static class SingletonHolder {
        public static final Singleton INSTANCE = new Singleton();
    }
    
    public static Singleton getSingleton() {
        return SingletonHolder.INSTANCE;
    }
}
```

### Producer-Consumer
```java
public class ProducerConsumer {
    private final BlockingQueue<Integer> queue = new LinkedBlockingQueue<>(10);
    
    public void producer() {
        for (int i = 0; i < 100; i++) {
            try {
                queue.put(i);
                System.out.println("Produced: " + i);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
    
    public void consumer() {
        while (true) {
            try {
                Integer item = queue.take();
                System.out.println("Consumed: " + item);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
}
```

### Thread Pool (ExecutorService)
```java
import java.util.concurrent.*;

public class ThreadPoolExample {
    public static void main(String[] args) {
        // Fixed thread pool
        ExecutorService executor = Executors.newFixedThreadPool(4);
        
        // Submit tasks
        for (int i = 0; i < 10; i++) {
            int taskId = i;
            executor.submit(() -> {
                System.out.println("Task " + taskId + " executing");
            });
        }
        
        executor.shutdown();  // No new tasks accepted
        executor.awaitTermination(10, TimeUnit.SECONDS);
        System.out.println("All tasks completed");
    }
}
```

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
