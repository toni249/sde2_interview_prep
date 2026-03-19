# Java DSA Syntax Revision (Ultra-Concise)

Format: **Type/DS → initialization → method (return | time | default/edge return | one-liner)**.

Notes:
- Hash DS ops are **avg/amortized**; worst-case can degrade with collisions.
- Tree DS ops are (log n).
- String/array ops often depend on length (m).

---

## Primitives + Wrappers

### Numeric (`byte/short/int/long/float/double`)
#### initialization
- literals: `int x=1; long y=1L; float f=1.0f; double d=1.0;`
- cast: `int x=(int) someLong;`
- parse: `int x=Integer.parseInt(s); long y=Long.parseLong(s); double z=Double.parseDouble(s);`

#### functions
- `Integer.parseInt(String)` → `int` | O(m) | **throws** `NumberFormatException` | parse base-10 int.
- `Long.parseLong(String)` → `long` | O(m) | **throws** `NumberFormatException` | parse base-10 long.
- `Double.parseDouble(String)` → `double` | O(m) | **throws** `NumberFormatException` | parse double.
- `String.valueOf(x)` → `String` | O(m) | never `null` | primitive → string.
- `Math.abs(int)` → `int` | O(1) | returns abs (watch `Integer.MIN_VALUE`) | absolute value.
- `Math.sqrt(double)` → `double` | O(1) | `NaN` if input < 0 | square root.
- operators `& | ^ ~ << >> >>>` → primitive | O(1) | — | bit masks/shifts.

### `char` / `Character`
#### initialization
- `char c='a'; char u='\u0061'; char x=s.charAt(i);`

#### functions
- `Character.isDigit(char)` → `boolean` | O(1) | `false` if not digit | digit check.
- `Character.isLetter(char)` → `boolean` | O(1) | `false` if not letter | letter check.
- `Character.toLowerCase(char)` → `char` | O(1) | unchanged if no mapping | lower-case.
- `Character.toUpperCase(char)` → `char` | O(1) | unchanged if no mapping | upper-case.

### `boolean` / `Boolean`
#### initialization
- `boolean b=true; boolean ok=(x>0); boolean p=Boolean.parseBoolean(s);`

#### functions
- `Boolean.parseBoolean(String)` → `boolean` | O(m) | `true` iff equalsIgnoreCase `"true"` else `false` | parse boolean.

---

## Arrays (`T[]`)

### initialization
- declare: `int[] a; String[] s;`
- sized: `int[] a=new int[n];`
- literal: `int[] a={1,2,3};`
- new+values: `int[] a=new int[]{1,2,3};`
- 2D: `int[][] g=new int[r][c]; int[][] jag=new int[r][];`

### `java.util.Arrays`
- `a.length` → `int` | O(1) | — | size property.
- `Arrays.sort(int[])` → `void` | O(n log n) | — | sort primitives.
- `Arrays.sort(T[])` → `void` | O(n log n) | — | sort objects (comparator overload exists).
- `Arrays.sort(arr, from, to)` → `void` | O(n log n) | — | sort range [from, to).
- `Arrays.parallelSort(arr)` → `void` | O(n log n / p) | — | parallel sort (multi-core).
- `Arrays.binarySearch(int[], key)` → `int` | O(log n) | `>=0` index; else `-(insertionPoint)-1` | search in sorted array.
- `Arrays.binarySearch(arr, from, to, key)` → `int` | O(log n) | same as above | search in range.
- `Arrays.fill(int[], val)` → `void` | O(n) | — | fill with value.
- `Arrays.fill(arr, from, to, val)` → `void` | O(to-from) | — | fill range.
- `Arrays.copyOf(int[], newLen)` → `int[]` | O(newLen) | pads defaults if bigger | resize/copy.
- `Arrays.copyOfRange(int[], from, to)` → `int[]` | O(to-from) | **throws** `AIOOBE` on bad range | slice copy.
- `Arrays.equals(int[], int[])` → `boolean` | O(n) | `false` if differ | shallow equality.
- `Arrays.deepEquals(Object[], Object[])` → `boolean` | O(total) | `false` if differ | deep equality.
- `Arrays.toString(int[])` → `String` | O(n) | `"null"` if arg null | stringify.
- `Arrays.deepToString(Object[])` → `String` | O(total) | `"null"` if arg null | stringify nested.
- `Arrays.asList(T...)` → `List<T>` | O(1) | fixed-size, backed by array | array → list view.
- `Arrays.stream(int[])` → `IntStream` | O(1) | — | sequential stream.
- `Arrays.parallelStream(int[])` → `IntStream` | O(1) | — | parallel stream.
- `Arrays.setAll(int[], generator)` → `void` | O(n) | — | set each element using generator function.
- `Arrays.mismatch(arr1, arr2)` → `int` | O(n) | -1 if equal | first differing index.

