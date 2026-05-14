# LLD: Splitwise (Expense Splitting)

> Frequency: Medium-High | Difficulty: High
> Tests: Graph algorithms (debt simplification), complex entity modeling, mathematical correctness

---

## Step 1 — Clarifying Questions

- Just track who owes whom, or also handle payments?
- Simplify debts (minimize transactions)?
- Multiple currencies?
- Groups or just individual splits?
- Split types: equal / exact amounts / percentage / shares?
- Real-time notifications?

---

## Step 2 — Core Concepts

```
Expense: User A paid ₹300 for a dinner. Split among A, B, C equally.
  → A owes A: ₹0 (they paid)
  → B owes A: ₹100
  → C owes A: ₹100

Debt Simplification:
  A → B: ₹100
  B → C: ₹50
  C → A: ₹150

  Instead of 3 transactions, simplify to:
  C → A: ₹50
  C → B: ₹50
```

---

## Step 3 — Class Diagram

```
User
├── userId, name, email
└── balance: Map<User, Double>  (net balance with each user)

Expense
├── expenseId, description, amount
├── paidBy: User
├── splits: List<Split>
└── group: Group (optional)

Split (abstract)
├── EqualSplit
├── ExactSplit
├── PercentSplit
└── ShareSplit

Group
├── groupId, name
├── members: List<User>
└── expenses: List<Expense>

BalanceManager (service)
└── userBalances: Map<String, Map<String, Double>>
```

---

## Step 4 — Full Java Code

