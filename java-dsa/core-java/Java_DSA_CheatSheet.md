# Java DSA Cheat Sheet: Primitive Data Types & Collections

## Table of Contents
1. [Primitive Data Types](#primitive-data-types)
2. [Arrays](#arrays)
3. [Collections Framework](#collections-framework)
4. [List Interface](#list-interface)
5. [Set Interface](#set-interface)
6. [Map Interface](#map-interface)
7. [Queue Interface](#queue-interface)
8. [Stack](#stack)
9. [Time & Space Complexity Summary](#time--space-complexity-summary)
10. [Common DSA Patterns](#common-dsa-patterns)

---

## Primitive Data Types

### Overview
Java has 8 primitive data types that are the building blocks for data manipulation.

| Type | Size (bits) | Range | Default Value | Wrapper Class |
|------|-------------|-------|---------------|---------------|
| `byte` | 8 | -128 to 127 | 0 | Byte |
| `short` | 16 | -32,768 to 32,767 | 0 | Short |
| `int` | 32 | -2³¹ to 2³¹-1 | 0 | Integer |
| `long` | 64 | -2⁶³ to 2⁶³-1 | 0L | Long |
| `float` | 32 | ±3.4E±38 (7 digits) | 0.0f | Float |
| `double` | 64 | ±1.7E±308 (15 digits) | 0.0d | Double |
| `char` | 16 | 0 to 65,535 (Unicode) | '\u0000' | Character |
| `boolean` | 1 | true/false | false | Boolean |




### DSA Applications

#### Integer Types (`byte`, `short`, `int`, `long`)
```java
// Array indexing - most common use
int[] arr = new int[1000];
for (int i = 0; i < arr.length; i++) {
    arr[i] = i * 2;
}

// Bit manipulation
int setBit = n | (1 << pos);        // Set bit at position
int clearBit = n & ~(1 << pos);     // Clear bit at position
int toggleBit = n ^ (1 << pos);     // Toggle bit at position
boolean checkBit = (n & (1 << pos)) != 0; // Check if bit is set

// Mathematical operations
long factorial = 1;
for (int i = 1; i <= n; i++) {
    factorial *= i;
}

// GCD using Euclidean algorithm
public static int gcd(int a, int b) {
    while (b != 0) {
        int temp = b;
        b = a % b;
        a = temp;
    }
    return a;
}
```

#### Floating Point (`float`, `double`)
```java
// Precision comparison
double epsilon = 1e-9;
boolean isEqual = Math.abs(a - b) < epsilon;

// Mathematical calculations
double distance = Math.sqrt((x2-x1)*(x2-x1) + (y2-y1)*(y2-y1));
double angle = Math.atan2(y, x);

// Geometric calculations
double area = 0.5 * base * height;
double circumference = 2 * Math.PI * radius;
```

#### Character (`char`)
```java
// Character operations
char ch = 'A';
boolean isDigit = Character.isDigit(ch);
boolean isLetter = Character.isLetter(ch);
boolean isUpperCase = Character.isUpperCase(ch);
char lowercase = Character.toLowerCase(ch);

// ASCII manipulation
int asciiValue = (int) ch;
char fromAscii = (char) (asciiValue + 1);

// String to char array conversion
String str = "hello";
char[] charArray = str.toCharArray();

// Character frequency counting
Map<Character, Integer> freq = new HashMap<>();
for (char c : str.toCharArray()) {
    freq.put(c, freq.getOrDefault(c, 0) + 1);
}
```

#### Boolean (`boolean`)
```java
// Logical operations
boolean result = (a > b) && (c < d);
boolean flag = !condition;

// Array of booleans for visited tracking in graphs/trees
boolean[] visited = new boolean[n];
visited[index] = true;

// Boolean algebra in bit manipulation
boolean isPowerOfTwo = (n > 0) && ((n & (n - 1)) == 0);
```

---

## Arrays

### Array Declaration and Initialization

#### Basic Array Declaration
```java
// Declaration only (creates null reference)
int[] arr1;
String[] arr2;
double[] arr3;

// Alternative syntax (less preferred)
int arr4[];
String arr5[];
```

#### Array Initialization Methods
```java
// Method 1: Declare and initialize with size
int[] arr = new int[5];           // Creates array of size 5, all elements = 0
String[] names = new String[10];  // Creates array of size 10, all elements = null
boolean[] flags = new boolean[3]; // Creates array of size 3, all elements = false

// Method 2: Initialize with values (array literal)
int[] numbers = {1, 2, 3, 4, 5};
String[] colors = {"red", "green", "blue"};
double[] prices = {10.5, 20.0, 15.75};

// Method 3: Using new keyword with values
int[] scores = new int[]{85, 90, 78, 92, 88};
String[] cities = new String[]{"New York", "London", "Tokyo"};

// Method 4: Anonymous array (useful for method parameters)
processArray(new int[]{1, 2, 3, 4, 5});

// Method 5: Using Arrays.fill() for same values
int[] arr = new int[10];
Arrays.fill(arr, 42);  // All elements = 42

// Method 6: Using streams for initialization
int[] range = IntStream.range(0, 10).toArray();        // [0, 1, 2, ..., 9]
int[] squares = IntStream.range(1, 6).map(x -> x * x).toArray(); // [1, 4, 9, 16, 25]
```

### Array Properties and Basic Operations
```java
int[] arr = {10, 20, 30, 40, 50};

// Array length (property, not method - no parentheses!)
int length = arr.length;  // Returns 5

// Access elements by index (0-based)
int first = arr[0];       // Gets first element (10)
int last = arr[arr.length - 1]; // Gets last element (50)

// Modify elements
arr[0] = 100;            // Changes first element to 100
arr[2] = arr[1] + arr[3]; // arr[2] = 20 + 40 = 60

// Iterate through array
// Method 1: Traditional for loop
for (int i = 0; i < arr.length; i++) {
    System.out.println("Index " + i + ": " + arr[i]);
}

// Method 2: Enhanced for loop (for-each)
for (int value : arr) {
    System.out.println("Value: " + value);
}

// Method 3: Using streams
Arrays.stream(arr).forEach(System.out::println);
```

### Multidimensional Arrays
```java
// 2D Array Declaration and Initialization
// Method 1: Declare with dimensions
int[][] matrix = new int[3][4];  // 3 rows, 4 columns

// Method 2: Initialize with values
int[][] grid = {
    {1, 2, 3},
    {4, 5, 6},
    {7, 8, 9}
};

// Method 3: Jagged arrays (different row lengths)
int[][] jagged = new int[3][];
jagged[0] = new int[2];  // First row has 2 elements
jagged[1] = new int[4];  // Second row has 4 elements
jagged[2] = new int[3];  // Third row has 3 elements

// Access 2D array elements
int element = matrix[1][2];  // Row 1, Column 2
matrix[0][0] = 10;          // Set element at row 0, column 0

// Get dimensions
int rows = matrix.length;        // Number of rows
int cols = matrix[0].length;     // Number of columns (assuming rectangular)

// Iterate through 2D array
for (int i = 0; i < matrix.length; i++) {
    for (int j = 0; j < matrix[i].length; j++) {
        System.out.print(matrix[i][j] + " ");
    }
    System.out.println();
}

// Enhanced for loop for 2D array
for (int[] row : matrix) {
    for (int value : row) {
        System.out.print(value + " ");
    }
    System.out.println();
}

// 3D Array example
int[][][] cube = new int[3][4][5];  // 3x4x5 cube
int value = cube[1][2][3];          // Access element
```

### Arrays Utility Class Methods
```java
import java.util.Arrays;

int[] arr = {5, 2, 8, 1, 9, 3};

// Sorting
Arrays.sort(arr);                    // Sorts in ascending order: [1, 2, 3, 5, 8, 9]
Arrays.sort(arr, 1, 4);             // Sort subarray from index 1 to 3

// For descending order (works with Integer[], not int[])
Integer[] boxedArr = {5, 2, 8, 1, 9, 3};
Arrays.sort(boxedArr, Collections.reverseOrder());

// Binary search (array must be sorted first)
int index = Arrays.binarySearch(arr, 5);  // Returns index of element 5
int notFound = Arrays.binarySearch(arr, 7); // Returns negative value if not found

// Fill array with specific value
Arrays.fill(arr, 0);                 // All elements become 0
Arrays.fill(arr, 2, 5, 10);         // Fill indices 2-4 with value 10

// Copy arrays
int[] original = {1, 2, 3, 4, 5};
int[] copy1 = Arrays.copyOf(original, original.length);     // Full copy
int[] copy2 = Arrays.copyOf(original, 10);                  // Copy with new size (padded with 0s)
int[] copy3 = Arrays.copyOfRange(original, 1, 4);          // Copy subarray [2, 3, 4]

// Array comparison
int[] arr1 = {1, 2, 3};
int[] arr2 = {1, 2, 3};
boolean equal = Arrays.equals(arr1, arr2);  // true

// Deep comparison for multidimensional arrays
int[][] matrix1 = {{1, 2}, {3, 4}};
int[][] matrix2 = {{1, 2}, {3, 4}};
boolean deepEqual = Arrays.deepEquals(matrix1, matrix2);  // true

// Convert array to string
String str = Arrays.toString(arr);      // "[1, 2, 3, 4, 5]"
String deepStr = Arrays.deepToString(matrix1); // "[[1, 2], [3, 4]]"

// Convert array to List
List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);
// Note: This creates a fixed-size list backed by the array

// Stream operations on arrays
int[] numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
int sum = Arrays.stream(numbers).sum();                    // Sum all elements
double average = Arrays.stream(numbers).average().orElse(0); // Average
int max = Arrays.stream(numbers).max().orElse(0);         // Maximum
int[] evens = Arrays.stream(numbers).filter(x -> x % 2 == 0).toArray(); // Filter evens
```

### Array vs Collections Comparison
```java
// Arrays - Fixed size, better performance for primitive types
int[] primitiveArray = new int[1000];     // Memory efficient
primitiveArray.length;                    // Property (no parentheses)
Arrays.sort(primitiveArray);              // Utility methods in Arrays class

// Collections - Dynamic size, more methods, only for objects
List<Integer> list = new ArrayList<>();   // Dynamic size
list.size();                              // Method (with parentheses)
Collections.sort(list);                   // Utility methods in Collections class

// When to use arrays:
// - Fixed size data
// - Performance critical applications
// - Working with primitive types
// - Memory constraints

// When to use Collections:
// - Dynamic size needed
// - Rich set of methods required
// - Working with objects
// - Need thread-safe versions
```

---

## String Manipulation

### String Basics
```java
// String Declaration and Initialization
String str1 = "Hello";                    // String literal (stored in pool)
String str2 = new String("Hello");       // String object
String str3 = str1 + " World";            // Concatenation

// String Immutability
String s = "Java";
s.concat(" Programming");                 // Returns new string "Java Programming"
System.out.println(s);                    // Still prints "Java"

// String Comparison
String a = "Hello";
String b = "Hello";
boolean equal1 = (a == b);                // true (same reference in pool)
boolean equal2 = a.equals(b);             // true (content comparison)

String c = new String("Hello");
boolean equal3 = (a == c);                // false (different references)
boolean equal4 = a.equals(c);             // true (content is same)

// Case-insensitive comparison
"Hello".equalsIgnoreCase("HELLO");        // true
```

### Common String Operations
```java
String str = "Java Programming";

// Length
int len = str.length();                   // 17

// Character access
char ch = str.charAt(0);                  // 'J'
char ch2 = str.charAt(5);                 // 'P'

// Substring
String sub1 = str.substring(5);            // "Programming"
String sub2 = str.substring(0, 4);         // "Java"
String sub3 = str.substring(5, 8);         // "Pro"

// Searching
boolean contains1 = str.contains("Pro");  // true
int indexOf1 = str.indexOf('a');          // 1 (first occurrence)
int indexOf2 = str.indexOf('a', 2);       // 3 (from index 2)
int indexOf3 = str.lastIndexOf('a');      // 3 (last occurrence)
int indexOf4 = str.indexOf("Pro");        // 5

// Case conversion
String upper = str.toUpperCase();         // "JAVA PROGRAMMING"
String lower = str.toLowerCase();         // "java programming"

// Trimming whitespace
String withSpaces = "  Hello World  ";
String trimmed = withSpaces.trim();       // "Hello World"
String stripped = "  Hello  ".strip();    // "Hello" (Java 11+)

// Replacing
String replaced = str.replace('a', 'A');  // "JAvA Programming"
String replaced2 = str.replace("Programming", "Development"); // "Java Development"
String replaced3 = str.replaceAll("a", "A"); // "JAvA ProgrAmming"
String replaced4 = str.replaceFirst("a", "A"); // "JAva Programming"

// Checking prefix/suffix
boolean starts = str.startsWith("Java");  // true
boolean starts2 = str.startsWith("Pro", 5); // true (from index 5)
boolean ends = str.endsWith("ing");       // true

// Checking if empty
boolean isEmpty = str.isEmpty();           // false
String empty = "";
boolean isEmpty2 = empty.isEmpty();       // true
```

### StringBuilder vs StringBuffer
```java
// StringBuilder (Not thread-safe, faster)
StringBuilder sb = new StringBuilder("Hello");
sb.append(" World");                      // Append
sb.insert(5, ", ");                       // Insert at position
sb.delete(5, 7);                          // Delete substring
sb.replace(0, "Hi", "He");               // Replace
sb.reverse();                             // Reverse string
int length = sb.length();                 // Length
char ch = sb.charAt(0);                   // Get character
String result = sb.toString();            // Convert to String

// Example: Building string efficiently
StringBuilder builder = new StringBuilder();
for (int i = 0; i < 100; i++) {
    builder.append(i).append(", ");
}
String numbers = builder.toString();

// StringBuffer (Thread-safe, slower)
StringBuffer buffer = new StringBuffer("Hello");
buffer.append(" World");                   // Same methods as StringBuilder
buffer.reverse();
String result = buffer.toString();
```

### String Conversion
```java
String str = "123";

// String to int
int num = Integer.parseInt(str);          // 123
int num2 = Integer.valueOf(str);          // 123 (returns Integer object)

// String to other types
double d = Double.parseDouble("3.14");
long l = Long.parseLong("999999999");
boolean b = Boolean.parseBoolean("true");

// int/double/etc to String
int n = 42;
String s1 = String.valueOf(n);            // "42"
String s2 = Integer.toString(n);          // "42"
String s3 = n + "";                        // "42" (implicit conversion)

// char array to String
char[] chars = {'H', 'e', 'l', 'l', 'o'};
String strFromChars = new String(chars);  // "Hello"
String strFromChars2 = String.valueOf(chars); // "Hello"

// String to char array
String text = "Hello";
char[] charArray = text.toCharArray();    // ['H', 'e', 'l', 'l', 'o']

// StringBuilder to String
StringBuilder sb = new StringBuilder("Hi");
String str = sb.toString();
```

### Splitting and Joining
```java
// Splitting strings
String data = "apple,banana,orange";
String[] fruits = data.split(",");        // ["apple", "banana", "orange"]
String[] fruits2 = data.split(",", 2);    // ["apple", "banana,orange"] (limit 2)
String line = "one\ntwo\nthree";
String[] lines = line.split("\n");        // Split by newline

// Splitting with regex
String text = "apple and banana";
String[] words = text.split("\\s+");      // Split by whitespace
String dates = "2023-01-01";
String[] parts = dates.split("-");        // ["2023", "01", "01"]

// Joining strings (Java 8+)
String[] arr = {"apple", "banana", "orange"};
String joined = String.join(", ", arr);   // "apple, banana, orange"

// Using StringBuilder for joining
StringBuilder sb = new StringBuilder();
for (int i = 0; i < arr.length; i++) {
    if (i > 0) sb.append(", ");
    sb.append(arr[i]);
}
String result = sb.toString();             // "apple, banana, orange"
```

### Pattern Matching (Basic Regex)
```java
String text = "The year is 2023 and 2024 is coming";

// Matches pattern (entire string must match)
boolean matches = "2023".matches("\\d{4}"); // true

// Find pattern using Pattern class
import java.util.regex.Pattern;
import java.util.regex.Matcher;

Pattern pattern = Pattern.compile("\\d{4}"); // 4 digits
Matcher matcher = pattern.matcher(text);
while (matcher.find()) {
    System.out.println(matcher.group());   // "2023", then "2024"
}

// Replace with regex
String result = text.replaceAll("\\d{4}", "XXXX"); // "The year is XXXX and XXXX is coming"

// Common regex patterns
String email = "user@example.com";
boolean validEmail = email.matches("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$");
```

### Common DSA String Patterns
```java
// Pattern 1: Character frequency counting
String s = "aabbbcd";
HashMap<Character, Integer> freq = new HashMap<>();
for (char c : s.toCharArray()) {
    freq.put(c, freq.getOrDefault(c, 0) + 1);
}

// Pattern 2: String palindrome check
public static boolean isPalindrome(String s) {
    int left = 0, right = s.length() - 1;
    while (left < right) {
        if (s.charAt(left) != s.charAt(right)) {
            return false;
        }
        left++;
        right--;
    }
    return true;
}

// Pattern 3: Anagram check
public static boolean isAnagram(String s1, String s2) {
    if (s1.length() != s2.length()) return false;
    
    char[] arr1 = s1.toCharArray();
    char[] arr2 = s2.toCharArray();
    Arrays.sort(arr1);
    Arrays.sort(arr2);
    
    return Arrays.equals(arr1, arr2);
}

// Pattern 4: Longest common prefix
public String longestCommonPrefix(String[] strs) {
    if (strs.length == 0) return "";
    
    String prefix = strs[0];
    for (int i = 1; i < strs.length; i++) {
        while (strs[i].indexOf(prefix) != 0) {
            prefix = prefix.substring(0, prefix.length() - 1);
            if (prefix.isEmpty()) return "";
        }
    }
    return prefix;
}

// Pattern 5: Valid parentheses
public boolean isValid(String s) {
    Stack<Character> stack = new Stack<>();
    for (char c : s.toCharArray()) {
        if (c == '(' || c == '{' || c == '[') {
            stack.push(c);
        } else {
            if (stack.isEmpty()) return false;
            char open = stack.pop();
            if ((c == ')' && open != '(') ||
                (c == '}' && open != '{') ||
                (c == ']' && open != '[')) {
                return false;
            }
        }
    }
    return stack.isEmpty();
}

// Pattern 6: Reverse string
public String reverseString(String s) {
    char[] chars = s.toCharArray();
    int left = 0, right = chars.length - 1;
    while (left < right) {
        char temp = chars[left];
        chars[left] = chars[right];
        chars[right] = temp;
        left++;
        right--;
    }
    return new String(chars);
}

// Pattern 7: Remove duplicates
public String removeDuplicates(String s) {
    StringBuilder sb = new StringBuilder();
    boolean[] seen = new boolean[26];
    
    for (char c : s.toCharArray()) {
        if (!seen[c - 'a']) {
            sb.append(c);
            seen[c - 'a'] = true;
        }
    }
    return sb.toString();
}
```

### String Performance Tips
```java
// ❌ BAD: String concatenation in loop (creates many objects)
String result = "";
for (int i = 0; i < 1000; i++) {
    result += i;  // Creates new String object each iteration
}

// ✅ GOOD: Use StringBuilder for concatenation in loops
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.append(i);
}
String result = sb.toString();

// ❌ BAD: Converting char to String inefficiently
String s = "" + ch;

// ✅ GOOD: Use String.valueOf() or Character.toString()
String s = String.valueOf(ch);
String s = Character.toString(ch);

// ✅ Comparing strings
if ("value".equals(variable)) { }  // Safe even if variable is null
// vs
if (variable.equals("value")) { }  // Throws NullPointerException if variable is null
```

---

## Collections Framework

### Hierarchy Overview
```
Collection (Interface)
├── List (Interface)
│   ├── ArrayList (Class)
│   ├── LinkedList (Class)
│   └── Vector (Class)
├── Set (Interface)
│   ├── HashSet (Class)
│   ├── LinkedHashSet (Class)
│   └── TreeSet (Class)
└── Queue (Interface)
    ├── PriorityQueue (Class)
    ├── ArrayDeque (Class)
    └── LinkedList (Class)

Map (Interface)
├── HashMap (Class)
├── LinkedHashMap (Class)
├── TreeMap (Class)
└── Hashtable (Class)
```

---

## List Interface

### ArrayList
**Best for:** Random access, frequent reads, dynamic resizing

```java
import java.util.*;

// Declaration and initialization
List<Integer> list = new ArrayList<>();
List<Integer> listWithCapacity = new ArrayList<>(100);
List<Integer> listFromArray = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));

// Basic operations
list.add(10);           // Add element - O(1) amortized
list.add(0, 5);         // Add at index - O(n)
list.get(0);            // Access by index - O(1)
list.set(0, 15);        // Update by index - O(1)
list.remove(0);         // Remove by index - O(n)
list.remove(Integer.valueOf(10)); // Remove by value - O(n)

// Useful methods
int size = list.size();
boolean isEmpty = list.isEmpty();
boolean contains = list.contains(10);
int index = list.indexOf(10);
int lastIndex = list.lastIndexOf(10);
list.clear();

// Iteration methods
for (int num : list) { /* process num */ }
for (int i = 0; i < list.size(); i++) { /* process list.get(i) */ }
list.forEach(System.out::println);

// Sorting
Collections.sort(list);                    // Natural order
Collections.sort(list, Collections.reverseOrder()); // Reverse order
Collections.sort(list, (a, b) -> a.compareTo(b));   // Custom comparator

// Binary search (requires sorted list)
int position = Collections.binarySearch(list, 10);

// Sublist operations
List<Integer> subList = list.subList(1, 4); // [1, 4)
```

### LinkedList
**Best for:** Frequent insertions/deletions, queue/deque operations

```java
import java.util.*;

LinkedList<Integer> linkedList = new LinkedList<>();

// List operations
linkedList.add(10);
linkedList.addFirst(5);     // O(1)
linkedList.addLast(15);     // O(1)
linkedList.removeFirst();   // O(1)
linkedList.removeLast();    // O(1)

// Queue operations
linkedList.offer(20);       // Add to tail
linkedList.poll();          // Remove from head
linkedList.peek();          // View head without removing

// Deque operations
linkedList.offerFirst(1);
linkedList.offerLast(2);
linkedList.pollFirst();
linkedList.pollLast();
linkedList.peekFirst();
linkedList.peekLast();

// Iterator for safe removal during iteration
Iterator<Integer> it = linkedList.iterator();
while (it.hasNext()) {
    Integer value = it.next();
    if (value % 2 == 0) {
        it.remove(); // Safe removal
    }
}
```

### Time Complexity Comparison

| Operation | ArrayList | LinkedList |
|-----------|-----------|------------|
| Access by index | O(1) | O(n) |
| Insert at beginning | O(n) | O(1) |
| Insert at end | O(1) amortized | O(1) |
| Insert at middle | O(n) | O(n) |
| Delete at beginning | O(n) | O(1) |
| Delete at end | O(1) | O(1) |
| Delete at middle | O(n) | O(n) |
| Search | O(n) | O(n) |

---

## Set Interface

### HashSet
**Best for:** Fast lookups, uniqueness checking, no ordering needed

```java
import java.util.*;

Set<Integer> hashSet = new HashSet<>();

// Basic operations
hashSet.add(10);        // O(1) average
hashSet.remove(10);     // O(1) average
hashSet.contains(10);   // O(1) average
hashSet.size();
hashSet.isEmpty();

// Set operations
Set<Integer> set1 = new HashSet<>(Arrays.asList(1, 2, 3, 4));
Set<Integer> set2 = new HashSet<>(Arrays.asList(3, 4, 5, 6));

// Union
Set<Integer> union = new HashSet<>(set1);
union.addAll(set2);     // {1, 2, 3, 4, 5, 6}

// Intersection
Set<Integer> intersection = new HashSet<>(set1);
intersection.retainAll(set2);  // {3, 4}

// Difference
Set<Integer> difference = new HashSet<>(set1);
difference.removeAll(set2);    // {1, 2}

// Check if subset
boolean isSubset = set1.containsAll(set2);

// Iteration (no guaranteed order)
for (int num : hashSet) { /* process num */ }

// Convert to array
Integer[] array = hashSet.toArray(new Integer[0]);
```

### LinkedHashSet
**Best for:** Maintaining insertion order with set properties

```java
Set<Integer> linkedHashSet = new LinkedHashSet<>();
// Maintains insertion order
// Same operations as HashSet but with predictable iteration order
// Useful when you need both uniqueness and order preservation
```

### TreeSet
**Best for:** Sorted set, range queries, ordered operations

```java
import java.util.*;

TreeSet<Integer> treeSet = new TreeSet<>();

// Basic operations - O(log n)
treeSet.add(10);
treeSet.remove(10);
treeSet.contains(10);

// Ordered operations
treeSet.first();        // Smallest element
treeSet.last();         // Largest element
treeSet.lower(10);      // Largest element < 10
treeSet.higher(10);     // Smallest element > 10
treeSet.floor(10);      // Largest element <= 10
treeSet.ceiling(10);    // Smallest element >= 10

// Range operations
SortedSet<Integer> subset = treeSet.subSet(5, 15);  // [5, 15)
SortedSet<Integer> headSet = treeSet.headSet(10);   // < 10
SortedSet<Integer> tailSet = treeSet.tailSet(10);   // >= 10

// Custom comparator
TreeSet<String> customSet = new TreeSet<>((a, b) -> a.length() - b.length());

// Descending operations
NavigableSet<Integer> descendingSet = treeSet.descendingSet();
Iterator<Integer> descendingIterator = treeSet.descendingIterator();
```

---

## Map Interface

### HashMap
**Best for:** Fast key-value lookups, caching, frequency counting

```java
import java.util.*;

Map<String, Integer> hashMap = new HashMap<>();

// Basic operations - O(1) average
hashMap.put("key1", 100);
hashMap.get("key1");
hashMap.remove("key1");
hashMap.containsKey("key1");
hashMap.containsValue(100);

// Useful methods
hashMap.putIfAbsent("key2", 200);  // Only put if key doesn't exist
hashMap.getOrDefault("key3", 0);   // Return default if key not found
hashMap.merge("key4", 1, Integer::sum);  // Merge values
hashMap.compute("key5", (k, v) -> v == null ? 1 : v + 1); // Compute new value
hashMap.computeIfAbsent("key6", k -> new ArrayList<>()); // Compute if absent

// Iteration
for (Map.Entry<String, Integer> entry : hashMap.entrySet()) {
    String key = entry.getKey();
    Integer value = entry.getValue();
}

for (String key : hashMap.keySet()) { /* process key */ }
for (Integer value : hashMap.values()) { /* process value */ }

// Common DSA patterns
// Frequency counting
Map<Character, Integer> freq = new HashMap<>();
for (char c : str.toCharArray()) {
    freq.put(c, freq.getOrDefault(c, 0) + 1);
}

// Two sum problem
Map<Integer, Integer> numToIndex = new HashMap<>();
for (int i = 0; i < nums.length; i++) {
    int complement = target - nums[i];
    if (numToIndex.containsKey(complement)) {
        return new int[]{numToIndex.get(complement), i};
    }
    numToIndex.put(nums[i], i);
}

// Group anagrams
Map<String, List<String>> groups = new HashMap<>();
for (String word : words) {
    char[] chars = word.toCharArray();
    Arrays.sort(chars);
    String key = new String(chars);
    groups.computeIfAbsent(key, k -> new ArrayList<>()).add(word);
}
```

### LinkedHashMap
**Best for:** Maintaining insertion/access order with map properties

```java
// Insertion order (default)
Map<String, Integer> insertionOrder = new LinkedHashMap<>();

// Access order (LRU cache implementation)
Map<String, Integer> accessOrder = new LinkedHashMap<>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<String, Integer> eldest) {
        return size() > MAX_CAPACITY; // Remove oldest when capacity exceeded
    }
};
```

### TreeMap
**Best for:** Sorted keys, range queries on keys

```java
import java.util.*;

TreeMap<Integer, String> treeMap = new TreeMap<>();

// Basic operations - O(log n)
treeMap.put(10, "ten");
treeMap.get(10);
treeMap.remove(10);

// Ordered operations
treeMap.firstKey();     // Smallest key
treeMap.lastKey();      // Largest key
treeMap.lowerKey(10);   // Largest key < 10
treeMap.higherKey(10);  // Smallest key > 10
treeMap.floorKey(10);   // Largest key <= 10
treeMap.ceilingKey(10); // Smallest key >= 10

// Entry operations
Map.Entry<Integer, String> firstEntry = treeMap.firstEntry();
Map.Entry<Integer, String> lastEntry = treeMap.lastEntry();
Map.Entry<Integer, String> lowerEntry = treeMap.lowerEntry(10);

// Range operations
SortedMap<Integer, String> subMap = treeMap.subMap(5, 15);
SortedMap<Integer, String> headMap = treeMap.headMap(10);
SortedMap<Integer, String> tailMap = treeMap.tailMap(10);

// Descending operations
NavigableMap<Integer, String> descendingMap = treeMap.descendingMap();
```

---

## Queue Interface

### PriorityQueue (Min-Heap by default)
**Best for:** Heap operations, finding min/max elements, priority-based processing

```java
import java.util.*;

// Min heap (default)
PriorityQueue<Integer> minHeap = new PriorityQueue<>();

// Max heap
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());

// Custom comparator for objects
PriorityQueue<int[]> customHeap = new PriorityQueue<>((a, b) -> a[0] - b[0]);

// String length comparator
PriorityQueue<String> stringHeap = new PriorityQueue<>((a, b) -> a.length() - b.length());

// Basic operations - O(log n) for add/remove, O(1) for peek
minHeap.offer(10);      // Add element
minHeap.poll();         // Remove and return smallest
minHeap.peek();         // View smallest without removing
minHeap.size();
minHeap.isEmpty();

// Common patterns
// K largest elements
public int[] findKLargest(int[] nums, int k) {
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();
    for (int num : nums) {
        minHeap.offer(num);
        if (minHeap.size() > k) {
            minHeap.poll();
        }
    }
    
    int[] result = new int[k];
    for (int i = k - 1; i >= 0; i--) {
        result[i] = minHeap.poll();
    }
    return result;
}

// Merge k sorted arrays
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
// Add first element of each array: [value, arrayIndex, elementIndex]

// Dijkstra's algorithm
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);
// [node, distance]
```

### ArrayDeque
**Best for:** Double-ended queue operations, stack operations

```java
import java.util.*;

Deque<Integer> deque = new ArrayDeque<>();

// Queue operations (FIFO)
deque.offerLast(10);    // Add to rear
deque.pollFirst();      // Remove from front
deque.peekFirst();      // View front

// Stack operations (LIFO)
deque.push(10);         // Add to front (stack top)
deque.pop();            // Remove from front (stack top)
deque.peek();           // View front (stack top)

// Deque operations
deque.offerFirst(5);    // Add to front
deque.offerLast(15);    // Add to rear
deque.pollFirst();      // Remove from front
deque.pollLast();       // Remove from rear
deque.peekFirst();      // View front
deque.peekLast();       // View rear

// Sliding window maximum using deque
public int[] maxSlidingWindow(int[] nums, int k) {
    Deque<Integer> deque = new ArrayDeque<>(); // stores indices
    int[] result = new int[nums.length - k + 1];
    
    for (int i = 0; i < nums.length; i++) {
        // Remove elements outside window
        while (!deque.isEmpty() && deque.peekFirst() < i - k + 1) {
            deque.pollFirst();
        }
        
        // Remove smaller elements
        while (!deque.isEmpty() && nums[deque.peekLast()] < nums[i]) {
            deque.pollLast();
        }
        
        deque.offerLast(i);
        
        if (i >= k - 1) {
            result[i - k + 1] = nums[deque.peekFirst()];
        }
    }
    
    return result;
}
```

---

## Stack

### Using ArrayDeque (Recommended)
```java
import java.util.*;

Deque<Integer> stack = new ArrayDeque<>();

stack.push(10);         // O(1)
stack.pop();            // O(1)
stack.peek();           // O(1)
stack.isEmpty();
stack.size();

// Common patterns
// Balanced parentheses
public boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    Map<Character, Character> map = Map.of(')', '(', '}', '{', ']', '[');
    
    for (char c : s.toCharArray()) {
        if (map.containsKey(c)) {
            if (stack.isEmpty() || stack.pop() != map.get(c)) {
                return false;
            }
        } else {
            stack.push(c);
        }
    }
    return stack.isEmpty();
}

// Next greater element
public int[] nextGreaterElement(int[] nums) {
    int[] result = new int[nums.length];
    Deque<Integer> stack = new ArrayDeque<>();
    
    for (int i = nums.length - 1; i >= 0; i--) {
        while (!stack.isEmpty() && stack.peek() <= nums[i]) {
            stack.pop();
        }
        result[i] = stack.isEmpty() ? -1 : stack.peek();
        stack.push(nums[i]);
    }
    
    return result;
}

// Largest rectangle in histogram
public int largestRectangleArea(int[] heights) {
    Deque<Integer> stack = new ArrayDeque<>();
    int maxArea = 0;
    
    for (int i = 0; i <= heights.length; i++) {
        int h = (i == heights.length) ? 0 : heights[i];
        
        while (!stack.isEmpty() && h < heights[stack.peek()]) {
            int height = heights[stack.pop()];
            int width = stack.isEmpty() ? i : i - stack.peek() - 1;
            maxArea = Math.max(maxArea, height * width);
        }
        
        stack.push(i);
    }
    
    return maxArea;
}
```

---

## Time & Space Complexity Summary

### List Operations
| Data Structure | Access | Search | Insertion | Deletion | Space |
|----------------|--------|--------|-----------|----------|-------|
| ArrayList | O(1) | O(n) | O(n) | O(n) | O(n) |
| LinkedList | O(n) | O(n) | O(1)* | O(1)* | O(n) |
| Vector | O(1) | O(n) | O(n) | O(n) | O(n) |

*At known position

### Set Operations
| Data Structure | Add | Remove | Contains | Space |
|----------------|-----|--------|----------|-------|
| HashSet | O(1) avg | O(1) avg | O(1) avg | O(n) |
| LinkedHashSet | O(1) avg | O(1) avg | O(1) avg | O(n) |
| TreeSet | O(log n) | O(log n) | O(log n) | O(n) |

### Map Operations
| Data Structure | Put | Get | Remove | Space |
|----------------|-----|-----|--------|-------|
| HashMap | O(1) avg | O(1) avg | O(1) avg | O(n) |
| LinkedHashMap | O(1) avg | O(1) avg | O(1) avg | O(n) |
| TreeMap | O(log n) | O(log n) | O(log n) | O(n) |
| Hashtable | O(1) avg | O(1) avg | O(1) avg | O(n) |

### Queue Operations
| Data Structure | Enqueue | Dequeue | Peek | Space |
|----------------|---------|---------|------|-------|
| ArrayDeque | O(1) | O(1) | O(1) | O(n) |
| LinkedList | O(1) | O(1) | O(1) | O(n) |
| PriorityQueue | O(log n) | O(log n) | O(1) | O(n) |

---

## Common DSA Patterns

### 1. Two Pointers
```java
// Array two pointers - Two Sum (sorted array)
public int[] twoSum(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left < right) {
        int sum = nums[left] + nums[right];
        if (sum == target) {
            return new int[]{left, right};
        } else if (sum < target) {
            left++;
        } else {
            right--;
        }
    }
    return new int[]{-1, -1};
}

// String two pointers - Palindrome check
public boolean isPalindrome(String s) {
    int left = 0, right = s.length() - 1;
    while (left < right) {
        if (s.charAt(left) != s.charAt(right)) {
            return false;
        }
        left++;
        right--;
    }
    return true;
}

// Remove duplicates from sorted array
public int removeDuplicates(int[] nums) {
    int writeIndex = 1;
    for (int readIndex = 1; readIndex < nums.length; readIndex++) {
        if (nums[readIndex] != nums[readIndex - 1]) {
            nums[writeIndex] = nums[readIndex];
            writeIndex++;
        }
    }
    return writeIndex;
}
```

### 2. Sliding Window
```java
// Fixed size window - Maximum sum subarray of size k
public int maxSumSubarray(int[] arr, int k) {
    int maxSum = Integer.MIN_VALUE;
    int windowSum = 0;
    
    // Calculate sum of first window
    for (int i = 0; i < k; i++) {
        windowSum += arr[i];
    }
    maxSum = windowSum;
    
    // Slide the window
    for (int i = k; i < arr.length; i++) {
        windowSum = windowSum - arr[i - k] + arr[i];
        maxSum = Math.max(maxSum, windowSum);
    }
    
    return maxSum;
}

// Variable size window - Longest substring without repeating characters
public int lengthOfLongestSubstring(String s) {
    int left = 0, maxLength = 0;
    Set<Character> seen = new HashSet<>();
    
    for (int right = 0; right < s.length(); right++) {
        while (seen.contains(s.charAt(right))) {
            seen.remove(s.charAt(left));
            left++;
        }
        seen.add(s.charAt(right));
        maxLength = Math.max(maxLength, right - left + 1);
    }
    
    return maxLength;
}

// Minimum window substring
public String minWindow(String s, String t) {
    Map<Character, Integer> need = new HashMap<>();
    Map<Character, Integer> window = new HashMap<>();
    
    for (char c : t.toCharArray()) {
        need.put(c, need.getOrDefault(c, 0) + 1);
    }
    
    int left = 0, right = 0;
    int valid = 0;
    int start = 0, len = Integer.MAX_VALUE;
    
    while (right < s.length()) {
        char c = s.charAt(right);
        right++;
        
        if (need.containsKey(c)) {
            window.put(c, window.getOrDefault(c, 0) + 1);
            if (window.get(c).equals(need.get(c))) {
                valid++;
            }
        }
        
        while (valid == need.size()) {
            if (right - left < len) {
                start = left;
                len = right - left;
            }
            
            char d = s.charAt(left);
            left++;
            
            if (need.containsKey(d)) {
                if (window.get(d).equals(need.get(d))) {
                    valid--;
                }
                window.put(d, window.get(d) - 1);
            }
        }
    }
    
    return len == Integer.MAX_VALUE ? "" : s.substring(start, start + len);
}
```

### 3. Frequency Counting
```java
// Character frequency
public Map<Character, Integer> getCharFrequency(String s) {
    Map<Character, Integer> freq = new HashMap<>();
    for (char c : s.toCharArray()) {
        freq.put(c, freq.getOrDefault(c, 0) + 1);
    }
    return freq;
}

// Check if two strings are anagrams
public boolean areAnagrams(String s1, String s2) {
    if (s1.length() != s2.length()) return false;
    
    Map<Character, Integer> freq = new HashMap<>();
    for (char c : s1.toCharArray()) {
        freq.put(c, freq.getOrDefault(c, 0) + 1);
    }
    
    for (char c : s2.toCharArray()) {
        freq.put(c, freq.getOrDefault(c, 0) - 1);
        if (freq.get(c) == 0) {
            freq.remove(c);
        }
    }
    
    return freq.isEmpty();
}

// Find all anagrams in a string
public List<Integer> findAnagrams(String s, String p) {
    List<Integer> result = new ArrayList<>();
    if (s.length() < p.length()) return result;
    
    Map<Character, Integer> pCount = new HashMap<>();
    Map<Character, Integer> sCount = new HashMap<>();
    
    for (char c : p.toCharArray()) {
        pCount.put(c, pCount.getOrDefault(c, 0) + 1);
    }
    
    int windowSize = p.length();
    
    for (int i = 0; i < s.length(); i++) {
        char rightChar = s.charAt(i);
        sCount.put(rightChar, sCount.getOrDefault(rightChar, 0) + 1);
        
        if (i >= windowSize) {
            char leftChar = s.charAt(i - windowSize);
            sCount.put(leftChar, sCount.get(leftChar) - 1);
            if (sCount.get(leftChar) == 0) {
                sCount.remove(leftChar);
            }
        }
        
        if (sCount.equals(pCount)) {
            result.add(i - windowSize + 1);
        }
    }
    
    return result;
}
```

### 4. Stack for Parsing/Validation
```java
// Evaluate postfix expression
public int evalRPN(String[] tokens) {
    Deque<Integer> stack = new ArrayDeque<>();
    
    for (String token : tokens) {
        if ("+-*/".contains(token)) {
            int b = stack.pop();
            int a = stack.pop();
            switch (token) {
                case "+": stack.push(a + b); break;
                case "-": stack.push(a - b); break;
                case "*": stack.push(a * b); break;
                case "/": stack.push(a / b); break;
            }
        } else {
            stack.push(Integer.parseInt(token));
        }
    }
    
    return stack.pop();
}

// Basic calculator
public int calculate(String s) {
    Deque<Integer> stack = new ArrayDeque<>();
    int num = 0;
    char sign = '+';
    
    for (int i = 0; i < s.length(); i++) {
        char c = s.charAt(i);
        
        if (Character.isDigit(c)) {
            num = num * 10 + (c - '0');
        }
        
        if (c == '+' || c == '-' || c == '*' || c == '/' || i == s.length() - 1) {
            switch (sign) {
                case '+': stack.push(num); break;
                case '-': stack.push(-num); break;
                case '*': stack.push(stack.pop() * num); break;
                case '/': stack.push(stack.pop() / num); break;
            }
            sign = c;
            num = 0;
        }
    }
    
    int result = 0;
    while (!stack.isEmpty()) {
        result += stack.pop();
    }
    
    return result;
}

// Decode string
public String decodeString(String s) {
    Deque<Integer> countStack = new ArrayDeque<>();
    Deque<StringBuilder> stringStack = new ArrayDeque<>();
    StringBuilder current = new StringBuilder();
    int k = 0;
    
    for (char c : s.toCharArray()) {
        if (Character.isDigit(c)) {
            k = k * 10 + (c - '0');
        } else if (c == '[') {
            countStack.push(k);
            stringStack.push(current);
            current = new StringBuilder();
            k = 0;
        } else if (c == ']') {
            StringBuilder temp = current;
            current = stringStack.pop();
            int count = countStack.pop();
            for (int i = 0; i < count; i++) {
                current.append(temp);
            }
        } else {
            current.append(c);
        }
    }
    
    return current.toString();
}
```

### 5. Heap for Top K Problems
```java
// Find K largest elements
public int[] findKLargest(int[] nums, int k) {
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();
    
    for (int num : nums) {
        minHeap.offer(num);
        if (minHeap.size() > k) {
            minHeap.poll();
        }
    }
    
    int[] result = new int[k];
    for (int i = k - 1; i >= 0; i--) {
        result[i] = minHeap.poll();
    }
    
    return result;
}

// Top K frequent elements
public int[] topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int num : nums) {
        freq.put(num, freq.getOrDefault(num, 0) + 1);
    }
    
    PriorityQueue<Map.Entry<Integer, Integer>> minHeap = 
        new PriorityQueue<>((a, b) -> a.getValue() - b.getValue());
    
    for (Map.Entry<Integer, Integer> entry : freq.entrySet()) {
        minHeap.offer(entry);
        if (minHeap.size() > k) {
            minHeap.poll();
        }
    }
    
    int[] result = new int[k];
    for (int i = k - 1; i >= 0; i--) {
        result[i] = minHeap.poll().getKey();
    }
    
    return result;
}

// Merge K sorted lists
public ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> pq = new PriorityQueue<>((a, b) -> a.val - b.val);
    
    for (ListNode list : lists) {
        if (list != null) {
            pq.offer(list);
        }
    }
    
    ListNode dummy = new ListNode(0);
    ListNode current = dummy;
    
    while (!pq.isEmpty()) {
        ListNode node = pq.poll();
        current.next = node;
        current = current.next;
        
        if (node.next != null) {
            pq.offer(node.next);
        }
    }
    
    return dummy.next;
}
```

### 6. BFS with Queue
```java
// Level order traversal
public List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    if (root == null) return result;
    
    Queue<TreeNode> queue = new ArrayDeque<>();
    queue.offer(root);
    
    while (!queue.isEmpty()) {
        int levelSize = queue.size();
        List<Integer> level = new ArrayList<>();
        
        for (int i = 0; i < levelSize; i++) {
            TreeNode node = queue.poll();
            level.add(node.val);
            
            if (node.left != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
        
        result.add(level);
    }
    
    return result;
}

// Shortest path in unweighted graph
public int shortestPath(int[][] grid, int[] start, int[] end) {
    int m = grid.length, n = grid[0].length;
    boolean[][] visited = new boolean[m][n];
    Queue<int[]> queue = new ArrayDeque<>();
    
    queue.offer(new int[]{start[0], start[1], 0}); // {row, col, distance}
    visited[start[0]][start[1]] = true;
    
    int[][] directions = {{-1, 0}, {1, 0}, {0, -1}, {0, 1}};
    
    while (!queue.isEmpty()) {
        int[] current = queue.poll();
        int row = current[0], col = current[1], dist = current[2];
        
        if (row == end[0] && col == end[1]) {
            return dist;
        }
        
        for (int[] dir : directions) {
            int newRow = row + dir[0];
            int newCol = col + dir[1];
            
            if (newRow >= 0 && newRow < m && newCol >= 0 && newCol < n 
                && !visited[newRow][newCol] && grid[newRow][newCol] == 0) {
                visited[newRow][newCol] = true;
                queue.offer(new int[]{newRow, newCol, dist + 1});
            }
        }
    }
    
    return -1; // No path found
}

// Word ladder
public int ladderLength(String beginWord, String endWord, List<String> wordList) {
    Set<String> wordSet = new HashSet<>(wordList);
    if (!wordSet.contains(endWord)) return 0;
    
    Queue<String> queue = new ArrayDeque<>();
    Set<String> visited = new HashSet<>();
    
    queue.offer(beginWord);
    visited.add(beginWord);
    int level = 1;
    
    while (!queue.isEmpty()) {
        int size = queue.size();
        
        for (int i = 0; i < size; i++) {
            String word = queue.poll();
            
            if (word.equals(endWord)) {
                return level;
            }
            
            for (int j = 0; j < word.length(); j++) {
                char[] chars = word.toCharArray();
                for (char c = 'a'; c <= 'z'; c++) {
                    chars[j] = c;
                    String newWord = new String(chars);
                    
                    if (wordSet.contains(newWord) && !visited.contains(newWord)) {
                        visited.add(newWord);
                        queue.offer(newWord);
                    }
                }
            }
        }
        
        level++;
    }
    
    return 0;
}
```

### 7. DFS with Stack (Iterative)
```java
// Iterative DFS for tree traversal
public void dfsIterative(TreeNode root) {
    if (root == null) return;
    
    Deque<TreeNode> stack = new ArrayDeque<>();
    stack.push(root);
    
    while (!stack.isEmpty()) {
        TreeNode node = stack.pop();
        System.out.println(node.val); // Process node
        
        // Push right first, then left (so left is processed first)
        if (node.right != null) stack.push(node.right);
        if (node.left != null) stack.push(node.left);
    }
}

// Path sum in binary tree
public boolean hasPathSum(TreeNode root, int targetSum) {
    if (root == null) return false;
    
    Deque<TreeNode> nodeStack = new ArrayDeque<>();
    Deque<Integer> sumStack = new ArrayDeque<>();
    
    nodeStack.push(root);
    sumStack.push(targetSum - root.val);
    
    while (!nodeStack.isEmpty()) {
        TreeNode node = nodeStack.pop();
        int currentSum = sumStack.pop();
        
        if (node.left == null && node.right == null && currentSum == 0) {
            return true;
        }
        
        if (node.left != null) {
            nodeStack.push(node.left);
            sumStack.push(currentSum - node.left.val);
        }
        
        if (node.right != null) {
            nodeStack.push(node.right);
            sumStack.push(currentSum - node.right.val);
        }
    }
    
    return false;
}

// Number of islands using DFS
public int numIslands(char[][] grid) {
    if (grid == null || grid.length == 0) return 0;
    
    int count = 0;
    int m = grid.length, n = grid[0].length;
    
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (grid[i][j] == '1') {
                count++;
                dfsMarkIsland(grid, i, j);
            }
        }
    }
    
    return count;
}

private void dfsMarkIsland(char[][] grid, int i, int j) {
    Deque<int[]> stack = new ArrayDeque<>();
    stack.push(new int[]{i, j});
    
    int[][] directions = {{-1, 0}, {1, 0}, {0, -1}, {0, 1}};
    
    while (!stack.isEmpty()) {
        int[] current = stack.pop();
        int row = current[0], col = current[1];
        
        if (row < 0 || row >= grid.length || col < 0 || col >= grid[0].length 
            || grid[row][col] != '1') {
            continue;
        }
        
        grid[row][col] = '0'; // Mark as visited
        
        for (int[] dir : directions) {
            stack.push(new int[]{row + dir[0], col + dir[1]});
        }
    }
}
```

---

## Memory Management Tips

### 1. Choose the Right Data Structure
```java
// Use ArrayList for frequent random access
List<Integer> list = new ArrayList<>(); // O(1) access by index

// Use LinkedList for frequent insertions/deletions at beginning/end
List<Integer> list = new LinkedList<>(); // O(1) add/remove at ends

// Use HashMap for fast lookups
Map<String, Integer> map = new HashMap<>(); // O(1) average lookup

// Use TreeMap when you need sorted keys
Map<String, Integer> sortedMap = new TreeMap<>(); // O(log n) operations, sorted keys

// Use HashSet for uniqueness checking
Set<Integer> uniqueElements = new HashSet<>(); // O(1) contains operation

// Use TreeSet for sorted unique elements
Set<Integer> sortedUnique = new TreeSet<>(); // O(log n) operations, sorted
```

### 2. Initialize with Proper Capacity
```java
// Avoid frequent resizing by setting initial capacity
List<Integer> list = new ArrayList<>(1000);
Map<String, Integer> map = new HashMap<>(1000);
Set<Integer> set = new HashSet<>(1000);

// For HashMap, also consider load factor
Map<String, Integer> map = new HashMap<>(1000, 0.75f);
```

### 3. Use Primitive Collections for Large Datasets
```java
// Instead of List<Integer>, consider primitive collections
// (requires external libraries like Eclipse Collections or Trove)

// Avoid autoboxing overhead
int[] primitiveArray = new int[1000]; // More memory efficient
Integer[] boxedArray = new Integer[1000]; // Uses more memory due to object overhead
```

### 4. Memory-Conscious String Operations
```java
// Good: Use StringBuilder for string concatenation
StringBuilder sb = new StringBuilder();
for (String s : strings) {
    sb.append(s);
}
String result = sb.toString();

// Bad: Creates many intermediate String objects
String result = "";
for (String s : strings) {
    result += s; // Creates new String each time
}

// Use StringJoiner for delimiter-separated strings
StringJoiner joiner = new StringJoiner(", ");
for (String s : strings) {
    joiner.add(s);
}
String result = joiner.toString();
```

### 5. Efficient Iteration
```java
// Use enhanced for loop when you don't need index
for (Integer num : list) {
    // Process num
}

// Use traditional for loop when you need index
for (int i = 0; i < list.size(); i++) {
    // Process list.get(i)
}

// Use iterator for safe removal during iteration
Iterator<Integer> it = list.iterator();
while (it.hasNext()) {
    Integer num = it.next();
    if (shouldRemove(num)) {
        it.remove(); // Safe removal
    }
}

// Use streams for functional operations
list.stream()
    .filter(x -> x > 0)
    .map(x -> x * 2)
    .collect(Collectors.toList());
```

### 6. Avoid Common Pitfalls
```java
// Don't modify collection while iterating (except with iterator.remove())
// Bad:
for (Integer num : list) {
    if (num < 0) {
        list.remove(num); // ConcurrentModificationException
    }
}

// Good:
list.removeIf(num -> num < 0);

// Be careful with null values
Map<String, List<Integer>> map = new HashMap<>();
// Always check for null or use computeIfAbsent
List<Integer> list = map.computeIfAbsent("key", k -> new ArrayList<>());

// Use appropriate equals() and hashCode() for custom objects in HashSet/HashMap
public class Person {
    private String name;
    private int age;
    
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        Person person = (Person) obj;
        return age == person.age && Objects.equals(name, person.name);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }
}
```

---

## Quick Reference for Common Operations

### Array to Collection Conversions
```java
// Array to List
int[] arr = {1, 2, 3, 4, 5};
List<Integer> list = Arrays.stream(arr).boxed().collect(Collectors.toList());

// List to Array
List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);
int[] arr = list.stream().mapToInt(Integer::intValue).toArray();

// Array to Set
Set<Integer> set = Arrays.stream(arr).boxed().collect(Collectors.toSet());
```

### Sorting Operations
```java
// Sort list
Collections.sort(list);
Collections.sort(list, Collections.reverseOrder());
Collections.sort(list, (a, b) -> a.compareTo(b));

// Sort array
Arrays.sort(arr);
Arrays.sort(arr, Collections.reverseOrder()); // For Integer[]
Arrays.sort(arr, (a, b) -> Integer.compare(a, b));

// Custom object sorting
Collections.sort(people, (p1, p2) -> p1.getName().compareTo(p2.getName()));
Collections.sort(people, Comparator.comparing(Person::getName));
```

### Binary Search
```java
// In sorted list
int index = Collections.binarySearch(list, target);

// In sorted array
int index = Arrays.binarySearch(arr, target);

// Custom binary search
public int binarySearch(int[] arr, int target) {
    int left = 0, right = arr.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (arr[mid] == target) return mid;
        else if (arr[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}
```

This comprehensive cheat sheet covers all the essential primitive data types and Collections framework components you'll need for DSA in Java. Practice implementing algorithms using these data structures to build proficiency and understand when to use each one!