### Array Iteration
- traditional: `for(int i=0; i<a.length; i++) { a[i]; }` | index access
- for-each: `for(int x : a) { x; }` | value only, no index
- streams: `Arrays.stream(a).forEach(x -> ...)` | functional, supports filter/map
- parallel: `Arrays.stream(a).parallel().forEach(x -> ...)` | parallel processing
- indexed stream: `IntStream.range(0, a.length).forEach(i -> a[i])` | index via stream

---

## `String` (immutable)

### initialization
- literal: `String s="abc";`
- object: `String s=new String("abc");` (avoid unless needed)
- from char[]: `new String(chars)`
- from builder: `sb.toString()`

### methods (m = length)
- `length()` → `int` | O(1) | — | length.
- `charAt(i)` → `char` | O(1) | **throws** `IOOBE` | char at index.
- `substring(b)` / `substring(b,e)` → `String` | O(m) | **throws** `IOOBE` | slice copy.
- `indexOf(ch/str)` / `lastIndexOf(...)` → `int` | O(m·p) worst | `-1` if not found | find match index.
- `contains(cs)` → `boolean` | O(m·p) worst | `false` if not found | substring check.
- `startsWith(p)` / `endsWith(p)` → `boolean` | O(p) | `false` if not match | prefix/suffix.
- `toLowerCase()` / `toUpperCase()` → `String` | O(m) | returns new or same | case convert.
- `trim()` / `strip()` → `String` | O(m) | returns new or same | edge whitespace remove.
- `replace(char,char)` / `replace(seq,seq)` → `String` | O(m) | returns same if no change | replace.
- `split(regex)` → `String[]` | O(m) to O(m·regex) | never `null` | regex split.
- `String.join(delim, parts...)` → `String` | O(total) | **throws** `NPE` if delim/parts null | join strings.
- `"x".equals(y)` → `boolean` | O(m) | `false` if `y==null` | safe-null equality.

### String Iteration
- charAt: `for(int i=0; i<s.length(); i++) { s.charAt(i); }` | index + char
- toCharArray: `for(char c : s.toCharArray()) { c; }` | value only, creates copy
- streams: `s.chars().forEach(c -> ...)` | int stream of char codes
- chars mapped: `s.chars().mapToObj(c -> (char)c).forEach(...)` | Character stream

---

## `StringBuilder` / `StringBuffer` (mutable)

### initialization
- `new StringBuilder()`, `new StringBuilder(cap)`, `new StringBuilder("seed")`
- `new StringBuffer()` (thread-safe, slower)

### methods (n = length)
- `append(x)` → builder | O(1) amortized | returns `this` | append.
- `insert(i,x)` / `delete(s,e)` / `replace(s,e,str)` → builder | O(n) | **throws** `IOOBE` | mutate by index/range.
- `reverse()` → builder | O(n) | returns `this` | reverse.
- `length()` → `int` | O(1) | — | length.
- `capacity()` → `int` | O(1) | — | current capacity.
- `ensureCapacity(cap)` → `void` | O(cap) | — | pre-allocate capacity.
- `charAt(i)` → `char` | O(1) | **throws** `IOOBE` | char at index.
- `setCharAt(i,ch)` → `void` | O(1) | **throws** `IOOBE` | modify char at index.
- `substring(s)` / `substring(s,e)` → `String` | O(e-s) | **throws** `IOOBE` | extract substring (copies).
- `indexOf(str)` / `lastIndexOf(str)` → `int` | O(n·m) | `-1` if not found | find substring index.
- `startsWith(prefix)` → `boolean` | O(p) | `false` if not match | prefix check.
- `isEmpty()` → `boolean` | O(1) | — | empty check.
- `trimToSize()` → `void` | O(n) | — | reduce capacity to length.
- `toString()` → `String` | O(n) | never `null` | freeze.