```java
// ─── USER ───
public class User {
    private final String userId;
    private final String name;
    private final String email;

    public User(String userId, String name, String email) {
        this.userId = userId;
        this.name = name;
        this.email = email;
    }

    public String getUserId() { return userId; }
    public String getName() { return name; }
    @Override public String toString() { return name; }
}

// ─── SPLIT TYPES ───
public abstract class Split {
    private final User user;
    private double amount;  // calculated amount owed

    public Split(User user) { this.user = user; }

    public User getUser() { return user; }
    public double getAmount() { return amount; }
    public void setAmount(double amount) { this.amount = amount; }
}

public class EqualSplit extends Split {
    public EqualSplit(User user) { super(user); }
}

public class ExactSplit extends Split {
    public ExactSplit(User user, double amount) {
        super(user);
        setAmount(amount);
    }
}

public class PercentSplit extends Split {
    private final double percentage;
    public PercentSplit(User user, double percentage) {
        super(user);
        this.percentage = percentage;
    }
    public double getPercentage() { return percentage; }
}

// ─── EXPENSE ───
public class Expense {
    private static final AtomicInteger counter = new AtomicInteger(0);

    private final String expenseId;
    private final String description;
    private final double amount;
    private final User paidBy;
    private final List<Split> splits;
    private final LocalDateTime createdAt;

    public Expense(String description, double amount, User paidBy, List<Split> splits) {
        this.expenseId = "EXP-" + counter.incrementAndGet();
        this.description = description;
        this.amount = amount;
        this.paidBy = paidBy;
        this.splits = new ArrayList<>(splits);
        this.createdAt = LocalDateTime.now();

        validateAndAssignSplits();
    }

    private void validateAndAssignSplits() {
        // Assign amounts based on split type
        long equalCount = splits.stream().filter(s -> s instanceof EqualSplit).count();
        double exactSum = splits.stream()
            .filter(s -> s instanceof ExactSplit)
            .mapToDouble(Split::getAmount).sum();
        double percentSum = splits.stream()
            .filter(s -> s instanceof PercentSplit)
            .mapToDouble(s -> ((PercentSplit) s).getPercentage()).sum();

        double remainingAmount = amount - exactSum;

        for (Split split : splits) {
            if (split instanceof EqualSplit) {
                split.setAmount(remainingAmount / equalCount);
            } else if (split instanceof PercentSplit) {
                split.setAmount(amount * ((PercentSplit) split).getPercentage() / 100.0);
            }
            // ExactSplit already has amount set
        }

        // Validate total
        double totalSplit = splits.stream().mapToDouble(Split::getAmount).sum();
        if (Math.abs(totalSplit - amount) > 0.01) {
            throw new IllegalArgumentException(
                "Split amounts (" + totalSplit + ") don't match expense amount (" + amount + ")");
        }
    }

    public String getExpenseId() { return expenseId; }
    public String getDescription() { return description; }
    public double getAmount() { return amount; }
    public User getPaidBy() { return paidBy; }
    public List<Split> getSplits() { return Collections.unmodifiableList(splits); }
}

// ─── BALANCE MANAGER ───
public class BalanceManager {
    // balances.get(userA).get(userB) = amount A owes B (positive)
    // if negative, B owes A
    private final Map<String, Map<String, Double>> balances = new ConcurrentHashMap<>();

    public synchronized void processExpense(Expense expense) {
        String payerId = expense.getPaidBy().getUserId();

        for (Split split : expense.getSplits()) {
            String owerId = split.getUser().getUserId();
            if (owerId.equals(payerId)) continue;  // payer doesn't owe themselves

            // ower → payer increases by split amount
            updateBalance(owerId, payerId, split.getAmount());
        }
    }

    public synchronized void settleDebt(User payer, User receiver, double amount) {
        // payer is paying receiver → reduce payer's debt to receiver
        updateBalance(payer.getUserId(), receiver.getUserId(), -amount);
    }

    private void updateBalance(String fromId, String toId, double delta) {
        balances.computeIfAbsent(fromId, k -> new ConcurrentHashMap<>());
        balances.computeIfAbsent(toId, k -> new ConcurrentHashMap<>());

        // from owes 'to' more
        double current = getBalance(fromId, toId);
        double newBalance = current + delta;

        if (newBalance > 0) {
            balances.get(fromId).put(toId, newBalance);
            balances.get(toId).put(fromId, -newBalance);  // mirror
        } else if (newBalance < 0) {
            balances.get(toId).put(fromId, -newBalance);
            balances.get(fromId).put(toId, newBalance);
        } else {
            balances.get(fromId).remove(toId);
            balances.get(toId).remove(fromId);
        }
    }

    public double getBalance(String fromId, String toId) {
        return balances.getOrDefault(fromId, Collections.emptyMap())
                       .getOrDefault(toId, 0.0);
    }

    public Map<String, Double> getBalancesFor(String userId) {
        return balances.getOrDefault(userId, Collections.emptyMap()).entrySet().stream()
            .filter(e -> Math.abs(e.getValue()) > 0.01)
            .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));
    }
}

// ─── DEBT SIMPLIFICATION (Graph Algorithm) ───
public class DebtSimplifier {
    /**
     * Minimizes the number of transactions to settle all debts.
     * Algorithm: reduce to net balances, then greedily match max creditor to max debtor.
     */
    public List<Transaction> simplify(Map<String, Double> netBalances) {
        // netBalances: userId → net amount (positive = owed to them, negative = they owe)
        List<Transaction> result = new ArrayList<>();

        // Split into creditors (owed money) and debtors (owe money)
        PriorityQueue<double[]> creditors = new PriorityQueue<>((a, b) -> Double.compare(b[0], a[0]));  // max heap
        PriorityQueue<double[]> debtors = new PriorityQueue<>((a, b) -> Double.compare(b[0], a[0]));   // max heap

        // [amount, userId encoded as double — use a Map in real code]
        // Simplified: use indexed approach
        List<double[]> credits = new ArrayList<>();  // [index, amount]
        List<double[]> debts = new ArrayList<>();

        List<String> userIds = new ArrayList<>(netBalances.keySet());
        for (int i = 0; i < userIds.size(); i++) {
            double balance = netBalances.get(userIds.get(i));
            if (balance > 0.01) credits.add(new double[]{i, balance});
            else if (balance < -0.01) debts.add(new double[]{i, -balance});
        }

        // Sort descending
        credits.sort((a, b) -> Double.compare(b[1], a[1]));
        debts.sort((a, b) -> Double.compare(b[1], a[1]));

        int ci = 0, di = 0;
        while (ci < credits.size() && di < debts.size()) {
            double[] credit = credits.get(ci);
            double[] debt = debts.get(di);

            double transfer = Math.min(credit[1], debt[1]);
            result.add(new Transaction(
                userIds.get((int) debt[0]),
                userIds.get((int) credit[0]),
                transfer
            ));

            credit[1] -= transfer;
            debt[1] -= transfer;

            if (credit[1] < 0.01) ci++;
            if (debt[1] < 0.01) di++;
        }

        return result;
    }
}

public class Transaction {
    public final String fromUserId;
    public final String toUserId;
    public final double amount;

    public Transaction(String from, String to, double amount) {
        this.fromUserId = from;
        this.toUserId = to;
        this.amount = Math.round(amount * 100.0) / 100.0;  // round to 2 decimal places
    }

    @Override
    public String toString() {
        return fromUserId + " → " + toUserId + ": ₹" + amount;
    }
}

// ─── SPLITWISE SERVICE (Orchestrator) ───
public class SplitwiseService {
    private final BalanceManager balanceManager = new BalanceManager();
    private final Map<String, User> users = new ConcurrentHashMap<>();
    private final Map<String, List<Expense>> expenseHistory = new ConcurrentHashMap<>();

    public User addUser(String name, String email) {
        User user = new User("U-" + users.size() + 1, name, email);
        users.put(user.getUserId(), user);
        return user;
    }

    public Expense addExpense(String description, double amount, User paidBy,
                               List<Split> splits) {
        Expense expense = new Expense(description, amount, paidBy, splits);
        balanceManager.processExpense(expense);
        expenseHistory.computeIfAbsent(paidBy.getUserId(), k -> new ArrayList<>()).add(expense);
        System.out.println("Expense added: " + description + " ₹" + amount + " by " + paidBy.getName());
        return expense;
    }

    public void settle(User payer, User receiver, double amount) {
        balanceManager.settleDebt(payer, receiver, amount);
        System.out.println(payer.getName() + " paid " + receiver.getName() + " ₹" + amount);
    }

    public void printBalances(User user) {
        Map<String, Double> balances = balanceManager.getBalancesFor(user.getUserId());
        if (balances.isEmpty()) { System.out.println(user.getName() + ": All settled!"); return; }
        balances.forEach((otherId, amount) -> {
            User other = users.get(otherId);
            if (amount > 0) System.out.println(user.getName() + " owes " + other.getName() + " ₹" + amount);
            else System.out.println(other.getName() + " owes " + user.getName() + " ₹" + (-amount));
        });
    }
}
```

