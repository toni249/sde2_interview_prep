# Java Collections Reference (Complete)

## 1. Collection Interface Hierarchy

```
java.util.Collection
├── List
│   ├── ArrayList          (dynamic array, random access O(1))
│   ├── LinkedList         (doubly-linked list, insert/delete O(1))
│   └── Vector             (synchronized, legacy - avoid)
│       └── Stack          (legacy - use ArrayDeque instead)
│
├── Set
│   ├── HashSet            (hash table, O(1) avg)
│   ├── LinkedHashSet      (hash + linked list, insertion order)
│   └── TreeSet            (red-black tree, sorted, O(log n))
│       └── ConcurrentSkipListSet (concurrent sorted set)
│
└── Queue
    ├── PriorityQueue      (heap-based, min-heap default)
    ├── ArrayDeque         (double-ended queue, stack/queue)
    └── LinkedList         (also implements Queue)

java.util.Map (NOT part of Collection interface)
├── HashMap                (hash table, O(1) avg)
├── LinkedHashMap          (hash + linked list, insertion/access order)
├── TreeMap                (red-black tree, sorted keys, O(log n))
├── Hashtable              (synchronized, legacy - avoid)
├── IdentityHashMap        (reference equality ==)
├── WeakHashMap            (weak keys, GC-friendly)
└── ConcurrentHashMap      (thread-safe, high concurrency)

Specialized Collections
├── EnumSet                (bit vector for enums, ultra-fast)
├── EnumMap                (array-based map for enum keys)
├── Collections            (utility class with static methods)
└── Arrays                 (utility class for array operations)
```

---

## 2. `Optional<T>` (Java 8+)

### Initialization
```java
Optional<String> empty = Optional.empty();
Optional<String> present = Optional.of("value");        // NPE if null
Optional<String> nullable = Optional.ofNullable(null);  // empty if null
```

### Methods
| Method | Returns | Description |
|--------|---------|-------------|
| `isPresent()` | `boolean` | `true` if value exists |
| `isEmpty()` | `boolean` | `true` if empty (Java 11+) |
| `get()` | `T` | **throws** `NoSuchElementException` if empty |
| `orElse(T other)` | `T` | value or default |
| `orElseGet(Supplier)` | `T` | value or lazy default |
| `orElseThrow()` | `T` | value or **throws** `NoSuchElementException` |
| `orElseThrow(Supplier)` | `T` | value or **throws** custom exception |
| `ifPresent(Consumer)` | `void` | execute if present |
| `ifPresentOrElse(Consumer, Runnable)` | `void` | execute if present else run (Java 11+) |
| `filter(Predicate)` | `Optional<T>` | filter value |
| `map(Function)` | `Optional<R>` | transform if present |
| `flatMap(Function)` | `Optional<R>` | transform returning Optional |
| `stream()` | `Stream<T>` | sequential stream (Java 9+) |

### Patterns
```java
// Avoid: if(opt.isPresent()) return opt.get();
// Use:
return opt.orElse("default");
return opt.orElseGet(() -> computeDefault());
opt.ifPresent(x -> process(x));

// Chaining:
Optional<String> result = Optional.ofNullable(user)
    .map(User::getAddress)
    .flatMap(Address::getCity)
    .filter(city -> !city.isEmpty())
    .map(String::toLowerCase);
```

---

## 3. `EnumSet` (Specialized Set for Enums)

### When to Use
- Enum types only
- Extremely fast (bit vector internally)
- Memory efficient
- Not thread-safe (wrap with `Collections.synchronizedSet()` if needed)

### Initialization
```java
enum Day { MON, TUE, WED, THU, FRI, SAT, SUN }

EnumSet<Day> empty = EnumSet.noneOf(Day.class);
EnumSet<Day> all = EnumSet.allOf(Day.class);
EnumSet<Day> single = EnumSet.of(Day.MON);
EnumSet<Day> range = EnumSet.range(Day.MON, Day.FRI);
EnumSet<Day> copy = EnumSet.copyOf(existingSet);
EnumSet<Day> complement = EnumSet.complementOf(weekdays);
```

### Methods (Same as `Set` + specialized)
| Method | Returns | Description |
|--------|---------|-------------|
| `allOf(Class<E>)` | `EnumSet<E>` | all enum values |
| `noneOf(Class<E>)` | `EnumSet<E>` | empty set |
| `of(E...)` | `EnumSet<E>` | specified values |
| `range(E from, E to)` | `EnumSet<E>` | inclusive range |
| `complementOf(EnumSet<E>)` | `EnumSet<E>` | complement |
| `copyOf(Collection/EnumSet)` | `EnumSet<E>` | copy |

### Example
```java
EnumSet<Day> weekdays = EnumSet.range(Day.MON, Day.FRI);
EnumSet<Day> weekend = EnumSet.complementOf(weekdays);
boolean isWeekend = weekend.contains(Day.SAT);  // true
```