---

## Collections (Quick chooser)
- `ArrayList`: random access.
- `HashMap/HashSet`: lookup/membership.
- `TreeMap/TreeSet`: sorted + range.
- `ArrayDeque`: stack/queue/deque.
- `PriorityQueue`: heap/top-k.
- `LinkedHashMap`: predictable order; access-order for LRU.

---

## `List` — `ArrayList<E>` / `LinkedList<E>`

### initialization
- `new ArrayList<>()`, `new ArrayList<>(cap)`, `new ArrayList<>(collection)`
- `new LinkedList<>()`

### methods
- `add(e)` → `boolean` | O(1) amortized | `true` | append.
- `add(i,e)` → `void` | O(n) | **throws** `IOOBE` | insert at index.
- `get(i)` → `E` | ArrayList O(1) / LinkedList O(n) | **throws** `IOOBE` | read.
- `set(i,e)` → `E` | O(n) worst | old value; **throws** `IOOBE` | overwrite.
- `remove(i)` → `E` | O(n) | removed; **throws** `IOOBE` | remove by index.
- `remove(obj)` → `boolean` | O(n) | `true` if removed | remove first match.
- `contains(obj)` → `boolean` | O(n) | `false` if absent | membership.
- `indexOf/lastIndexOf(obj)` → `int` | O(n) | `-1` if absent | find index.
- `subList(f,t)` → `List<E>` | O(1) | **throws** `IOOBE` | backed view.
- `size()/isEmpty()` → `int/boolean` | O(1) | — | size checks.
- `clear()` → `void` | O(n) | — | remove all.
- `toArray()` / `toArray(T[] arr)` → `T[]` | O(n) | new array | convert to array.
- `iterator()` → `Iterator<E>` | O(1) | — | iterator for traversal.
- `listIterator()` / `listIterator(i)` → `ListIterator<E>` | O(1) | — | bidirectional iterator.
- `spliterator()` → `Spliterator<E>` | O(1) | — | parallel iteration (Java 8+).
- `stream()` / `parallelStream()` → `Stream<E>` | O(1) | — | stream for functional ops.
- `containsAll(c)` → `boolean` | O(n·m) | `false` if any missing | subset check.
- `addAll(i,c)` / `addAll(c)` → `boolean` | O(n) | `true` if changed | add collection.
- `removeAll(c)` / `retainAll(c)` → `boolean` | O(n) | `true` if changed | remove/keep intersection.
- `sort(cmp)` → `void` | O(n log n) | — | sort in place (Java 8+).
- `removeIf(pred)` → `boolean` | O(n) | `true` if changed | conditional remove (Java 8+).
- `replaceAll(unaryOp)` → `void` | O(n) | — | transform all elements (Java 8+).

### `Collections` utilities
- `Collections.sort(list)` → `void` | O(n log n) | — | sort.
- `Collections.binarySearch(list,key)` → `int` | O(log n) | `>=0` index; else `-(insertionPoint)-1` | search sorted.
- `Collections.reverse(list)` → `void` | O(n) | — | reverse.
- `Collections.swap(list, i, j)` → `void` | O(1) | — | swap elements at indices.
- `Collections.shuffle(list)` → `void` | O(n) | — | random shuffle.
- `Collections.min(list)` / `max(list)` → `E` | O(n) | **throws** `NoSuchElementException` if empty | extremes.
- `Collections.frequency(list, o)` → `int` | O(n) | 0 if not found | count occurrences.
- `Collections.disjoint(c1, c2)` → `boolean` | O(n) | `false` if any common | no common elements.
- `Collections.fill(list, val)` → `void` | O(n) | — | fill list with value.
- `Collections.copy(dest, src)` → `void` | O(n) | **throws** `IndexOutOfBoundsException` | copy src to dest.
- `Collections.rotate(list, dist)` → `void` | O(n) | — | rotate by distance.
- `Collections.indexOfSubList(list, sub)` / `lastIndexOfSubList(...)` → `int` | O(n·m) | -1 if not found | find sublist.
- `Collections.unmodifiableList/Set/Map(...)` → immutable view | O(1) | **throws** `UnsupportedOperationException` on modify | read-only wrapper.
- `Collections.synchronizedList/Set/Map(...)` → synchronized view | O(1) | thread-safe wrapper | concurrent access.
- `Collections.emptyList/Set/Map()` → immutable empty | O(1) | — | constant empty collection.
- `Collections.singleton(e)` / `singletonList(e)` / `singletonMap(k,v)` → immutable | O(1) | — | single-element collection.
- `Collections.nCopies(n, e)` → immutable list | O(1) | — | list with n copies of e.

