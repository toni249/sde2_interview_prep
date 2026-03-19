# Java Map Internals: Complete Guide

## Table of Contents
1. [Map Interface Overview](#map-interface-overview)
2. [HashMap Internals](#hashmap-internals)
3. [LinkedHashMap Internals](#linkedhashmap-internals)
4. [TreeMap Internals](#treemap-internals)
5. [Thread-Safe Maps](#thread-safe-maps)
6. [Java Version Changes (8 → 21)](#java-version-changes-8--21)
7. [Performance Comparison](#performance-comparison)
8. [Common Interview Questions](#common-interview-questions)

---

## Map Interface Overview

```java
public interface Map<K, V> {
    // Key-value pairs, no duplicate keys
    // Each key maps to at most one value
    // Optional: maintains insertion order (LinkedHashMap) or sorted order (TreeMap)
}
```

**Key Characteristics:**
- No duplicate keys (uses `equals()` for comparison)
- At most one `null` key (HashMap/LinkedHashMap allow, TreeMap does not)
- Multiple `null` values allowed
- No guaranteed order (except LinkedHashMap & TreeMap)

---

## HashMap Internals

### Architecture

```
HashMap<K, V>
    ↓ (uses)
Array<Node<K, V>> (buckets)
    ↓ (collision resolution)
Linked List → Balanced Tree (Java 8+)
```

### Internal Structure (Java 8+)

```
HashMap [1→"one", 2→"two", 12→"twelve"]

Internal Table (capacity = 16):
┌─────────────────────────────────────────────────┐
│  Node<K,V>[] table                              │
│  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┐         │
│  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │...│         │  ← Buckets
│  ├───┼───┼───┼───┼───┼───┼───┼───┼───┤         │
│  │   │   │   │   │   │   │   │   │   │         │
│  │   │ A │ B │   │   │   │   │   │   │         │
│  │   │ │ │ │   │   │   │   │   │   │         │
│  │   │ │ └─→ [12,"twelve"]                     │
│  │   │ └───→ [2,"two"]                         │
│  │   └─────→ [1,"one"]                         │
│  └───┴───┴───┴───┴───┴───┴───┴───┴───┘         │
└─────────────────────────────────────────────────┘

Node Structure:
┌────────────────────────────────────────┐
│  Node<K,V>                             │
│  ├── final int hash                    │
│  ├── final K key                       │
│  ├── V value                           │
│  └── Node<K,V> next                    │
└────────────────────────────────────────┘

TreeNode Structure (when treeified):
┌────────────────────────────────────────┐
│  TreeNode<K,V> extends Node<K,V>      │
│  ├── TreeNode<K,V> parent             │
│  ├── TreeNode<K,V> left               │
│  ├── TreeNode<K,V> right              │
│  ├── TreeNode<K,V> prev               │
│  └── boolean red                      │
└────────────────────────────────────────┘
```

### Core Implementation

```java
// HashMap source code (simplified)
public class HashMap<K, V> extends AbstractMap<K, V> implements Map<K, V> {
    transient Node<K, V>[] table;
    transient int size;
    int threshold;
    final float loadFactor;
    
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;  // 16
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    static final int TREEIFY_THRESHOLD = 8;
    static final int UNTREEIFY_THRESHOLD = 6;
    static final int MIN_TREEIFY_CAPACITY = 64;
    
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
    
    public V get(Object key) {
        Node<K, V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
}
```

### Hash Calculation (Perturbation)

```java
// Java 8+ hash calculation
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// Why perturbation?
// 1. Mixes upper bits with lower bits
// 2. Reduces collisions when table length is small
// 3. Protects against poor hashCode() implementations

// Example:
// hashCode = 0x12345678
// h >>> 16 = 0x00001234
// h ^ (h >>> 16) = 0x1234444C  (better distribution)
```

### Bucket Index Calculation

```java
// Index calculation (Java 8+)
int indexFor(int hash, int tableLength) {
    return hash & (tableLength - 1);  // Bitwise AND (faster than modulo)
}

// Why power of 2?
// - tableLength - 1 creates a mask
// - Example: 16 - 1 = 15 = 0b1111
// - hash & 15 gives values 0-15 (perfect distribution)

// Example:
// hash = 0x1234444C = 305419340
// tableLength = 16
// index = 305419340 & 15 = 12
```

### put() Operation (Step by Step)

```java
// HashMap.put(key, value) - Java 8+

final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab = table;
    Node<K,V> p;
    int n, i;
    
    // Step 1: Initialize table if null or empty
    if (tab == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    
    // Step 2: Check if bucket is empty
    if ((p = tab[i = (n - 1) & hash]) == null) {
        tab[i] = newNode(hash, key, value, null);  // Direct insert
    } else {
        // Step 3: Bucket has entries (collision)
        Node<K,V> e; K k;
        
        // Step 3a: Check first node
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;  // Key exists
        // Step 3b: Check if tree node
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // Step 3c: Traverse linked list
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);  // Add to end
                    // Step 4: Check if treeify needed
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                    break;
                }
                // Check if key exists in chain
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        
        // Step 5: Update existing key
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;  // Return old value
        }
    }
    
    // Step 6: Update size and check resize
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;  // New key inserted
}
```

### get() Operation (Step by Step)

```java
// HashMap.get(key) - Java 8+

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e;
    int n; K k;
    
    // Step 1: Check table and bucket
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        
        // Step 2a: Check first node
        if (first.hash == hash && ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        
        // Step 2b: Check rest of chain
        if ((e = first.next) != null) {
            // If tree node, search tree
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // Otherwise traverse linked list
            do {
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;  // Key not found
}
```

### Collision Resolution: Treeify

```java
// When linked list → tree conversion happens
// Threshold: 8+ entries in same bucket
// Minimum capacity: 64

void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    
    // Check if table is large enough
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();  // Prefer resizing over treeifying
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        // Convert linked list to red-black tree
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            // ... link tree nodes ...
        } while ((e = e.next) != null);
        
        // Replace bucket head with tree root
        tab[index] = hd;
        if (hd != null)
            hd.treeify(tab);  // Balance the tree
    }
}

// Tree operations: O(log n) vs List: O(n)
// But tree has higher memory overhead
// So only treeify when beneficial
```

### Resize (Rehashing)

```java
// When size > threshold (capacity × loadFactor)
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    
    // Calculate new capacity
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;  // Can't grow further
        }
        newCap = oldCap << 1;  // Double (must be power of 2)
        newThr = oldThr << 1;  // Double threshold
    }
    
    // Create new table
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    threshold = newThr;
    
    // Rehash all entries
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;  // Help GC
                
                if (e.next == null) {
                    // Single node: simple rehash
                    newTab[e.hash & (newCap - 1)] = e;
                } else if (e instanceof TreeNode) {
                    // Tree: split or untreeify
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                } else {
                    // Linked list: split into two lists
                    // Java 8 optimization: no rehash needed!
                    // Nodes stay in same relative order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    
                    do {
                        next = e.next;
                        // Check if index changes with new capacity
                        if ((e.hash & oldCap) == 0) {
                            // Same index
                            if (loTail == null) loHead = e;
                            else loTail.next = e;
                            loTail = e;
                        } else {
                            // Index + oldCap
                            if (hiTail == null) hiHead = e;
                            else hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

### Load Factor Trade-offs

```java
// Default: 0.75 (good balance)

// Lower load factor (e.g., 0.5):
// ✅ Less collisions, faster operations
// ❌ More memory waste, more frequent resizing

// Higher load factor (e.g., 1.0):
// ✅ Better memory utilization
// ❌ More collisions, slower operations

// Choose based on use case:
HashMap<K, V> fastMap = new HashMap<>(1000, 0.5f);    // Speed priority
HashMap<K, V> compactMap = new HashMap<>(1000, 1.0f); // Memory priority
```

---

## LinkedHashMap Internals

### Architecture

```
LinkedHashMap<K, V>
    ↓ (extends)
HashMap<K, V>
    ↓ (maintains)
Doubly Linked List (insertion or access order)
```

### Internal Structure

```
LinkedHashMap [1→"one", 2→"two", 3→"three"]

HashMap Buckets + Doubly Linked List:
┌──────────────────────────────────────────────┐
│  HashMap Buckets                             │
│  ┌───┬───┬───┬───┐                          │
│  │ 0 │ 1 │ 2 │ 3 │                          │
│  ├───┼───┼───┼───┤                          │
│  │   │ A │ B │ C │  ← Entries               │
│  └───┴─┬─┴─┬─┴─┬─┘                          │
│        │   │   │                             │
│        ↓   ↓   ↓                             │
│  ┌─────────────────────────────────┐        │
│  │  Doubly Linked List             │        │
│  │  ┌─────┐   ┌─────┐   ┌─────┐   │        │
│  │  │  1  │ ↔ │  2  │ ↔ │  3  │   │        │
│  │  │"one"│   │"two"│   │"three"│  │        │
│  │  │pred │   │pred │   │pred │   │        │
│  │  │next │   │next │   │next │   │        │
│  │  └─────┘   └─────┘   └─────┘   │        │
│  │     ↑                   ↑       │        │
│  │  header              tail       │        │
│  └─────────────────────────────────┘        │
└──────────────────────────────────────────────┘

Entry Structure:
class LinkedHashMap.Entry<K,V> extends HashMap.Node<K,V> {
    LinkedHashMap.Entry<K,V> before, after;
    
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```

### Core Implementation

```java
// LinkedHashMap source code (simplified)
public class LinkedHashMap<K, V> extends HashMap<K, V> implements Map<K, V> {
    
    // Access order flag
    final boolean accessOrder;
    
    // Doubly linked list pointers
    transient LinkedHashMap.Entry<K,V> head, tail;
    
    public LinkedHashMap(int initialCapacity, float loadFactor, boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
    
    // Callbacks for maintaining linked list
    void afterNodeAccess(Node<K,V> e) {
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            // Move accessed node to tail (for access order)
            LinkedHashMap.Entry<K,V> p = (LinkedHashMap.Entry<K,V>)e;
            // ... relink to end ...
        }
    }
    
    void afterNodeInsertion(Node<K,V> e) {
        // Add new node to tail
        LinkedHashMap.Entry<K,V> p = (LinkedHashMap.Entry<K,V>)e;
        linkNodeLast(p);
    }
    
    void afterNodeRemoval(Node<K,V> e) {
        // Remove from linked list
        LinkedHashMap.Entry<K,V> p = (LinkedHashMap.Entry<K,V>)e;
        unlinkNode(p);
    }
}
```

### Insertion Order vs Access Order

```java
// Insertion Order (default)
LinkedHashMap<String, Integer> insertionOrder = new LinkedHashMap<>();
insertionOrder.put("A", 1);
insertionOrder.put("B", 2);
insertionOrder.put("C", 3);
insertionOrder.get("A");  // Access A

// Iteration: A → B → C (order of insertion)

// Access Order (LRU cache)
LinkedHashMap<String, Integer> accessOrder = new LinkedHashMap<>(16, 0.75f, true);
accessOrder.put("A", 1);
accessOrder.put("B", 2);
accessOrder.put("C", 3);
accessOrder.get("A");  // Access A

// Iteration: B → C → A (A moved to end)
```

### LRU Cache Implementation

```java
// Classic LRU Cache using LinkedHashMap
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxEntries;
    
    public LRUCache(int capacity) {
        super(capacity, 0.75f, true);  // Access order!
        this.maxEntries = capacity;
    }
    
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxEntries;  // Remove oldest when over capacity
    }
}

// Usage
LRUCache<String, String> cache = new LRUCache<>(100);
cache.put("key1", "value1");
cache.put("key2", "value2");
// ... after 100 entries, oldest automatically removed
```

---

## TreeMap Internals

### Architecture

```
TreeMap<K, V>
    ↓ (uses)
Red-Black Tree (self-balancing BST)
    ↓ (maintains)
Sorted order by natural ordering or Comparator
```

### Internal Structure

```
TreeMap [3→"three", 1→"one", 5→"five", 2→"two", 4→"four"]

Red-Black Tree:
         3(B)
        /   \
      1(R)   5(R)
        \    /
        2(B) 4(B)

B = Black, R = Red

Entry Structure:
static final class Entry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    Entry<K,V> left;
    Entry<K,V> right;
    Entry<K,V> parent;
    boolean color = BLACK;
    
    Entry(K key, V value, Entry<K,V> parent) {
        this.key = key;
        this.value = value;
        this.parent = parent;
    }
}
```

### Red-Black Tree Properties

```
1. Every node is either RED or BLACK
2. Root is always BLACK
3. All leaves (NULL) are BLACK
4. RED nodes must have BLACK children (no red-red)
5. All paths from any node to its leaves have same number of BLACK nodes

These properties ensure:
- Tree height ≤ 2 × log₂(n)
- Operations: O(log n) worst case
```

### put() Operation

```java
// TreeMap.put(key, value)

public V put(K key, V value) {
    Entry<K,V> t = root;
    
    if (t == null) {
        // Empty tree: first entry
        addEntryToEmptyMap(key, value);
        return null;
    }
    
    int cmp;
    Entry<K,V> parent;
    Comparator<? super K> cpr = comparator;
    
    // Find insertion point
    if (cpr != null) {
        // Custom comparator
        do {
            parent = t;
            cmp = cpr.compare(key, t.key);
            if (cmp < 0) t = t.left;
            else if (cmp > 0) t = t.right;
            else return t.setValue(value);  // Key exists
        } while (t != null);
    } else {
        // Natural ordering
        if (key == null) throw new NullPointerException();
        @SuppressWarnings("unchecked")
        Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0) t = t.left;
            else if (cmp > 0) t = t.right;
            else return t.setValue(value);
        } while (t != null);
    }
    
    // Insert new node
    Entry<K,V> e = new Entry<>(key, value, parent);
    if (cmp < 0) parent.left = e;
    else parent.right = e;
    
    // Fix red-black violations
    fixAfterInsertion(e);
    
    size++;
    modCount++;
    return null;
}
```

### Red-Black Tree Fix After Insertion

```java
// Fix red-black violations after insertion
private void fixAfterInsertion(Entry<K,V> x) {
    x.color = RED;  // New node is always red
    
    // While not root and parent is red
    while (x != null && x != root && x.parent.color == RED) {
        if (parentOf(x) == leftOf(grandparentOf(x))) {
            // Parent is left child of grandparent
            Entry<K,V> y = rightOf(grandparentOf(x));  // Uncle
            
            if (colorOf(y) == RED) {
                // Case 1: Uncle is RED → Recolor
                setParentColor(x, BLACK);
                setColor(y, BLACK);
                setGrandparentColor(x, RED);
                x = grandparentOf(x);
            } else {
                if (x == rightOf(parentOf(x))) {
                    // Case 2: Triangle → Rotate left
                    x = parentOf(x);
                    rotateLeft(x);
                }
                // Case 3: Line → Rotate right + recolor
                setParentColor(x, BLACK);
                setGrandparentColor(x, RED);
                rotateRight(grandparentOf(x));
            }
        } else {
            // Mirror case (parent is right child)
            // ... same logic mirrored ...
        }
    }
    root.color = BLACK;  // Root must be black
}
```

### NavigableMap Operations

```java
TreeMap<Integer, String> map = new TreeMap<>();
map.put(10, "ten");
map.put(20, "twenty");
map.put(30, "thirty");
map.put(40, "forty");
map.put(50, "fifty");

// Search operations
map.floorKey(25);    // 20 (largest key <= 25)
map.ceilingKey(25);  // 30 (smallest key >= 25)
map.lowerKey(30);    // 20 (largest key < 30)
map.higherKey(30);   // 40 (smallest key > 30)

// Entry operations
map.firstEntry();  // 10→"ten"
map.lastEntry();   // 50→"fifty"
map.floorEntry(25);   // 20→"twenty"
map.ceilingEntry(25); // 30→"thirty"

// Poll operations (retrieve + remove)
map.pollFirstEntry();  // 10→"ten", map now starts at 20
map.pollLastEntry();   // 50→"fifty", map now ends at 40

// Range views (backed by original map)
map.subMap(20, 40);         // {20→"twenty", 30→"thirty"} (40 exclusive)
map.subMap(20, true, 40, true);  // {20, 30, 40} (inclusive)
map.headMap(30);            // {10, 20}
map.tailMap(30);            // {30, 40, 50}

// Descending view
map.descendingMap();  // {50, 40, 30, 20, 10}
map.descendingKeySet();  // [50, 40, 30, 20, 10]
```

---

## Thread-Safe Maps

### 1. Collections.synchronizedMap()

```java
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());

// Implementation (simplified)
static <K,V> Map<K,V> synchronizedMap(Map<K,V> m) {
    return new SynchronizedMap<>(m);
}

class SynchronizedMap<K,V> implements Map<K,V> {
    final Map<K,V> m;
    final Object mutex;
    
    public V put(K key, V value) {
        synchronized (mutex) {
            return m.put(key, value);
        }
    }
    
    public V get(Object key) {
        synchronized (mutex) {
            return m.get(key);
        }
    }
    
    public Set<K> keySet() {
        synchronized (mutex) {
            return new SynchronizedSet<>(m.keySet(), mutex);
        }
    }
}

// ⚠️ Manual synchronization required for iteration!
synchronized (syncMap) {
    for (Map.Entry<String, Integer> e : syncMap.entrySet()) {
        // process
    }
}
```

**Characteristics:**
- Uses intrinsic locks (`synchronized`)
- All operations mutually exclusive
- Poor concurrency (one thread at a time)
- Iteration requires manual synchronization

### 2. ConcurrentHashMap (Java 8+)

```java
Map<String, Integer> concurrentMap = new ConcurrentHashMap<>();

// Internal Structure (Java 8+):
┌─────────────────────────────────────────┐
│  ConcurrentHashMap                       │
│  ┌───┬───┬───┬───┬───┬───┬───┬───┐     │
│  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │     │  ← Striped locks
│  ├───┼───┼───┼───┼───┼───┼───┼───┤     │
│  │ L │ L │ L │ L │ L │ L │ L │ L │     │  ← Lock per bucket
│  ├───┼───┼───┼───┼───┼───┼───┼───┤     │
│  │   │ A │   │ B │   │   │ C │   │     │  ← Nodes
│  └───┴─┬─┴───┴─┬─┴───┴───┴─┬─┴───┘     │
│        │       │           │             │
│        ↓       ↓           ↓             │
│    Linked   Tree       Linked           │
│    List     (8+)       List             │
└─────────────────────────────────────────┘

// Java 8+ uses:
// - CAS (Compare-And-Swap) for lock-free operations
// - Synchronized on bucket head for collisions
// - No global lock!
```

### ConcurrentHashMap Key Operations

```java
// put() implementation (simplified)
public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    
    int hash = spread(key.hashCode());
    int binCount = 0;
    
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();  // CAS initialization
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // Empty bucket: CAS insert
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);  // Help with resize
        else {
            V oldVal = null;
            synchronized (f) {  // Lock only this bucket
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        // Linked list
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            if (e.hash == hash && e.key.equals(key)) {
                                if (onlyIfAbsent) return e.value;
                                oldVal = e.value;
                                e.value = value;
                                break;
                            }
                            if (e.next == null) {
                                e.next = new Node<K,V>(hash, key, value, null);
                                break;
                            }
                        }
                    } else if (f instanceof TreeBin) {
                        // Tree node
                        Node<K,V> p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value);
                        if (p != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent) p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null) return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

### ConcurrentHashMap Features

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// Atomic operations
map.putIfAbsent("key", 1);           // Only put if key doesn't exist
map.remove("key", 1);                // Only remove if value matches
map.replace("key", 1, 2);            // Only replace if old value matches
map.compute("key", (k, v) -> v + 1); // Atomic compute
map.computeIfAbsent("key", k -> 0);  // Atomic compute if absent
map.computeIfPresent("key", (k, v) -> v + 1);  // Atomic compute if present
map.merge("key", 1, Integer::sum);   // Atomic merge

// Bulk operations (Java 8+)
map.forEach(1, (k, v) -> System.out.println(k + "=" + v));  // Parallel threshold
map.search(1, (k, v) -> v > 10 ? k : null);  // Find matching
map.reduce(1, (k, v) -> v, Integer::sum);    // Aggregate

// Views (weakly consistent, no ConcurrentModificationException)
Set<String> keySet = map.keySet();  // Backed by map
Collection<Integer> values = map.values();
Set<Map.Entry<String, Integer>> entrySet = map.entrySet();
```

### 3. ConcurrentSkipListMap

```java
Map<String, Integer> skipListMap = new ConcurrentSkipListMap<>();

// Internal structure: Skip List
┌─────────────────────────────────────────┐
│  Level 3:  10 ─────→ 30 ─────→ 50      │
│  Level 2:  10 ─→ 20 ─→ 30 ─→ 40 ─→ 50  │
│  Level 1:  10 ─→ 15 ─→ 20 ─→ 25 ─→...  │
│  Level 0:  10 → 15 → 17 → 20 → 25 →... │
└─────────────────────────────────────────┘

// Search: Start at highest level, drop down
// Insert/Delete: Use CAS for lock-free operations
```

**Characteristics:**
- Thread-safe, non-blocking
- **Operations**: O(log n) average
- **Sorted**: Maintains natural/comparator order
- **Iterator**: Weakly consistent
- **Best for**: Concurrent access + sorted order

### Comparison Table

| Feature | synchronizedMap | ConcurrentHashMap | ConcurrentSkipListMap |
|---------|----------------|-------------------|----------------------|
| **Locking** | Intrinsic (global) | CAS + bucket locks | CAS (lock-free) |
| **Read Performance** | O(1) blocked | O(1) mostly lock-free | O(log n) |
| **Write Performance** | O(1) blocked | O(1) mostly lock-free | O(log n) |
| **Ordering** | None | None | Sorted |
| **Null Keys/Values** | Yes | No | No |
| **Iterator** | Fail-fast (with sync) | Weakly consistent | Weakly consistent |
| **Best For** | Simple sync | General concurrent | Sorted + concurrent |

---

## Java Version Changes (8 → 21)

### Java 8 (2014) - Major Changes

**1. ConcurrentHashMap Redesign**
```java
// Before Java 8: Segment-based locking (16 segments)
// Java 8+: Node-level locking + CAS

// Old (Java 7):
ConcurrentHashMap<K,V> {
    Segment<K,V>[] segments;  // 16 segments, each with ReentrantLock
}

// New (Java 8+):
ConcurrentHashMap<K,V> {
    Node<K,V>[] table;  // Striped locks via synchronized on bucket head
    // + CAS operations for lock-free insert
}

// Performance improvement:
// - Java 7: 16 concurrent threads max (one per segment)
// - Java 8+: Up to table.length concurrent threads
```

**2. Treeify in HashMap/ConcurrentHashMap**
```java
// Java 8+: Tree when bucket has 8+ entries
static final int TREEIFY_THRESHOLD = 8;
static final int UNTREEIFY_THRESHOLD = 6;

// Improves worst-case from O(n) to O(log n)
// Protects against hash collision attacks
```

**3. Bulk Operations in ConcurrentHashMap**
```java
// Java 8+ additions
concurrentMap.forEach(parallelismThreshold, (k, v) -> process(k, v));
concurrentMap.search(parallelismThreshold, (k, v) -> predicate);
concurrentMap.reduce(parallelismThreshold, (k, v) -> mapper, reducer);
concurrentMap.mappingCount();  // Returns long (can exceed int range)
```

**4. Default Methods in Map Interface**
```java
// Java 8+ default methods
map.putIfAbsent(key, value);
map.remove(key, value);  // Conditional remove
map.replace(key, oldValue, newValue);
map.compute(key, (k, v) -> ...);
map.computeIfAbsent(key, k -> ...);
map.computeIfPresent(key, (k, v) -> ...);
map.merge(key, value, (oldVal, newVal) -> ...);
map.forEach((k, v) -> ...);
map.replaceAll((k, v) -> ...);
```

### Java 9 (2017)

**1. Map.of() Factory Methods**
```java
// Immutable maps (Java 9+)
Map<String, Integer> empty = Map.of();
Map<String, Integer> single = Map.of("one", 1);
Map<String, Integer> multi = Map.of("one", 1, "two", 2, "three", 3);

// Characteristics:
// - Immutable (UnsupportedOperationException on modify)
// - Compact memory footprint
// - Rejects null keys and values
// - Rejects duplicate keys
```

**2. Map Entries with ofEntries()**
```java
Map<String, Integer> map = Map.ofEntries(
    Map.entry("one", 1),
    Map.entry("two", 2),
    Map.entry("three", 3)
);
```

### Java 10 (2018)

**1. Local Variable Type Inference**
```java
// var keyword (Java 10+)
var map = new HashMap<String, Integer>();
var concurrentMap = new ConcurrentHashMap<>();
```

### Java 11 (2018)

**1. Collection.toArray(IntFunction)**
```java
// Java 11+ (no need for dummy array)
Integer[] arr = map.values().toArray(Integer[]::new);
```

**2. Map.nullKey() and Map.nullValue()**
```java
// Not added (proposal rejected)
// Use getOrDefault() or containsKey() instead
```

### Java 12-16 (2019-2021)

**1. Collectors.toUnmodifiableMap() (Java 10+)**
```java
Map<String, Integer> immutable = stream.collect(Collectors.toUnmodifiableMap());
```

**2. ConcurrentHashMap Improvements**
- Better performance under high contention
- Improved treeify logic

### Java 17 (2021) - LTS

**1. Sealed Classes Impact**
```java
// Better pattern matching potential
public sealed interface MapOperation permits Put, Get, Remove {}
```

**2. Performance Improvements**
- Better inlining for HashMap operations
- G1 GC improvements affect iteration

### Java 21 (2023) - LTS

**1. Record Patterns (Standard)**
```java
// Pattern matching for records in Map operations
Map<String, Point> map = new HashMap<>();
for (Map.Entry<String, Point(int x, int y)> e : map.entrySet()) {
    System.out.println(e.getKey() + ": " + x + ", " + y);
}
```

**2. Virtual Threads Impact**
```java
// Better concurrency with virtual threads
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    Map<String, Integer> map = new ConcurrentHashMap<>();
    // Thousands of virtual threads can safely access
}
```

**3. SequencedMap Interface (Java 21)**
```java
// New interface for ordered maps
interface SequencedMap<K,V> extends Map<K,V> {
    K firstKey();
    K lastKey();
    SequencedMap<K,V> reversed();
    // ...
}

