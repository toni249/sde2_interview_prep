# LLD: LRU Cache

> Frequency: Very High | Difficulty: Medium-High
> Tests: Data structure design, O(1) operations, thread safety, cache patterns

---

## Step 1 — Clarifying Questions

- Fixed capacity? What to evict when full?
- TTL (time-to-live) per entry?
- Thread-safe? Read-heavy or write-heavy?
- Should it notify on eviction?
- Single machine or distributed?

---

## Step 2 — Core Algorithm

LRU (Least Recently Used) evicts the entry that was accessed least recently.

**Required operations, both O(1):**
- `get(key)` → return value, mark as most recently used
- `put(key, value)` → insert/update, evict LRU if at capacity

**Data structure:**
```
HashMap<K, Node> → O(1) lookup by key
Doubly Linked List → O(1) move-to-front, O(1) remove from tail

[ HEAD ↔ most_recent ↔ ... ↔ least_recent ↔ TAIL ]
   (sentinel)                                (sentinel)

get(k): find node via map → move to front
put(k,v): add to front → if full, remove tail → update map
```

---

## Step 3 — Implementation 1: Using LinkedHashMap (Interview Fast Answer)

```java
public class LRUCache<K, V> {
    private final int capacity;
    // LinkedHashMap with access-order mode = LRU order
    private final LinkedHashMap<K, V> cache;

    public LRUCache(int capacity) {
        this.capacity = capacity;
        // true = access-order (most-recently-accessed at end)
        this.cache = new LinkedHashMap<K, V>(capacity, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                return size() > capacity;  // evict when over capacity
            }
        };
    }

    public synchronized V get(K key) {
        return cache.getOrDefault(key, null);
    }

    public synchronized void put(K key, V value) {
        cache.put(key, value);
    }

    public synchronized boolean containsKey(K key) {
        return cache.containsKey(key);
    }

    public int size() { return cache.size(); }
}
```

**When to use this:** When you want a quick, correct implementation and don't need custom behavior.

---

## Step 4 — Implementation 2: Custom (What Interviewers Want to See)

```java
public class LRUCache<K, V> {
    private final int capacity;
    private final Map<K, Node<K, V>> map;

    // Sentinel nodes — avoid null checks
    private final Node<K, V> head;  // most recently used end
    private final Node<K, V> tail;  // least recently used end
    private int size;

    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.map = new HashMap<>();
        this.head = new Node<>(null, null);
        this.tail = new Node<>(null, null);
        head.next = tail;
        tail.prev = head;
        this.size = 0;
    }

    // ─── O(1) GET ───
    public V get(K key) {
        Node<K, V> node = map.get(key);
        if (node == null) return null;
        moveToFront(node);  // mark as recently used
        return node.value;
    }

    // ─── O(1) PUT ───
    public void put(K key, V value) {
        Node<K, V> existing = map.get(key);
        if (existing != null) {
            existing.value = value;
            moveToFront(existing);
            return;
        }

        // New entry
        Node<K, V> newNode = new Node<>(key, value);
        map.put(key, newNode);
        addToFront(newNode);
        size++;

        if (size > capacity) {
            Node<K, V> lru = removeTail();  // evict LRU
            map.remove(lru.key);
            size--;
        }
    }

    // ─── DOUBLY LINKED LIST OPERATIONS ───
    private void addToFront(Node<K, V> node) {
        node.prev = head;
        node.next = head.next;
        head.next.prev = node;
        head.next = node;
    }

    private void removeNode(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void moveToFront(Node<K, V> node) {
        removeNode(node);
        addToFront(node);
    }

    private Node<K, V> removeTail() {
        Node<K, V> lru = tail.prev;  // last real node before sentinel tail
        removeNode(lru);
        return lru;
    }

    public int size() { return size; }

    // ─── NODE ───
    private static class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev;
        Node<K, V> next;

        Node(K key, V value) { this.key = key; this.value = value; }
    }
}
```

**Test it:**
```java
LRUCache<Integer, String> cache = new LRUCache<>(3);
cache.put(1, "one");
cache.put(2, "two");
cache.put(3, "three");
cache.get(1);           // access 1 → order: 1, 3, 2
cache.put(4, "four");   // evicts 2 (LRU)
System.out.println(cache.get(2));  // null — evicted
System.out.println(cache.get(1));  // "one"
System.out.println(cache.get(4));  // "four"
```

---

## Step 5 — Thread-Safe LRU Cache

```java
public class ThreadSafeLRUCache<K, V> {
    private final int capacity;
    private final Map<K, Node<K, V>> map = new HashMap<>();
    private final Node<K, V> head;
    private final Node<K, V> tail;
    private int size;
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    public ThreadSafeLRUCache(int capacity) {
        this.capacity = capacity;
        this.head = new Node<>(null, null);
        this.tail = new Node<>(null, null);
        head.next = tail; tail.prev = head;
    }

    // get() modifies the list (moveToFront), so needs WRITE lock
    public V get(K key) {
        writeLock.lock();
        try {
            Node<K, V> node = map.get(key);
            if (node == null) return null;
            moveToFront(node);
            return node.value;
        } finally {
            writeLock.unlock();
        }
    }

    public void put(K key, V value) {
        writeLock.lock();
        try {
            Node<K, V> existing = map.get(key);
            if (existing != null) {
                existing.value = value;
                moveToFront(existing);
                return;
            }
            Node<K, V> newNode = new Node<>(key, value);
            map.put(key, newNode);
            addToFront(newNode);
            size++;
            if (size > capacity) { map.remove(removeTail().key); size--; }
        } finally {
            writeLock.unlock();
        }
    }

    private void addToFront(Node<K, V> node) { /* same as before */ }
    private void removeNode(Node<K, V> node) { /* same as before */ }
    private void moveToFront(Node<K, V> node) { removeNode(node); addToFront(node); }
    private Node<K, V> removeTail() { Node<K, V> lru = tail.prev; removeNode(lru); return lru; }

    private static class Node<K, V> { K key; V value; Node<K, V> prev, next;
        Node(K k, V v) { key = k; value = v; } }
}
```

