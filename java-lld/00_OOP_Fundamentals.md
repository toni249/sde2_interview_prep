# OOP Fundamentals — Deep Dive for SDE2 Interviews

> These aren't just definitions. Every question below is a real follow-up an interviewer can ask.
> Master the "why" behind each concept.

---

## The 4 Pillars

---

## 1. Encapsulation

**Core idea:** Bundle data + behavior that operates on that data inside one unit. Hide internal state; expose only what's necessary through a controlled interface.

**Mental Model:** A capsule (pill). The medicine (data) is inside. You don't directly touch it — the capsule controls how it's released.

```java
// BAD — direct field access, anyone can corrupt state
class BankAccount {
    public double balance;  // anyone can set balance = -999
}

// GOOD — data is hidden, access is controlled
class BankAccount {
    private double balance;    // private = hidden
    private String owner;

    public BankAccount(String owner, double initialBalance) {
        this.owner = owner;
        if (initialBalance < 0) throw new IllegalArgumentException("Balance can't be negative");
        this.balance = initialBalance;
    }

    public double getBalance() { return balance; }

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Deposit must be positive");
        this.balance += amount;
    }

    public void withdraw(double amount) {
        if (amount > balance) throw new IllegalStateException("Insufficient funds");
        this.balance -= amount;
    }
}
```

**Why it matters in LLD:**
- Invariants are enforced in one place (`balance >= 0` rule lives only in `BankAccount`)
- You can change internal representation (e.g., store in paise instead of rupees) without touching callers
- Enables thread safety: `synchronized` on the method protects the encapsulated state

### Follow-up Questions

**Q: Encapsulation vs Abstraction — what's the difference?**
> Encapsulation is about **hiding data** (the "how it's stored"). Abstraction is about **hiding complexity** (the "how it works"). Encapsulation is implemented via `private` fields. Abstraction is implemented via interfaces/abstract classes.
>
> Example: A `List` interface abstracts "ordered collection". `ArrayList` encapsulates a `Object[]` array internally.

**Q: Can you have encapsulation without OOP?**
> Yes. In C, you can use a struct with functions that operate on it, keeping the struct definition in a `.c` file and exposing only the function headers in `.h`. This is encapsulation without OOP language constructs.

**Q: How does encapsulation help with thread safety?**
> If state is private and only modified through synchronized methods, no external code can bypass the synchronization. Without encapsulation, even if you synchronize internally, external code can still read/write `balance` directly, creating race conditions.

**Q: What's the problem with JavaBeans (getters/setters for every field)?**
> Getters/setters on mutable fields break encapsulation. A `setBalance()` method with no validation is as dangerous as a `public` field. True encapsulation means exposing **behavior** (`deposit`, `withdraw`), not **data accessors**.

---

## 2. Abstraction

**Core idea:** Expose WHAT an object does, hide HOW it does it. Work with contracts (interfaces), not implementations.

**Mental Model:** You drive a car without knowing how the engine works. The steering wheel is the interface; the engine internals are the abstraction.

```java
// The CONTRACT — callers know "what" but not "how"
interface PaymentGateway {
    boolean processPayment(double amount, String currency);
    boolean refund(String transactionId);
}

// ONE implementation — Razorpay
class RazorpayGateway implements PaymentGateway {
    private final RazorpayClient client;  // internal detail, hidden

    public boolean processPayment(double amount, String currency) {
        // Complex Razorpay API calls hidden here
        return client.charge(amount, currency).isSuccessful();
    }

    public boolean refund(String transactionId) {
        return client.createRefund(transactionId).isSuccessful();
    }
}

// ANOTHER implementation — Stripe
class StripeGateway implements PaymentGateway {
    private final Stripe stripeClient;

    public boolean processPayment(double amount, String currency) {
        // Stripe-specific logic, completely different internally
        PaymentIntent intent = PaymentIntent.create(amount, currency);
        return intent.getStatus().equals("succeeded");
    }
    // ...
}

// CheckoutService doesn't care which gateway — it works with the abstraction
class CheckoutService {
    private final PaymentGateway gateway;  // depends on abstraction!

    public CheckoutService(PaymentGateway gateway) { this.gateway = gateway; }

    public void checkout(Order order) {
        if (!gateway.processPayment(order.getTotal(), "INR")) {
            throw new PaymentFailedException("Payment failed");
        }
        order.markPaid();
    }
}
```

### Follow-up Questions

**Q: Interface vs Abstract Class — when to use which?**

