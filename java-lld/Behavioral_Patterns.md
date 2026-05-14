# Behavioral Patterns — Deep Dive

> Behavioral patterns define **how objects communicate and assign responsibilities**.
> These are the most commonly tested in LLD interviews.

---

## Pattern Quick Reference

| Pattern | Core Idea | Key Signal |
|---|---|---|
| Strategy | Pluggable algorithms | "different ways to do X", swappable at runtime |
| Observer | One-to-many notification | "when X happens, notify all subscribers" |
| Command | Request as object | "undo/redo", "queue operations", "log commands" |
| Chain of Responsibility | Pipeline of handlers | "validation chain", "middleware", "escalation" |
| State | Behavior changes with state | "state machine", "workflow", "lifecycle" |
| Template Method | Skeleton + pluggable steps | "same steps, different implementations" |
| Iterator | Sequential access | "traverse without knowing structure" |
| Mediator | Central coordinator | "reduce direct connections between objects" |

---

## 1. Strategy

**Problem:** Multiple algorithms/behaviors for the same operation. Select at runtime without if-else.
**Mental model:** GPS navigation — same destination, choose between fastest/shortest/no-tolls.

```java
// ─── STRATEGY interface ───
public interface SortStrategy {
    void sort(int[] data);
}

// ─── CONCRETE STRATEGIES ───
public class BubbleSort implements SortStrategy {
    @Override
    public void sort(int[] data) {
        System.out.println("Bubble sort (small datasets)");
        // O(n²) — good for nearly sorted, small arrays
        for (int i = 0; i < data.length - 1; i++)
            for (int j = 0; j < data.length - i - 1; j++)
                if (data[j] > data[j+1]) { int tmp = data[j]; data[j] = data[j+1]; data[j+1] = tmp; }
    }
}

public class QuickSort implements SortStrategy {
    @Override
    public void sort(int[] data) {
        System.out.println("Quick sort (large datasets)");
        // O(n log n) average
    }
}

public class MergeSort implements SortStrategy {
    @Override
    public void sort(int[] data) {
        System.out.println("Merge sort (stable, guaranteed O(n log n))");
    }
}

// ─── CONTEXT: uses a strategy ───
public class DataProcessor {
    private SortStrategy strategy;

    public DataProcessor(SortStrategy strategy) { this.strategy = strategy; }

    // Swap strategy at runtime
    public void setStrategy(SortStrategy strategy) { this.strategy = strategy; }

    public void processData(int[] data) {
        // Choose strategy based on data characteristics
        if (data.length < 10) {
            setStrategy(new BubbleSort());
        } else if (needsStableSort()) {
            setStrategy(new MergeSort());
        }
        strategy.sort(data);
    }

    private boolean needsStableSort() { return false; }
}

// ─── Payment Strategy example (fintech classic) ───
public interface PaymentStrategy {
    boolean pay(double amount);
    String getMethodName();
}

public class UPIPayment implements PaymentStrategy {
    private final String upiId;
    public UPIPayment(String upiId) { this.upiId = upiId; }

    @Override
    public boolean pay(double amount) {
        System.out.println("Paying ₹" + amount + " via UPI: " + upiId);
        return true;
    }

    @Override public String getMethodName() { return "UPI"; }
}

public class CardPayment implements PaymentStrategy {
    private final String cardNumber;
    public CardPayment(String cardNumber) { this.cardNumber = cardNumber; }

    @Override
    public boolean pay(double amount) {
        System.out.println("Paying ₹" + amount + " via Card: ***" + cardNumber.substring(12));
        return true;
    }

    @Override public String getMethodName() { return "CARD"; }
}
```

---

### Strategy — Deep Follow-up Questions

**Q: Strategy vs State vs Template Method — explain all three differences clearly.**

| Aspect | Strategy | State | Template Method |
|---|---|---|---|
| Who changes behavior | Client swaps strategy | Object changes its own state | Subclass overrides specific steps |
| Trigger | External (client calls `setStrategy`) | Internal (state transitions itself) | None — compile-time inheritance |
| Awareness | Strategies don't know about each other | States often transition to each other | Subclasses fill in steps |
| Pattern type | Behavioral (runtime composition) | Behavioral (state machine) | Behavioral (inheritance) |
| Typical use | Payment methods, sort algorithms | Vending machine, ATM, order lifecycle | Data parsers, game loops |