---

## Step 6 — LRU Cache with TTL (Expiration)

```java
public class TTLLRUCache<K, V> {
    private final int capacity;
    private final long ttlMs;  // time-to-live per entry
    private final Map<K, CacheEntry<K, V>> map = new HashMap<>();
    private final Node<K, V> head, tail;
    private int size;

    public TTLLRUCache(int capacity, long ttlMs) {
        this.capacity = capacity;
        this.ttlMs = ttlMs;
        this.head = new Node<>(null, null);
        this.tail = new Node<>(null, null);
        head.next = tail; tail.prev = head;
    }

    public synchronized V get(K key) {
        CacheEntry<K, V> entry = map.get(key);
        if (entry == null) return null;

        // Check expiry
        if (System.currentTimeMillis() > entry.expiryTime) {
            removeNode(entry.node);
            map.remove(key);
            size--;
            return null;  // expired = miss
        }

        moveToFront(entry.node);
        return entry.node.value;
    }

    public synchronized void put(K key, V value) {
        long expiryTime = System.currentTimeMillis() + ttlMs;

        CacheEntry<K, V> existing = map.get(key);
        if (existing != null) {
            existing.node.value = value;
            existing.expiryTime = expiryTime;
            moveToFront(existing.node);
            return;
        }

        Node<K, V> node = new Node<>(key, value);
        CacheEntry<K, V> entry = new CacheEntry<>(node, expiryTime);
        map.put(key, entry);
        addToFront(node);
        size++;

        if (size > capacity) {
            Node<K, V> lru = removeTail();
            map.remove(lru.key);
            size--;
        }
    }

    private static class CacheEntry<K, V> {
        Node<K, V> node;
        long expiryTime;
        CacheEntry(Node<K, V> node, long expiryTime) { this.node = node; this.expiryTime = expiryTime; }
    }

    private static class Node<K, V> { K key; V value; Node<K, V> prev, next;
        Node(K k, V v) { key = k; value = v; } }

    // List operations same as before...
    private void addToFront(Node<K, V> n) { n.prev = head; n.next = head.next; head.next.prev = n; head.next = n; }
    private void removeNode(Node<K, V> n) { n.prev.next = n.next; n.next.prev = n.prev; }
    private void moveToFront(Node<K, V> n) { removeNode(n); addToFront(n); }
    private Node<K, V> removeTail() { Node<K, V> lru = tail.prev; removeNode(lru); return lru; }
}
```

---

## Step 7 — Concurrency Follow-up Questions

**Q: Why does `get()` need a write lock? It's just reading a value.**
> Because `get()` calls `moveToFront()` which **modifies the doubly linked list**. Even though we're reading the value, we're writing to the list structure. If two threads both call `get()`, both would try to rearrange the list, causing corruption.
>
> This is why LRU is deceptively hard to parallelize — every read mutates structure.

**Q: Can you make LRU cache non-blocking (lock-free)?**
> True lock-free LRU is very complex. Practical approach: use `ConcurrentLinkedHashMap` library or "concurrent LRU" variants that approximate LRU behavior. One approach: use multiple segments (like ConcurrentHashMap) with independent LRU lists. Not perfectly LRU across segments but good enough.

**Q: In `get()`, between checking expiry and removing from map, another thread could also call `get()` for the same key. Is there a race?**
> With `synchronized` on the method (or using a write lock), only one thread runs `get()` at a time, so no race. Without synchronization: yes — two threads could both see the key as expired, both try to `map.remove(key)`, and both return `null`. The second `map.remove()` just returns null (idempotent), so the result is still correct. But the list could be corrupted without synchronization.

**Q: How does Java's `LinkedHashMap` implement LRU internally?**
> `LinkedHashMap` extends `HashMap` and adds doubly-linked list pointers (`before`/`after`) on each entry. With `accessOrder=true`, `get()` internally calls `afterNodeAccess()` which moves the accessed entry to the tail. `removeEldestEntry()` is called after each `put()` to check if the oldest entry (head) should be removed. It's the same idea as our custom implementation, just built into the JDK.

**Q: What are the trade-offs of LRU vs LFU (Least Frequently Used)?**
| | LRU | LFU |
|---|---|---|
| Evicts | Least recently accessed | Least frequently accessed |
| Implementation | O(1) with hashmap + DLL | O(1) with hashmap + frequency lists |
| Best for | Temporal locality (recently used = likely used again) | Frequency locality (popular items should stay) |
| Weakness | Cold start: new popular items evict old ones initially | Frequency count never forgets old popularity |
| Example | Web page cache | CDN content cache |

**Q: How would you implement a cache that evicts based on size of value (not count)?**
> Track total bytes used instead of count. Each entry stores its byte size. On `put()`, calculate the size of the new value. Evict LRU entries (from tail) until `totalBytes + newSize <= capacity`. This is how `Guava CacheBuilder.maximumWeight()` works.