| | Interface | Abstract Class |
|---|---|---|
| Purpose | Pure contract / capability | Partial implementation + contract |
| Multiple inheritance | Yes (Java allows multiple interfaces) | No (single class inheritance) |
| State (fields) | No (only constants) | Yes |
| Constructor | No | Yes |
| When to use | "Can do" relationships (`Flyable`, `Serializable`) | "Is a" with shared code (`AbstractAnimal`) |

> Rule of thumb: **Default to interfaces**. Use abstract class only when you have shared state or a genuine "is-a" + partial implementation.

**Q: What's the problem with using abstract classes everywhere?**
> Java doesn't support multiple inheritance for classes. If you extend `AbstractAnimal`, you can't also extend `AbstractVehicle`. Interfaces don't have this constraint. Over-relying on abstract classes creates inflexible hierarchies.

**Q: Can an interface have concrete methods in Java?**
> Yes, since Java 8, interfaces can have `default` methods (concrete) and `static` methods. Since Java 9, they can have `private` methods (helpers for defaults). This was added to allow adding methods to interfaces without breaking all implementing classes.

---

## 3. Inheritance

**Core idea:** A child class acquires the properties and behaviors of a parent class. Enables code reuse and IS-A relationships.

**Mental Model:** A `SavingsAccount` IS-A `BankAccount`. It inherits all basic account behavior and adds interest calculation.

```java
class BankAccount {
    protected double balance;
    protected String accountNumber;

    public BankAccount(String accountNumber, double initialBalance) {
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
    }

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException();
        balance += amount;
    }

    public double getBalance() { return balance; }

    public String getAccountInfo() {
        return "Account: " + accountNumber + ", Balance: " + balance;
    }
}

class SavingsAccount extends BankAccount {
    private double interestRate;

    public SavingsAccount(String accountNumber, double balance, double interestRate) {
        super(accountNumber, balance);  // call parent constructor
        this.interestRate = interestRate;
    }

    public void addInterest() {
        double interest = balance * interestRate;
        deposit(interest);  // reuses parent's deposit logic
    }

    @Override
    public String getAccountInfo() {
        return super.getAccountInfo() + ", Rate: " + interestRate;  // extends parent behavior
    }
}
```

### Follow-up Questions

**Q: "Composition over Inheritance" — explain this principle.**
> Inheritance creates tight coupling between parent and child. If you change the parent, it can break all subclasses. Composition (HAS-A) is more flexible: you inject a collaborator and can swap it at runtime.
>
> Example: Instead of `LoggingOrderService extends OrderService`, prefer:
> ```java
> class LoggingOrderService {
>     private final OrderService delegate;   // composition
>     private final Logger logger;
>
>     public void process(Order o) {
>         logger.log("Processing: " + o.getId());
>         delegate.process(o);               // delegate, not override
>     }
> }
> ```
> The Decorator pattern is formalization of this idea.

**Q: What is the Fragile Base Class problem?**
> If a base class method is called internally within the base class, and a subclass overrides it, the subclass method gets called from the base class context — which can cause unexpected behavior or infinite loops.
>
> Example: `HashSet.addAll()` calls `add()` internally. If you subclass `HashSet` and override `add()` to count additions, `addAll(5 items)` will count 10 instead of 5. This is why `@Override` is important and why you should prefer composition.

**Q: Is-A vs Has-A — how do you decide?**
> Ask: "Is this relationship always true regardless of context?" A `Dog` IS-A `Animal` forever. But a `Car` is not necessarily `IS-A Engine`; it HAS-A Engine.
> If you're unsure, choose Has-A (composition). You can always add an interface later if needed.

**Q: What's the difference between method overloading and overriding?**
> - **Overloading:** Same method name, different parameters. Resolved at **compile time** (static dispatch). Not polymorphism.
> - **Overriding:** Child redefines parent method with same signature. Resolved at **runtime** (dynamic dispatch). This IS polymorphism.

---

## 4. Polymorphism

**Core idea:** One interface, many forms. The same method call behaves differently depending on the actual object type at runtime.

**Mental Model:** A `Shape.area()` call works correctly whether the shape is a Circle, Rectangle, or Triangle — each computes area differently.