### List Iteration
- traditional: `for(int i=0; i<list.size(); i++) { list.get(i); }` | index access (ArrayList O(1), LinkedList O(n))
- for-each: `for(E x : list) { x; }` | value only, uses iterator
- iterator: `Iterator<E> it=list.iterator(); while(it.hasNext()) { it.next(); }` | safe removal via `it.remove()`
- list-iterator: `ListIterator<E> it=list.listIterator();` | bidirectional, `hasPrevious()/previous()/add()/set()`
- streams: `list.stream().forEach(x -> ...)` | functional, supports filter/map
- parallel: `list.parallelStream().forEach(x -> ...)` | parallel processing
- indexed stream: `IntStream.range(0, list.size()).forEach(i -> list.get(i))` | index via stream

---

## `Set` — `HashSet<E>` / `LinkedHashSet<E>` / `TreeSet<E>`

### initialization
- `new HashSet<>()`, `new HashSet<>(cap,lf)`, `new HashSet<>(collection)`
- `new LinkedHashSet<>()`
- `new TreeSet<>()`, `new TreeSet<>(comparator)`

### methods
- `add(e)` → `boolean` | Hash O(1) avg / Tree O(log n) | `true` if added else `false` | insert unique.
- `contains(o)` → `boolean` | Hash O(1) avg / Tree O(log n) | `false` if absent | membership.
- `remove(o)` → `boolean` | Hash O(1) avg / Tree O(log n) | `true` if removed else `false` | delete.
- `addAll/retainAll/removeAll(c)` → `boolean` | O(n) | `true` if changed | set algebra.
- `containsAll(c)` → `boolean` | O(n) | `false` if any missing | subset check.
- `size()/isEmpty()` → `int/boolean` | O(1) | — | size.
- `clear()` → `void` | O(n) | — | remove all.
- `toArray()` / `toArray(T[] arr)` → `T[]` | O(n) | new array | convert to array.
- `iterator()` → `Iterator<E>` | O(1) | — | iterator for traversal.
- `spliterator()` → `Spliterator<E>` | O(1) | — | parallel iteration (Java 8+).
- `stream()` / `parallelStream()` → `Stream<E>` | O(1) | — | stream for functional ops.
- `removeIf(pred)` → `boolean` | O(n) | `true` if changed | conditional remove (Java 8+).

### `TreeSet` ordered ops
- `first()/last()` → `E` | O(log n) | **throws** `NoSuchElementException` if empty | extremes.
- `lower/higher/floor/ceiling(e)` → `E` | O(log n) | `null` if none | neighbor.
- `subSet(from,to)` / `headSet(to)` / `tailSet(from)` → `NavigableSet<E>` | O(log n) | view | range views.
- `descendingSet()` → `NavigableSet<E>` | O(1) | — | reverse order view.
- `descendingIterator()` → `Iterator<E>` | O(1) | — | reverse iterator.
- `pollFirst()/pollLast()` → `E` | O(log n) | `null` if empty | retrieve+remove extremes.
- `size()/isEmpty()` → `int/boolean` | O(1) | — | size.
- `clear()` → `void` | O(n) | — | remove all.
- `toArray()` / `toArray(T[] arr)` → `T[]` | O(n) | new array | convert to array.
- `comparator()` → `Comparator<? super E>` | O(1) | `null` if natural ordering | get comparator.

### Set Iteration
- for-each: `for(E x : set) { x; }` | value only, uses iterator
- iterator: `Iterator<E> it=set.iterator(); while(it.hasNext()) { it.next(); }` | safe removal via `it.remove()`
- streams: `set.stream().forEach(x -> ...)` | functional, supports filter/map
- parallel: `set.parallelStream().forEach(x -> ...)` | parallel processing
- `TreeSet` specific: `for(E x : treeSet.descendingSet()) { x; }` | reverse order iteration