// LinkedHashMap implements SequencedMap
LinkedHashMap<String, Integer> map = new LinkedHashMap<>();
String first = map.firstKey();    // Java 21+
String last = map.lastKey();      // Java 21+
Map<String, Integer> reversed = map.reversed();  // Java 21+
```

### Version Comparison Table

| Feature | Java 8 | Java 11 | Java 17 | Java 21 |
|---------|--------|---------|---------|---------|
| **Treeify** | ✅ | ✅ | ✅ | ✅ |
| **bulk operations** | ✅ | ✅ | ✅ | ✅ |
| **Map.of()** | ❌ | ✅ | ✅ | ✅ |
| **toArray(IntFunction)** | ❌ | ✅ | ✅ | ✅ |
| **SequencedMap** | ❌ | ❌ | ❌ | ✅ |
| **Record Patterns** | ❌ | ❌ | ❌ | ✅ |
| **Virtual Threads** | ❌ | ❌ | ❌ | ✅ |

---

## Performance Comparison

### Time Complexity

| Operation | HashMap | LinkedHashMap | TreeMap | ConcurrentHashMap |
|-----------|---------|---------------|---------|-------------------|
| **get** | O(1)* | O(1)* | O(log n) | O(1)* |
| **put** | O(1)* | O(1)* | O(log n) | O(1)* |
| **remove** | O(1)* | O(1)* | O(log n) | O(1)* |
| **containsKey** | O(1)* | O(1)* | O(log n) | O(1)* |
| **iteration** | O(n) | O(n) | O(n) | O(n) |
| **first/last** | ❌ | O(1) | O(log n) | ❌ |

*Amortized, worst case O(log n) with treeify

### Space Complexity

| Map Type | Space Overhead |
|----------|---------------|
| HashMap | O(n) + bucket array |
| LinkedHashMap | O(n) + bucket array + doubly-linked list |
| TreeMap | O(n) + Red-Black tree pointers |
| ConcurrentHashMap | O(n) + striped locks + tree nodes |
| ConcurrentSkipListMap | O(n × levels) ≈ O(n × log n) |

### Memory Footprint (Java 21)

```
HashMap with 1000 Integer→String entries:
- Bucket array: ~4 KB (16-64 buckets)
- Nodes: ~48 KB (1000 × 48 bytes per Node)
- Total: ~52 KB

