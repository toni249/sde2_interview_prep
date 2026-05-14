# LLD: ATM Machine

> Frequency: Medium | Difficulty: Medium-High
> Tests: State pattern (lifecycle), Chain of Responsibility (validation), concurrency

---

## Step 1 — Clarifying Questions

- Operations: withdraw, deposit, balance check, PIN change?
- Cash dispensing algorithm? (which denominations)
- Multiple accounts? Linked to a bank system?
- Session timeout?
- Concurrent access (multiple users at same ATM)?

---

## Step 2 — State Machine

```
  IDLE ──insertCard──► CARD_INSERTED
                            │
                         enterPIN
                            │
                    ┌───────▼────────┐
                    │  PIN_VERIFIED  │──selectTransaction──► TRANSACTION
                    └───────┬────────┘                           │
                         (wrong PIN × 3)                     complete
                            │                                    │
                         CARD_BLOCKED                        back to PIN_VERIFIED
                                                                 │
                                                             ejectCard
                                                                 ▼
                                                               IDLE
```

---

## Step 3 — Class Diagram

```
ATM (Context)
├── currentState: ATMState
├── cardReader: CardReader
├── cashDispenser: CashDispenser
├── currentAccount: Account

ATMState (interface)
├── IdleState
├── CardInsertedState
├── PinVerifiedState
└── TransactionState

CashDispenser
├── denominations: Map<Integer, Integer>  (₹500: 10, ₹100: 5)
└── dispensing algorithm

WithdrawalValidationChain (CoR)
├── AuthenticationValidator
├── BalanceValidator
└── WithdrawalLimitValidator

Account
├── accountNumber, PIN, balance
└── dailyWithdrawalLimit
```

---

## Step 4 — Full Java Code