---

## 4. `EnumMap` (Specialized Map for Enum Keys)

### When to Use
- Enum keys only
- Faster than `HashMap` (array internally)
- Maintains natural enum order
- Not thread-safe

### Initialization
```java
enum Priority { LOW, MEDIUM, HIGH }

EnumMap<Priority, String> map = new EnumMap<>(Priority.class);
EnumMap<Priority, String> copy = new EnumMap<>(otherMap);
```

### Methods (Same as `Map` + specialized)
| Method | Returns | Description |
|--------|---------|-------------|
| `firstKey()` | `K` | **throws** `NoSuchElementException` |
| `lastKey()` | `K` | **throws** `NoSuchElementException` |
| `lowerKey(K)` | `K` | `null` if none |
| `higherKey(K)` | `K` | `null` if none |
| `comparator()` | `Comparator` | always `null` (natural order) |

### Example
```java
EnumMap<Priority, String> tasks = new EnumMap<>(Priority.class);
tasks.put(Priority.HIGH, "Urgent");
tasks.put(Priority.LOW, "Later");

for (Priority p : tasks.keySet()) {  // always LOW → MEDIUM → HIGH
    System.out.println(p + ": " + tasks.get(p));
}
```

---

## 5. Method Reference by Interface

### `Collection<E>` — Root Interface (15 methods)
| Method | All Implementations? |
|--------|---------------------|
| `size()` | ✅ |
| `isEmpty()` | ✅ |
| `contains(Object)` | ✅ |
| `iterator()` | ✅ |
| `toArray()` / `toArray(T[])` | ✅ |
| `add(E)` | ✅ (optional) |
| `remove(Object)` | ✅ (optional) |
| `containsAll(Collection)` | ✅ |
| `addAll(Collection)` | ✅ (optional) |
| `removeAll(Collection)` | ✅ (optional) |
| `retainAll(Collection)` | ✅ (optional) |
| `clear()` | ✅ (optional) |
| `equals(Object)` | ✅ |
| `hashCode()` | ✅ |
| `stream()` / `parallelStream()` | ✅ (Java 8+) |
| `removeIf(Predicate)` | ✅ (Java 8+, optional) |
| `spliterator()` | ✅ (Java 8+) |

---

### `List<E>` — Extends `Collection` (11 additional methods)
| Method | ArrayList | LinkedList |
|--------|-----------|------------|
| `get(int)` | ✅ O(1) | ✅ O(n) |
| `set(int, E)` | ✅ O(1) | ✅ O(n) |
| `add(int, E)` | ✅ O(n) | ✅ O(1) |
| `remove(int)` | ✅ O(n) | ✅ O(1) |
| `indexOf(Object)` / `lastIndexOf(Object)` | ✅ | ✅ |
| `subList(int, int)` | ✅ (view) | ✅ (view) |
| `listIterator()` / `listIterator(int)` | ✅ | ✅ |
| `sort(Comparator)` | ✅ (Java 8+) | ✅ (Java 8+) |
| `replaceAll(UnaryOperator)` | ✅ (Java 8+) | ✅ (Java 8+) |

---

### `Set<E>` — Extends `Collection` (no additional methods)
| Implementation | Unique Behavior |
|----------------|-----------------|
| `HashSet` | O(1) avg, no order |
| `LinkedHashSet` | Insertion order |
| `TreeSet` | Sorted, O(log n), has navigation methods |

### `TreeSet` Additional Methods (`NavigableSet<E>`)
| Method | Returns |
|--------|---------|
| `first()` / `last()` | `E` |
| `lower(E)` / `higher(E)` | `E` |
| `floor(E)` / `ceiling(E)` | `E` |
| `pollFirst()` / `pollLast()` | `E` |
| `subSet(from, to)` / `headSet(to)` / `tailSet(from)` | `NavigableSet<E>` |
| `descendingSet()` | `NavigableSet<E>` |
| `descendingIterator()` | `Iterator<E>` |

---

### `Queue<E>` — Extends `Collection` (6 additional methods)
| Method | Throws Exception | Returns Special Value |
|--------|-----------------|----------------------|
| **Insert** | `add(E)` | `offer(E)` |
| **Remove** | `remove()` | `poll()` |
| **Examine** | `element()` | `peek()` |

### `Deque<E>` — Extends `Queue` (12 additional methods)
| Method | First (Head) | Last (Tail) |
|--------|--------------|-------------|
| **Insert** | `addFirst(E)` / `offerFirst(E)` | `addLast(E)` / `offerLast(E)` |
| **Remove** | `removeFirst()` / `pollFirst()` | `removeLast()` / `pollLast()` |
| **Examine** | `getFirst()` / `peekFirst()` | `getLast()` / `peekLast()` |
| **Stack** | `push(E)` | `pop()` |

---

