# UML & Class Relationships — Interview Crash Course

> You don't need perfect UML notation. You need to clearly communicate class structure.
> Learn the 6 relationships and when each applies.

---

## The 6 Relationships (Weakest → Strongest Coupling)

```
Dependency ──────── (dashed arrow)         weakest
Association ─────── (solid arrow)
Aggregation ─────── (open diamond)
Composition ─────── (filled diamond)
Realization ─────── (dashed + triangle)    interface implementation
Inheritance ─────── (solid + triangle)     strongest
```

---

## 1. Dependency (Uses-A, Weakest)

Class A *uses* class B but doesn't hold a reference. B appears as a method parameter, local variable, or return type.

```java
class OrderService {
    // B appears ONLY in method signature — no field reference
    public Receipt process(Order order, PaymentGateway gateway) {
        gateway.charge(order.getTotal());
        return new Receipt(order);
    }
}
```

**UML:** `OrderService ----> PaymentGateway` (dashed arrow)

**Interview note:** Minimize dependencies. High dependency = high coupling = hard to test and change.

---

## 2. Association (Knows-A)

Class A holds a reference to B as a field. No lifecycle tie — B exists independently of A.

```java
class Student {
    private List<Course> enrolledCourses;  // A holds reference to B

    public void enroll(Course course) {
        enrolledCourses.add(course);
    }
}
```

**Bidirectional Association** — both sides hold references:
```java
class Teacher { List<Student> students; }
class Student { Teacher teacher; }
// Be careful: bidirectional = more coupling, need to keep both sides in sync
```

**UML:** `Student ──── Course`

---

## 3. Aggregation (Has-A, Weak Ownership)

Class A "has" class B. B can exist without A. B's lifetime is NOT controlled by A.

```java
class Department {
    private List<Employee> employees;  // Employees exist independently

    public Department(List<Employee> employees) {
        this.employees = employees;  // passed in, not created here
    }

    public void dissolve() {
        // Department is gone, but Employee objects still exist
        employees = null;
    }
}
```

**UML:** `Department <>──── Employee` (open diamond on Department)

**Key test:** If A is destroyed, does B get destroyed too? No → Aggregation.

---

## 4. Composition (Owns-A, Strong Ownership)

Class A creates and owns B. B cannot exist without A. Lifecycle is tied.

```java
class House {
    private final List<Room> rooms;

    public House(int numRooms) {
        rooms = new ArrayList<>();
        for (int i = 0; i < numRooms; i++) {
            rooms.add(new Room());  // created INSIDE House, owned by House
        }
    }
    // When House is GC'd, all Rooms are GC'd
}
```

**UML:** `House ◆──── Room` (filled diamond on House)

**Key test:** If A is destroyed, is B also destroyed? Yes → Composition.

---

## 5. Realization / Implementation

Class A implements interface B.

```java
class Sparrow implements Flyable {
    @Override
    public void fly() { ... }
}
```

**UML:** `Sparrow - - - ▷ Flyable` (dashed line with open triangle)

---

## 6. Inheritance (Is-A, Strongest)

Class A extends class B. A is a specialized form of B.

```java
class SavingsAccount extends BankAccount { ... }
```

**UML:** `SavingsAccount ──── ▷ BankAccount` (solid line with open triangle)

---

## Common Confusion: Aggregation vs Composition

| | Aggregation | Composition |
|---|---|---|
| Ownership | Shared / none | Exclusive |
| Lifetime | Independent | Dependent |
| Null allowed? | Yes (member can be null) | No (must always exist) |
| Example | `University` has `Professors` | `Order` has `OrderItems` |
| UML | Open diamond | Filled diamond |

**Rule of thumb:** Can the part be transferred to another whole? If yes → Aggregation. If no → Composition.

---

## Quick Class Diagram Template

For any LLD problem, use this template in interviews:

```
+---------------------------+
|       ClassName           |  ← Class name (bold if concrete, italic if abstract)
+---------------------------+
| - privateField: Type      |  ← Fields (- private, + public, # protected)
| + publicField: Type       |
+---------------------------+
| + methodName(): ReturnType|  ← Methods
| - helper(): void          |
+---------------------------+
```

---

## Parking Lot — Quick Class Diagram Example

```
         ParkingLot
         ──────────────
         - floors: List<Floor>           ◆ (composition)
         - ticketCounter: int
         + park(Vehicle): Ticket
         + leave(Ticket): Payment
              |
              ▼ (composition)
           Floor
           ──────────────
           - slots: List<ParkingSlot>    ◆ (composition)
           - floorNumber: int
              |
              ▼ (composition)
         ParkingSlot
         ──────────────
         - slotType: SlotType           (enum)
         - status: SlotStatus           (enum)
         - vehicle: Vehicle             <> (aggregation — vehicle exists outside)

    Vehicle (abstract)
    ──────────────
    - licensePlate: String
    - vehicleType: VehicleType
        △ (inheritance)
        |
    ┌──────────┬──────────┐
  Car       Bike       Truck
```

---

## OOP Relationship Interview Questions

**Q: An Order has OrderItems. When an Order is deleted, its items are deleted too. What relationship is this?**
> Composition. OrderItems don't exist independently of their Order.

**Q: A Teacher teaches multiple Students. What relationship?**
> Association (bidirectional). Both exist independently; neither owns the other.

**Q: What's the difference between `has-a` implemented via field vs via method parameter?**
> Field = Association or Aggregation or Composition (depending on lifecycle).
> Method parameter = Dependency (weakest form). A method parameter doesn't mean the class "has" that object.

**Q: In your design, `PaymentService` takes `EmailNotifier` in its constructor. What relationship is this? How does it differ from creating `EmailNotifier` inside the method?**
> Constructor injection → Association (or Aggregation if `EmailNotifier` exists elsewhere too).
> Creating inside the method → Composition (owned, lifecycle-tied) OR just Dependency if it's a local variable.
> Constructor injection is preferred because it makes dependencies explicit and enables testing with mocks.

**Q: When would you use Inheritance vs Composition for a logging feature?**
> Prefer Composition. If you use inheritance (`class LoggingOrderService extends OrderService`), you break if `OrderService` changes, and you can't apply logging to multiple classes. With composition (Decorator pattern), you wrap any service without coupling.

---

## Multiplicity in Relationships

```
1      — exactly one
0..1   — zero or one (optional)
*      — zero or more
1..*   — one or more
2..5   — specific range
```

**Example:** `Order (1) ◆──── (1..*) OrderItem`
- One Order must have at least one OrderItem
- One OrderItem belongs to exactly one Order

**Interview:** "An invoice can have many line items. A line item belongs to exactly one invoice."
> `Invoice (1) ◆──── (1..*) LineItem` — Composition with multiplicity.