```java
// ─── ACCOUNT ───
public class Account {
    private final String accountNumber;
    private final String pin;  // In real system: hashed PIN
    private double balance;
    private double dailyWithdrawn;
    private static final double DAILY_LIMIT = 50000.0;

    public Account(String accountNumber, String pin, double balance) {
        this.accountNumber = accountNumber;
        this.pin = pin;
        this.balance = balance;
        this.dailyWithdrawn = 0.0;
    }

    public boolean validatePin(String inputPin) { return pin.equals(inputPin); }
    public double getBalance() { return balance; }
    public double getDailyWithdrawn() { return dailyWithdrawn; }
    public double getDailyLimit() { return DAILY_LIMIT; }
    public String getAccountNumber() { return accountNumber; }

    public synchronized boolean debit(double amount) {
        if (balance < amount) return false;
        balance -= amount;
        dailyWithdrawn += amount;
        return true;
    }

    public synchronized void credit(double amount) { balance += amount; }
}

// ─── CASH DISPENSER ───
public class CashDispenser {
    // denomination → count available
    private final TreeMap<Integer, Integer> denominations;  // TreeMap = sorted by key

    public CashDispenser() {
        denominations = new TreeMap<>(Collections.reverseOrder());  // high denominations first
        denominations.put(2000, 10);
        denominations.put(500, 20);
        denominations.put(100, 50);
        denominations.put(50, 100);
    }

    public boolean canDispense(double amount) {
        return calculateDispensePlan(amount) != null;
    }

    public synchronized Map<Integer, Integer> dispense(double amount) {
        Map<Integer, Integer> plan = calculateDispensePlan(amount);
        if (plan == null) throw new InsufficientCashException("ATM cash insufficient for ₹" + amount);

        // Deduct from ATM
        plan.forEach((denom, count) ->
            denominations.merge(denom, -count, Integer::sum));

        return plan;
    }

    private Map<Integer, Integer> calculateDispensePlan(double amount) {
        int remaining = (int) amount;
        Map<Integer, Integer> plan = new LinkedHashMap<>();

        for (Map.Entry<Integer, Integer> entry : denominations.entrySet()) {
            int denom = entry.getKey();
            int available = entry.getValue();
            if (remaining <= 0 || denom > remaining) continue;

            int needed = remaining / denom;
            int use = Math.min(needed, available);
            if (use > 0) {
                plan.put(denom, use);
                remaining -= denom * use;
            }
        }

        return remaining == 0 ? plan : null;  // null if can't make exact amount
    }

    public int getTotalCash() {
        return denominations.entrySet().stream()
            .mapToInt(e -> e.getKey() * e.getValue()).sum();
    }
}

// ─── WITHDRAWAL VALIDATION CHAIN ───
public abstract class WithdrawalValidator {
    protected WithdrawalValidator next;

    public WithdrawalValidator setNext(WithdrawalValidator next) {
        this.next = next;
        return next;
    }

    public final ValidationResult validate(Account account, double amount, CashDispenser dispenser) {
        ValidationResult result = doValidate(account, amount, dispenser);
        if (!result.isValid()) return result;
        return next != null ? next.validate(account, amount, dispenser) : ValidationResult.success();
    }

    protected abstract ValidationResult doValidate(Account account, double amount, CashDispenser dispenser);
}

public class BalanceValidator extends WithdrawalValidator {
    @Override
    protected ValidationResult doValidate(Account account, double amount, CashDispenser d) {
        if (account.getBalance() < amount)
            return ValidationResult.failure("Insufficient balance. Available: ₹" + account.getBalance());
        return ValidationResult.success();
    }
}

public class DailyLimitValidator extends WithdrawalValidator {
    @Override
    protected ValidationResult doValidate(Account account, double amount, CashDispenser d) {
        double remaining = account.getDailyLimit() - account.getDailyWithdrawn();
        if (amount > remaining)
            return ValidationResult.failure("Daily limit exceeded. Remaining: ₹" + remaining);
        return ValidationResult.success();
    }
}

public class CashAvailabilityValidator extends WithdrawalValidator {
    @Override
    protected ValidationResult doValidate(Account account, double amount, CashDispenser dispenser) {
        if (!dispenser.canDispense(amount))
            return ValidationResult.failure("ATM cannot dispense ₹" + amount + " (denomination mismatch)");
        return ValidationResult.success();
    }
}

public class ValidationResult {
    private final boolean valid;
    private final String message;

    private ValidationResult(boolean valid, String message) { this.valid = valid; this.message = message; }
    public static ValidationResult success() { return new ValidationResult(true, "OK"); }
    public static ValidationResult failure(String msg) { return new ValidationResult(false, msg); }
    public boolean isValid() { return valid; }
    public String getMessage() { return message; }
}

// ─── ATM STATES ───
public interface ATMState {
    void insertCard(ATM atm, String cardNumber);
    void enterPin(ATM atm, String pin);
    void selectWithdraw(ATM atm, double amount);
    void selectBalance(ATM atm);
    void ejectCard(ATM atm);
    default String getName() { return this.getClass().getSimpleName(); }
}

public class IdleState implements ATMState {
    @Override
    public void insertCard(ATM atm, String cardNumber) {
        Account account = atm.findAccount(cardNumber);
        if (account == null) { System.out.println("Card not recognized"); return; }
        atm.setCurrentAccount(account);
        System.out.println("Card inserted. Enter your PIN.");
        atm.setState(new CardInsertedState());
    }

    @Override public void enterPin(ATM atm, String pin) { System.out.println("Insert card first"); }
    @Override public void selectWithdraw(ATM atm, double amount) { System.out.println("Insert card first"); }
    @Override public void selectBalance(ATM atm) { System.out.println("Insert card first"); }
    @Override public void ejectCard(ATM atm) { System.out.println("No card inserted"); }
}

public class CardInsertedState implements ATMState {
    private int pinAttempts = 0;

    @Override public void insertCard(ATM atm, String card) { System.out.println("Card already inserted"); }

    @Override
    public void enterPin(ATM atm, String pin) {
        if (atm.getCurrentAccount().validatePin(pin)) {
            System.out.println("PIN verified. Select transaction.");
            pinAttempts = 0;
            atm.setState(new PinVerifiedState());
        } else {
            pinAttempts++;
            System.out.println("Wrong PIN. Attempts: " + pinAttempts + "/3");
            if (pinAttempts >= 3) {
                System.out.println("Card blocked after 3 wrong attempts.");
                atm.blockCard();
                atm.setState(new IdleState());
            }
        }
    }

    @Override public void selectWithdraw(ATM atm, double a) { System.out.println("Enter PIN first"); }
    @Override public void selectBalance(ATM atm) { System.out.println("Enter PIN first"); }

    @Override
    public void ejectCard(ATM atm) {
        System.out.println("Card ejected");
        atm.setCurrentAccount(null);
        atm.setState(new IdleState());
    }
}

public class PinVerifiedState implements ATMState {
    @Override public void insertCard(ATM atm, String card) { System.out.println("Session active"); }
    @Override public void enterPin(ATM atm, String pin) { System.out.println("PIN already verified"); }

    @Override
    public void selectWithdraw(ATM atm, double amount) {
        // Build validation chain
        WithdrawalValidator chain = new BalanceValidator();
        chain.setNext(new DailyLimitValidator()).setNext(new CashAvailabilityValidator());

        ValidationResult result = chain.validate(atm.getCurrentAccount(), amount, atm.getCashDispenser());

        if (!result.isValid()) {
            System.out.println("Withdrawal failed: " + result.getMessage());
            return;
        }

        atm.setState(new TransactionState());
        atm.processWithdrawal(amount);
    }

    @Override
    public void selectBalance(ATM atm) {
        System.out.println("Balance: ₹" + atm.getCurrentAccount().getBalance());
    }

    @Override
    public void ejectCard(ATM atm) {
        System.out.println("Thank you. Card ejected.");
        atm.setCurrentAccount(null);
        atm.setState(new IdleState());
    }
}

public class TransactionState implements ATMState {
    @Override public void insertCard(ATM atm, String c) { System.out.println("Transaction in progress"); }
    @Override public void enterPin(ATM atm, String p) { System.out.println("Transaction in progress"); }
    @Override public void selectWithdraw(ATM atm, double a) { System.out.println("Transaction in progress"); }
    @Override public void selectBalance(ATM atm) { System.out.println("Transaction in progress"); }
    @Override public void ejectCard(ATM atm) { System.out.println("Please wait for transaction to complete"); }
}

// ─── ATM CONTEXT ───
public class ATM {
    private ATMState currentState;
    private Account currentAccount;
    private final CashDispenser cashDispenser;
    private final Map<String, Account> accounts = new HashMap<>();  // cardNumber → Account

    public ATM() {
        this.cashDispenser = new CashDispenser();
        this.currentState = new IdleState();
    }

    // Load test accounts
    public void loadAccount(String cardNumber, Account account) {
        accounts.put(cardNumber, account);
    }

    // Delegate to state
    public synchronized void insertCard(String cardNumber) { currentState.insertCard(this, cardNumber); }
    public synchronized void enterPin(String pin) { currentState.enterPin(this, pin); }
    public synchronized void withdraw(double amount) { currentState.selectWithdraw(this, amount); }
    public synchronized void checkBalance() { currentState.selectBalance(this); }
    public synchronized void ejectCard() { currentState.ejectCard(this); }

    public void processWithdrawal(double amount) {
        try {
            Map<Integer, Integer> dispensed = cashDispenser.dispense(amount);
            boolean debited = currentAccount.debit(amount);

            if (!debited) {
                // This shouldn't happen (validation passed) but defensive check
                System.out.println("Debit failed — please contact support");
                atm.setState(new PinVerifiedState());
                return;
            }

            System.out.println("Dispensing ₹" + amount + ":");
            dispensed.forEach((denom, count) ->
                System.out.println("  ₹" + denom + " × " + count));
        } catch (Exception e) {
            System.out.println("Dispensing error: " + e.getMessage());
        } finally {
            currentState = new PinVerifiedState();  // back to menu
        }
    }

    public void blockCard() { /* notify bank to block card */ }
    public Account findAccount(String cardNumber) { return accounts.get(cardNumber); }

    // Accessors for states
    public void setState(ATMState state) {
        System.out.println("[ATM] " + currentState.getName() + " → " + state.getName());
        this.currentState = state;
    }
    public Account getCurrentAccount() { return currentAccount; }
    public void setCurrentAccount(Account account) { this.currentAccount = account; }
    public CashDispenser getCashDispenser() { return cashDispenser; }
}
```