### `Map<K,V>` — NOT a Collection (16 methods)
| Method | All Implementations? |
|--------|---------------------|
| `size()` | ✅ |
| `isEmpty()` | ✅ |
| `containsKey(Object)` | ✅ |
| `containsValue(Object)` | ✅ |
| `get(Object)` | ✅ |
| `put(K, V)` | ✅ (optional) |
| `remove(Object)` | ✅ (optional) |
| `putAll(Map)` | ✅ (optional) |
| `clear()` | ✅ (optional) |
| `keySet()` | ✅ |
| `values()` | ✅ |
| `entrySet()` | ✅ |
| `equals(Object)` / `hashCode()` | ✅ |
| `getOrDefault(Object, V)` | ✅ (Java 8+) |
| `forEach(BiConsumer)` | ✅ (Java 8+) |

### Java 8+ Default Methods (All Maps)
| Method | Description |
|--------|-------------|
| `putIfAbsent(K, V)` | Insert if missing |
| `remove(Object, Object)` | Remove key-value pair |
| `replace(K, V, V)` | Conditional replace |
| `replaceAll(BiFunction)` | Transform all values |
| `compute(K, BiFunction)` | Compute value |
| `computeIfAbsent(K, Function)` | Compute if missing |
| `computeIfPresent(K, BiFunction)` | Compute if present |
| `merge(K, V, BiFunction)` | Merge values |

### `TreeMap` Additional Methods (`NavigableMap<K,V>`)
| Method | Returns |
|--------|---------|
| `firstKey()` / `lastKey()` | `K` |
| `lowerKey(K)` / `higherKey(K)` | `K` |
| `floorKey(K)` / `ceilingKey(K)` | `K` |
| `firstEntry()` / `lastEntry()` | `Map.Entry<K,V>` |
| `pollFirstEntry()` / `pollLastEntry()` | `Map.Entry<K,V>` |
| `subMap(from, to)` / `headMap(to)` / `tailMap(from)` | `NavigableMap<K,V>` |
| `descendingMap()` | `NavigableMap<K,V>` |
| `descendingKeySet()` | `NavigableSet<K>` |

---

## 6. `Collections` Utility Class (Quick Reference)

### Synchronization (Thread Safety)
```java
Collections.synchronizedList(list)
Collections.synchronizedSet(set)
Collections.synchronizedMap(map)
// Must synchronize manually during iteration!
```

### Unmodifiable (Read-Only)
```java
Collections.unmodifiableList(list)
Collections.unmodifiableSet(set)
Collections.unmodifiableMap(map)
```

### Singleton / Empty
```java
Collections.singleton(e)
Collections.singletonList(e)
Collections.singletonMap(k, v)
Collections.emptyList()
Collections.emptySet()
Collections.emptyMap()
```

### Operations
```java
Collections.sort(list)
Collections.binarySearch(list, key)
Collections.reverse(list)
Collections.shuffle(list)
Collections.swap(list, i, j)
Collections.rotate(list, dist)
Collections.fill(list, val)
Collections.copy(dest, src)
Collections.min(list) / max(list)
Collections.frequency(list, o)
Collections.disjoint(c1, c2)
Collections.indexOfSubList(list, sub)
Collections.nCopies(n, e)
```

---

## 7. Quick Method Lookup Table

| Need | Use This |
|------|----------|
| **Sorted iteration** | `TreeSet`, `TreeMap` |
| **Insertion order** | `LinkedHashSet`, `LinkedHashMap` |
| **Access order (LRU)** | `new LinkedHashMap<>(16, 0.75f, true)` |
| **Enum keys/values** | `EnumMap`, `EnumSet` |
| **Thread-safe map** | `ConcurrentHashMap` |
| **Stack** | `ArrayDeque` (not `Stack`) |
| **Priority queue** | `PriorityQueue` |
| **Double-ended queue** | `ArrayDeque` |
| **Null keys/values** | `HashMap`, `LinkedHashMap` (1 null key) |
| **No nulls** | `TreeMap`, `TreeSet`, `HashMap` (keys) |
| **Fast iteration** | `ArrayList`, `HashSet`, `HashMap` |
| **Frequent insert/delete** | `LinkedList`, `ArrayDeque` |

---

## 8. Common Pitfalls

```java
// ❌ ArrayList remove by value vs index
list.remove(0);      // removes index 0
list.remove(Integer.valueOf(0));  // removes value 0

// ❌ ConcurrentModificationException
for (String s : list) {
    list.remove(s);  // throws!
}
// ✅ Use iterator
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (condition) it.remove();
}
// ✅ Or use removeIf (Java 8+)
list.removeIf(s -> condition);

// ❌ HashMap with mutable keys
// If key's hashCode changes after insertion, get() will fail

// ❌ TreeSet/TreeMap without Comparable or Comparator
// throws ClassCastException

// ❌ PriorityQueue iteration order ≠ sorted order
// Use poll() repeatedly for sorted access
```
