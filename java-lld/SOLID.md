# 🎓 Tutor's Guide: SOLID Principles

Hey! Let's nail the fundamentals. SOLID isn't about memorizing definitions; it's about **Cohesion** (things that belong together stay together) and **Coupling** (things shouldn't depend on each other too much).

---

## 1. Single Responsibility Principle (SRP)
**Mental Model:** "One class, one actor."
If HR and Finance both use a class, and HR wants a change, Finance shouldn't even know it happened.

### 💻 Code Brief
```java
// ❌ BAD: This class is a "God Object". It knows too much.
class OrderManager {
    void processOrder() { /* Logic */ }
    void saveToDB() { /* Logic */ }
    void sendEmail() { /* Logic */ }
}

// ✅ GOOD: Each class has one reason to change.
class OrderService { void process() { ... } }
class OrderRepository { void save() { ... } }
class NotificationService { void send() { ... } }
```

### 🗣️ Interviewer Drill:
*   **Q:** "Doesn't SRP lead to too many small classes?"
*   **A:** "Yes, it increases the number of classes, but it significantly reduces the cost of change. Small classes are easier to test in isolation and more reusable."

---

## 2. Open/Closed Principle (OCP)
**Mental Model:** "Plugin Architecture."
You should be able to add a new "plugin" without opening the "core" engine.

### 💻 Code Brief
```java
// ✅ GOOD: Use Interfaces. The Engine stays closed.
interface PaymentMethod { void pay(double amount); }

class UPI implements PaymentMethod { public void pay(double a) { ... } }
class CreditCard implements PaymentMethod { public void pay(double a) { ... } }

class CheckoutEngine {
    void checkout(PaymentMethod method, double amount) {
        method.pay(amount); // No if-else needed!
    }
}
```

### 🗣️ Interviewer Drill:
*   **Q:** "How do you achieve OCP in an existing legacy codebase with many `if-else` blocks?"
*   **A:** "By using the **Strategy Pattern**. I'd move each condition into its own class implementing a common interface."

---

## 3. Liskov Substitution Principle (LSP)
**Mental Model:** "No Surprises."
If I give you a `Child` instead of a `Parent`, your code shouldn't crash or behave weirdly.

### 💻 Code Brief
```java
// ❌ BAD: Ostrich is a Bird, but it can't fly. 
// Calling fly() on an Ostrich throws an Exception -> Breaks the system.
class Bird { void fly() { ... } }
class Ostrich extends Bird { void fly() { throw new RuntimeException(); } }

// ✅ GOOD: Split interfaces based on capabilities.
interface Flyable { void fly(); }
class Sparrow implements Flyable { ... }
class Ostrich { /* Only methods it actually supports */ }
```

---

## 4. Interface Segregation Principle (ISP)
**Mental Model:** "Don't force clients to implement dummy methods."

### 💻 Code Brief
```java
// ❌ BAD: One giant interface.
interface Worker { void work(); void eat(); }

// ✅ GOOD: Lean, specific interfaces.
interface Workable { void work(); }
interface Eatable { void eat(); }

class Robot implements Workable { public void work() { ... } } // No dummy eat()!
```

---

## 5. Dependency Inversion Principle (DIP)
**Mental Model:** "Depend on abstractions, not concretions."

### 💻 Code Brief
```java
// ✅ GOOD: The Service doesn't care IF it's MySQL or MongoDB.
interface Database { void save(String data); }

class MyService {
    private final Database db;
    public MyService(Database db) { this.db = db; } // Constructor Injection
    void perform() { db.save("data"); }
}
```

### 🗣️ Interviewer Drill:
*   **Q:** "What’s the difference between DIP and Dependency Injection (DI)?"
*   **A:** "DIP is the **High-level Design Principle** (the 'What'). DI is a **Design Pattern/Technique** (the 'How') used to achieve DIP."