---

## Design Decisions

| Decision | Choice | Why |
|---|---|---|
| State pattern | Yes | ATM has 4 distinct states with completely different behavior |
| Validation | Chain of Responsibility | Each check is independent; adding new check = new class |
| CashDispenser algorithm | Greedy (largest denomination first) | Minimizes notes dispensed |
| `synchronized` on ATM methods | Yes | ATM is physically single-user, but software may have concurrent admin calls |

---

## Concurrency Follow-up Questions

**Q: `Account.debit()` is `synchronized`. Why? Can't we rely on ATM-level synchronization?**
> Defense in depth. ATM methods are synchronized, but the Account object might be used from multiple places (online banking, other ATMs linked to same account). The account's `debit()` being synchronized ensures no race condition regardless of which code path calls it.

**Q: What happens if the ATM dispenses cash but `account.debit()` fails (DB unreachable)?**
> This is a **distributed transaction** problem (two-phase commit). Cash dispensed but DB not updated = money given for free. Practical solutions:
> (1) Debit FIRST (pessimistic), then dispense. If dispense fails, re-credit account.
> (2) Use a saga pattern: record "pending withdrawal", dispense, confirm. If confirm fails (timeout), ATM-level reconciliation job detects and re-credits.
> (3) Keep local transaction log in ATM hardware — ATM records everything; bank syncs periodically.

**Q: The PIN attempt counter is in `CardInsertedState` instance. What if the state is recreated?**
> Good catch! If `setState(new CardInsertedState())` is called for any reason, the counter resets. The counter should be in the Account or ATM context, not the State object. Fix:
> ```java
> // In ATM context:
> private int pinAttempts = 0;
>
> // In CardInsertedState:
> atm.incrementPinAttempts();
> if (atm.getPinAttempts() >= 3) { ... }
> ```

**Q: How do you handle session timeout (user inserts card but walks away)?**
> Use a `ScheduledExecutorService` to schedule a timeout task when entering `CardInsertedState`. If no interaction within 60 seconds, auto-eject the card:
> ```java
> ScheduledFuture<?> timeoutTask = scheduler.schedule(() -> {
>     atm.ejectCard();
>     System.out.println("Session timed out");
> }, 60, TimeUnit.SECONDS);
> ```
> Cancel the task when the user successfully moves to `PinVerifiedState`.