LinkedHashMap with 1000 entries:
- HashMap overhead: ~52 KB
- Doubly-linked list: ~16 KB (before/after pointers)
- Total: ~68 KB

TreeMap with 1000 entries:
- TreeNodes: ~64 KB (1000 × 64 bytes, more pointers)
- Total: ~64 KB

ConcurrentHashMap with 1000 entries:
- Bucket array: ~32 KB (larger for concurrency)
- Nodes: ~48 KB
- Total: ~80 KB
```

---

## Common Interview Questions

### Q1: How does HashMap handle collisions?

```java
// Answer: Java 8+ uses hybrid approach
// 1. Linked list for 0-7 entries (O(n) lookup)
// 2. Balanced tree for 8+ entries (O(log n) lookup)
// 3. Converts back to list when ≤ 6 entries

// This prevents DoS attacks via hash collisions
// and maintains O(1) average case
```

### Q2: Why is load factor 0.75 by default?

```java
// Answer: Trade-off between space and time
// - Higher (e.g., 1.0): Better space, more collisions
// - Lower (e.g., 0.5): Fewer collisions, more space waste
// - 0.75: Good balance (statistically optimal)

// From HashMap source:
// "As a general rule, the default load factor (.75) 
// offers a good tradeoff between time and space costs."
```

### Q3: HashMap vs Hashtable?

```java
// HashMap:
// - Non-synchronized (not thread-safe)
// - Allows one null key
// - Faster (no locking overhead)
// - Iterator is fail-fast