**Q: Could you replace Strategy with lambdas in Java 8+?**
> Yes, if the strategy interface has a single method (functional interface). `SortStrategy` can be a `@FunctionalInterface`, and strategies become lambdas:
> ```java
> DataProcessor p = new DataProcessor(data -> Arrays.sort(data));  // lambda AS strategy
> ```
> Lambdas are great for simple, stateless strategies. Use classes when the strategy needs state (like `UPIPayment` with `upiId`).

**Q: Is Strategy thread-safe when `setStrategy()` is called concurrently?**
> No. If thread A is reading `strategy` while thread B calls `setStrategy()`, you can get a race condition. Solutions:
> (1) Make `strategy` `volatile` — ensures visibility but not atomicity.
> (2) Make `setStrategy` `synchronized`.
> (3) Make the Context immutable — no `setStrategy()`, pass strategy in constructor.
> Option 3 is best if strategy doesn't need to change after construction.

---

## 2. Observer

**Problem:** One object (subject) needs to notify multiple dependents when it changes state.
**Mental model:** YouTube subscriptions — creator (Subject) posts, all subscribers (Observers) are notified.

```java
// ─── OBSERVER interface ───
public interface EventObserver {
    void onEvent(String eventType, Object data);
}

// ─── SUBJECT: manages observers + publishes events ───
public class PaymentEventBus {
    private final Map<String, List<EventObserver>> observers = new ConcurrentHashMap<>();

    public void subscribe(String eventType, EventObserver observer) {
        observers.computeIfAbsent(eventType, k -> new CopyOnWriteArrayList<>()).add(observer);
    }

    public void unsubscribe(String eventType, EventObserver observer) {
        List<EventObserver> list = observers.get(eventType);
        if (list != null) list.remove(observer);
    }

    public void publish(String eventType, Object data) {
        List<EventObserver> list = observers.getOrDefault(eventType, Collections.emptyList());
        for (EventObserver observer : list) {
            try {
                observer.onEvent(eventType, data);  // notify each observer
            } catch (Exception e) {
                // don't let one failing observer break others
                System.err.println("Observer failed: " + e.getMessage());
            }
        }
    }
}

// ─── CONCRETE OBSERVERS ───
public class EmailNotifier implements EventObserver {
    @Override
    public void onEvent(String eventType, Object data) {
        if ("PAYMENT_SUCCESS".equals(eventType)) {
            Payment p = (Payment) data;
            System.out.println("Email: Payment of ₹" + p.getAmount() + " confirmed");
        }
    }
}

public class FraudDetector implements EventObserver {
    @Override
    public void onEvent(String eventType, Object data) {
        if ("PAYMENT_INITIATED".equals(eventType)) {
            Payment p = (Payment) data;
            System.out.println("Fraud check on payment: " + p.getId());
        }
    }
}

public class AuditLogger implements EventObserver {
    @Override
    public void onEvent(String eventType, Object data) {
        System.out.println("[AUDIT] " + eventType + ": " + data);
    }
}

// Usage
PaymentEventBus bus = new PaymentEventBus();
bus.subscribe("PAYMENT_INITIATED", new FraudDetector());
bus.subscribe("PAYMENT_SUCCESS", new EmailNotifier());
bus.subscribe("PAYMENT_SUCCESS", new AuditLogger());

bus.publish("PAYMENT_INITIATED", new Payment("pay123", 5000));
bus.publish("PAYMENT_SUCCESS", new Payment("pay123", 5000));
```

---

### Observer — Deep Follow-up Questions

**Q: Observer in a single JVM vs distributed systems (Pub-Sub) — what changes?**
> In JVM: direct method calls, synchronous by default, in-memory list of subscribers.
> In distributed: subscribers are separate services. You need a message broker (Kafka, RabbitMQ, SNS) to deliver events. Key differences: events are serialized (JSON/Avro), delivery is asynchronous, you get durability/replay, but also eventual consistency and potential duplicate delivery.

