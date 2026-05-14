# LLD: Vending Machine

> Frequency: Very High | Difficulty: Medium
> Tests: State pattern (this IS the canonical State pattern problem), concurrency

---

## Step 1 — Clarifying Questions

- What operations? Insert money, select item, cancel, collect change?
- Multiple currencies / coin denominations?
- Admin operations? (restock items, collect money)
- Concurrent users? (one machine, one user at a time by design, but...?)
- What happens if item is out of stock? If machine has no change?

---

## Step 2 — State Machine (Draw This First in Interview)

```
    ┌─────────────────────────────────────────┐
    │                                         │
    ▼                                         │
  IDLE ──insertMoney──► HAS_MONEY             │
    ▲                      │                  │
    │                   selectItem            │
    │                      ▼                  │
    │               ITEM_SELECTED             │
    │                  │       │              │
    │              dispense   cancel          │
    │                  │       │              │
    │                  ▼       ▼              │
    └────────────── DISPENSING ──────────────►│
                       │                  (return to IDLE)
                    (done)
```

---

## Step 3 — Class Diagram

```
VendingMachine (Context)
├── currentState: VendingMachineState   [Aggregation — state is an object]
├── inventory: Map<String, Item>
├── insertedMoney: double
└── selectedItemCode: String

VendingMachineState (interface)
├── IdleState
├── HasMoneyState
├── ItemSelectedState
└── DispensingState

Item
├── code: String
├── name: String
├── price: double
└── quantity: int
```

---

## Step 4 — Full Java Code

```java
// ─── ITEM ───
public class Item {
    private final String code;
    private final String name;
    private final double price;
    private int quantity;

    public Item(String code, String name, double price, int quantity) {
        this.code = code;
        this.name = name;
        this.price = price;
        this.quantity = quantity;
    }

    public String getCode() { return code; }
    public String getName() { return name; }
    public double getPrice() { return price; }
    public int getQuantity() { return quantity; }

    public boolean isAvailable() { return quantity > 0; }

    public void decrementQuantity() {
        if (quantity <= 0) throw new IllegalStateException("Item out of stock: " + code);
        quantity--;
    }
}

// ─── STATE INTERFACE ───
public interface VendingMachineState {
    void insertMoney(VendingMachine machine, double amount);
    void selectItem(VendingMachine machine, String itemCode);
    void dispenseItem(VendingMachine machine);
    void cancel(VendingMachine machine);

    default String getStateName() { return this.getClass().getSimpleName(); }
}

// ─── STATE 1: IDLE ───
public class IdleState implements VendingMachineState {
    @Override
    public void insertMoney(VendingMachine machine, double amount) {
        if (amount <= 0) { System.out.println("Invalid amount"); return; }
        machine.addMoney(amount);
        System.out.println("Inserted: ₹" + amount);
        machine.setState(new HasMoneyState());
    }

    @Override
    public void selectItem(VendingMachine machine, String code) {
        System.out.println("Please insert money first");
    }

    @Override
    public void dispenseItem(VendingMachine machine) {
        System.out.println("Please insert money and select an item first");
    }

    @Override
    public void cancel(VendingMachine machine) {
        System.out.println("Nothing to cancel");
    }
}

// ─── STATE 2: HAS_MONEY ───
public class HasMoneyState implements VendingMachineState {
    @Override
    public void insertMoney(VendingMachine machine, double amount) {
        machine.addMoney(amount);
        System.out.println("Added: ₹" + amount + ". Total: ₹" + machine.getInsertedMoney());
    }

    @Override
    public void selectItem(VendingMachine machine, String code) {
        Item item = machine.getItem(code);
        if (item == null) { System.out.println("Item not found: " + code); return; }
        if (!item.isAvailable()) { System.out.println("Item out of stock: " + item.getName()); return; }
        if (machine.getInsertedMoney() < item.getPrice()) {
            double needed = item.getPrice() - machine.getInsertedMoney();
            System.out.println("Need ₹" + needed + " more for " + item.getName());
            return;
        }
        machine.setSelectedItem(code);
        System.out.println("Selected: " + item.getName() + " @ ₹" + item.getPrice());
        machine.setState(new ItemSelectedState());
    }

    @Override
    public void dispenseItem(VendingMachine machine) {
        System.out.println("Please select an item first");
    }

    @Override
    public void cancel(VendingMachine machine) {
        System.out.println("Returning ₹" + machine.getInsertedMoney());
        machine.resetMoney();
        machine.setState(new IdleState());
    }
}

// ─── STATE 3: ITEM_SELECTED ───
public class ItemSelectedState implements VendingMachineState {
    @Override
    public void insertMoney(VendingMachine machine, double amount) {
        System.out.println("Item already selected. Please dispense or cancel first.");
    }

    @Override
    public void selectItem(VendingMachine machine, String code) {
        // Allow changing selection before dispensing
        Item item = machine.getItem(code);
        if (item != null && item.isAvailable() && machine.getInsertedMoney() >= item.getPrice()) {
            machine.setSelectedItem(code);
            System.out.println("Changed selection to: " + item.getName());
        } else {
            System.out.println("Cannot switch to " + code);
        }
    }

    @Override
    public void dispenseItem(VendingMachine machine) {
        machine.setState(new DispensingState());
        machine.dispenseItem();  // actual dispensing in Dispensing state
    }

    @Override
    public void cancel(VendingMachine machine) {
        System.out.println("Returning ₹" + machine.getInsertedMoney());
        machine.resetMoney();
        machine.setSelectedItem(null);
        machine.setState(new IdleState());
    }
}

// ─── STATE 4: DISPENSING ───
public class DispensingState implements VendingMachineState {
    @Override
    public void insertMoney(VendingMachine machine, double amount) {
        System.out.println("Please wait — dispensing in progress");
    }

    @Override
    public void selectItem(VendingMachine machine, String code) {
        System.out.println("Please wait — dispensing in progress");
    }

    @Override
    public void dispenseItem(VendingMachine machine) {
        Item item = machine.getItem(machine.getSelectedItem());
        item.decrementQuantity();  // reduce inventory

        double change = machine.getInsertedMoney() - item.getPrice();
        System.out.println("Dispensing: " + item.getName());
        if (change > 0) System.out.println("Change returned: ₹" + change);

        machine.resetMoney();
        machine.setSelectedItem(null);
        machine.setState(new IdleState());
    }

    @Override
    public void cancel(VendingMachine machine) {
        System.out.println("Cannot cancel — already dispensing");
    }
}

// ─── CONTEXT: VENDING MACHINE ───
public class VendingMachine {
    private VendingMachineState currentState;
    private double insertedMoney;
    private String selectedItemCode;
    private final Map<String, Item> inventory;

    public VendingMachine() {
        this.inventory = new HashMap<>();
        this.currentState = new IdleState();
        this.insertedMoney = 0;
    }

    // Admin: add items
    public void addItem(Item item) { inventory.put(item.getCode(), item); }

    // Public operations — all delegated to current state
    public synchronized void insertMoney(double amount) {
        currentState.insertMoney(this, amount);
    }

    public synchronized void selectItem(String code) {
        currentState.selectItem(this, code);
    }

    public synchronized void dispenseItem() {
        currentState.dispenseItem(this);
    }

    public synchronized void cancel() {
        currentState.cancel(this);
    }

    // State-accessible methods (package-visible or public)
    public void setState(VendingMachineState state) {
        System.out.println("[State] " + currentState.getStateName() + " → " + state.getStateName());
        this.currentState = state;
    }

    public void addMoney(double amount) { this.insertedMoney += amount; }
    public void resetMoney() { this.insertedMoney = 0; }
    public double getInsertedMoney() { return insertedMoney; }
    public void setSelectedItem(String code) { this.selectedItemCode = code; }
    public String getSelectedItem() { return selectedItemCode; }
    public Item getItem(String code) { return inventory.get(code); }

    public void displayStatus() {
        System.out.println("State: " + currentState.getStateName() +
            " | Money: ₹" + insertedMoney);
        inventory.values().forEach(i ->
            System.out.println("  " + i.getCode() + ": " + i.getName() +
                " @ ₹" + i.getPrice() + " [" + i.getQuantity() + " left]"));
    }
}
```