---

## `Map` — `HashMap<K,V>` / `LinkedHashMap<K,V>` / `TreeMap<K,V>`

### initialization
- `new HashMap<>()`, `new HashMap<>(cap,lf)`, `new HashMap<>(otherMap)`
- `new LinkedHashMap<>()`, `new LinkedHashMap<>(16,0.75f,true)` (access-order)
- `new TreeMap<>()`, `new TreeMap<>(comparator)`

### methods
- `put(k,v)` → `V` | O(1) avg / O(log n) | previous value or `null` | insert/replace.
- `get(k)` → `V` | O(1) avg / O(log n) | value or `null` | lookup.
- `containsKey(k)` → `boolean` | O(1) avg / O(log n) | `false` if absent | key check.
- `containsValue(v)` → `boolean` | O(n) | `false` if absent | value check (slow).
- `remove(k)` → `V` | O(1) avg / O(log n) | removed value or `null` | delete.
- `getOrDefault(k,def)` → `V` | O(1) avg / O(log n) | `def` if absent | default.
- `putIfAbsent(k,v)` → `V` | O(1) avg / O(log n) | existing value or `null` | insert if missing.
- `compute/computeIfAbsent/merge(...)` → `V` | O(1) avg / O(log n) | new value or `null` (if removed) | functional update.
- `entrySet()/keySet()/values()` → view | O(1) | empty view if empty | iterate O(n).
- `size()/isEmpty()` → `int/boolean` | O(1) | — | size.
- `clear()` → `void` | O(n) | — | remove all.
- `forEach(consumer)` → `void` | O(n) | — | lambda iteration.
- `replace(k,v)` / `replace(k,oldV,newV)` → `V` | O(1) | `null` if not replaced | conditional replace.
- `replaceAll(biFunction)` → `void` | O(n) | — | transform all values.

### `TreeMap` ordered ops
- `firstKey()/lastKey()` → `K` | O(log n) | **throws** `NoSuchElementException` if empty | extremes.
- `lowerKey/higherKey/floorKey/ceilingKey(k)` → `K` | O(log n) | `null` if none | neighbor keys.
- `subMap(from,to)` / `headMap(to)` / `tailMap(from)` → `NavigableMap<K,V>` | O(log n) | view | range views.
- `descendingMap()` → `NavigableMap<K,V>` | O(1) | — | reverse order view.
- `descendingKeySet()` → `NavigableSet<K>` | O(1) | — | reverse key set.
- `pollFirstEntry()/pollLastEntry()` → `Map.Entry<K,V>` | O(log n) | `null` if empty | retrieve+remove extremes.
- `firstEntry()/lastEntry()` → `Map.Entry<K,V>` | O(log n) | `null` if empty | view extremes.
- `lowerEntry/higherEntry/floorEntry/ceilingEntry(k)` → `Map.Entry<K,V>` | O(log n) | `null` if none | neighbor entries.
- `size()/isEmpty()` → `int/boolean` | O(1) | — | size.
- `clear()` → `void` | O(n) | — | remove all.
- `comparator()` → `Comparator<? super K>` | O(1) | `null` if natural ordering | get comparator.

### Map Iteration
- entrySet: `for(Map.Entry<K,V> e : map.entrySet()) { e.getKey(); e.getValue(); }` | key+value, most efficient
- keySet: `for(K k : map.keySet()) { map.get(k); }` | key only, extra lookup O(n) total
- values: `for(V v : map.values()) { v; }` | value only
- forEach: `map.forEach((k,v) -> ...)` → `void` | lambda iteration
- streams entries: `map.entrySet().stream().forEach(e -> ...)` | functional on entries
- streams keys: `map.keySet().stream().forEach(k -> ...)` | functional on keys
- streams values: `map.values().stream().forEach(v -> ...)` | functional on values
- `TreeMap` specific: `for(K k : treeMap.descendingKeySet()) { k; }` | reverse key order
- `LinkedHashMap` specific: iteration order = insertion order (or access order if configured)

---

## Queue / Deque / Heap