// Hashtable:
// - Synchronized (thread-safe)
// - Does not allow null keys/values
// - Slower (global lock)
// - Enumeration is not fail-fast
// - Legacy class (use ConcurrentHashMap instead)

// Modern alternative:
Map<K,V> syncMap = new ConcurrentHashMap<>();
```

### Q4: How does ConcurrentHashMap achieve thread-safety?

```java
// Java 8+ approach:
// 1. CAS (Compare-And-Swap) for lock-free insert
// 2. Synchronized only on bucket head (not global)
// 3. No locking for reads (volatile reads)
// 4. Tree bins use tree locks

// This allows:
// - Multiple threads to write different buckets concurrently
// - Reads to proceed without blocking
// - Much higher throughput than Hashtable
```

### Q5: Why doesn't ConcurrentHashMap allow null keys/values?

```java
// Answer: Ambiguity in concurrent operations

// In HashMap:
V value = map.get(key);
if (value == null) {
    // Could mean: key not found OR value is null
}

// In ConcurrentHashMap:
// This ambiguity would break atomic operations
V value = concurrentMap.get(key);
if (value == null) {
    // Must mean: key not found (for atomic operations to work)
}

// Also: null values would break compute/merge operations
```

### Q6: Fail-fast vs Weakly Consistent Iterators

```java
// Fail-fast (HashMap, TreeMap)
Map<String, Integer> map = new HashMap<>();
map.put("a", 1);
Iterator<Map.Entry<String, Integer>> it = map.entrySet().iterator();
while (it.hasNext()) {
    it.next();
    map.put("b", 2);  // Throws ConcurrentModificationException!
}