**Q: Memory leaks in Observer — explain the problem and solution.**
> If the Subject holds strong references to Observers, and observers are "temporary" (e.g., UI components that get destroyed), the Subject keeps them alive even after they're logically dead (they can never be GC'd). This is the classic **lapsed listener** problem.
>
> Solutions:
> (1) Always call `unsubscribe()` in cleanup code (teardown/destructor).
> (2) Use `WeakReference<Observer>` in the Subject's list — GC can collect the observer.
> (3) Use `removeEventListener`-style APIs or auto-cleanup on scope exit.

**Q: How to make the Observer pattern thread-safe?**
> (1) Use `CopyOnWriteArrayList` for the observer list — thread-safe reads without locking, safe iteration even if observer is added/removed during notification.
> (2) Use `ConcurrentHashMap` for event-type → observer list mapping.
> (3) Catch exceptions per observer (shown above) — one observer failure shouldn't stop others.
> (4) Consider whether notification should be synchronous or async (via `ExecutorService`).

**Q: Push vs Pull model in Observer — difference?**
> **Push:** Subject pushes all data to the Observer in the callback (`onEvent(data)`). Observer gets what the subject decides. Simpler but observers get data they may not need.
> **Pull:** Subject calls `onEvent()` with just a reference to itself. Observer pulls only the data it needs from the subject. More flexible but requires subject's state to be accessible.

**Q: What if observer notification is slow and we need to not block the publisher?**
> Make notifications async:
> ```java
> ExecutorService executor = Executors.newFixedThreadPool(4);
>
> public void publish(String eventType, Object data) {
>     List<EventObserver> list = observers.getOrDefault(eventType, emptyList());
>     for (EventObserver observer : list) {
>         executor.submit(() -> {                // async — doesn't block publisher
>             try { observer.onEvent(eventType, data); }
>             catch (Exception e) { /* log */ }
>         });
>     }
> }
> ```
> Trade-off: observers now run out of order, and errors don't propagate to publisher.

---

## 3. Command

**Problem:** Encapsulate a request as an object. Enables: undo/redo, queuing, logging, scheduling.
**Mental model:** A restaurant order — the waiter takes your order (Command) and the chef executes it later.

```java
// ─── COMMAND interface ───
public interface Command {
    void execute();
    void undo();
}

// ─── RECEIVER: knows how to perform the work ───
public class TextEditor {
    private final StringBuilder content = new StringBuilder();
    private int cursorPosition = 0;

    public void insertText(String text, int position) {
        content.insert(position, text);
        cursorPosition = position + text.length();
    }

    public void deleteText(int start, int length) {
        content.delete(start, start + length);
        cursorPosition = start;
    }

    public String getContent() { return content.toString(); }
    public int getCursorPosition() { return cursorPosition; }
}

// ─── CONCRETE COMMANDS ───
public class InsertTextCommand implements Command {
    private final TextEditor editor;
    private final String text;
    private final int position;

    public InsertTextCommand(TextEditor editor, String text, int position) {
        this.editor = editor;
        this.text = text;
        this.position = position;
    }

    @Override
    public void execute() { editor.insertText(text, position); }

    @Override
    public void undo() { editor.deleteText(position, text.length()); }  // reverse!
}

// ─── INVOKER: manages command history ───
public class CommandHistory {
    private final Deque<Command> history = new ArrayDeque<>();
    private final Deque<Command> redoStack = new ArrayDeque<>();

    public void execute(Command command) {
        command.execute();
        history.push(command);
        redoStack.clear();  // new command clears redo history
    }

    public void undo() {
        if (history.isEmpty()) return;
        Command command = history.pop();
        command.undo();
        redoStack.push(command);
    }

    public void redo() {
        if (redoStack.isEmpty()) return;
        Command command = redoStack.pop();
        command.execute();
        history.push(command);
    }
}

// ─── ASYNC COMMAND QUEUE (for task scheduling) ───
public class CommandQueue {
    private final BlockingQueue<Command> queue = new LinkedBlockingQueue<>();
    private final ExecutorService executor = Executors.newSingleThreadExecutor();

    public CommandQueue() {
        executor.submit(this::processQueue);  // single consumer thread
    }

    public void enqueue(Command command) {
        queue.offer(command);
    }

    private void processQueue() {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                Command command = queue.take();  // blocks until command available
                command.execute();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

---

### Command — Deep Follow-up Questions

**Q: How do you implement undo for a command that modifies shared state?**
> Each command must save enough state BEFORE `execute()` to restore it in `undo()`. Two approaches:
> (1) **Snapshot:** Store a copy of the entire object state before change (works for small objects).
> (2) **Inverse operation:** Store parameters to reverse the operation (more efficient — `InsertText` undone by `DeleteText`). Prefer inverse operation for large objects.

**Q: What are the threading concerns when using a Command Queue?**
> (1) `BlockingQueue` is thread-safe — producers and consumer can operate concurrently.
> (2) Commands themselves must be thread-safe if shared state is accessed (the Receiver object).
> (3) If multiple consumer threads process commands, order is not guaranteed — use single consumer for ordered execution.
> (4) Unhandled exceptions in commands kill the consumer thread — always wrap in try-catch.

**Q: Command vs Strategy — both encapsulate behavior in an object. Difference?**
> **Strategy:** Encapsulates an **algorithm** that the context uses. The context typically holds ONE strategy at a time.
> **Command:** Encapsulates a **request/action** with receiver and arguments. Multiple commands are typically queued/stored/executed sequentially. Command enables undo/redo/scheduling; Strategy does not.

---

## 4. Chain of Responsibility

**Problem:** Pass a request along a chain of handlers. Each handler decides to process or pass to the next.
**Mental model:** Customer support escalation (L1 → L2 → L3). Middleware pipelines.

```java
// ─── HANDLER interface ───
public abstract class TransactionValidator {
    private TransactionValidator next;

