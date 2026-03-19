# Java Core Concepts

## Table of Contents
1. [Pass by Value vs Pass by Reference](#1-pass-by-value-vs-pass-by-reference)
2. [Interface vs Class Initialization](#2-interface-vs-class-initialization)
3. [String Immutability](#3-string-immutability)
4. [equals() vs ==](#4-equals-vs-)
5. [final, finally, finalize](#5-final-finally-finalize)
6. [Static vs Instance](#6-static-vs-instance)
7. [Autoboxing & Unboxing](#7-autoboxing--unboxing)

---

## 1. Pass by Value vs Pass by Reference

### Java is ALWAYS Pass by Value

```java
// Common misconception: Java passes objects by reference
// Reality: Java passes THE REFERENCE VALUE by value
```

### Primitive Types

```java
public class PassByValuePrimitive {
    public static void main(String[] args) {
        int x = 10;
        modifyPrimitive(x);
        System.out.println(x);  // Output: 10 (unchanged)
    }
    
    static void modifyPrimitive(int value) {
        value = 100;  // Only modifies the copy
    }
}

// What happens:
// 1. x = 10 stored in main's stack frame
// 2. Call modifyPrimitive(x):
//    - COPY of value (10) is passed
//    - value = 100 in modifyPrimitive's stack frame
//    - x in main remains 10
```

### Reference Types (Objects)

```java
public class PassByValueReference {
    static class Person {
        String name;
        Person(String name) { this.name = name; }
    }
    
    public static void main(String[] args) {
        Person p = new Person("Alice");
        
        // Case 1: Modifying object's state
        modifyObject(p);
        System.out.println(p.name);  // Output: "Bob" (CHANGED!)
        
        // Case 2: Reassigning reference
        reassignReference(p);
        System.out.println(p.name);  // Output: "Bob" (unchanged)
    }
    
    static void modifyObject(Person person) {
        person.name = "Bob";  // Modifies the SAME object
    }
    
    static void reassignReference(Person person) {
        person = new Person("Charlie");  // Only changes local copy
    }
}

// What happens:
// Memory layout:
// Stack (main)          Heap                    Stack (modifyObject)
// ┌─────────┐          ┌─────────────┐         ┌─────────────┐
// │   p     │ ────────→│ Person      │         │   person    │ ──┐
// │ ref=100 │          │ name="Alice"│         │ ref=100     │   │
// └─────────┘          └─────────────┘         └─────────────┘   │
//                                                                │
// After modifyObject(p):                                         │
// Both p and person point to SAME object (ref=100) ←────────────┘
// So person.name = "Bob" changes the object
//
// After reassignReference(p):
// Stack (main)          Heap                    Stack (reassignReference)
// ┌─────────┐          ┌─────────────┐         ┌─────────────┐
// │   p     │ ────────→│ Person      │         │   person    │ ──→ new Person("Charlie")
// │ ref=100 │          │ name="Bob"  │         │ ref=200     │
// └─────────┘          └─────────────┘         └─────────────┘
// person = new Person() only changes local copy (ref=200)
// p still points to original object (ref=100)
```

### Visual Comparison

```java
// PASS BY VALUE (what Java does):
// ┌──────────┐         ┌──────────┐
// │  Caller  │         │  Callee  │
// │   x = 10 │ ──copy──│ value=10 │
// └──────────┘         └──────────┘

// PASS BY REFERENCE (what Java does NOT do):
// ┌──────────┐         ┌──────────┐
// │  Caller  │         │  Callee  │
// │   x = 10 │ ──ref──→│ value    │ ──→ 10
// └──────────┘         └──────────┘

// PASS BY REFERENCE VALUE (what Java DOES for objects):
// ┌──────────┐         ┌──────────┐
// │  Caller  │         │  Callee  │
// │  obj ref │ ──copy──│ obj ref  │ ──both point to same object
// └──────────┘         └──────────┘      in heap
```

### Common Interview Question

```java
public class SwapExample {
    public static void main(String[] args) {
        Integer a = 1;
        Integer b = 2;
        swap(a, b);
        System.out.println("a = " + a + ", b = " + b");  // a = 1, b = 2 (NOT swapped!)
    }
    
    // This CANNOT swap in Java
    public static void swap(Integer x, Integer y) {
        Integer temp = x;
        x = y;
        y = temp;
        // Only swaps local copies of references
    }
    
    // To actually swap, you need to swap within a container
    static class Container {
        Integer a, b;
    }
    
    public static void swap(Container c) {
        Integer temp = c.a;
        c.a = c.b;
        c.b = temp;
        // This works! Modifies the same object
    }
}
```

### Summary Table

| Aspect | Primitive Types | Reference Types |
|--------|----------------|-----------------|
| **What's passed** | Copy of value | Copy of reference value |
| **Can modify original?** | No | Yes (object state) |
| **Can reassign original?** | No | No |
| **Example** | `int x = 5` → copy of 5 | `Person p` → copy of reference |

---

## 2. Interface vs Class Initialization

### Two Ways to Initialize

```java
// Way 1: Interface type (Recommended)
List<String> list = new ArrayList<>();

// Way 2: Concrete class type
ArrayList<String> list = new ArrayList<>();
```

### Interface Initialization (Recommended)

```java
// ✅ RECOMMENDED: Program to interface
List<String> list = new ArrayList<>();
Map<String, Integer> map = new HashMap<>();
Set<Integer> set = new HashSet<>();

// Benefits:
// 1. Flexibility - Easy to change implementation
List<String> list = new ArrayList<>();  // Today
list = new LinkedList<>();  // Tomorrow (no other code changes)

// 2. Loose coupling
public void process(List<String> items) {
    // Works with ArrayList, LinkedList, etc.
}

// 3. API design - Expose minimal contract
public class Service {
    // Clients only see List interface methods
    public List<String> getItems() {
        return new ArrayList<>();
    }
}

// 4. Testability - Easy to mock
List<String> mockList = Mockito.mock(List.class);
```

### Concrete Class Initialization

```java
// ⚠️ Use only when you need class-specific methods
ArrayList<String> arrayList = new ArrayList<>();

// When this is acceptable:
// 1. Need ArrayList-specific methods
arrayList.ensureCapacity(1000);  // Not in List interface
arrayList.trimToSize();

// 2. Need LinkedList-specific methods
LinkedList<String> linked = new LinkedList<>();
linked.addFirst("head");  // Not in List interface
linked.addLast("tail");

// 3. Need HashMap-specific methods
HashMap<String, Integer> hashMap = new HashMap<>();
hashMap.clone();  // Not in Map interface

// 4. Need TreeMap-specific methods
TreeMap<String, Integer> treeMap = new TreeMap<>();
treeMap.floorKey("key");  // NavigableMap, but often use TreeMap directly
```

### Real-World Example

```java
// ❌ BAD: Tied to specific implementation
public class UserService {
    private ArrayList<String> users = new ArrayList<>();
    
    public ArrayList<String> getUsers() {
        return users;
    }
}

// If you need to change to LinkedList later:
// - Change field type
// - Change return type
// - ALL client code using getUsers() must change!

// ✅ GOOD: Program to interface
public class UserService {
    private List<String> users = new ArrayList<>();
    
    public List<String> getUsers() {
        return users;
    }
}

// Can change implementation without affecting clients:
private List<String> users = new LinkedList<>();  // No other changes needed
```

### Factory Methods (Java 9+)

```java
// Even better: Use factory methods when implementation doesn't matter
List<String> list = List.of("A", "B", "C");  // Immutable
Set<Integer> set = Set.of(1, 2, 3);          // Immutable
Map<String, Integer> map = Map.of("A", 1);   // Immutable

// Or with collectors (implementation chosen by collector)
List<String> list = stream.collect(Collectors.toList());
Set<String> set = stream.collect(Collectors.toSet());
```

### When to Use Which

| Scenario | Recommended Approach |
|----------|---------------------|
| General use | `List<T> list = new ArrayList<>()` |
| Need specific methods | `ArrayList<T> list = new ArrayList<>()` |
| Factory/Builder | `List<T> list = List.of(...)` |
| Method parameters | `List<T>` (interface) |
| Method return types | `List<T>` (interface) |
| Fields (internal) | `List<T>` (interface) |
| Need thread-safe | `List<T> list = Collections.synchronizedList(...)` |

### Code Example: Best Practice

```java
// ✅ GOOD DESIGN
public class OrderService {
    // Interface for flexibility
    private List<Order> orders = new ArrayList<>();
    private Map<String, Customer> customers = new HashMap<>();
    
    // Interface in method signature
    public void processOrders(List<Order> newOrders) {
        orders.addAll(newOrders);
    }
    
    // Interface in return type
    public List<Order> getOrders() {
        return Collections.unmodifiableList(orders);  // Defensive copy
    }
    
    // Can change implementation without breaking API
    private void internalRefactor() {
        orders = new LinkedList<>();  // No client code affected
    }
}

// ❌ POOR DESIGN
public class OrderService {
    private ArrayList<Order> orders = new ArrayList<>();
    
    public ArrayList<Order> getOrders() {
        return orders;  // Clients depend on ArrayList
    }
    
    // Changing to LinkedList breaks all clients!
}
```

### Summary

```java
// ✅ DO: Program to interface
List<String> list = new ArrayList<>();
Map<String, Integer> map = new HashMap<>();
Set<Integer> set = new HashSet<>();

// ⚠️ ONLY WHEN NEEDED: Concrete class for specific methods
ArrayList<String> arrayList = new ArrayList<>();  // Need ensureCapacity()
LinkedList<String> linked = new LinkedList<>();   // Need addFirst()
TreeMap<String, Integer> treeMap = new TreeMap<>(); // Need floorKey()

// ✅ MODERN: Factory methods (Java 9+)
List<String> list = List.of("A", "B", "C");
```

---

## 3. String Immutability

### Why String is Immutable

```java
String s1 = "Hello";
String s2 = "Hello";
String s3 = new String("Hello");

// Memory layout:
// String Pool          Heap
// ┌─────────┐         ┌─────────────┐
// │ "Hello" │ ←───────│ s3 (new)    │
// └─────────┘         └─────────────┘
//     ↑
//     ├────── s1
//     └────── s2

// s1 and s2 point to SAME object in pool
// s3 points to DIFFERENT object in heap
```

### Immutability Benefits

```java
// 1. String Pool (Memory efficiency)
String a = "Java";
String b = "Java";  // Reuses same object
System.out.println(a == b);  // true (same reference)

// 2. Security (Method parameters)
void connect(String url) {
    // url cannot be changed by caller after passing
}

// 3. Thread Safety (No synchronization needed)
String shared = "shared";
// Multiple threads can read without locking

// 4. HashCode Caching
String key = "key";
int hash = key.hashCode();  // Computed once, cached
```

### String Operations Create New Objects

```java
String s = "Hello";
s.concat(" World");  // Returns NEW string, s unchanged
s.toUpperCase();     // Returns NEW string
s.replace("e", "a"); // Returns NEW string

// Correct usage:
s = s.concat(" World");  // Reassign
```

### StringBuilder for Mutable Operations

```java
// ❌ BAD: Creates many intermediate strings
String result = "";
for (int i = 0; i < 100; i++) {
    result += i;  // Creates new String each iteration
}

// ✅ GOOD: StringBuilder is mutable
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 100; i++) {
    sb.append(i);
}
String result = sb.toString();
```

---

## 4. equals() vs ==

### == Operator

```java
// Primitives: Compares VALUES
int a = 5;
int b = 5;
System.out.println(a == b);  // true

// Reference types: Compares REFERENCES (memory addresses)
String s1 = new String("Hello");
String s2 = new String("Hello");
System.out.println(s1 == s2);  // false (different objects)

String s3 = "Hello";
String s4 = "Hello";
System.out.println(s3 == s4);  // true (same pool object)
```

### equals() Method

```java
// Object class default (same as ==)
class Person {
    String name;
}

Person p1 = new Person();
Person p2 = new Person();
System.out.println(p1.equals(p2));  // false (default compares references)

// Override for content comparison
class Person {
    String name;
    int age;
    
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;  // Same reference
        if (obj == null || getClass() != obj.getClass()) return false;
        Person p = (Person) obj;
        return age == p.age && Objects.equals(name, p.name);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }
}

Person p1 = new Person("Alice", 25);
Person p2 = new Person("Alice", 25);
System.out.println(p1.equals(p2));  // true (same content)
```

### String equals() Example

```java
String s1 = new String("Hello");
String s2 = new String("Hello");

System.out.println(s1 == s2);      // false (different references)
System.out.println(s1.equals(s2)); // true (same content)

// Best practice: Call equals on literal (null-safe)
String userInput = null;
// userInput.equals("admin")  // NullPointerException!
"admin".equals(userInput);  // false (safe)
```

### Summary Table

| Comparison | == | equals() |
|------------|-----|----------|
| **Primitives** | Compares values | N/A |
| **Objects (default)** | Compares references | Compares references |
| **Objects (overridden)** | Compares references | Compares content |
| **String** | Reference check | Content check |

---

## 5. final, finally, finalize

### final Keyword

```java
// 1. final variable (cannot reassign)
final int x = 10;
x = 20;  // Compilation error!

final List<String> list = new ArrayList<>();
list.add("A");  // OK (object can change)
list = new LinkedList<>();  // Error! (cannot reassign reference)

// 2. final method (cannot override)
class Parent {
    public final void show() {}
}
class Child extends Parent {
    public void show() {}  // Error! Cannot override final method
}

// 3. final class (cannot extend)
final class Utility {}
class MyUtil extends Utility {}  // Error!

// 4. final parameter (cannot modify)
void process(final int value) {
    value = 100;  // Error!
}
```

### finally Block

```java
// Always executes (even with return/exception)
try {
    int result = 10 / 0;
} catch (ArithmeticException e) {
    System.out.println("Caught exception");
} finally {
    System.out.println("Always executes");
}

// Common use: Resource cleanup
FileInputStream fis = null;
try {
    fis = new FileInputStream("file.txt");
    // read file
} catch (IOException e) {
    e.printStackTrace();
} finally {
    if (fis != null) {
        try { fis.close(); } catch (IOException e) {}
    }
}

// Modern Java 7+: try-with-resources (better)
try (FileInputStream fis = new FileInputStream("file.txt")) {
    // read file
} catch (IOException e) {
    e.printStackTrace();
}  // Auto-closes, no finally needed
```

### finalize() Method (Deprecated)

```java
// ❌ DEPRECATED (Java 9+)
// Called by GC before object destruction
// DO NOT RELY ON IT!

class MyClass {
    @Override
    protected void finalize() throws Throwable {
        // Cleanup code (unreliable!)
        super.finalize();
    }
}

// Problems:
// 1. No guarantee when (or if) it runs
// 2. Performance overhead
// 3. Unpredictable behavior

// ✅ USE INSTEAD:
// - try-with-resources for AutoCloseable
// - Explicit close() methods
// - Cleaner API (Java 9+)
```

### Summary

| Keyword | Purpose | Use Case |
|---------|---------|----------|
| `final` | Make immutable | Constants, prevent override |
| `finally` | Always execute | Resource cleanup |
| `finalize()` | GC hook | ❌ Deprecated, avoid |

---

## 6. Static vs Instance

### Static Members

```java
class Counter {
    // Static variable (shared across ALL instances)
    static int count = 0;
    
    // Instance variable (per object)
    int id;
    
    Counter() {
        id = ++count;
    }
    
    // Static method (can only access static members)
    static int getCount() {
        return count;
        // return id;  // Error! Cannot access instance variable
    }
    
    // Instance method (can access both)
    int getId() {
        return id;
        return count;  // OK
    }
}

// Usage
Counter c1 = new Counter();  // id = 1
Counter c2 = new Counter();  // id = 2

System.out.println(Counter.count);  // 2 (shared)
System.out.println(c1.id);  // 1 (instance)
System.out.println(c2.id);  // 2 (instance)
```

### When to Use Static

```java
// 1. Utility methods (no state needed)
class MathUtils {
    public static int add(int a, int b) {
        return a + b;
    }
}
MathUtils.add(5, 3);  // No object needed

// 2. Constants
class Constants {
    public static final String APP_NAME = "MyApp";
}

// 3. Singleton pattern
class Singleton {
    private static Singleton instance;
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}

// 4. Factory methods
List<String> list = List.of("A", "B");
```

### Static vs Instance Comparison

| Aspect | Static | Instance |
|--------|--------|----------|
| **Belongs to** | Class | Object |
| **Memory** | Once per class | Once per object |
| **Access** | Classname.member | object.member |
| **Can access** | Only static | Static + instance |
| **Use case** | Utilities, constants | Object state/behavior |

---

## 7. Autoboxing & Unboxing

### What is Autoboxing?

```java
// Automatic conversion between primitive and wrapper

// Autoboxing: primitive → wrapper
int primitive = 5;
Integer wrapper = primitive;  // Auto: Integer.valueOf(primitive)

// Unboxing: wrapper → primitive
Integer wrapper = 10;
int primitive = wrapper;  // Auto: wrapper.intValue()
```

### Behind the Scenes

```java
// What compiler does:
Integer auto = 5;           // Integer.valueOf(5)
Integer manual = Integer.valueOf(5);

int autoPrimitive = wrapper;  // wrapper.intValue()
int manualPrimitive = wrapper.intValue();
```

### Common Pitfalls

```java
// 1. NullPointerException
Integer wrapper = null;
int primitive = wrapper;  // NullPointerException!

// 2. == compares references (not values)
Integer a = 128;
Integer b = 128;
System.out.println(a == b);  // false (different objects)
System.out.println(a.equals(b));  // true (same value)

// 3. Cache for -128 to 127
Integer x = 127;
Integer y = 127;
System.out.println(x == y);  // true (cached)

Integer p = 128;
Integer q = 128;
System.out.println(p == q);  // false (not cached)

// 4. Performance in loops
// ❌ BAD: Creates many Integer objects
int sum = 0;
List<Integer> list = new ArrayList<>();
for (int i = 0; i < 10000; i++) {
    list.add(i);  // Autoboxing creates Integer each time
    sum += list.get(i);  // Unboxing each time
}

// ✅ GOOD: Use primitives when possible
int[] array = new int[10000];
for (int i = 0; i < 10000; i++) {
    array[i] = i;  // No boxing
    sum += array[i];  // No unboxing
}
```

### When Autoboxing Happens

```java
// 1. Collections (only accept objects)
List<Integer> list = new ArrayList<>();
list.add(5);  // Autoboxes int → Integer

// 2. Generics
Map<String, Integer> map = new HashMap<>();
map.put("key", 100);  // Autoboxes

// 3. Method calls
void process(Integer n) {}
process(42);  // Autoboxes

// 4. Return values
Integer getValue() { return 10; }  // Autoboxes
```

### Best Practices

```java
// 1. Use primitives by default
int count = 0;  // Not Integer

// 2. Use wrappers when:
// - Need null (e.g., database NULL)
Integer age = null;  // Valid

// - Working with collections
List<Integer> list = new ArrayList<>();

// - Working with generics
Map<String, Integer> map;

// 3. Never use == for wrapper comparison
Integer a = 1000;
Integer b = 1000;
if (a.equals(b)) {}  // ✅ Correct
if (a == b) {}       // ❌ Wrong (compares references)

// 4. Watch for null unboxing
Integer value = getValue();  // Might return null
int primitive = (value != null) ? value : 0;  // Safe
```

---

## Quick Reference

```java
// Pass by Value: Always!
// - Primitives: Copy of value
// - Objects: Copy of reference value

// Interface vs Class:
List<T> list = new ArrayList<>();  // ✅ Recommended
ArrayList<T> list = new ArrayList<>();  // ⚠️ Only for specific methods

// String:
// - Immutable, uses string pool
// - Use StringBuilder for mutable operations

// equals() vs ==:
// - == : Reference comparison (or value for primitives)
// - equals() : Content comparison (when overridden)

// final/finally/finalize:
// - final : Make immutable
// - finally : Always execute
// - finalize() : ❌ Deprecated

// Static vs Instance:
// - Static : Class-level, shared
// - Instance : Object-level, per-instance

// Autoboxing:
// - Automatic primitive ↔ wrapper conversion
// - Watch for null, ==, and performance
```