### initialization
- `Deque<E> dq = new ArrayDeque<>();`
- `Queue<E> q = new ArrayDeque<>();`
- `PriorityQueue<E> pq = new PriorityQueue<>();` (min-heap)
- `PriorityQueue<E> pq = new PriorityQueue<>(cmp);`

### `Deque` (ArrayDeque)
- `offerFirst/offerLast(e)` → `boolean` | O(1) | usually `true` | enqueue.
- `pollFirst/pollLast()` → `E` | O(1) | `null` if empty | dequeue.
- `peekFirst/peekLast()` / `peek()` → `E` | O(1) | `null` if empty | view.
- `push(e)` → `void` | O(1) | — | stack push.
- `pop()` → `E` | O(1) | **throws** `NoSuchElementException` if empty | stack pop.
- `addFirst/addLast(e)` → `void` | O(1) | **throws** `IllegalStateException` if capacity restricted | add (strict).
- `removeFirst/removeLast()` → `E` | O(1) | **throws** `NoSuchElementException` if empty | remove (strict).
- `getFirst/getLast()` → `E` | O(1) | **throws** `NoSuchElementException` if empty | view (strict).
- `add(e)` / `offer(e)` → `boolean` | O(1) | `true` | add to tail (Queue interface).
- `remove()` / `poll()` → `E` | O(1) | head element | remove from head (Queue interface).
- `contains(o)` → `boolean` | O(n) | `false` if absent | membership check.
- `remove(o)` → `boolean` | O(n) | `true` if removed | remove first occurrence.
- `size()/isEmpty()` → `int/boolean` | O(1) | — | size.
- `clear()` → `void` | O(n) | — | remove all.
- `toArray()` / `toArray(T[] arr)` → `T[]` | O(n) | new array | convert to array.
- `iterator()` / `descendingIterator()` → `Iterator<E>` | O(1) | — | forward/reverse iterator.
- `stream()` / `parallelStream()` → `Stream<E>` | O(1) | — | stream for functional ops.

### `PriorityQueue`
- `offer(e)` → `boolean` | O(log n) | `true` | insert.
- `poll()` → `E` | O(log n) | `null` if empty | remove top.
- `peek()` → `E` | O(1) | `null` if empty | view top.
- `add(e)` → `boolean` | O(log n) | **throws** `IllegalStateException` if capacity restricted | insert (strict).
- `remove()` → `E` | O(n) | **throws** `NoSuchElementException` if empty | remove head.
- `element()` → `E` | O(1) | **throws** `NoSuchElementException` if empty | view head (strict).
- `contains(o)` → `boolean` | O(n) | `false` if absent | membership check.
- `remove(o)` → `boolean` | O(n) | `true` if removed | remove specific element (rebuilds heap).
- `addAll(c)` → `boolean` | O(n) | `true` if changed | add all elements.
- `clear()` → `void` | O(n) | — | remove all.
- `size()/isEmpty()` → `int/boolean` | O(1) | — | size.
- `toArray()` / `toArray(T[] arr)` → `T[]` | O(n) | new array | convert to array.
- `iterator()` → `Iterator<E>` | O(1) | — | iterator (no ordering guarantee).
- `stream()` / `parallelStream()` → `Stream<E>` | O(1) | — | stream for functional ops.
- `comparator()` → `Comparator<? super E>` | O(1) | `null` if natural ordering | get comparator.

### Queue/Deque Iteration
- for-each: `for(E x : queue/deque) { x; }` | value only, no ordering guarantee for PQ
- iterator: `Iterator<E> it=queue/deque.iterator(); while(it.hasNext()) { it.next(); }` | safe removal
- streams: `queue/deque.stream().forEach(x -> ...)` | functional
- `Deque` specific: `for(E x : deque.descendingIterator()) { x; }` | reverse order
- `PriorityQueue` note: iteration order ≠ priority order; use `poll()` for sorted access

---

## Lambda & Functional Interfaces (Java 8+)