    public TransactionValidator setNext(TransactionValidator next) {
        this.next = next;
        return next;  // fluent — enables chaining
    }

    // Template method: subclass implements doValidate(), this manages chain
    public final ValidationResult validate(Transaction tx) {
        ValidationResult result = doValidate(tx);
        if (!result.isPassed()) return result;  // short-circuit on failure
        if (next != null) return next.validate(tx);  // pass to next
        return ValidationResult.success();
    }

    protected abstract ValidationResult doValidate(Transaction tx);
}

// ─── CONCRETE HANDLERS ───
public class AuthenticationValidator extends TransactionValidator {
    @Override
    protected ValidationResult doValidate(Transaction tx) {
        if (!tx.getUserSession().isValid()) {
            return ValidationResult.failure("Session expired");
        }
        System.out.println("Auth check passed");
        return ValidationResult.success();
    }
}

public class FraudValidator extends TransactionValidator {
    private final FraudDetectionService fraudService;

    public FraudValidator(FraudDetectionService fraudService) {
        this.fraudService = fraudService;
    }

    @Override
    protected ValidationResult doValidate(Transaction tx) {
        if (fraudService.isSuspicious(tx)) {
            return ValidationResult.failure("Transaction flagged as suspicious");
        }
        System.out.println("Fraud check passed");
        return ValidationResult.success();
    }
}

public class BalanceValidator extends TransactionValidator {
    @Override
    protected ValidationResult doValidate(Transaction tx) {
        double balance = tx.getAccount().getBalance();
        if (balance < tx.getAmount()) {
            return ValidationResult.failure("Insufficient balance: " + balance);
        }
        System.out.println("Balance check passed");
        return ValidationResult.success();
    }
}

public class DailyLimitValidator extends TransactionValidator {
    private static final double DAILY_LIMIT = 100000.0;

    @Override
    protected ValidationResult doValidate(Transaction tx) {
        double todaySpend = tx.getAccount().getTodaySpend();
        if (todaySpend + tx.getAmount() > DAILY_LIMIT) {
            return ValidationResult.failure("Daily limit exceeded");
        }
        System.out.println("Daily limit check passed");
        return ValidationResult.success();
    }
}

// ─── BUILDING THE CHAIN ───
public class TransactionValidationChain {
    public static TransactionValidator build(FraudDetectionService fraudService) {
        TransactionValidator chain = new AuthenticationValidator();
        chain.setNext(new FraudValidator(fraudService))
             .setNext(new BalanceValidator())
             .setNext(new DailyLimitValidator());
        return chain;
    }
}

