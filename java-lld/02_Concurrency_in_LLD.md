# Concurrency in LLD — Deep Dive

> This is the #1 differentiator between SDE1 and SDE2 answers.
> When you say "this is thread-safe", you must be able to PROVE it.

---

## Java Memory Model (JMM) — What You Must Know

The JMM defines when changes made by one thread are visible to another.

**The problem — CPU caches:**
```
Thread 1 (CPU Core 1)     Thread 2 (CPU Core 2)
    L1 Cache                   L1 Cache
       |                           |
       └──────── Main Memory ──────┘

Thread 1 writes x = 5 to its L1 cache.
Thread 2 reads x → still sees x = 0 (stale value in its own cache).
```

**Three guarantees you need:**
1. **Visibility** — when thread A writes, thread B eventually sees it
2. **Atomicity** — operation completes fully or not at all (no partial state)
3. **Ordering** — CPU/compiler can reorder instructions; we need to control this

---

## synchronized

**What it provides:** Atomicity + Visibility + Ordering (the complete package).

```java
public class SafeCounter {
    private int count = 0;

    // Mutual exclusion: only one thread at a time
    public synchronized void increment() {
        count++;  // read-modify-write — now atomic
    }

    public synchronized int get() {
        return count;  // synchronized on read too! (visibility)
    }
}
```

**synchronized on different locks:**
```java
public class BankAccount {
    private double balance;
    private final Object lock = new Object();  // explicit lock object

    public void deposit(double amount) {
        synchronized (lock) {            // instance-level lock
            balance += amount;
        }
    }

    public static synchronized void processGlobal() {  // class-level lock
        // ...
    }
}
```

**Interview: What is `synchronized` locking on?**
> `synchronized(this)` / `synchronized void method()` → locks on the **instance** object.
> `synchronized(ClassName.class)` / `static synchronized` → locks on the **Class** object.
> Two threads calling `synchronized` instance methods on DIFFERENT objects DON'T block each other.

**Deadlock with synchronized:**
```java
// DEADLOCK: Thread1 holds lockA, waits for lockB
//           Thread2 holds lockB, waits for lockA
class DeadlockExample {
    private final Object lockA = new Object();
    private final Object lockB = new Object();

    void method1() {
        synchronized (lockA) {
            synchronized (lockB) { /* work */ }  // always acquire in order: A then B
        }
    }

    void method2() {
        synchronized (lockB) {       // WRONG: different order from method1
            synchronized (lockA) { /* work */ }
        }
    }
}

// FIX: Always acquire locks in the same order
void method2Fixed() {
    synchronized (lockA) {           // same order as method1
        synchronized (lockB) { /* work */ }
    }
}
```

---

## volatile

**What it provides:** Visibility + Ordering. NOT atomicity.

```java
class StatusFlag {
    private volatile boolean running = true;  // volatile: writes visible across threads

    // Thread 1
    public void stop() {
        running = false;  // write is immediately visible to all threads
    }

    // Thread 2
    public void doWork() {
        while (running) {  // reads fresh value from main memory
            // do work
        }
    }
}
```

**When volatile is NOT enough:**
```java
private volatile int count = 0;

public void increment() {
    count++;  // STILL NOT THREAD-SAFE
    // count++ = read + modify + write (3 ops)
    // volatile only makes individual read/write visible
    // it does NOT make compound operations atomic
}
```

**When to use `volatile`:**
- Simple flags (`boolean running`)
- Publishing an immutable object reference (double-checked locking)
- Single writer, multiple readers scenario

---

## Atomic Classes

**What they provide:** Atomicity via CAS (Compare-And-Swap) — **no locking needed!**

```java
import java.util.concurrent.atomic.*;

public class AtomicExamples {
    private final AtomicInteger counter = new AtomicInteger(0);
    private final AtomicLong requestId = new AtomicLong(0);
    private final AtomicBoolean initialized = new AtomicBoolean(false);
    private final AtomicReference<String> status = new AtomicReference<>("PENDING");

    public void increment() {
        counter.incrementAndGet();  // atomic increment, no lock
    }

    public int getAndReset() {
        return counter.getAndSet(0);  // atomic swap
    }

    public boolean tryInitialize() {
        // CAS: only succeeds if current value is false
        return initialized.compareAndSet(false, true);
    }

    public void updateStatus(String expected, String newStatus) {
        // Only updates if current value matches expected
        boolean updated = status.compareAndSet(expected, newStatus);
        System.out.println("Update " + (updated ? "succeeded" : "failed (concurrent modification)"));
    }
}
```

**How CAS works internally:**
```
compareAndSet(expected, newValue):
1. Read current value
2. If current == expected → write newValue, return true
3. If current != expected → don't write, return false (another thread changed it)

This is a single CPU instruction (CMPXCHG on x86) — hardware-level atomicity, no OS lock needed.
```

**AtomicReference for complex state:**
```java
// Immutable config — update atomically
public class ConfigManager {
    private final AtomicReference<AppConfig> config =
        new AtomicReference<>(new AppConfig("v1", 100));

    public void updateConfig(AppConfig newConfig) {
        config.set(newConfig);  // atomic reference swap
    }

    public AppConfig getConfig() {
        return config.get();  // always returns consistent snapshot
    }
}
```

