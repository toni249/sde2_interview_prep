# Top 20 Core Java Interview Questions with In-Depth Explanations

## Table of Contents
1. [What is the difference between JDK, JRE, and JVM?](#1-what-is-the-difference-between-jdk-jre-and-jvm)
2. [Explain JVM Architecture and How Garbage Collection Works](#2-explain-jvm-architecture-and-how-garbage-collection-works)
3. [Difference between Stack and Heap Memory](#3-difference-between-stack-and-heap-memory)
4. [What are the main principles of OOP?](#4-what-are-main-principles-of-oop)
5. [Difference between Method Overloading and Method Overriding](#5-difference-between-method-overloading-and-method-overriding)
6. [What is the difference between Abstract Class and Interface?](#6-what-is-the-difference-between-abstract-class-and-interface)
7. [Explain String Pool and String Immutability](#7-explain-string-pool-and-string-immutability)
8. [What are Wrapper Classes and Autoboxing/Unboxing?](#8-what-are-wrapper-classes-and-autoboxingunboxing)
9. [Explain Collection Framework Hierarchy](#9-explain-collection-framework-hierarchy)
10. [Difference between ArrayList and LinkedList](#10-difference-between-arraylist-and-linkedlist)
11. [Difference between HashSet, LinkedHashSet, and TreeSet](#11-difference-between-hashset-linkedhashset-and-treeset)
12. [Difference between HashMap, LinkedHashMap, and TreeMap](#12-difference-between-hashmap-linkedhashmap-and-treemap)
13. [What is Thread and what are different ways to create it?](#13-what-is-thread-and-different-ways-to-create-it)
14. [Difference between String, StringBuffer, and StringBuilder](#14-difference-between-string-stringbuffer-and-stringbuilder)
15. [Explain Exception Handling and Hierarchy](#15-explain-exception-handling-and-hierarchy)
16. [Difference between final, finally, and finalize](#16-difference-between-final-finally-and-finalize)
17. [What is the difference between == and equals()](#17-what-is-the-difference-between--and-equals)
18. [Explain Generics in Java](#18-explain-generics-in-java)
19. [What are Lambdas and Streams?](#19-what-are-lambdas-and-streams)
20. [Explain Java Memory Model and Concurrency](#20-explain-java-memory-model-and-concurrency)

---

## 1. What is the difference between JDK, JRE, and JVM?

**JDK (Java Development Kit)**
- Complete development environment
- Includes: JRE + Development tools (javac, jdb, javap, jar, etc.)
- Used to develop and run Java applications
- Contains compilers and debuggers
- Size: ~200-250 MB

**JRE (Java Runtime Environment)**
- Environment to run Java applications
- Includes: JVM + Class libraries + Java class loader
- Cannot compile Java source code
- Size: ~40-60 MB

**JVM (Java Virtual Machine)**
- Abstract machine that executes Java bytecode
- Part of JRE
- Converts bytecode to machine code
- Platform-specific implementation
- Provides garbage collection, memory management

**Relationship:**
```
Application → JRE → JVM → OS
         ↓
      JDK (for development)
```

**Example:**
```java
// To compile (need JDK)
javac Hello.java

// To run (need JRE/JVM)
java Hello
```

---

## 2. Explain JVM Architecture and How Garbage Collection Works

### JVM Architecture Components:

**1. Class Loader Subsystem**
- Bootstrap ClassLoader: Loads core Java classes
- Extension ClassLoader: Loads extensions
- Application ClassLoader: Loads application classes

**2. Runtime Data Areas**

**a) Method Area (Shared)**
- Stores class metadata, static variables
- Only one per JVM

**b) Heap Area (Shared)**
- Stores objects and arrays
- Divided into Young Generation (Eden, S0, S1) and Old Generation
- Garbage collected area

**c) Stack Area (Per Thread)**
- Stores local variables, method calls
- Each thread has its own stack
- Frame created for each method call

**d) PC Registers (Per Thread)**
- Stores address of next instruction

**e) Native Method Area (Per Thread)**
- For native methods

**3. Execution Engine**
- Interpreter: Executes bytecode line by line
- JIT Compiler: Compiles frequently used code to native
- Garbage Collector: Manages heap memory

### Garbage Collection Process:

```java
// Example demonstrating GC
public class GCDemo {
    public static void main(String[] args) {
        // Object created in Young Generation (Eden space)
        String s1 = new String("Hello");
        
        // After minor GC, if not collected, moved to Survivor space
        // After major GC, if not collected, moved to Old Generation
        
        s1 = null; // Object becomes eligible for garbage collection
        System.gc(); // Request GC (not guaranteed)
    }
}
```

**Types of GC:**
- **Serial GC:** Single threaded, suitable for small apps
- **Parallel GC:** Multiple threads, default in Java 8
- **G1 GC:** For large heap (>4GB), low latency
- **ZGC/Shenandoah:** Ultra-low latency (Java 11+)

---

## 3. Difference between Stack and Heap Memory

| Stack | Heap |
|-------|------|
| Stores local variables and references | Stores actual objects |
| Fast access | Slower access |
| Limited size (smaller) | Larger size |
| Fixed size per thread | Shared across all threads |
| Last In First Out (LIFO) | Managed by GC |
| Automatically managed | Garbage collected |
| Thread-safe (one per thread) | Not thread-safe (shared) |

**Example:**

```java
public class StackHeapDemo {
    // Static variable - stored in Method Area
    static int staticVar = 10;
    
    public static void main(String[] args) {
        // Local primitive - stored in Stack
        int localVar = 20;
        
        // Local reference - stored in Stack
        // Object itself stored in Heap
        String str = new String("Hello"); // str in stack, "Hello" object in heap
        
        // Pass by value (reference copied)
        modifyString(str);
        System.out.println(str); // "Hello" (unchanged due to immutability)
        
        // Pass by value (primitive)
        modifyInt(localVar);
        System.out.println(localVar); // 20 (unchanged)
        
        // Heap object
        MyClass obj = new MyClass();
        modifyObject(obj);
        System.out.println(obj.value); // 100 (changed)
    }
    
    static void modifyString(String s) {
        s = "World"; // Doesn't affect original
    }
    
    static void modifyInt(int i) {
        i = 100; // Doesn't affect original
    }
    
    static void modifyObject(MyClass o) {
        o.value = 100; // Changes original object
    }
}

class MyClass {
    int value = 10;
}
```

**Memory Layout:**
```
┌─────────────────┐
│  Stack Memory   │ ← Each thread has own stack
│  - local vars   │
│  - references   │
└─────────────────┘
        ↓
┌─────────────────┐
│  Heap Memory      │ ← Shared across threads
│  - Objects        │
│  - Arrays         │
└─────────────────┘
```

---

## 4. What are Main Principles of OOP?

### 1. **Encapsulation**
- Binding data and methods together
- Data hiding using access modifiers
- Protection from unauthorized access

```java
public class BankAccount {
    // Private data - hidden from outside
    private double balance;
    
    // Public methods - controlled access
    public void deposit(double amount) {
        if (amount > 0) {
            balance += amount;
        }
    }
    
    public double getBalance() {
        return balance;
    }
    
    // Data validation and security
}
```

### 2. **Inheritance**
- Acquiring properties from parent class
- Code reusability
- IS-A relationship

```java
// Base class
class Animal {
    void eat() {
        System.out.println("Eating...");
    }
}

// Derived class
class Dog extends Animal {
    void bark() {
        System.out.println("Barking...");
    }
}
// Dog IS-A Animal
```

### 3. **Polymorphism**
- One interface, multiple implementations
- Method overloading (compile-time)
- Method overriding (runtime)

```java
// Runtime Polymorphism
class Shape {
    void draw() {
        System.out.println("Drawing shape");
    }
}

class Circle extends Shape {
    @Override
    void draw() {
        System.out.println("Drawing circle");
    }
}

// Method Resolution at runtime
Shape s = new Circle();
s.draw(); // Output: "Drawing circle" (depends on object)
```

### 4. **Abstraction**
- Hiding implementation details
- Showing only essential features

```java
// Abstract class
abstract class Vehicle {
    abstract void start(); // Abstract method
    
    void display() { // Concrete method
        System.out.println("This is a vehicle");
    }
}

class Car extends Vehicle {
    @Override
    void start() {
        System.out.println("Car started with key");
    }
}
```

---

## 5. Difference between Method Overloading and Method Overriding

| Feature | Overloading | Overriding |
|---------|------------|------------|
| Definition | Same method name, different parameters | Same method signature in child class |
| Resolution | Compile-time | Runtime |
| Return type | Can be different | Must be same/subtype |
| Access modifier | Can be different | Must be same or more accessible |
| Exceptions | Can be different | Child can't throw broader exception |
| Inheritance | Not required | Required |
| Location | Same class | Parent and child classes |

### Method Overloading Examples:

```java
class Calculator {
    // Different number of parameters
    int add(int a, int b) {
        return a + b;
    }
    
    int add(int a, int b, int c) {
        return a + b + c;
    }
    
    // Different types
    double add(double a, double b) {
        return a + b;
    }
    
    // Different order
    void display(String name, int age) { }
    void display(int age, String name) { }
}
```

### Method Overriding Examples:

```java
class Parent {
    void show() {
        System.out.println("Parent show");
    }
    
    int getValue() {
        return 10;
    }
}

class Child extends Parent {
    @Override
    void show() {
        System.out.println("Child show");
    }
    
    // Covariant return type
    @Override
    Integer getValue() { // Wrapper type
        return 20;
    }
}

// Runtime polymorphism
Parent p = new Child();
p.show(); // Output: "Child show"
```

**Rules for Overriding:**
- Method signature must match
- Return type must be same or subtype (covariant)
- Access modifier must be same or more visible
- Cannot reduce visibility (public → private not allowed)
- Cannot override static, final, or private methods

---

## 6. What is the difference between Abstract Class and Interface?

| Feature | Abstract Class | Interface (Java 8+) |
|---------|---------------|---------------------|
| Keyword | `abstract` | `interface` |
| Variables | Any type | Public static final (constants) |
| Methods | Abstract and concrete | Abstract, default, static |
| Inheritance | Single | Multiple |
| Constructor | Yes | No |
| Access modifiers | Any | Public (implicitly) |
| Main method | Yes | Yes (since Java 8) |
| Use when | IS-A relationship | Contract/HAS-A behavior |

### Abstract Class Example:

```java
abstract class Animal {
    // Concrete method
    void sleep() {
        System.out.println("Sleeping...");
    }
    
    // Abstract method
    abstract void makeSound();
    
    // Constructor
    public Animal() {
        System.out.println("Animal created");
    }
}

class Dog extends Animal {
    @Override
    void makeSound() {
        System.out.println("Woof!");
    }
}
```

### Interface Examples (Java 8+):

```java
// Interface with multiple method types
interface Drawable {
    // Constant
    int MAX_SIZE = 100;
    
    // Abstract method
    void draw();
    
    // Default method (Java 8+)
    default void resize() {
        System.out.println("Resizing...");
    }
    
    // Static method (Java 8+)
    static void display() {
        System.out.println("Displaying...");
    }
}

// Multiple interfaces
interface Printable {
    void print();
}

class Circle implements Drawable, Printable {
    @Override
    public void draw() {
        System.out.println("Drawing circle");
    }
    
    @Override
    public void print() {
        System.out.println("Printing circle");
    }
}

// Using default method
Circle c = new Circle();
c.draw();    // Abstract method
c.resize();  // Default method
Drawable.display(); // Static method
```

### When to Use What?

**Use Abstract Class when:**
- Sharing code between related classes
- Want to enforce specific behaviors
- Need non-static, non-final variables
- Need constructors
- IS-A relationship

**Use Interface when:**
- Multiple unrelated classes need same behavior
- Need multiple inheritance
- Defining contracts/APIs
- HAS-A behavior (can-do relationship)

---

## 7. Explain String Pool and String Immutability

### String Immutability

String objects are immutable - once created, cannot be changed.

```java
String str = "Hello";
str = str + " World"; // Creates new String object
// Original "Hello" becomes eligible for GC
```

### String Pool

Special memory area in heap for storing string literals to save memory.

```java
// Both s1 and s2 point to same object in String Pool
String s1 = "Hello";
String s2 = "Hello";
System.out.println(s1 == s2); // true (same reference)

// New String object created in heap
String s3 = new String("Hello");
System.out.println(s1 == s3); // false (different references)
System.out.println(s1.equals(s3)); // true (same content)
```

### Important Points:

```java
public class StringDemo {
    public static void main(String[] args) {
        // String Pool
        String s1 = "Hello";
        String s2 = "Hello";
        System.out.println(s1 == s2); // true
        
        // String Pool (intern)
        String s3 = new String("Hello").intern();
        System.out.println(s1 == s3); // true
        
        // New object in heap
        String s4 = new String("Hello");
        System.out.println(s1 == s4); // false
        
        // Concatenation creates new object
        String s5 = "Hel" + "lo"; // Compiler optimization: "Hello"
        System.out.println(s1 == s5); // true
        
        // Runtime concatenation
        String s6 = "Hel";
        String s7 = s6 + "lo"; // New object
        System.out.println(s1 == s7); // false
        
        // Immutability proof
        String original = "Hello";
        String modified = original.concat(" World");
        System.out.println(original); // "Hello" (unchanged)
        System.out.println(modified); // "Hello World" (new object)
    }
}
```

**Benefits of Immutability:**
- Thread-safe
- Cache-friendly (String Pool)
- Security in network operations
- Prevents accidental modifications
- Safe for use as keys in HashMap

---

## 8. What are Wrapper Classes and Autoboxing/Unboxing?

### Wrapper Classes

Wrapper classes convert primitives to objects (required for Collections).

```java
Primitive → Wrapper
int       → Integer
char      → Character
byte      → Byte
short     → Short
long      → Long
float     → Float
double    → Double
boolean   → Boolean
```

### Autoboxing and Unboxing

```java
// Autoboxing: Primitive → Wrapper
int primitiveInt = 10;
Integer wrapperInt = primitiveInt; // Autoboxing
// Equivalent to: Integer wrapperInt = Integer.valueOf(primitiveInt);

// Unboxing: Wrapper → Primitive
Integer wrapperInt = new Integer(10);
int primitiveInt = wrapperInt; // Unboxing
// Equivalent to: int primitiveInt = wrapperInt.intValue();
```

### Real Examples:

```java
import java.util.ArrayList;

public class WrapperDemo {
    public static void main(String[] args) {
        // Collections only work with objects
        ArrayList<Integer> list = new ArrayList<>();
        list.add(5); // Autoboxing: int → Integer
        
        int value = list.get(0); // Unboxing: Integer → int
        
        // Behind the scenes
        list.add(Integer.valueOf(10)); // Manual boxing
        int v = list.get(1).intValue(); // Manual unboxing
        
        // Wrapper class methods
        Integer num = Integer.parseInt("123"); // String to int
        String str = Integer.toString(123); // int to String
        System.out.println(Integer.MAX_VALUE);
        
        // Comparison
        Integer a = 100;
        Integer b = 100;
        System.out.println(a == b); // true (cached -128 to 127)
        
        Integer c = 200;
        Integer d = 200;
        System.out.println(c == d); // false (not cached)
        System.out.println(c.equals(d)); // true (equals comparison)
    }
}
```

**Important Notes:**
- Primitive types stored in stack, faster
- Wrapper objects stored in heap, slower
- Use primitives for performance-critical code
- Use wrappers when objects required (Collections, generics)

---

## 9. Explain Collection Framework Hierarchy

```
Collection (Interface)
├── List (Interface)
│   ├── ArrayList (Implementation)
│   ├── LinkedList (Implementation)
│   └── Vector (Legacy - Thread-safe)
│       └── Stack
│
├── Set (Interface)
│   ├── HashSet (Implementation - Unordered)
│   │   └── LinkedHashSet (Ordered)
│   ├── TreeSet (Implementation - Sorted)
│   └── EnumSet
│
└── Queue (Interface)
    ├── PriorityQueue
    └── Deque (Interface)
        ├── ArrayDeque
        └── LinkedList

Map (Separate hierarchy)
├── HashMap (Unordered)
│   └── LinkedHashMap (Ordered)
├── TreeMap (Sorted)
├── Hashtable (Legacy - Thread-safe)
└── ConcurrentHashMap
```

### Key Interfaces:

**Collection (Root interface)**
```java
List<E>       // Ordered, duplicates allowed
Set<E>        // Unordered, no duplicates
Queue<E>      // FIFO
Deque<E>      // Double-ended queue
Map<K, V>     // Key-value pairs (separate hierarchy)
```

### Code Example:

```java
import java.util.*;

public class CollectionHierarchy {
    public static void main(String[] args) {
        // List - Ordered, allows duplicates
        List<String> list = new ArrayList<>();
        list.add("Apple");
        list.add("Banana");
        list.add("Apple"); // Duplicate allowed
        System.out.println("List: " + list); // [Apple, Banana, Apple]
        
        // Set - No duplicates
        Set<String> set = new HashSet<>();
        set.add("Apple");
        set.add("Banana");
        set.add("Apple"); // Duplicate ignored
        System.out.println("Set: " + set); // [Apple, Banana]
        
        // Queue - FIFO
        Queue<String> queue = new LinkedList<>();
        queue.offer("First");
        queue.offer("Second");
        System.out.println("Queue: " + queue.poll()); // First
        
        // Map - Key-value pairs
        Map<String, Integer> map = new HashMap<>();
        map.put("Apple", 1);
        map.put("Banana", 2);
        System.out.println("Map: " + map); // {Apple=1, Banana=2}
    }
}
```

---

## 10. Difference between ArrayList and LinkedList

| Feature | ArrayList | LinkedList |
|---------|-----------|------------|
| Underlying data | Dynamic array | Doubly linked list |
| Access | O(1) - Fast | O(n) - Slow |
| Insert/Delete | O(n) - Slow | O(1) - Fast |
| Memory | Contiguous | Extra pointers overhead |
| Search | O(1) by index | O(n) traversal |
| Best for | Frequent access | Frequent insert/delete |

### Performance Comparison:

```java
import java.util.*;

public class ListComparison {
    public static void main(String[] args) {
        int size = 100000;
        
        // Create lists
        List<Integer> arrayList = new ArrayList<>();
        List<Integer> linkedList = new LinkedList<>();
        
        // Add at end - both O(1) amortized
        long start = System.currentTimeMillis();
        for (int i = 0; i < size; i++) {
            arrayList.add(i);
        }
        System.out.println("ArrayList add end: " + (System.currentTimeMillis() - start));
        
        start = System.currentTimeMillis();
        for (int i = 0; i < size; i++) {
            linkedList.add(i);
        }
        System.out.println("LinkedList add end: " + (System.currentTimeMillis() - start));
        
        // Access by index
        start = System.currentTimeMillis();
        for (int i = 0; i < size; i++) {
            arrayList.get(i);
        }
        System.out.println("ArrayList get: " + (System.currentTimeMillis() - start));
        
        start = System.currentTimeMillis();
        for (int i = 0; i < size; i++) {
            linkedList.get(i);
        }
        System.out.println("LinkedList get: " + (System.currentTimeMillis() - start));
        
        // Insert at beginning
        start = System.currentTimeMillis();
        for (int i = 0; i < 1000; i++) {
            arrayList.add(0, i); // Shifts all elements
        }
        System.out.println("ArrayList insert begin: " + (System.currentTimeMillis() - start));
        
        start = System.currentTimeMillis();
        for (int i = 0; i < 1000; i++) {
            linkedList.add(0, i); // Just changes pointers
        }
        System.out.println("LinkedList insert begin: " + (System.currentTimeMillis() - start));
    }
}
```

**When to use:**
- **ArrayList:** When you need frequent random access
- **LinkedList:** When you need frequent insertions/deletions in middle

---

## 11. Difference between HashSet, LinkedHashSet, and TreeSet

| Feature | HashSet | LinkedHashSet | TreeSet |
|---------|---------|---------------|---------|
| Order | No order | Insertion order | Sorted order |
| Null | One null allowed | One null allowed | No null |
| Performance | O(1) add/remove | O(1) add/remove | O(log n) add/remove |
| Implementation | HashMap | LinkedHashMap | TreeMap (Red-Black Tree) |
| Use when | No order needed | Order matters | Sorted needed |

```java
import java.util.*;

public class SetComparison {
    public static void main(String[] args) {
        // HashSet - No order
        Set<String> hashSet = new HashSet<>();
        hashSet.add("Apple");
        hashSet.add("Banana");
        hashSet.add("Cherry");
        hashSet.add("Date");
        System.out.println("HashSet: " + hashSet); // No guarantee of order
        
        // LinkedHashSet - Insertion order preserved
        Set<String> linkedHashSet = new LinkedHashSet<>();
        linkedHashSet.add("Apple");
        linkedHashSet.add("Banana");
        linkedHashSet.add("Cherry");
        linkedHashSet.add("Date");
        System.out.println("LinkedHashSet: " + linkedHashSet); // [Apple, Banana, Cherry, Date]
        
        // TreeSet - Sorted order
        Set<String> treeSet = new TreeSet<>();
        treeSet.add("Apple");
        treeSet.add("Banana");
        treeSet.add("Cherry");
        treeSet.add("Date");
        System.out.println("TreeSet: " + treeSet); // [Apple, Banana, Cherry, Date] - alphabetical
        
        // Custom comparator
        Set<Integer> treeSetDesc = new TreeSet<>(Collections.reverseOrder());
        treeSetDesc.add(5);
        treeSetDesc.add(2);
        treeSetDesc.add(8);
        treeSetDesc.add(1);
        System.out.println("TreeSet Desc: " + treeSetDesc); // [8, 5, 2, 1]
    }
}
```

---

## 12. Difference between HashMap, LinkedHashMap, and TreeMap

| Feature | HashMap | LinkedHashMap | TreeMap |
|---------|----------|---------------|---------|
| Order | No order | Insertion order | Sorted order |
| Null | One null key | One null key | No null key |
| Performance | O(1) average | O(1) average | O(log n) |
| Implementation | Hash table | Hash table + LinkedList | Red-Black Tree |
| Thread-safe | No | No | No |

```java
import java.util.*;

public class MapComparison {
    public static void main(String[] args) {
        // HashMap - No order
        Map<String, Integer> hashMap = new HashMap<>();
        hashMap.put("Zebra", 26);
        hashMap.put("Apple", 1);
        hashMap.put("Banana", 2);
        System.out.println("HashMap: " + hashMap); // No guaranteed order
        
        // LinkedHashMap - Insertion order
        Map<String, Integer> linkedHashMap = new LinkedHashMap<>();
        linkedHashMap.put("Apple", 1);
        linkedHashMap.put("Banana", 2);
        linkedHashMap.put("Zebra", 26);
        System.out.println("LinkedHashMap: " + linkedHashMap); // {Apple=1, Banana=2, Zebra=26}
        
        // TreeMap - Sorted by keys
        Map<String, Integer> treeMap = new TreeMap<>();
        treeMap.put("Zebra", 26);
        treeMap.put("Apple", 1);
        treeMap.put("Banana", 2);
        System.out.println("TreeMap: " + treeMap); // {Apple=1, Banana=2, Zebra=26} - sorted
        
        // Custom comparator
        Map<String, Integer> reverseTreeMap = new TreeMap<>(Collections.reverseOrder());
        reverseTreeMap.put("Zebra", 26);
        reverseTreeMap.put("Apple", 1);
        reverseTreeMap.put("Banana", 2);
        System.out.println("Reverse TreeMap: " + reverseTreeMap); // {Zebra=26, Banana=2, Apple=1}
        
        // Access order LinkedHashMap
        Map<String, Integer> accessOrderMap = new LinkedHashMap<>(16, 0.75f, true);
        accessOrderMap.put("A", 1);
        accessOrderMap.put("B", 2);
        accessOrderMap.put("C", 3);
        accessOrderMap.get("A"); // Accessing A moves it to end
        System.out.println("Access Ordered: " + accessOrderMap); // {B=2, C=3, A=1}
    }
}
```

---

## 13. What is Thread and Different Ways to Create it?

A thread is a lightweight process that allows concurrent execution of code.

### Three Ways to Create Threads:

### 1. Extending Thread Class

```java
class MyThread extends Thread {
    @Override
    public void run() {
        for (int i = 0; i < 5; i++) {
            System.out.println("Thread 1: " + i);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

public class ThreadDemo {
    public static void main(String[] args) {
        MyThread t1 = new MyThread();
        t1.start(); // Starts new thread
    }
}
```

### 2. Implementing Runnable Interface

```java
class MyRunnable implements Runnable {
    @Override
    public void run() {
        for (int i = 0; i < 5; i++) {
            System.out.println("Thread 2: " + i);
        }
    }
}

public class RunnableDemo {
    public static void main(String[] args) {
        MyRunnable runnable = new MyRunnable();
        Thread t2 = new Thread(runnable);
        t2.start();
        
        // Or using Lambda (Java 8+)
        Thread t3 = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                System.out.println("Thread 3: " + i);
            }
        });
        t3.start();
    }
}
```

### 3. Implementing Callable Interface (Returns value)

```java
import java.util.concurrent.*;

class MyCallable implements Callable<String> {
    @Override
    public String call() throws Exception {
        Thread.sleep(2000);
        return "Task completed";
    }
}

public class CallableDemo {
    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(3);
        Future<String> future = executor.submit(new MyCallable());
        
        System.out.println("Waiting for result...");
        String result = future.get(); // Blocks until result available
        System.out.println("Result: " + result);
        
        executor.shutdown();
    }
}
```

### Thread States:

```java
NEW          - Thread created, not started
RUNNABLE     - Thread is executing
BLOCKED      - Waiting for monitor lock
WAITING      - Waiting indefinitely
TIMED_WAITING - Waiting with timeout
TERMINATED   - Thread finished
```

### Complete Example:

```java
public class ThreadDemo {
    public static void main(String[] args) {
        // Method 1: Extend Thread
        Thread t1 = new Thread(() -> System.out.println("Thread 1 running"));
        
        // Method 2: Implement Runnable
        Runnable task = () -> System.out.println("Thread 2 running");
        Thread t2 = new Thread(task);
        
        // Method 3: Using ExecutorService (Best practice)
        ExecutorService executor = Executors.newFixedThreadPool(5);
        executor.submit(() -> System.out.println("Thread 3 running"));
        
        t1.start();
        t2.start();
        executor.shutdown();
    }
}
```

---

## 14. Difference between String, StringBuffer, and StringBuilder

| Feature | String | StringBuffer | StringBuilder |
|---------|--------|--------------|---------------|
| Mutability | Immutable | Mutable | Mutable |
| Thread-safe | Yes (immutable) | Yes (synchronized) | No (not synchronized) |
| Performance | Slow for concatenation | Medium | Fast |
| Storage | String Pool | Heap | Heap |
| Use when | Immutability needed | Thread-safe changes | Single-threaded changes |

### Examples:

```java
public class StringComparison {
    public static void main(String[] args) {
        // String - Immutable
        String str = "Hello";
        str = str + " World"; // Creates new object
        System.out.println(str); // "Hello World"
        
        // StringBuffer - Mutable, Thread-safe
        StringBuffer sb = new StringBuffer("Hello");
        sb.append(" World"); // Modifies same object
        System.out.println(sb); // "Hello World"
        
        // StringBuilder - Mutable, NOT Thread-safe
        StringBuilder strb = new StringBuilder("Hello");
        strb.append(" World"); // Modifies same object
        System.out.println(strb); // "Hello World"
        
        // Performance comparison
        int iterations = 100000;
        
        // String concatenation
        long start = System.currentTimeMillis();
        String result = "";
        for (int i = 0; i < iterations; i++) {
            result += "a"; // Creates new object each time
        }
        System.out.println("String: " + (System.currentTimeMillis() - start));
        
        // StringBuffer
        start = System.currentTimeMillis();
        StringBuffer sb2 = new StringBuffer();
        for (int i = 0; i < iterations; i++) {
            sb2.append("a");
        }
        System.out.println("StringBuffer: " + (System.currentTimeMillis() - start));
        
        // StringBuilder
        start = System.currentTimeMillis();
        StringBuilder strb2 = new StringBuilder();
        for (int i = 0; i < iterations; i++) {
            strb2.append("a");
        }
        System.out.println("StringBuilder: " + (System.currentTimeMillis() - start));
    }
}
```

**When to use:**
- **String:** When immutability is required, small concatenations
- **StringBuffer:** Multi-threaded environment with string modifications
- **StringBuilder:** Single-threaded environment, frequent string modifications

---

## 15. Explain Exception Handling and Hierarchy

```
Throwable (Top level)
├── Error (Unchecked)
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── VirtualMachineError
│
└── Exception (Parent)
    ├── RuntimeException (Unchecked)
    │   ├── NullPointerException
    │   ├── ArrayIndexOutOfBoundsException
    │   ├── IllegalArgumentException
    │   └── ArithmeticException
    │
    └── Checked Exception
        ├── IOException
        ├── SQLException
        ├── FileNotFoundException
        └── ClassNotFoundException
```

### Types of Exceptions:

**1. Checked Exceptions** (Compile-time)
- Must be handled or declared
- Extend Exception (not RuntimeException)

**2. Unchecked Exceptions** (Runtime)
- RuntimeException and its subclasses
- Don't need to be declared

**3. Errors**
- Serious problems beyond control
- Should not be caught typically

### Exception Handling Example:

```java
import java.io.*;

public class ExceptionHandling {
    public static void main(String[] args) {
        // Checked exception handling
        try {
            readFile("nonexistent.txt");
        } catch (FileNotFoundException e) {
            System.out.println("File not found: " + e.getMessage());
        } catch (IOException e) {
            System.out.println("IO Error: " + e.getMessage());
        } finally {
            System.out.println("Finally block always executes");
        }
        
        // Unchecked exception
        try {
            int result = divide(10, 0);
        } catch (ArithmeticException e) {
            System.out.println("Cannot divide by zero: " + e.getMessage());
        }
        
        // Custom exception
        try {
            validateAge(15);
        } catch (InvalidAgeException e) {
            System.out.println(e.getMessage());
        }
    }
    
    static void readFile(String filename) throws IOException {
        // Checked exception must be declared
    }
    
    static int divide(int a, int b) {
        return a / b; // May throw ArithmeticException
    }
    
    static void validateAge(int age) throws InvalidAgeException {
        if (age < 18) {
            throw new InvalidAgeException("Age must be 18 or above");
        }
    }
}

// Custom exception
class InvalidAgeException extends Exception {
    public InvalidAgeException(String message) {
        super(message);
    }
}
```

### Best Practices:

```java
// 1. Specific exceptions first
try {
    // code
} catch (FileNotFoundException e) {
    // specific
} catch (IOException e) {
    // general
}

// 2. Try-with-resources (Java 7+)
try (BufferedReader br = new BufferedReader(new FileReader("file.txt"))) {
    // resource automatically closed
} catch (IOException e) {
    e.printStackTrace();
}

// 3. Multiple catch blocks (Java 7+)
try {
    // code
} catch (FileNotFoundException | IOException e) {
    // handle both
}

// 4. Proper exception handling
try {
    riskyMethod();
} catch (SpecificException e) {
    log.error("Error occurred", e);
    // Don't swallow, handle properly
}
```

---

## 16. Difference between final, finally, and finalize

### 1. **final** (Keyword)

Used to restrict:
- Variables: cannot be reassigned
- Methods: cannot be overridden
- Classes: cannot be extended

```java
public class FinalDemo {
    // Final variable - constant
    final int x = 10;
    
    // Final method - cannot be overridden
    final void display() {
        System.out.println("Cannot override");
    }
}

final class MyClass { // Cannot be extended
    final int MAX = 100; // Constant
}

// Blank final
class Example {
    final int value;
    
    Example(int value) {
        this.value = value; // Must be initialized in constructor
    }
}
```

### 2. **finally** (Block)

Executes regardless of exception occurs or not.

```java
public class FinallyDemo {
    public static void main(String[] args) {
        try {
            int result = 10 / 0;
        } catch (ArithmeticException e) {
            System.out.println("Exception caught");
        } finally {
            System.out.println("Finally block always executes");
        }
        
        // Even with return statement
        System.out.println(test()); // Output: 10
    }
    
    static int test() {
        try {
            return 5;
        } finally {
            return 10; // Finally return overrides try return
        }
    }
}
```

### 3. **finalize()** (Method - Deprecated in Java 9)

Cleanup method called by garbage collector before object is destroyed.

```java
public class FinalizeDemo {
    public static void main(String[] args) {
        FinalizeDemo obj = new FinalizeDemo();
        obj = null; // Eligible for GC
        System.gc(); // Request garbage collection
        System.out.println("Main method");
    }
    
    @Override
    protected void finalize() throws Throwable {
        System.out.println("Finalize method called");
        super.finalize();
    }
}

// ⚠️ Note: finalize() is deprecated in Java 9+
// Use Cleaner API or try-with-resources instead
```

### Summary Table:

| Keyword/Method | Purpose | When Used |
|---------------|---------|-----------|
| final | Make variable/method/class constant | When value/method/class shouldn't change |
| finally | Cleanup code | Always in try-catch blocks |
| finalize | GC cleanup | Before object destruction (deprecated) |

**Note:** Don't rely on finalize(). Use try-with-resources or Cleaner API instead.

---

## 17. What is the difference between == and equals()?

### == Operator

Compares references (addresses) of objects in memory.

```java
public class EqualityComparison {
    public static void main(String[] args) {
        // Primitive comparison
        int a = 10;
        int b = 10;
        System.out.println(a == b); // true (compares values)
        
        // Object comparison
        String s1 = "Hello";
        String s2 = "Hello";
        System.out.println(s1 == s2); // true (same object in String Pool)
        
        String s3 = new String("Hello");
        System.out.println(s1 == s3); // false (different objects)
        
        // Reference comparison
        Object obj1 = new Object();
        Object obj2 = new Object();
        System.out.println(obj1 == obj2); // false (different references)
        
        // Same object
        Object obj3 = obj1;
        System.out.println(obj1 == obj3); // true (same reference)
    }
}
```

### equals() Method

Compares content/values of objects.

```java
public class EqualsDemo {
    public static void main(String[] args) {
        String s1 = "Hello";
        String s2 = new String("Hello");
        
        System.out.println(s1 == s2); // false (different references)
        System.out.println(s1.equals(s2)); // true (same content)
        
        // Custom class
        Person p1 = new Person("John", 25);
        Person p2 = new Person("John", 25);
        
        System.out.println(p1 == p2); // false
        System.out.println(p1.equals(p2)); // true (if equals overridden)
    }
}

class Person {
    String name;
    int age;
    
    Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        
        Person person = (Person) obj;
        return age == person.age && name.equals(person.name);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }
}
```

### Comparison Table:

| Operator/Method | Compares | Usage |
|----------------|----------|-------|
| == | References/addresses | Object references, primitive values |
| equals() | Content/values | Object content comparison |
| hashCode() | Hash code | Used in HashMap, HashSet |

**Important Rules:**
1. If `equals()` is overridden, `hashCode()` must also be overridden
2. If `a.equals(b)` is true, then `a.hashCode() == b.hashCode()` must be true
3. Vice versa is not necessary (hash collision)

---

## 18. Explain Generics in Java

Generics provide type safety and eliminate type casting.

### Benefits:
- Type safety
- No type casting
- Compile-time type checking
- Code reusability

### Basic Syntax:

```java
// Generic class
class Box<T> {
    private T item;
    
    public void setItem(T item) {
        this.item = item;
    }
    
    public T getItem() {
        return item;
    }
}

// Usage
Box<String> stringBox = new Box<>();
stringBox.setItem("Hello");
String value = stringBox.getItem(); // No casting needed

Box<Integer> intBox = new Box<>();
intBox.setItem(42);
int number = intBox.getItem(); // Type safe
```

### Generic Methods:

```java
public class GenericMethods {
    // Generic method
    public static <T> void printArray(T[] array) {
        for (T element : array) {
            System.out.println(element);
        }
    }
    
    // Bounded generic
    public static <T extends Number> double sum(T num1, T num2) {
        return num1.doubleValue() + num2.doubleValue();
    }
    
    public static void main(String[] args) {
        Integer[] intArray = {1, 2, 3};
        String[] strArray = {"A", "B", "C"};
        
        printArray(intArray);
        printArray(strArray);
        
        System.out.println(sum(10, 20));
        System.out.println(sum(10.5, 20.5));
    }
}
```

### Bounded Type Parameters:

```java
// Upper bound
class NumberBox<T extends Number> {
    private T number;
    // T must be Number or subclass
}

// Multiple bounds
interface Readable { }
interface Closeable { }
class Reader<T extends Readable & Closeable> { }
```

### Wildcards:

```java
import java.util.*;

public class Wildcards {
    // Unbounded wildcard
    public static void printList(List<?> list) {
        for (Object obj : list) {
            System.out.println(obj);
        }
    }
    
    // Upper bounded wildcard
    public static double sumList(List<? extends Number> list) {
        double sum = 0;
        for (Number num : list) {
            sum += num.doubleValue();
        }
        return sum;
    }
    
    // Lower bounded wildcard
    public static void addNumbers(List<? super Integer> list) {
        list.add(1);
        list.add(2);
    }
    
    public static void main(String[] args) {
        List<Double> doubles = Arrays.asList(1.0, 2.0, 3.0);
        System.out.println(sumList(doubles)); // 6.0
        
        List<Object> objects = new ArrayList<>();
        addNumbers(objects);
    }
}
```

### Real-world Example:

```java
// Generic Repository pattern
interface Repository<T, ID> {
    T findById(ID id);
    List<T> findAll();
    void save(T entity);
    void delete(ID id);
}

class UserRepository implements Repository<User, Long> {
    @Override
    public User findById(Long id) {
        // implementation
        return null;
    }
    
    @Override
    public List<User> findAll() {
        // implementation
        return null;
    }
    
    @Override
    public void save(User entity) {
        // implementation
    }
    
    @Override
    public void delete(Long id) {
        // implementation
    }
}
```

---

## 19. What are Lambdas and Streams?

### Lambda Expressions (Java 8+)

Lambda provides a clear and concise way to implement functional interfaces.

```java
@FunctionalInterface
interface Operation {
    int calculate(int a, int b);
}

public class LambdaDemo {
    public static void main(String[] args) {
        // Old way - Anonymous class
        Operation oldWay = new Operation() {
            @Override
            public int calculate(int a, int b) {
                return a + b;
            }
        };
        
        // Lambda way - Concise
        Operation addition = (a, b) -> a + b;
        Operation multiplication = (a, b) -> a * b;
        
        System.out.println(addition.calculate(5, 3)); // 8
        System.out.println(multiplication.calculate(5, 3)); // 15
        
        // Common functional interfaces
        Runnable runnable = () -> System.out.println("Running");
        
        List<String> names = Arrays.asList("Alice", "Bob", "Charlie");
        
        // Filter and print
        names.stream()
             .filter(name -> name.startsWith("A"))
             .forEach(System.out::println); // Method reference
        
        // Map to uppercase
        names.stream()
             .map(String::toUpperCase)
             .forEach(System.out::println);
    }
}
```

### Stream API (Java 8+)

Provides functional-style operations on sequences of elements.

```java
import java.util.*;
import java.util.stream.*;

public class StreamDemo {
    public static void main(String[] args) {
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        // Intermediate operations (lazy)
        // Terminal operations (eager)
        
        // Filter - Get even numbers
        List<Integer> evens = numbers.stream()
            .filter(n -> n % 2 == 0)
            .collect(Collectors.toList());
        System.out.println(evens); // [2, 4, 6, 8, 10]
        
        // Map - Square each number
        List<Integer> squares = numbers.stream()
            .map(n -> n * n)
            .collect(Collectors.toList());
        System.out.println(squares); // [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
        
        // Reduce - Sum all numbers
        int sum = numbers.stream()
            .reduce(0, Integer::sum);
        System.out.println(sum); // 55
        
        // Aggregate operations
        List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "David");
        
        // Count
        long count = names.stream()
            .filter(name -> name.length() > 4)
            .count();
        System.out.println(count); // 2
        
        // Find first
        Optional<String> first = names.stream()
            .filter(name -> name.startsWith("C"))
            .findFirst();
        System.out.println(first.get()); // "Charlie"
        
        // Any match, All match
        boolean anyMatch = names.stream()
            .anyMatch(name -> name.startsWith("A")); // true
        boolean allMatch = names.stream()
            .allMatch(name -> name.length() > 3); // true
        
        // Collectors
        Map<Integer, List<String>> byLength = names.stream()
            .collect(Collectors.groupingBy(String::length));
        System.out.println(byLength);
        
        // Parallel streams
        long sumParallel = numbers.parallelStream()
            .mapToInt(Integer::intValue)
            .sum();
        System.out.println(sumParallel); // 55
    }
}
```

### Common Use Cases:

```java
// 1. Filtering
list.stream()
    .filter(x -> x > 10)
    .collect(Collectors.toList());

// 2. Transformation
list.stream()
    .map(String::toUpperCase)
    .collect(Collectors.toList());

// 3. Aggregation
Optional<Integer> max = list.stream()
    .max(Integer::compare);

// 4. Grouping
Map<String, List<Person>> byCity = people.stream()
    .collect(Collectors.groupingBy(Person::getCity));

// 5. FlatMap
List<String> words = lines.stream()
    .flatMap(line -> Arrays.stream(line.split(" ")))
    .collect(Collectors.toList());
```

---

## 20. Explain Java Memory Model and Concurrency

### Java Memory Model (JMM)

JMM defines how threads interact through memory.

```java
public class MemoryModelDemo {
    // Shared variable in heap
    private int sharedVar = 0;
    private volatile int volatileVar = 0;
    
    // Each thread has its own stack
    public void method() {
        int localVar = 10; // Stored in stack
        sharedVar++; // Access to heap
    }
}
```

### Concurrency Concepts:

**1. Thread Safety**

```java
import java.util.concurrent.atomic.AtomicInteger;

public class ThreadSafety {
    private int unsafeCounter = 0;
    private volatile int volatileCounter = 0;
    private AtomicInteger safeCounter = new AtomicInteger(0);
    
    // Unsafe - not synchronized
    public void incrementUnsafe() {
        unsafeCounter++; // Not atomic
    }
    
    // Volatile - visibility but not atomicity
    public void incrementVolatile() {
        volatileCounter++; // Still not atomic
    }
    
    // Safe - atomic operation
    public void incrementSafe() {
        safeCounter.incrementAndGet(); // Atomic
    }
    
    // Synchronized method
    public synchronized void incrementSync() {
        unsafeCounter++; // Thread-safe
    }
}
```

**2. Synchronization**

```java
public class SynchronizationDemo {
    private final Object lock = new Object();
    
    // Method synchronization
    public synchronized void method1() {
        // Code here
    }
    
    // Block synchronization
    public void method2() {
        synchronized (this) {
            // Code here
        }
    }
    
    // Custom lock
    public void method3() {
        synchronized (lock) {
            // Code here
        }
    }
}
```

**3. Concurrent Collections**

```java
import java.util.concurrent.*;

public class ConcurrentCollectionsDemo {
    public static void main(String[] args) {
        // Thread-safe collections
        ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
        CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
        BlockingQueue<String> queue = new LinkedBlockingQueue<>();
        
        // Thread pools
        ExecutorService executor = Executors.newFixedThreadPool(5);
        
        // Submit tasks
        for (int i = 0; i < 10; i++) {
            int taskId = i;
            executor.submit(() -> {
                System.out.println("Task " + taskId + " executed by " + 
                    Thread.currentThread().getName());
            });
        }
        
        executor.shutdown();
    }
}
```

**4. Locks and Condition**

```java
import java.util.concurrent.locks.*;

public class LockDemo {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition condition = lock.newCondition();
    
    public void criticalSection() {
        lock.lock(); // Acquire lock
        try {
            // Critical section
            condition.await(); // Wait for signal
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            lock.unlock(); // Release lock
        }
    }
    
    public void signalMethod() {
        lock.lock();
        try {
            condition.signal(); // Signal waiting thread
        } finally {
            lock.unlock();
        }
    }
}
```

**5. Concurrent Utilities**

```java
import java.util.concurrent.*;

public class ConcurrentUtilities {
    public static void main(String[] args) throws Exception {
        // CountDownLatch - Wait for multiple tasks
        CountDownLatch latch = new CountDownLatch(3);
        
        for (int i = 0; i < 3; i++) {
            new Thread(() -> {
                // Do work
                latch.countDown();
            }).start();
        }
        latch.await(); // Wait until count reaches 0
        
        // CyclicBarrier - Multiple threads wait at barrier
        CyclicBarrier barrier = new CyclicBarrier(3, () -> 
            System.out.println("All threads reached barrier"));
        
        // Semaphore - Limited resources
        Semaphore semaphore = new Semaphore(3); // 3 permits
        
        semaphore.acquire(); // Get permit
        try {
            // Use resource
        } finally {
            semaphore.release(); // Release permit
        }
        
        // CompletableFuture - Asynchronous programming
        CompletableFuture<String> future = CompletableFuture
            .supplyAsync(() -> "Hello")
            .thenApply(s -> s + " World")
            .thenApply(String::toUpperCase);
        
        System.out.println(future.get()); // "HELLO WORLD"
    }
}
```

### Common Concurrency Issues:

**1. Race Condition**
```java
// Unsafe
if (count > 0) {
    count--; // Another thread might change count here
}

// Safe
synchronized (this) {
    if (count > 0) {
        count--;
    }
}
```

**2. Deadlock**
```java
// Thread 1: Lock A then B
// Thread 2: Lock B then A
// → Deadlock!
```

**3. Livelock** - Threads keep changing state

### Best Practices:

1. Use volatile for visibility
2. Use Atomic classes for atomicity
3. Prefer concurrent collections
4. Use thread pools instead of creating threads
5. Avoid shared mutable state
6. Use immutable objects when possible

---

## Additional Resources

- **Books:** Effective Java, Java Concurrency in Practice
- **Official Docs:** Oracle Java Documentation
- **Practice:** LeetCode, HackerRank
- **Design Patterns:** GOF Design Patterns

---

## Interview Tips

1. **Understand WHY** not just WHAT
2. **Draw diagrams** for architecture questions
3. **Write code** during explanations
4. **Ask clarifying questions**
5. **Think out loud** during problem-solving
6. **Know trade-offs** of different approaches
7. **Remember edge cases**
8. **Discuss time/space complexity**

**Good luck with your interviews!** 🚀