// Usage
TransactionValidator validator = TransactionValidationChain.build(fraudService);
ValidationResult result = validator.validate(transaction);
```

---

### Chain of Responsibility — Deep Follow-up Questions

**Q: What happens if no handler in the chain handles the request?**
> If the chain ends without a handler processing it, the request is "dropped" silently. You have two options: (1) Add a default handler at the end that handles everything (catch-all). (2) Return a specific result indicating "not handled". Never silently ignore — always have a defined outcome.

**Q: CoR vs Strategy for validation pipelines — when to use which?**
> **CoR:** Each handler independently decides to handle-or-pass. Order matters; handlers are decoupled from each other. Short-circuits on failure. Good for middleware-style pipelines where each step has the option to stop processing.
> **Strategy:** One handler chosen based on type. Not a pipeline — it's a single algorithm selected. Use for "which algorithm to use", not "how to pipeline processing".

**Q: How do you add logging/metrics to every step without modifying handlers?**
> Wrap each handler in a Decorator:
> ```java
> public class LoggingValidatorDecorator extends TransactionValidator {
>     private final TransactionValidator delegate;
>     private final String name;
>
>     @Override
>     protected ValidationResult doValidate(Transaction tx) {
>         long start = System.currentTimeMillis();
>         ValidationResult result = delegate.validate(tx);
>         long elapsed = System.currentTimeMillis() - start;
>         System.out.println(name + " took " + elapsed + "ms, result: " + result);
>         return result;
>     }
> }
> ```

**Q: Is CoR thread-safe when multiple requests traverse the chain concurrently?**
> If handlers are **stateless** (no instance variables that change per-request), yes. If handlers have state (e.g., `FraudValidator` keeps a cache), you need to synchronize that state or use request-scoped instances. The chain structure itself (linked list of handlers) is safe if not modified during operation.

---

## 5. State

**Problem:** An object's behavior changes completely based on its internal state. Eliminates large if-else/switch chains.
**Mental model:** A vending machine — same `pressButton()` action behaves completely differently depending on whether it has money inserted, is dispensing, etc.

```java
// ─── STATE interface ───
public interface VendingMachineState {
    void insertMoney(VendingMachine machine, double amount);
    void selectItem(VendingMachine machine, String itemCode);
    void dispenseItem(VendingMachine machine);
    void cancel(VendingMachine machine);
    String getStateName();
}

// ─── CONTEXT ───
public class VendingMachine {
    private VendingMachineState currentState;
    private double insertedMoney = 0;
    private String selectedItem = null;
    private Map<String, Item> inventory = new HashMap<>();

    public VendingMachine() {
        this.currentState = new IdleState();  // initial state
    }

    // Delegate all actions to current state
    public void insertMoney(double amount) { currentState.insertMoney(this, amount); }
    public void selectItem(String code) { currentState.selectItem(this, code); }
    public void dispenseItem() { currentState.dispenseItem(this); }
    public void cancel() { currentState.cancel(this); }

    // State can call these to transition
    public void setState(VendingMachineState state) {
        System.out.println("Transitioning: " + currentState.getStateName() + " → " + state.getStateName());
        this.currentState = state;
    }

    public double getInsertedMoney() { return insertedMoney; }
    public void setInsertedMoney(double money) { this.insertedMoney = money; }
    public String getSelectedItem() { return selectedItem; }
    public void setSelectedItem(String code) { this.selectedItem = code; }
    public Map<String, Item> getInventory() { return inventory; }
}

// ─── CONCRETE STATES ───
public class IdleState implements VendingMachineState {
    @Override
    public void insertMoney(VendingMachine machine, double amount) {
        machine.setInsertedMoney(amount);
        System.out.println("Money inserted: ₹" + amount);
        machine.setState(new HasMoneyState());  // transition!
    }

    @Override
    public void selectItem(VendingMachine machine, String code) {
        System.out.println("Please insert money first");
    }

    @Override
    public void dispenseItem(VendingMachine machine) {
        System.out.println("No item selected");
    }

    @Override
    public void cancel(VendingMachine machine) {
        System.out.println("Nothing to cancel");
    }

    @Override public String getStateName() { return "IDLE"; }
}

public class HasMoneyState implements VendingMachineState {
    @Override
    public void insertMoney(VendingMachine machine, double amount) {
        machine.setInsertedMoney(machine.getInsertedMoney() + amount);
        System.out.println("Added ₹" + amount + ". Total: ₹" + machine.getInsertedMoney());
    }