---

## ReentrantLock vs synchronized

```java
import java.util.concurrent.locks.*;

public class BankTransfer {
    private double balance;
    private final ReentrantLock lock = new ReentrantLock();

    // ReentrantLock gives you MORE control than synchronized
    public boolean transfer(BankTransfer target, double amount) {
        // tryLock: non-blocking, won't wait forever (prevents deadlock)
        if (lock.tryLock()) {
            try {
                if (target.lock.tryLock()) {
                    try {
                        if (this.balance < amount) return false;
                        this.balance -= amount;
                        target.balance += amount;
                        return true;
                    } finally {
                        target.lock.unlock();
                    }
                }
            } finally {
                lock.unlock();
            }
        }
        return false;  // couldn't acquire both locks
    }

    // Lock with timeout — avoid waiting forever
    public void withdraw(double amount) throws InterruptedException {
        if (lock.tryLock(5, TimeUnit.SECONDS)) {
            try {
                balance -= amount;
            } finally {
                lock.unlock();  // ALWAYS unlock in finally!
            }
        } else {
            throw new RuntimeException("Couldn't acquire lock in 5 seconds");
        }
    }
}
```

| Feature | synchronized | ReentrantLock |
|---|---|---|
| Automatic unlock | Yes (exits block) | No — must call unlock() in finally |
| Try to acquire | No | `tryLock()` |
| Timed acquire | No | `tryLock(timeout, unit)` |
| Interruptible wait | No | `lockInterruptibly()` |
| Fairness | No | `new ReentrantLock(true)` |
| Multiple conditions | No (one wait set) | `lock.newCondition()` multiple |
| Performance | Similar (JVM optimized) | Similar |

**Rule of thumb:** Default to `synchronized`. Only use `ReentrantLock` when you need tryLock, timeout, or multiple conditions.

---

## ReadWriteLock

**Problem:** Many reads are fine concurrently, but writes need exclusive access.

```java
public class SafeUserCache {
    private final Map<Long, User> cache = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    public User get(Long userId) {
        readLock.lock();   // multiple readers can hold this simultaneously
        try {
            return cache.get(userId);
        } finally {
            readLock.unlock();
        }
    }

    public void put(Long userId, User user) {
        writeLock.lock();  // exclusive — no readers or writers while this is held
        try {
            cache.put(userId, user);
        } finally {
            writeLock.unlock();
        }
    }
}
```

**Use case:** Caches, configuration stores — heavy reads, occasional writes.
**Note:** `ConcurrentHashMap` is often better for simple cases — it has internal segment-level locking.

---

## Thread-Safe Collections — Choosing the Right One

| Collection | Thread Safe? | Notes |
|---|---|---|
| `ArrayList`, `HashMap` | No | Never use in shared context |
| `Vector`, `Hashtable` | Yes (legacy) | Every method synchronized — very slow |
| `Collections.synchronizedList(list)` | Yes | Synchronized on every operation; iteration still needs external sync |
| `CopyOnWriteArrayList` | Yes | Write = copy entire array. Great for read-heavy, rare writes |
| `ConcurrentHashMap` | Yes | Best for high-concurrency maps; no full lock |
| `ConcurrentLinkedQueue` | Yes | Lock-free queue for producer-consumer |
| `LinkedBlockingQueue` | Yes | Blocking queue — ideal for producer-consumer |
| `PriorityBlockingQueue` | Yes | Thread-safe priority queue |

```java
// CopyOnWriteArrayList — for observer lists
private final List<Observer> observers = new CopyOnWriteArrayList<>();

// SAFE to iterate while another thread adds/removes:
for (Observer o : observers) {  // iterates over snapshot copy
    o.update();
}

// ConcurrentHashMap — for caches
private final Map<String, Object> cache = new ConcurrentHashMap<>();

// computeIfAbsent is atomic — won't duplicate DB calls for same key
Object value = cache.computeIfAbsent(key, k -> loadFromDB(k));
```

---

## Immutability — The Best Thread Safety

An immutable object can be safely shared across threads with **zero synchronization**.

```java
// IMMUTABLE class recipe:
// 1. final class (no subclassing)
// 2. All fields final and private
// 3. No setters
// 4. Deep copy of mutable fields in constructor and getters
public final class Money {
    private final long amountInPaise;
    private final String currency;

    public Money(long amountInPaise, String currency) {
        if (amountInPaise < 0) throw new IllegalArgumentException();
        this.amountInPaise = amountInPaise;
        this.currency = Objects.requireNonNull(currency);
    }

    public long getAmountInPaise() { return amountInPaise; }
    public String getCurrency() { return currency; }

    // Operations return NEW objects, never modify this
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) throw new IllegalArgumentException("Currency mismatch");
        return new Money(this.amountInPaise + other.amountInPaise, this.currency);
    }

    public Money multiply(int factor) {
        return new Money(this.amountInPaise * factor, this.currency);
    }

    @Override
    public String toString() { return amountInPaise / 100.0 + " " + currency; }
}

// Thread-safe without any locks — both threads read the same immutable object
Money price = new Money(50000, "INR");  // ₹500.00
Money discounted = price.multiply(9).divide(10);  // ₹450.00, new object
```