// Weakly consistent (ConcurrentHashMap)
Map<String, Integer> map = new ConcurrentHashMap<>();
map.put("a", 1);
Iterator<Map.Entry<String, Integer>> it = map.entrySet().iterator();
while (it.hasNext()) {
    it.next();
    map.put("b", 2);  // OK! May or may not see "b"
}
```

### Q7: How to choose between HashMap, LinkedHashMap, and TreeMap?

```java
// Use HashMap when:
// - No ordering needed
// - Need O(1) operations
// - Most common use case

// Use LinkedHashMap when:
// - Need insertion order (iteration)
// - Need access order (LRU cache)
// - Still want O(1) operations

// Use TreeMap when:
// - Need sorted keys
// - Need range queries (subMap, headMap, tailMap)
// - Need floor/ceiling/lower/higher operations
// - Can accept O(log n) operations
```

### Q8: How to create an LRU cache?

```java
// Using LinkedHashMap
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;
    
    public LRUCache(int capacity) {
        super(capacity, 0.75f, true);  // Access order!
        this.capacity = capacity;
    }
    
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}

// Usage
LRUCache<String, String> cache = new LRUCache<>(100);
cache.put("key1", "value1");
cache.get("key1");  // Moves to end (most recently used)
// When size > 100, eldest (least recently used) auto-removed
```

---

## Best Practices

### 1. Choose Initial Capacity Wisely

```java
// Bad: Frequent resizing
Map<String, Integer> map = new HashMap<>();
for (int i = 0; i < 10000; i++) {
    map.put("key" + i, i);  // Multiple resizes
}