    @Override
    public void selectItem(VendingMachine machine, String code) {
        Item item = machine.getInventory().get(code);
        if (item == null) { System.out.println("Item not found"); return; }
        if (machine.getInsertedMoney() < item.getPrice()) {
            System.out.println("Insufficient money. Need ₹" + (item.getPrice() - machine.getInsertedMoney()) + " more");
            return;
        }
        machine.setSelectedItem(code);
        machine.setState(new ItemSelectedState());
    }

    @Override
    public void dispenseItem(VendingMachine machine) { System.out.println("Select an item first"); }

    @Override
    public void cancel(VendingMachine machine) {
        System.out.println("Returning ₹" + machine.getInsertedMoney());
        machine.setInsertedMoney(0);
        machine.setState(new IdleState());
    }

    @Override public String getStateName() { return "HAS_MONEY"; }
}

public class ItemSelectedState implements VendingMachineState {
    @Override
    public void insertMoney(VendingMachine machine, double amount) {
        System.out.println("Item already selected. Dispense or cancel first.");
    }

    @Override
    public void selectItem(VendingMachine machine, String code) {
        System.out.println("Item already selected: " + machine.getSelectedItem());
    }

    @Override
    public void dispenseItem(VendingMachine machine) {
        System.out.println("Dispensing: " + machine.getSelectedItem());
        // calculate change
        Item item = machine.getInventory().get(machine.getSelectedItem());
        double change = machine.getInsertedMoney() - item.getPrice();
        if (change > 0) System.out.println("Returning change: ₹" + change);
        machine.setInsertedMoney(0);
        machine.setSelectedItem(null);
        machine.setState(new IdleState());
    }

    @Override
    public void cancel(VendingMachine machine) {
        System.out.println("Returning ₹" + machine.getInsertedMoney());
        machine.setInsertedMoney(0);
        machine.setSelectedItem(null);
        machine.setState(new IdleState());
    }

    @Override public String getStateName() { return "ITEM_SELECTED"; }
}
```

---

### State — Deep Follow-up Questions

**Q: State vs Strategy — most common interview question. Nail this answer.**
> Key differences:
> (1) **Who changes the behavior:** In State, the object changes its own behavior as it transitions between states (states call `machine.setState()`). In Strategy, the **client** swaps the strategy.
> (2) **Awareness:** State objects often know about other states (they cause transitions). Strategy objects are completely independent — one strategy doesn't know others exist.
> (3) **Intent:** State models an object's lifecycle/workflow. Strategy models a single pluggable algorithm.
> (4) **Number of "behaviors":** State has many behaviors that change together (all methods change). Strategy has one behavior that's swappable.

**Q: Why use State pattern instead of a simple `switch (state)` block?**
> Switch/if-else approach puts ALL behavior for ALL states in one class. Adding a new state means modifying the switch in every method (OCP violation). State pattern: each state is a class. Adding a new state = adding a new class, zero modification to existing classes. Also, State classes are independently testable.

**Q: State transitions in concurrent environments — what can go wrong?**
> Race condition: Two threads call `insertMoney()` simultaneously when the machine is in `HasMoneyState`. Both could read the same `insertedMoney` value, both add to it, resulting in only one addition being counted.
> Solution: Synchronize the transition methods on the VendingMachine context. Use `synchronized` on all public methods in VendingMachine, or use a `ReentrantLock` for finer control.
> ```java
> public synchronized void insertMoney(double amount) {
>     currentState.insertMoney(this, amount);
> }
> ```

---

## 6. Template Method

**Problem:** Define the skeleton of an algorithm in a base class. Let subclasses override specific steps without changing the overall structure.
**Mental model:** A recipe template — "mix dry ingredients, mix wet ingredients, combine, bake" — the steps are fixed, but each cake type fills in the specifics differently.

```java
// ─── ABSTRACT CLASS with template ───
public abstract class DataMigrator {
    // Template Method — final so subclasses can't change the algorithm structure
    public final void migrate() {
        System.out.println("=== Starting migration ===");
        connect();
        List<Object> data = extractData();
        List<Object> transformed = transformData(data);
        validateData(transformed);
        loadData(transformed);
        disconnect();
        System.out.println("=== Migration complete ===");
    }