### Common Functional Interfaces (`java.util.function`)
- `Predicate<T>` → `boolean test(T)` | condition check | `x -> x > 0`
- `Function<T,R>` → `R apply(T)` | transform | `s -> s.length()`
- `Consumer<T>` → `void accept(T)` | side-effect | `x -> System.out.println(x)`
- `Supplier<T>` → `T get()` | lazy supply | `() -> new ArrayList<>()`
- `Comparator<T>` → `int compare(T,T)` | sort order | `(a,b) -> a-b`
- `UnaryOperator<T>` → `T apply(T)` | same-type transform | `x -> x*2`
- `BinaryOperator<T>` → `T apply(T,T)` | same-type combine | `(a,b) -> a+b`

### Lambda Syntax Variations
- full: `(int x, int y) -> { return x + y; }`
- concise: `(x, y) -> x + y`
- single param: `x -> x * 2` (no parens needed)
- no param: `() -> Math.random()`
- method ref: `String::valueOf`, `System::out::println`, `Class::new`

### Comparator Lambda Patterns
- natural: `Comparator.naturalOrder()` | ascending
- reverse: `Comparator.reverseOrder()` | descending
- custom: `(a,b) -> a-b` | ascending int
- custom: `(a,b) -> b-a` | descending int
- chaining: `Comparator.comparing(Person::getAge).thenComparing(Person::getName)`
- reverse chaining: `Comparator.comparing(Person::getAge).reversed()`

### Collections + Lambda (Streams API)
- `list.stream()` → `Stream<T>` | sequential stream
- `list.parallelStream()` → `Stream<T>` | parallel stream
- `stream.filter(pred)` → `Stream<T>` | keep matching | `nums.stream().filter(x -> x > 0)`
- `stream.map(func)` → `Stream<R>` | transform | `nums.stream().map(x -> x*2)`
- `stream.flatMap(func)` → `Stream<R>` | flatten+map | `list.stream().flatMap(Collection::stream)`
- `stream.sorted(cmp)` → `Stream<T>` | sort | `stream.sorted(Comparator.naturalOrder())`
- `stream.distinct()` → `Stream<T>` | unique | remove duplicates
- `stream.limit(n)` → `Stream<T>` | first n | truncate
- `stream.skip(n)` → `Stream<T>` | skip n | drop first n
- `stream.min(cmp)/max(cmp)` → `Optional<T>` | extremes
- `stream.count()` → `long` | count elements
- `stream.anyMatch/allMatch/noneMatch(pred)` → `boolean` | predicate checks
- `stream.findFirst()/findAny()` → `Optional<T>` | get element
- `stream.collect(toList()/toSet()/toMap(k,v))` → Collection/Map | terminal collect
- `stream.reduce(identity, accumulator)` → `T` | fold | `nums.stream().reduce(0, (a,b) -> a+b)`
- `stream.forEach(consumer)` → `void` | iterate | `list.forEach(System.out::println)`

### Map + Lambda Methods
- `map.forEach((k,v) -> ...)` → `void` | iterate entries
- `map.compute(k, (k,v) -> ...)` → `V` | update or remove
- `map.computeIfAbsent(k, key -> ...)` → `V` | lazy insert | `map.computeIfAbsent(k, key -> new ArrayList<>())`
- `map.computeIfPresent(k, (k,v) -> ...)` → `V` | update if exists
- `map.merge(k, v, (oldVal,newVal) -> ...)` → `V` | merge values | `map.merge(k, 1, Integer::sum)`
- `map.removeIf(k, (k,v) -> ...)` → `boolean` | conditional remove (Java 8+)

### List + Lambda Methods
- `list.removeIf(pred)` → `boolean` | conditional remove | `list.removeIf(x -> x < 0)`
- `list.replaceAll(unaryOp)` → `void` | transform all | `list.replaceAll(x -> x*2)`
- `list.sort(cmp)` → `void` | sort | `list.sort(Comparator.naturalOrder())`
- `list.parallelStream()` → `Stream<T>` | parallel processing

---

## Patterns (names + use)
- Two pointers: ends / slow-fast.
- Sliding window: subarray/substring constraints.
- Prefix sum + hashmap: range/subarray counts.
- Monotonic stack: next greater/smaller, histogram.
- Monotonic deque: sliding window max/min.
- Binary search: sorted array or answer space.
- BFS/DFS: traversal, components, unweighted shortest paths.
- Top K: heap or quickselect.
- DSU: connectivity.
- DP: overlapping subproblems.