---

## Common Concurrency Bugs in LLD Designs

### Bug 1: Lazy Initialization Race

```java
// BROKEN — two threads can both create the object
public Connection getConnection() {
    if (connection == null) {                    // Thread A and B both see null
        connection = new Connection(dbUrl);       // both create! one is lost
    }
    return connection;
}

// FIX 1 — synchronized (simple but blocks every call)
public synchronized Connection getConnection() {
    if (connection == null) connection = new Connection(dbUrl);
    return connection;
}

// FIX 2 — double-checked locking
public Connection getConnection() {
    if (connection == null) {
        synchronized (this) {
            if (connection == null) connection = new Connection(dbUrl);
        }
    }
    return connection;
}
// Note: `connection` MUST be volatile for this to be correct!
```

### Bug 2: Check-Then-Act

```java
// BROKEN — gap between check and act
Map<String, Integer> inventory = new HashMap<>();

public void sell(String item) {
    if (inventory.getOrDefault(item, 0) > 0) {    // Thread A checks: 1 item left
        // context switch! Thread B also checks: 1 item left, also proceeds
        inventory.put(item, inventory.get(item) - 1);  // BOTH decrement: -1 items!
    }
}

// FIX — atomic check-and-act with ConcurrentHashMap
ConcurrentHashMap<String, Integer> inventory = new ConcurrentHashMap<>();

public boolean sell(String item) {
    // compute is atomic — replaces check-then-act
    int[] sold = {0};
    inventory.compute(item, (k, v) -> {
        if (v == null || v <= 0) return v;  // out of stock
        sold[0] = 1;
        return v - 1;
    });
    return sold[0] == 1;
}
```

### Bug 3: Partially Constructed Object

```java
// BROKEN — other thread sees object before constructor finishes
class Config {
    String url;
    int port;
    boolean initialized;

    Config() {
        url = "localhost";
        port = 5432;
        initialized = true;  // JVM CAN reorder! initialized=true might be written before url is set
    }
}

volatile Config sharedConfig = null;

// FIX — use volatile for the reference (guarantees ordering)
volatile Config sharedConfig = null;
// All writes in constructor complete before volatile write to sharedConfig
sharedConfig = new Config();  // safe with volatile on sharedConfig
```

### Bug 4: Iterator + ConcurrentModification

```java
List<String> items = new ArrayList<>();

// BROKEN — iterating while another thread modifies
for (String item : items) {  // ConcurrentModificationException if items changes
    process(item);
}

// FIX 1 — snapshot
for (String item : new ArrayList<>(items)) {  // iterate over copy
    process(item);
}

// FIX 2 — CopyOnWriteArrayList
List<String> items = new CopyOnWriteArrayList<>();
for (String item : items) {  // iterates over snapshot, safe
    process(item);
}
```

---

## Concurrency Interview Questions

**Q: Explain `happens-before` in the Java Memory Model.**
> A `happens-before` relationship guarantees that all memory writes by one specific action are visible to another action. Key rules:
> - Thread start: `thread.start()` happens-before first action in that thread.
> - Monitor unlock: Unlocking a `synchronized` block happens-before every subsequent lock of that same monitor.
> - volatile write: Writing to a volatile variable happens-before every subsequent read of that variable.
> - Thread termination: All actions in a thread happen-before `thread.join()` returns.

**Q: What is false sharing? How does it affect performance?**
> Modern CPUs transfer data in cache lines (~64 bytes). If two variables are on the same cache line, and two threads update different variables, the cache line bounces between cores — each write invalidates the other core's cache, causing massive slowdown.
>
> Fix: Pad variables to ensure they're on separate cache lines (Java 8+ `@Contended` annotation on fields).

**Q: Explain the ABA problem in CAS operations.**
> Thread 1 reads value A. Thread 2 changes A → B → A (back to A). Thread 1 does CAS(A, newValue) — succeeds! But the value WAS changed, even though it looks the same.
> Solution: Use `AtomicStampedReference` — adds a version counter. CAS checks both the value AND the version stamp.

**Q: What's the difference between `CountDownLatch` and `CyclicBarrier`?**
> `CountDownLatch`: A gate that counts down from N to 0. One or more threads wait at the gate; N other threads call `countDown()`. Gate opens once. Not reusable.
> `CyclicBarrier`: N threads all wait at a barrier; only when all N have arrived do they all proceed together. Reusable (cyclic).
> Use Latch for "wait for N events to complete." Use Barrier for "wait for N threads to reach the same checkpoint."

**Q: Thread starvation vs deadlock vs livelock?**
> - **Deadlock:** Threads are blocked forever — each holds a resource the other needs.
> - **Starvation:** Thread is never scheduled — other threads always get priority.
> - **Livelock:** Threads are NOT blocked but keep responding to each other and making no progress (two people in a corridor both stepping aside in the same direction forever).