**Usage:**
```java
VendingMachine vm = new VendingMachine();
vm.addItem(new Item("A1", "Cola", 30.0, 5));
vm.addItem(new Item("B2", "Chips", 25.0, 3));

vm.insertMoney(20);    // IDLE → HAS_MONEY
vm.insertMoney(15);    // total ₹35
vm.selectItem("A1");   // HAS_MONEY → ITEM_SELECTED
vm.dispenseItem();     // ITEM_SELECTED → DISPENSING → IDLE, change ₹5 returned
vm.displayStatus();
```

---

## Step 5 — Concurrency Follow-up Questions

**Q: The public methods in VendingMachine are `synchronized`. Why is this necessary?**
> A vending machine is physically a single resource — only one user interaction at a time makes sense. The `synchronized` keyword on the Context methods ensures that even if two threads represent two users trying to interact simultaneously (perhaps via a remote API), state transitions are serialized. Without synchronization, two threads could both check `insertedMoney < item.getPrice()`, both see it's enough, and both try to dispense.

**Q: What if dispensing is a slow operation (mechanical movement)? Would you keep the lock during dispensing?**
> No — holding the lock during slow I/O blocks all other operations. Better approach: set state to `DISPENSING` (which blocks new interactions), release the lock, do the slow work asynchronously, then re-acquire lock to transition back to `IDLE`.
>
> ```java
> public synchronized void dispenseItem() {
>     currentState.dispenseItem(this);  // sets state to DISPENSING, schedules async work
> }
>
> // In DispensingState.dispenseItem():
> // - set state to DISPENSING
> // - submit slow work to executor
> // - executor callback: machine.completeDispensing() which resets to IDLE
> ```

**Q: Why are State objects stateless in this design? Is that always the case?**
> In this design, `IdleState`, `HasMoneyState` etc. have no fields — all state is in the `VendingMachine` context. This means State objects can be **shared** (e.g., use singleton State objects, reducing allocation):
> ```java
> public class VendingMachine {
>     private static final IdleState IDLE_STATE = new IdleState();     // shared, stateless
>     private static final HasMoneyState HAS_MONEY_STATE = new HasMoneyState();
>     // ...
> }
> ```
> If states needed their own data (e.g., a timer), they'd need to be created fresh per transition.

**Q: How would you persist state so the machine survives a power outage?**
> (1) After every state transition, serialize the machine state to durable storage (DB, file).
> (2) On startup, reload state from storage.
> Critical: `insertedMoney` and `selectedItemCode` must be persisted — a customer could lose money otherwise.

**Q: Why State pattern over a switch-case approach here?**
> With switch/if-else, `VendingMachine` would have 4 methods each with 4 states = 16 blocks of logic, all in one class. When you add a new state (e.g., `MAINTENANCE_MODE`), you modify all 4 methods. The State pattern: each state class has all 4 methods for that state. Adding `MaintenanceState` = one new class, no modification to existing classes. OCP-compliant, independently testable.
