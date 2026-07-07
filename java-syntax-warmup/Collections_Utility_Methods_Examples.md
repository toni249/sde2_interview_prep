## `java.util.Collections` – Practical Examples

```java
import java.util.*;
```

---

### 1. Sorting and Reversing

```java
List<Integer> nums = new ArrayList<>(Arrays.asList(5, 1, 4, 2, 3));

// 1.1 sort (natural order)
Collections.sort(nums);                    // [1, 2, 3, 4, 5]

// 1.2 sort with custom Comparator
Collections.sort(nums, (a, b) -> b - a);   // [5, 4, 3, 2, 1]  (descending)

// 1.3 reverse (in-place)
Collections.reverse(nums);                 // Reverse current order

// 1.4 swap and rotate
Collections.swap(nums, 0, nums.size() - 1);
Collections.rotate(nums, 2);               // Right rotate by 2 positions
```

---

### 2. Shuffling (Randomizing Order)

```java
List<String> cards = new ArrayList<>(Arrays.asList("A", "K", "Q", "J", "10"));

Collections.shuffle(cards);                // Random order each time

Random random = new Random(42);            // Deterministic shuffling
Collections.shuffle(cards, random);
```

---

### 3. Min, Max, Binary Search, Frequency, Disjoint

```java
List<Integer> nums = Arrays.asList(10, 20, 30, 20, 40, 20);

// 3.1 min / max
int min = Collections.min(nums);           // 10
int max = Collections.max(nums);           // 40

// 3.2 frequency
int count20 = Collections.frequency(nums, 20); // 3

// 3.3 binarySearch (requires sorted list!)
List<Integer> sorted = new ArrayList<>(nums);
Collections.sort(sorted);                  // [10, 20, 20, 20, 30, 40]
int index = Collections.binarySearch(sorted, 20); // any valid index of 20

// 3.4 disjoint
List<Integer> a = Arrays.asList(1, 2, 3);
List<Integer> b = Arrays.asList(4, 5, 6);
List<Integer> c = Arrays.asList(3, 4, 5);

boolean abDisjoint = Collections.disjoint(a, b);  // true
boolean acDisjoint = Collections.disjoint(a, c);  // false (3 is common)
```

---

### 4. Copy, Fill, `nCopies`, `addAll`

```java
// 4.1 copy (destination must be at least src.size())
List<String> src  = Arrays.asList("A", "B", "C");
List<String> dest = new ArrayList<>(Arrays.asList("X", "Y", "Z", "W"));

Collections.copy(dest, src);   // dest becomes ["A", "B", "C", "W"]

// 4.2 fill (replace every element with same value)
Collections.fill(dest, "X");   // dest becomes ["X", "X", "X", "X"]

// 4.3 nCopies (immutable list with n copies of value)
List<String> copies = Collections.nCopies(5, "Hi");
// copies = ["Hi", "Hi", "Hi", "Hi", "Hi"]; cannot add/remove

// 4.4 addAll (varargs add)
List<Integer> list = new ArrayList<>();
Collections.addAll(list, 1, 2, 3, 4, 5);  // list = [1, 2, 3, 4, 5]
```

---

### 5. Unmodifiable Views

```java
List<String> modifiable = new ArrayList<>();
modifiable.add("A");
modifiable.add("B");

List<String> unmodifiable = Collections.unmodifiableList(modifiable);

System.out.println(unmodifiable.get(0));   // "A" works

// unmodifiable.add("C");  // throws UnsupportedOperationException at runtime

// If underlying list changes, view reflects it
modifiable.add("C");
System.out.println(unmodifiable);          // [A, B, C]
```

Similar methods exist for other types:

```java
Set<Integer> s       = new HashSet<>();
Set<Integer> roSet   = Collections.unmodifiableSet(s);

Map<String, Integer> m     = new HashMap<>();
Map<String, Integer> roMap = Collections.unmodifiableMap(m);
```

---

### 6. Synchronized Views (Thread-Safe Wrappers)

```java
List<Integer> list = new ArrayList<>();

// Wrap with synchronizedList for thread-safe access
List<Integer> syncList = Collections.synchronizedList(list);

// For iteration, still need external synchronization
synchronized (syncList) {
    for (int x : syncList) {
        // process x
    }
}
```

Other synchronized wrappers:

```java
Set<Integer> syncSet         = Collections.synchronizedSet(new HashSet<>());
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
```

---

### 7. Checked (Runtime Type-Safe) Collections

```java
List raw = new ArrayList(); // raw list (legacy)
List<String> safe = Collections.checkedList(raw, String.class);

safe.add("ok");             // works

Object o = 123;
// raw.add(o);              // allowed for raw, but:
// safe.add((String) o);    // ClassCastException at runtime via checkedList wrapper
```

Useful to catch heap-pollution issues when mixing generics with legacy/raw code.

---

### 8. Singleton and Empty Collections

```java
// Singleton
Set<String> singleSet   = Collections.singleton("ONE");
List<String> singleList = Collections.singletonList("ONE");
Map<String, Integer> singleMap = Collections.singletonMap("ONE", 1);

// singleSet.add("TWO");  // UnsupportedOperationException

// Empty
List<String> emptyList  = Collections.emptyList();
Set<String> emptySet    = Collections.emptySet();
Map<String, String> emptyMap = Collections.emptyMap();
```

---

### 9. Enumeration and List Conversion

```java
Vector<Integer> v = new Vector<>();
v.add(1);
v.add(2);
v.add(3);

// 9.1 enumeration from collection
Enumeration<Integer> e = Collections.enumeration(v);
while (e.hasMoreElements()) {
    System.out.println(e.nextElement());
}

// 9.2 list from enumeration
Enumeration<Integer> e2 = v.elements();
List<Integer> fromEnum = Collections.list(e2); // [1, 2, 3]
```

---

### 10. `indexOfSubList` / `lastIndexOfSubList`

```java
List<Integer> source = Arrays.asList(1, 2, 3, 2, 3, 4);
List<Integer> target = Arrays.asList(2, 3);

int firstIndex = Collections.indexOfSubList(source, target);     // 1
int lastIndex  = Collections.lastIndexOfSubList(source, target); // 3
```

---

### 11. `newSetFromMap` – Backed Set

```java
Map<String, Boolean> backingMap = new HashMap<>();
Set<String> customSet = Collections.newSetFromMap(backingMap);

customSet.add("A");
customSet.add("B");

System.out.println(customSet);  // [A, B]
System.out.println(backingMap); // {A=true, B=true}
```