**Usage:**
```java
SplitwiseService sw = new SplitwiseService();
User alice = sw.addUser("Alice", "a@x.com");
User bob = sw.addUser("Bob", "b@x.com");
User charlie = sw.addUser("Charlie", "c@x.com");

// Alice paid ₹300, split equally among all 3
sw.addExpense("Dinner", 300.0, alice, List.of(
    new EqualSplit(alice), new EqualSplit(bob), new EqualSplit(charlie)));

// Bob owes Alice ₹100, Charlie owes Alice ₹100
sw.printBalances(alice);
sw.printBalances(bob);

// Bob pays Alice ₹100
sw.settle(bob, alice, 100.0);
sw.printBalances(bob);  // settled
```

---

## Design Decisions

| Decision | Choice | Why |
|---|---|---|
| Balance storage | `Map<userId, Map<userId, Double>>` | O(1) lookup for any pair |
| Mirror balance update | Update both A→B and B→A | Consistency — both directions always sum to zero |
| Debt simplification | Greedy (max creditor + max debtor) | Provably minimizes transaction count |
| processExpense | `synchronized` | Multiple expenses could be added concurrently; balance state must be consistent |

---

## Concurrency Follow-up Questions

**Q: Two threads process expenses simultaneously and both update the same user pair's balance. What happens?**
> Without synchronization: classic race condition. Thread A reads balance = ₹100, Thread B reads balance = ₹100, both add ₹50, both write ₹150 — one ₹50 is lost.
>
> With `synchronized` on `processExpense`: all balance updates are serialized. The only performance cost is sequential expense processing. For most use cases, this is fine.

**Q: `updateBalance` updates TWO map entries (A→B and B→A). If an exception occurs between them, the state is inconsistent. How do you handle this?**
> This is a classic "compound operation needs to be atomic" problem. Options:
> (1) Wrap in `synchronized` block (what we do) — no exception between the two writes if we don't throw.
> (2) Use a `lock.lock()` / `lock.unlock()` with try-catch ensuring rollback.
> (3) Use immutable balance records and CAS for atomic updates.
> In our design, `synchronized` prevents concurrency; exceptions within would leave partial state. Add transaction-like rollback if needed.

**Q: The debt simplification algorithm — does it always give the optimal solution?**
> The greedy approach (match max creditor to max debtor) is provably optimal for minimizing the NUMBER of transactions. However, it may not be optimal if you have constraints (e.g., Bob doesn't have Venmo, so Bob can't pay Alice directly). With constraints, it becomes NP-hard (subset sum variant).

**Q: How would you add real-time notifications (Observer pattern) to Splitwise?**
> After `processExpense()`, publish events to an `ExpenseEventBus`:
> ```java
> eventBus.publish("EXPENSE_ADDED", expense);
> // Observers: NotificationService (sends email/SMS), ActivityFeedService, StatisticsService
> ```
> Users subscribe when they join a group. Observer pattern decouples expense processing from notification delivery.