// Good: Pre-allocate
int expectedSize = 10000;
float loadFactor = 0.75f;
Map<String, Integer> map = new HashMap<>((int) (expectedSize / loadFactor) + 1);
```

### 2. Implement hashCode() and equals() Correctly

```java
// Good implementation
class Key {
    String name;
    int id;
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Key)) return false;
        Key k = (Key) o;
        return id == k.id && Objects.equals(name, k.name);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(name, id);
    }
}
```

### 3. Use Immutable Maps When Possible

```java
// Java 9+
Map<String, Integer> statusCodes = Map.of(
    "OK", 200,
    "NOT_FOUND", 404,
    "ERROR", 500
);

// Benefits:
// - Thread-safe by design
// - Lower memory footprint
// - Clear intent
```

### 4. Choose Right Map for Concurrency

```java
// General concurrent use
Map<String, Integer> map = new ConcurrentHashMap<>();

// Sorted + concurrent access
Map<String, Integer> sortedMap = new ConcurrentSkipListMap<>();

// Read-heavy with iteration
Map<String, Integer> map = new ConcurrentHashMap<>();
// Iterate without locking, may see stale data
```

### 5. Use compute/merge for Atomic Operations

```java
// Bad: Race condition
if (!map.containsKey(key)) {
    map.put(key, 1);  // Another thread might put between check and put
}

// Good: Atomic
map.computeIfAbsent(key, k -> 1);

// Bad: Race condition in increment
Integer count = map.get(key);
map.put(key, count == null ? 1 : count + 1);

// Good: Atomic
map.merge(key, 1, Integer::sum);
```

---

## Quick Reference

```java
// HashMap - Most common, no ordering
Map<K, V> map = new HashMap<>();

// LinkedHashMap - Insertion/access order
Map<K, V> map = new LinkedHashMap<>();
Map<K, V> map = new LinkedHashMap<>(16, 0.75f, true);  // Access order (LRU)

// TreeMap - Sorted order
Map<K, V> map = new TreeMap<>();
Map<K, V> map = new TreeMap<>(Comparator.reverseOrder());

// Thread-safe options
Map<K, V> syncMap = Collections.synchronizedMap(new HashMap<>());
Map<K, V> concurrentMap = new ConcurrentHashMap<>();
Map<K, V> skipListMap = new ConcurrentSkipListMap<>();

// Immutable (Java 9+)
Map<K, V> immutable = Map.of(k1, v1, k2, v2, k3, v3);
Map<K, V> immutable = Map.ofEntries(Map.entry(k1, v1), Map.entry(k2, v2));
```