```java
// Compile-time polymorphism (Method Overloading)
class Calculator {
    int add(int a, int b) { return a + b; }
    double add(double a, double b) { return a + b; }            // same name, different params
    int add(int a, int b, int c) { return a + b + c; }          // resolved at compile time
}

// Runtime polymorphism (Method Overriding) — THE IMPORTANT ONE
abstract class Shape {
    abstract double area();  // contract

    void printInfo() {
        System.out.println("Area = " + area());  // polymorphic call
    }
}

class Circle extends Shape {
    private double radius;
    Circle(double r) { this.radius = r; }

    @Override
    double area() { return Math.PI * radius * radius; }
}

class Rectangle extends Shape {
    private double width, height;
    Rectangle(double w, double h) { this.width = w; this.height = h; }

    @Override
    double area() { return width * height; }
}

// Usage — caller doesn't need to know which shape
List<Shape> shapes = List.of(new Circle(5), new Rectangle(3, 4), new Circle(2));
for (Shape s : shapes) {
    s.printInfo();  // Java resolves the correct area() at RUNTIME via vtable lookup
}
```

### Follow-up Questions

**Q: How does Java implement runtime polymorphism internally?**
> Through a **Virtual Method Table (vtable)**. Every class has a vtable — a table of pointers to its method implementations. When you call `shape.area()`, the JVM looks up the vtable of the actual object's class (not the reference type) and dispatches to the correct method. This is called **dynamic dispatch**.

**Q: What is covariant return type?**
> In Java 5+, an overriding method can return a more specific (subtype) return type than the parent.
> ```java
> class Animal { Animal create() { return new Animal(); } }
> class Dog extends Animal {
>     @Override Dog create() { return new Dog(); }  // covariant return — Dog is subtype of Animal
> }
> ```

**Q: Can constructors be polymorphic?**
> No. Constructors are not inherited and cannot be overridden. However, a parent class constructor can call an overridden method (polymorphism), which is a common pitfall — the child method runs before the child's constructor, potentially accessing uninitialized fields.

**Q: `instanceof` vs polymorphism — when should you use each?**
> Using `instanceof` followed by a cast is a code smell — it means you're not leveraging polymorphism properly. Prefer adding a method to the interface/abstract class. Use `instanceof` only at system boundaries (deserialization, external input parsing) where you genuinely don't know the type.

---

## Key Relationships: Association, Aggregation, Composition

These determine how classes relate to each other — critical for drawing class diagrams.

```
Association:  A uses B (loose, no ownership)
Aggregation:  A has B (B can exist without A — "weak" has-a)
Composition:  A owns B (B cannot exist without A — "strong" has-a)
Inheritance:  A is-a B
```

```java
// ASSOCIATION — Teacher "knows" Student, but doesn't own them
class Teacher {
    void teach(Student student) { ... }  // just a method parameter
}

// AGGREGATION — Department has Professors, but Professors exist independently
class Department {
    private List<Professor> professors;  // Professors can belong to multiple departments
    void addProfessor(Professor p) { professors.add(p); }
}

// COMPOSITION — House contains Rooms, Rooms cannot exist without a House
class House {
    private final List<Room> rooms;
    House() {
        rooms = new ArrayList<>();
        rooms.add(new Room("Bedroom"));   // created inside House
        rooms.add(new Room("Kitchen"));   // lifecycle tied to House
    }
    // When House is destroyed, Rooms are destroyed too
}
```

### Follow-up Questions

**Q: When would you choose Aggregation over Composition?**
> When the contained objects have an independent lifecycle or are shared. A `Team` has `Players`, but `Players` exist even without that team (they can be transferred). If you model this as composition, you'd have to destroy and recreate the player when transferring — wrong.

**Q: How does Composition relate to dependency injection?**
> Composition via constructor injection is the recommended approach. The composed object is passed in (not created inside), which makes it testable (you can pass a mock). This is the DIP principle in action.

---

## OOP in Concurrent Contexts

**Key insight for interviews:** OOP doesn't automatically give you thread safety. These are orthogonal concerns.

```java
// This looks well-encapsulated but is NOT thread-safe
class Counter {
    private int count = 0;  // private field

    public void increment() { count++; }  // count++ is NOT atomic! (read-modify-write)
    public int get() { return count; }
}

// Thread-safe option 1: synchronized
class SynchronizedCounter {
    private int count = 0;
    public synchronized void increment() { count++; }
    public synchronized int get() { return count; }
}

// Thread-safe option 2: AtomicInteger (preferred for simple counters)
class AtomicCounter {
    private final AtomicInteger count = new AtomicInteger(0);
    public void increment() { count.incrementAndGet(); }  // CAS operation, no lock
    public int get() { return count.get(); }
}
```

**Q: Why is `count++` not thread-safe even if `count` is `private`?**
> `count++` compiles to 3 bytecode instructions: READ count, ADD 1, WRITE count. Two threads can both READ the same value, both add 1, and both WRITE the same incremented value — net result: count increased by 1 instead of 2. Encapsulation hides the field; it doesn't synchronize the operations.