    // Steps that MUST be implemented by subclasses (abstract)
    protected abstract void connect();
    protected abstract List<Object> extractData();
    protected abstract void loadData(List<Object> data);

    // Steps with default implementation (hook methods — subclasses MAY override)
    protected List<Object> transformData(List<Object> data) {
        System.out.println("Default: no transformation");
        return data;  // default: pass through
    }

    protected void validateData(List<Object> data) {
        System.out.println("Default: basic null check");
        if (data == null) throw new IllegalStateException("Data cannot be null");
    }

    protected void disconnect() {
        System.out.println("Default: closing connection");
    }
}

// ─── CONCRETE IMPLEMENTATIONS ───
public class MySQLToPostgresMigrator extends DataMigrator {
    @Override
    protected void connect() { System.out.println("Connected to MySQL and PostgreSQL"); }

    @Override
    protected List<Object> extractData() {
        System.out.println("Extracting from MySQL");
        return List.of("row1", "row2");
    }

    @Override
    protected List<Object> transformData(List<Object> data) {
        System.out.println("Converting MySQL syntax to PostgreSQL");
        return data;  // transform
    }

    @Override
    protected void loadData(List<Object> data) {
        System.out.println("Loading into PostgreSQL");
    }
}

public class CSVToDBMigrator extends DataMigrator {
    @Override
    protected void connect() { System.out.println("Opening CSV file"); }

    @Override
    protected List<Object> extractData() {
        System.out.println("Reading CSV rows");
        return List.of("line1", "line2");
    }

    @Override
    protected void loadData(List<Object> data) {
        System.out.println("Inserting into DB");
    }
    // Uses default transformData and validateData
}
```

---

### Template Method — Deep Follow-up Questions

**Q: Template Method vs Strategy — which is more flexible?**
> Strategy is more flexible at runtime. Template Method uses inheritance — you choose the variant at compile time by picking a subclass. Strategy uses composition — you can swap algorithms at runtime.
>
> Template Method violates "Composition over Inheritance" but is simpler for stable algorithms with well-defined variation points. Use Template Method when the structure is fixed and variations are predictable. Use Strategy when you need runtime flexibility.

**Q: What is a "hook method" in Template Method?**
> A hook is a method in the abstract class with a **default (often empty) implementation**. Subclasses may (but don't have to) override it. This gives subclasses optional extension points without requiring them to implement every step. In the example, `transformData()` and `validateData()` are hooks.

**Q: Why is the template method `final`?**
> To prevent subclasses from overriding the algorithm skeleton itself. The whole point of Template Method is that the structure is fixed — only the steps vary. If subclasses could override `migrate()`, they could completely bypass the algorithm structure.

---

## 7. Mediator

**Problem:** Reduce direct connections between objects by routing communication through a central coordinator.
**Mental model:** Air traffic control — planes don't communicate directly with each other; they all communicate with ATC.

```java
// ─── MEDIATOR interface ───
public interface ChatMediator {
    void sendMessage(String message, User sender);
    void addUser(User user);
}

// ─── CONCRETE MEDIATOR ───
public class ChatRoom implements ChatMediator {
    private final List<User> users = new ArrayList<>();

    @Override
    public void addUser(User user) { users.add(user); }

    @Override
    public void sendMessage(String message, User sender) {
        for (User user : users) {
            if (user != sender) {  // don't send to yourself
                user.receive(message, sender.getName());
            }
        }
    }
}

// ─── COLLEAGUE ───
public class User {
    private final String name;
    private final ChatMediator mediator;

    public User(String name, ChatMediator mediator) {
        this.name = name;
        this.mediator = mediator;
    }

    public void send(String message) {
        System.out.println(name + " sends: " + message);
        mediator.sendMessage(message, this);  // go through mediator
    }

    public void receive(String message, String from) {
        System.out.println(name + " received from " + from + ": " + message);
    }

    public String getName() { return name; }
}
```

**Q: Mediator vs Observer — both decouple senders from receivers.**
> **Observer:** One subject notifies many observers. Observers don't communicate with each other — the subject just broadcasts.
> **Mediator:** Many objects communicate through ONE central hub. The mediator contains the interaction logic. Objects know the mediator but not each other.
> Use Observer for broadcast (event bus). Use Mediator for complex multi-way interactions (chat room, air traffic control, complex UI forms where fields affect each other).
