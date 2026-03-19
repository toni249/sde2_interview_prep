# Java Set Internals: Complete Guide

## Table of Contents
1. [Set Interface Overview](#set-interface-overview)
2. [HashSet Internals](#hashset-internals)
3. [LinkedHashSet Internals](#linkedhashset-internals)
4. [TreeSet Internals](#treeset-internals)
5. [Thread-Safe Sets](#thread-safe-sets)
6. [Java Version Changes (8 → 21)](#java-version-changes-8--21)
7. [Performance Comparison](#performance-comparison)
8. [Common Interview Questions](#common-interview-questions)

---

## Set Interface Overview

```java
public interface Set<E> extends Collection<E> {
    // No duplicate elements allowed
    // Optional: maintains insertion order (LinkedHashSet) or sorted order (TreeSet)
}
```

**Key Characteristics:**
- No duplicate elements (uses `equals()` for comparison)
- At most one `null` element (HashSet/LinkedHashSet allow, TreeSet does not)
- No guaranteed order (except LinkedHashSet & TreeSet)

---

## HashSet Internals

### Architecture

```
HashSet<E>
    ↓ (backed by)
HashMap<K, V>
    where K = element, V = PRESENT (static final Object)
```

### Internal Structure (Java 8+)

```
HashSet ["apple", "banana", "cherry"]

Internal HashMap:
┌─────────────────────────────────────────┐
│  HashMap<Node<K,V>>                     │
│  ┌───┬───┬───┬───┬───┬───┬───┬───┐     │
│  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │     │  ← Buckets (array)
│  ├───┼───┼───┼───┼───┼───┼───┼───┤     │
│  │   │   │ A │   │   │ B │   │   │     │
│  │   │   │ │ │   │   │ │ │   │   │     │
│  │   │   │ │ └─→ [cherry]  │   │   │     │
│  │   │   │ └───→ [apple]   │   │   │     │
│  │   │   │               │   │   │     │
│  │   │   │               └───→ [banana]│
│  └───┴───┴───┴───┴───┴───┴───┴───┘     │
└─────────────────────────────────────────┘

Node Structure:
┌────────────────────────────────────────┐
│  Node<K,V>                             │
│  ├── hash: int                         │
│  ├── key: E (the element)              │
│  ├── value: Object (PRESENT)           │
│  └── next: Node<K,V> (collision chain) │
└────────────────────────────────────────┘
```

### Core Implementation

```java
// HashSet source code (simplified)
public class HashSet<E> implements Set<E>, Cloneable, Serializable {
    private transient HashMap<E, Object> map;
    private static final Object PRESENT = new Object();
    
    public HashSet() {
        this.map = new HashMap<>();
    }
    
    public boolean add(E e) {
        return map.put(e, PRESENT) == null;  // Returns true if key was new
    }
    
    public boolean contains(Object o) {
        return map.containsKey(o);
    }
    
    public boolean remove(Object o) {
        return map.remove(o) == PRESENT;
    }
}
```

### How add() Works

```java
// Step-by-step: hashSet.add("apple")

1. Calculate hash: hash = hashCode("apple")
2. Apply hash perturbation: hash = hash ^ (hash >>> 16)
3. Find bucket index: index = (n - 1) & hash  (n = table.length)
4. Check collision:
   - If bucket empty → create new Node
   - If bucket has entries → traverse linked list/tree
     - If key exists → return false (duplicate)
     - If key not found → add to end
5. Check load factor: if size > threshold * loadFactor → resize()
```

### How contains() Works

```java
// Step-by-step: hashSet.contains("apple")

1. Calculate hash: hash = hashCode("apple")
2. Apply perturbation: hash = hash ^ (hash >>> 16)
3. Find bucket: index = (n - 1) & hash
4. Search bucket:
   - If bucket empty → return false
   - Traverse linked list/tree:
     - Compare hash AND equals()
     - If match found → return true
   - End of chain → return false
```

### Collision Resolution

**Java 8+ Improvement:**
- **0-7 entries**: Linked List (O(n) lookup)
- **8+ entries**: Balanced Tree (TreeMap - O(log n) lookup)
- **6 or fewer**: Convert back to Linked List

```java
// Treeify threshold
static final int TREEIFY_THRESHOLD = 8;
static final int UNTREEIFY_THRESHOLD = 6;
static final int MIN_TREEIFY_CAPACITY = 64;

// When collision occurs at index i:
Bucket[i] → Node1 → Node2 → ... → Node8 (convert to TreeNode)
                ↓
Bucket[i] → TreeNode
              ├── left: TreeNode
              ├── right: TreeNode
              └── parent: TreeNode
```

### Load Factor & Resizing

```java
// Default settings
DEFAULT_INITIAL_CAPACITY = 16;  // Must be power of 2
DEFAULT_LOAD_FACTOR = 0.75f;
threshold = capacity * loadFactor;  // 16 * 0.75 = 12

// Resizing (rehashing):
When size > threshold:
1. newCapacity = oldCapacity * 2
2. Create new table
3. Rehash ALL elements to new positions
4. Update threshold

// Time complexity: O(n) for resize, but amortized O(1) for add
```

### hashCode() and equals() Contract

```java
// Required for correct HashSet behavior
@Override
public int hashCode() {
    // Must return same value for equal objects
}

@Override
public boolean equals(Object obj) {
    // Must be consistent with hashCode
}

// Example: Custom class in HashSet
class Person {
    String name;
    int age;
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Person)) return false;
        Person p = (Person) o;
        return age == p.age && name.equals(p.name);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }
}
```

---

## LinkedHashSet Internals

### Architecture

```
LinkedHashSet<E>
    ↓ (extends)
HashSet<E>
    ↓ (backed by)
LinkedHashMap<K, V>
    ↓ (maintains)
Doubly Linked List (insertion order)
```

### Internal Structure

```
LinkedHashSet ["apple", "banana", "cherry"]

HashMap Buckets + Doubly Linked List:
┌──────────────────────────────────────────────┐
│  HashMap Buckets                             │
│  ┌───┬───┬───┬───┐                          │
│  │ 0 │ 1 │ 2 │ 3 │                          │
│  ├───┼───┼───┼───┤                          │
│  │   │ A │   │ B │  ← Entries               │
│  └───┴─┬─┴───┴─┬─┘                          │
│        │       │                             │
│        ↓       ↓                             │
│  ┌─────────────────────────────────┐        │
│  │  Doubly Linked List             │        │
│  │  ┌─────┐   ┌─────┐   ┌─────┐   │        │
│  │  │apple│ ↔ │banana│ ↔ │cherry│  │        │
│  │  │pred │   │pred │   │pred │   │        │
│  │  │next │   │next │   │next │   │        │
│  │  └─────┘   └─────┘   └─────┘   │        │
│  │     ↑                   ↑       │        │
│  │  header              tail       │        │
│  └─────────────────────────────────┘        │
└──────────────────────────────────────────────┘

Entry Structure:
class LinkedHashMap.Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;  // Doubly linked list pointers
}
```

### Core Implementation

```java
// LinkedHashSet source code (simplified)
public class LinkedHashSet<E> extends HashSet<E> {
    public LinkedHashSet() {
        super(new LinkedHashMap<>());  // Uses LinkedHashMap
    }
    
    // All operations delegated to HashSet/HashMap
}

// LinkedHashMap maintains insertion order
class LinkedHashMap<K,V> extends HashMap<K,V> {
    transient LinkedHashMap.Entry<K,V> head, tail;
    
    void afterNodeInsertion(boolean evict) {
        // Add new entry to tail of linked list
    }
    
    void afterNodeRemoval(Node<K,V> e) {
        // Remove from linked list
    }
}
```

### Iteration Order Guarantee

```java
LinkedHashSet<String> set = new LinkedHashSet<>();
set.add("apple");
set.add("banana");
set.add("cherry");

// Iteration ALWAYS returns: apple → banana → cherry
for (String s : set) {
    System.out.println(s);
}
```

---

## TreeSet Internals

### Architecture

```
TreeSet<E>
    ↓ (backed by)
TreeMap<K, V>
    where K = element, V = PRESENT
    ↓ (uses)
Red-Black Tree (self-balancing BST)
```

### Internal Structure

```
TreeSet [3, 1, 5, 2, 4]

Red-Black Tree:
        3(B)
       /   \
     1(R)   5(R)
       \    /
       2(B) 4(B)

B = Black, R = Red

Properties:
1. Every node is either red or black
2. Root is black
3. All leaves (NULL) are black
4. Red nodes have black children (no red-red)
5. All paths from node to leaves have same black count
```

### Core Implementation

```java
// TreeSet source code (simplified)
public class TreeSet<E> implements NavigableSet<E> {
    private transient TreeMap<E, Object> m;
    private static final Object PRESENT = new Object();
    
    public TreeSet() {
        this(new TreeMap<>());
    }
    
    public TreeSet(Comparator<? super E> comparator) {
        this(new TreeMap<>(comparator));
    }
    
    public boolean add(E e) {
        return m.put(e, PRESENT) == null;
    }
    
    public boolean contains(Object o) {
        return m.containsKey(o);
    }
}
```

### Red-Black Tree Operations

**Insertion (O(log n)):**
```
1. Insert as in BST (find correct position)
2. Color new node RED
3. Fix violations:
   - Case 1: Uncle is RED → recolor
   - Case 2: Uncle is BLACK, triangle → rotate + recolor
   - Case 3: Uncle is BLACK, line → rotate + recolor
4. Ensure root is BLACK
```

**Deletion (O(log n)):**
```
1. Delete as in BST
2. If deleted node was BLACK, fix violations:
   - Case 1: Sibling is RED → rotate + recolor
   - Case 2: Sibling is BLACK, both children BLACK → recolor
   - Case 3: Sibling is BLACK, outer child RED → rotate + recolor
   - Case 4: Sibling is BLACK, inner child RED → rotate + recolor
```

### Comparator Requirements

```java
// TreeSet with natural ordering
TreeSet<Integer> set = new TreeSet<>();  // Uses Comparable

// TreeSet with custom comparator
TreeSet<String> set = new TreeSet<>((a, b) -> a.length() - b.length());

// Important: Comparator must be consistent with equals
// If compare(a,b) == 0, then a.equals(b) should be true
```

### NavigableSet Operations

```java
TreeSet<Integer> set = new TreeSet<>(Arrays.asList(10, 20, 30, 40, 50));

// Search operations
set.floor(25);    // 20 (largest <= 25)
set.ceiling(25);  // 30 (smallest >= 25)
set.lower(30);    // 20 (largest < 30)
set.higher(30);   // 40 (smallest > 30)

// Range views (backed by original set - changes reflect!)
set.subSet(20, 40);      // [20, 30] (40 exclusive)
set.subSet(20, true, 40, true);  // [20, 30, 40]
set.headSet(30);         // [10, 20]
set.tailSet(30);         // [30, 40, 50]

// Reverse order
set.descendingSet();     // [50, 40, 30, 20, 10]

// Poll operations (retrieve + remove)
set.pollFirst();  // 10, set becomes [20, 30, 40, 50]
set.pollLast();   // 50, set becomes [20, 30, 40]
```

---

## Thread-Safe Sets

### 1. Collections.synchronizedSet()

```java
Set<String> syncSet = Collections.synchronizedSet(new HashSet<>());

// Implementation (simplified)
static <E> Set<E> synchronizedSet(Set<E> s) {
    return new SynchronizedSet<>(s);
}

class SynchronizedSet<E> implements Set<E> {
    final Set<E> s;
    final Object mutex;
    
    public boolean add(E e) {
        synchronized (mutex) {
            return s.add(e);
        }
    }
    
    public Iterator<E> iterator() {
        synchronized (mutex) {
            return new SynchronizedIterator<>(s.iterator(), mutex);
        }
    }
}

// ⚠️ Manual synchronization required for iteration!
synchronized (syncSet) {
    for (String s : syncSet) {
        // process
    }
}
```

**Characteristics:**
- Uses intrinsic locks (`synchronized`)
- All operations are mutually exclusive
- Poor concurrency (only one thread at a time)
- Iteration requires manual synchronization

### 2. CopyOnWriteArraySet

```java
Set<String> cowSet = new CopyOnWriteArraySet<>();

// Implementation
public class CopyOnWriteArraySet<E> implements Set<E> {
    private final CopyOnWriteArrayList<E> al;
    
    public boolean add(E e) {
        return al.addIfAbsent(e);
    }
}

// CopyOnWriteArrayList internal
public void add(E e) {
    ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);  // Atomic reference update
    } finally {
        lock.unlock();
    }
}
```

**Characteristics:**
- Thread-safe without external synchronization
- **Read operations**: Lock-free, very fast
- **Write operations**: Copy entire array (slow O(n))
- **Iterator**: Snapshot at creation time (no ConcurrentModificationException)
- **Best for**: Read-heavy, rare writes

**Use Case:**
```java
// Good: Listener pattern with many reads, few writes
Set<Listener> listeners = new CopyOnWriteArraySet<>();

void notifyListeners() {
    for (Listener l : listeners) {  // No synchronization needed
        l.onEvent();
    }
}
```

### 3. ConcurrentSkipListSet

```java
Set<String> concurrentSet = new ConcurrentSkipListSet<>();

// Internal structure: Skip List
┌─────────────────────────────────────────┐
│  Level 3:  10 ─────→ 30 ─────→ 50      │
│  Level 2:  10 ─→ 20 ─→ 30 ─→ 40 ─→ 50  │
│  Level 1:  10 → 15 → 20 → 25 → 30 →... │
│  Level 0:  10  15  17  20  25  27  30  │
└─────────────────────────────────────────┘

// Search: Start at highest level, drop down when needed
```

**Characteristics:**
- Thread-safe, non-blocking
- **Operations**: O(log n) average
- **Sorted**: Maintains natural/comparator order
- **Iterator**: Weakly consistent (may not reflect recent changes)
- **Best for**: Concurrent access + sorted order needed

### Comparison Table

| Feature | synchronizedSet | CopyOnWriteArraySet | ConcurrentSkipListSet |
|---------|----------------|---------------------|----------------------|
| **Locking** | Intrinsic (synchronized) | ReentrantLock (write only) | CAS + locks |
| **Read Performance** | O(1) blocked | O(1) lock-free | O(log n) |
| **Write Performance** | O(1) blocked | O(n) copy | O(log n) |
| **Ordering** | None | Insertion | Sorted |
| **Iterator** | Fail-fast (with sync) | Snapshot | Weakly consistent |
| **Null Elements** | Yes | No | No |
| **Best For** | Simple sync | Read-heavy | Sorted + concurrent |

---

## Java Version Changes (8 → 21)

### Java 8 (2014) - Major Changes

**1. HashMap Treeify (affects HashSet)**
```java
// Before Java 8: Linked list for all collisions (O(n) worst case)
// Java 8+: Treeify when collisions >= 8 (O(log n) worst case)

static final int TREEIFY_THRESHOLD = 8;
static final int UNTREEIFY_THRESHOLD = 6;
static final int MIN_TREEIFY_CAPACITY = 64;
```

**2. Spliterator Support**
```java
// All Sets now support parallel streams
Set<String> set = new HashSet<>();
set.parallelStream().forEach(s -> process(s));

// Spliterator characteristics
set.spliterator().characteristics();
// HashSet: DISTINCT, SIZED, SUBSIZED
// LinkedHashSet: DISTINCT, SIZED, SUBSIZED, ORDERED
// TreeSet: DISTINCT, SIZED, SUBSIZED, ORDERED, SORTED, COMPARED
```

**3. Default Methods in Collection Interface**
```java
// New methods available on all Sets
set.removeIf(s -> s.startsWith("A"));  // Java 8+
set.forEach(s -> process(s));          // Java 8+
```

### Java 9 (2017)

**1. Set.of() Factory Methods**
```java
// Immutable sets (Java 9+)
Set<String> empty = Set.of();
Set<String> single = Set.of("apple");
Set<String> multi = Set.of("apple", "banana", "cherry");

// Characteristics:
// - Immutable (UnsupportedOperationException on modify)
// - Compact memory footprint
// - Rejects null elements
// - Rejects duplicates at creation
```

**2. JShell for Quick Testing**
```bash
jshell> Set<String> set = new HashSet<>();
jshell> set.add("test");
jshell> set.stream().collect(Collectors.toList());
```

### Java 10 (2018)

**1. Local Variable Type Inference**
```java
// var keyword (Java 10+)
var set = new HashSet<String>();  // Infers Set<String>
var treeSet = new TreeSet<>(Comparator.reverseOrder());
```

### Java 11 (2018)

**1. Collection.toArray(IntFunction)**
```java
// Before (Java 8-10)
String[] arr = set.toArray(new String[0]);

// Java 11+ (no need for dummy array)
String[] arr = set.toArray(String[]::new);
```

**2. String Methods Affecting Set Operations**
```java
// isBlank() useful for filtering
set.removeIf(String::isBlank);
```

### Java 12-16 (2019-2021)

**1. Collectors.toUnmodifiableSet() (Java 10+)**
```java
Set<String> immutable = stream.collect(Collectors.toUnmodifiableSet());
```

**2. List/Set.of() Performance Improvements**
- Smaller memory footprint
- Faster creation

### Java 17 (2021) - LTS

**1. Sealed Classes Impact**
```java
// Better pattern matching potential
public sealed interface SetOperation permits Add, Remove, Contains {}
```

**2. Performance Improvements**
- G1 GC improvements affect Set iteration
- Better inlining for HashSet operations

### Java 18-20 (2022-2023)

**1. Small Improvements**
- Better error messages
- Performance optimizations in HashMap (affects HashSet)

### Java 21 (2023) - LTS

**1. Record Patterns (Preview → Standard)**
```java
// Pattern matching for records in Set operations
Set<Point> points = new HashSet<>();
for (Point(int x, int y) : points) {  // Java 21+
    System.out.println(x + ", " + y);
}
```

**2. Sequenced Collections (Java 21)**
```java
// New interface for ordered collections
interface SequencedSet<E> extends Set<E> {
    E getFirst();
    E getLast();
    SequencedSet<E> reversed();
    void addFirst(E e);
    void addLast(E e);
}

// LinkedHashSet implements SequencedSet
LinkedHashSet<String> set = new LinkedHashSet<>();
set.addFirst("first");   // Java 21+
set.addLast("last");     // Java 21+
String first = set.getFirst();  // Java 21+
```

**3. Virtual Threads Impact**
```java
// Better concurrency with virtual threads
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    Set<String> set = ConcurrentHashMap.newKeySet();
    // Thousands of virtual threads can safely access
}
```

### Version Comparison Table

| Feature | Java 8 | Java 11 | Java 17 | Java 21 |
|---------|--------|---------|---------|---------|
| **Treeify** | ✅ | ✅ | ✅ | ✅ |
| **Spliterator** | ✅ | ✅ | ✅ | ✅ |
| **Set.of()** | ❌ | ✅ | ✅ | ✅ |
| **toArray(IntFunction)** | ❌ | ✅ | ✅ | ✅ |
| **SequencedSet** | ❌ | ❌ | ❌ | ✅ |
| **Record Patterns** | ❌ | ❌ | ❌ | ✅ |
| **Virtual Threads** | ❌ | ❌ | ❌ | ✅ |

---

## Performance Comparison

### Time Complexity

| Operation | HashSet | LinkedHashSet | TreeSet | ConcurrentSkipListSet |
|-----------|---------|---------------|---------|----------------------|
| **add** | O(1)* | O(1)* | O(log n) | O(log n) |
| **contains** | O(1)* | O(1)* | O(log n) | O(log n) |
| **remove** | O(1)* | O(1)* | O(log n) | O(log n) |
| **iteration** | O(n) | O(n) | O(n) | O(n) |
| **first/last** | ❌ | ❌ | O(log n) | O(log n) |

*Amortized, worst case O(log n) with treeify

### Space Complexity

| Set Type | Space Overhead |
|----------|---------------|
| HashSet | O(n) + HashMap overhead |
| LinkedHashSet | O(n) + HashMap + doubly-linked list |
| TreeSet | O(n) + TreeMap + Red-Black tree pointers |
| CopyOnWriteArraySet | O(n) per write (temporary) |
| ConcurrentSkipListSet | O(n × levels) ≈ O(n × log n) |

### Memory Footprint (Java 21)

```
HashSet with 1000 Integer elements:
- HashMap: ~32 KB (buckets)
- Nodes: ~48 KB (1000 × 48 bytes per Node)
- Total: ~80 KB

LinkedHashSet with 1000 elements:
- HashSet overhead: ~80 KB
- Doubly-linked list: ~16 KB (before/after pointers)
- Total: ~96 KB

TreeSet with 1000 elements:
- TreeMap: ~32 KB
- TreeNodes: ~64 KB (1000 × 64 bytes, more pointers)
- Total: ~96 KB
```

---

## Common Interview Questions

### Q1: How does HashSet prevent duplicates?

```java
// Answer: Uses HashMap.put() return value
public boolean add(E e) {
    return map.put(e, PRESENT) == null;
    // put() returns null if key was new
    // put() returns old value if key existed
}
```

### Q2: What happens if two objects have same hashCode?

```java
// Answer: Collision resolution
// Java 8+: 
// - 0-7 collisions: Linked list (O(n) lookup)
// - 8+ collisions: Balanced tree (O(log n) lookup)

// Example: Poor hashCode() causes collisions
class BadHash {
    @Override
    public int hashCode() {
        return 42;  // All objects go to same bucket!
    }
}
```

### Q3: HashSet vs TreeSet - When to use which?

```java
// Use HashSet when:
// - Order doesn't matter
// - Need O(1) operations
// - Don't need range queries

// Use TreeSet when:
// - Need sorted order
// - Need floor/ceiling/lower/higher
// - Need range queries (subSet, headSet, tailSet)
// - Can accept O(log n) operations
```

### Q4: Why doesn't TreeSet allow null?

```java
// Answer: compareTo() throws NullPointerException
TreeSet<String> set = new TreeSet<>();
set.add(null);  // Throws ClassCastException or NPE

// Reason: null.compareTo(obj) is undefined
// HashSet allows one null (uses hashCode/equals)
```

### Q5: How to make HashSet thread-safe?

```java
// Option 1: Synchronized wrapper
Set<String> syncSet = Collections.synchronizedSet(new HashSet<>());

// Option 2: ConcurrentSkipListSet (if sorted OK)
Set<String> concurrentSet = new ConcurrentSkipListSet<>();

// Option 3: CopyOnWriteArraySet (read-heavy)
Set<String> cowSet = new CopyOnWriteArraySet<>();

// Option 4: ConcurrentHashMap.newKeySet() (Java 8+)
Set<String> keySet = ConcurrentHashMap.newKeySet();
```

### Q6: What is ConcurrentHashMap.newKeySet()?

```java
// Java 8+ utility method
Set<String> set = ConcurrentHashMap.newKeySet();

// Internally uses ConcurrentHashMap with dummy values
// Better performance than synchronizedSet()
// Thread-safe, non-blocking

// Implementation:
public static <K> KeySetView<K,Boolean> newKeySet() {
    return new ConcurrentHashMap<K,Boolean>().keySet();
}
```

### Q7: Fail-fast vs Fail-safe Iterators

```java
// Fail-fast (HashSet, TreeSet)
Set<String> set = new HashSet<>();
set.add("a");
Iterator<String> it = set.iterator();
while (it.hasNext()) {
    String s = it.next();
    set.add("b");  // Throws ConcurrentModificationException!
}

// Fail-safe (CopyOnWriteArraySet, ConcurrentSkipListSet)
Set<String> set = new CopyOnWriteArraySet<>();
set.add("a");
Iterator<String> it = set.iterator();
while (it.hasNext()) {
    String s = it.next();
    set.add("b");  // OK! Iterator has snapshot
}
```

---

## Best Practices

### 1. Choose Initial Capacity Wisely

```java
// Bad: Frequent resizing
Set<String> set = new HashSet<>();
for (int i = 0; i < 10000; i++) {
    set.add("item" + i);  // Multiple resizes
}

// Good: Pre-allocate
int expectedSize = 10000;
float loadFactor = 0.75f;
Set<String> set = new HashSet<>((int) (expectedSize / loadFactor) + 1);
```

### 2. Implement hashCode() and equals() Correctly

```java
// Good implementation
class Person {
    String name;
    int age;
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Person)) return false;
        Person p = (Person) o;
        return age == p.age && Objects.equals(name, p.name);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }
}
```

### 3. Use Immutable Sets When Possible

```java
// Java 9+
Set<String> roles = Set.of("ADMIN", "USER", "GUEST");

// Benefits:
// - Thread-safe by design
// - Lower memory footprint
// - Clear intent
```

### 4. Choose Right Set for Concurrency

```java
// Read-heavy, rare writes
Set<Listener> listeners = new CopyOnWriteArraySet<>();

// Sorted + concurrent access
Set<Integer> sortedSet = new ConcurrentSkipListSet<>();

// General concurrent use
Set<String> set = ConcurrentHashMap.newKeySet();
```

---

## Quick Reference

```java
// HashSet - Most common, no ordering
Set<String> hashSet = new HashSet<>();

// LinkedHashSet - Insertion order
Set<String> linkedSet = new LinkedHashSet<>();

// TreeSet - Sorted order
Set<String> treeSet = new TreeSet<>();
Set<String> treeSet = new TreeSet<>(Comparator.reverseOrder());

// Thread-safe options
Set<String> syncSet = Collections.synchronizedSet(new HashSet<>());
Set<String> cowSet = new CopyOnWriteArraySet<>();
Set<String> skipListSet = new ConcurrentSkipListSet<>();
Set<String> keySet = ConcurrentHashMap.newKeySet();

// Immutable (Java 9+)
Set<String> immutable = Set.of("a", "b", "c");
```
