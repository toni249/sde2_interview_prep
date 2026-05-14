# Creational Patterns — Deep Dive

> Creational patterns answer: **Who creates the object, How, and When.**
> SDE2 focus: Thread safety, object lifecycle, avoiding tight coupling on construction.

---

## Pattern Quick Reference

| Pattern | Problem it Solves | Key Signal in Interview |
|---|---|---|
| Singleton | One instance needed globally | "only one", "global access" |
| Factory Method | Caller shouldn't know concrete type | "create based on type" |
| Abstract Factory | Families of related objects | "consistent family", "themes" |
| Builder | Complex object construction | "many optional params", "immutable" |
| Prototype | Expensive to create from scratch | "clone", "copy config" |

---

## 1. Singleton

**Problem:** Ensure only one instance exists. Provide a global access point.
**Use cases:** DB connection pool, Logger, Config manager, Thread pool.

### Implementation 1 — Double-Checked Locking (Most Common Interview Answer)

```java
public class DatabaseConnectionPool {
    // volatile prevents instruction reordering during construction
    private static volatile DatabaseConnectionPool instance;
    private final List<Connection> connections;

    private DatabaseConnectionPool() {
        connections = new ArrayList<>();
        // initialize pool
        for (int i = 0; i < 10; i++) {
            connections.add(createConnection());
        }
    }

    public static DatabaseConnectionPool getInstance() {
        if (instance == null) {                          // Check 1: fast path, no lock
            synchronized (DatabaseConnectionPool.class) {
                if (instance == null) {                  // Check 2: only one thread creates
                    instance = new DatabaseConnectionPool();
                }
            }
        }
        return instance;
    }

    public Connection getConnection() { /* ... */ return connections.get(0); }
    private Connection createConnection() { return null; /* real impl */ }
}
```

**Why `volatile`?**
Without `volatile`, instruction reordering can make `instance` non-null before the constructor finishes. Thread 2 could see `instance != null`, return a partially constructed object, and crash when using it.

**Why double-check?**
First check (outside `synchronized`) avoids locking on every call after initialization — critical for performance since `getInstance()` may be called millions of times. Second check (inside `synchronized`) prevents two threads that both passed the first check from both initializing.

### Implementation 2 — Enum Singleton (Bill Pugh, Most Robust)

```java
public enum AppConfig {
    INSTANCE;

    private final String dbUrl;
    private final int maxConnections;

    AppConfig() {
        // JVM guarantees this runs once, thread-safely
        this.dbUrl = System.getenv("DB_URL");
        this.maxConnections = Integer.parseInt(System.getenv("MAX_CONN"));
    }

    public String getDbUrl() { return dbUrl; }
    public int getMaxConnections() { return maxConnections; }
}

// Usage
String url = AppConfig.INSTANCE.getDbUrl();
```

**Why Enum is the best:**
- JVM guarantees single instantiation
- Serialization-safe (no new instance on deserialization)
- Reflection-safe (can't call private constructor via reflection on enums)
- Thread-safe by JVM class loading guarantees

### Implementation 3 — Initialization-on-Demand Holder (Lazy, Thread-Safe, No `volatile`)

```java
public class Logger {
    private Logger() {}

    // Inner class is loaded ONLY when getInstance() is first called
    private static class Holder {
        static final Logger INSTANCE = new Logger();
    }

    public static Logger getInstance() {
        return Holder.INSTANCE;  // class loading is thread-safe
    }

    public void log(String msg) { System.out.println("[LOG] " + msg); }
}
```

This is lazy initialization without any explicit synchronization — JVM class loading is inherently thread-safe.

---

### Singleton — Deep Follow-up Questions

**Q: How can Singleton be broken? How to prevent each?**

| Attack | How it works | Prevention |
|---|---|---|
| Reflection | `Constructor c = clazz.getDeclaredConstructor(); c.setAccessible(true);` | Throw exception in constructor if instance already exists |
| Serialization | `readObject()` creates a new instance | Override `readResolve()` to return existing instance |
| Cloning | `clone()` creates a copy | Override `clone()` to throw `CloneNotSupportedException` |
| Multiple ClassLoaders | Each ClassLoader creates its own instance | Use Enum (immune) or ensure single ClassLoader |

```java
// Reflection attack prevention
private DatabaseConnectionPool() {
    if (instance != null) {
        throw new IllegalStateException("Use getInstance()");
    }
}

// Serialization attack prevention
protected Object readResolve() {
    return instance;
}
```

**Q: Is Singleton an anti-pattern?**
> It can be. Problems: (1) Hard to unit test — you can't inject a mock singleton. (2) Hidden global state makes code hard to reason about. (3) Violates SRP if used as a service locator.
>
> Better alternative: Use dependency injection frameworks (Spring) to manage single-instance beans. Same single instance guarantee but injectable and testable.

**Q: Singleton vs Static class — what's the difference?**
> Static class: all methods/fields are static, no object created, can't implement interfaces, can't be lazy.
> Singleton: a real object (implements interfaces, can be mocked/replaced, can be lazy, can have state with proper encapsulation). Prefer Singleton for testability and polymorphism.

**Q: Your Singleton holds a `List`. Is that thread-safe?**
> No. The Singleton instance itself is created safely (once). But reading/writing the list from multiple threads still requires synchronization. Use `Collections.synchronizedList()` or `CopyOnWriteArrayList` for the list.

---

## 2. Factory Method

**Problem:** A class needs to create objects, but shouldn't be coupled to specific concrete classes.
**Use cases:** Different notification types (SMS, Email, Push), different DB drivers.

```java
// Product interface
public interface Notification {
    void send(String to, String message);
}

// Concrete products
public class SMSNotification implements Notification {
    @Override
    public void send(String to, String message) {
        System.out.println("SMS to " + to + ": " + message);
        // Twilio API call
    }
}

public class EmailNotification implements Notification {
    @Override
    public void send(String to, String message) {
        System.out.println("Email to " + to + ": " + message);
        // SMTP call
    }
}

public class PushNotification implements Notification {
    @Override
    public void send(String to, String message) {
        System.out.println("Push to " + to + ": " + message);
        // FCM call
    }
}

// Creator — defines the factory method
public abstract class NotificationService {
    public abstract Notification createNotification();  // ← Factory Method

    // Template method uses the factory method
    public void notifyUser(String userId, String message) {
        Notification n = createNotification();  // polymorphic creation
        String contact = getContactFor(userId, n.getClass());
        n.send(contact, message);
    }

    private String getContactFor(String userId, Class<?> type) { return "user@example.com"; }
}

// Concrete creators
public class SMSNotificationService extends NotificationService {
    @Override
    public Notification createNotification() { return new SMSNotification(); }
}

public class EmailNotificationService extends NotificationService {
    @Override
    public Notification createNotification() { return new EmailNotification(); }
}
```

**Simpler variant — Static Factory (when inheritance isn't needed):**
```java
public class NotificationFactory {
    public static Notification create(String type) {
        return switch (type.toUpperCase()) {
            case "SMS"   -> new SMSNotification();
            case "EMAIL" -> new EmailNotification();
            case "PUSH"  -> new PushNotification();
            default      -> throw new IllegalArgumentException("Unknown type: " + type);
        };
    }
}
```

---

### Factory — Deep Follow-up Questions

**Q: Simple Factory vs Factory Method vs Abstract Factory — clear difference?**

| | Simple Factory | Factory Method | Abstract Factory |
|---|---|---|---|
| Structure | One class with if/switch | Inheritance (abstract creator + concrete creators) | Interface with multiple factory methods |
| Extensibility | Violates OCP (modify factory to add type) | OCP-compliant (add new subclass) | OCP-compliant |
| Use case | Few types, unlikely to change | One product, many variants | Multiple related products that must go together |
| Example | `NotificationFactory.create(type)` | `SMSNotificationService` | `UIComponentFactory` with `createButton()` + `createScrollbar()` |

**Q: Why use Factory instead of just calling `new SMSNotification()` directly?**
> (1) Decoupling: Caller code doesn't reference the concrete class name → easy to swap implementations. (2) Centralized creation logic: validation, caching, initialization in one place. (3) Supports polymorphism: code works with interface `Notification`, not a specific class.

**Q: Can you make the Simple Factory thread-safe?**
> Yes. If the factory is stateless (just creates new objects), it's already thread-safe — each call creates its own local objects. If it has shared state (like a counter or cache), you need synchronization.

**Q: How would you add a new notification type without modifying existing code?**
> With Factory Method pattern, add a new class `WhatsAppNotificationService extends NotificationService`. Zero changes to existing code. This is why Factory Method is OCP-compliant while Simple Factory is not.

---

## 3. Builder

**Problem:** Constructing complex objects with many optional parameters. Avoids "telescoping constructors" and mutable setters.
**Use cases:** HTTP Request, SQL Query builder, complex config objects.

```java
public class HttpRequest {
    // Required
    private final String url;
    private final String method;

    // Optional
    private final Map<String, String> headers;
    private final String body;
    private final int timeoutMs;
    private final boolean followRedirects;
    private final int retryCount;

    // Private constructor — only Builder can call this
    private HttpRequest(Builder builder) {
        this.url = builder.url;
        this.method = builder.method;
        this.headers = Collections.unmodifiableMap(builder.headers);
        this.body = builder.body;
        this.timeoutMs = builder.timeoutMs;
        this.followRedirects = builder.followRedirects;
        this.retryCount = builder.retryCount;
    }

    // Getters only — object is IMMUTABLE
    public String getUrl() { return url; }
    public String getMethod() { return method; }
    public Map<String, String> getHeaders() { return headers; }
    public String getBody() { return body; }

    public static class Builder {
        // Required fields
        private final String url;
        private final String method;

        // Optional fields with defaults
        private Map<String, String> headers = new HashMap<>();
        private String body = null;
        private int timeoutMs = 30000;
        private boolean followRedirects = true;
        private int retryCount = 0;

        public Builder(String url, String method) {  // required in constructor
            if (url == null || url.isEmpty()) throw new IllegalArgumentException("URL required");
            this.url = url;
            this.method = method;
        }

        public Builder header(String key, String value) {
            this.headers.put(key, value);
            return this;  // fluent API
        }

        public Builder body(String body) {
            this.body = body;
            return this;
        }

        public Builder timeout(int ms) {
            if (ms <= 0) throw new IllegalArgumentException("Timeout must be positive");
            this.timeoutMs = ms;
            return this;
        }

        public Builder followRedirects(boolean follow) {
            this.followRedirects = follow;
            return this;
        }

        public Builder retryCount(int count) {
            this.retryCount = count;
            return this;
        }

        public HttpRequest build() {
            // Cross-field validation before building
            if ("POST".equals(method) && body == null) {
                throw new IllegalStateException("POST request must have a body");
            }
            return new HttpRequest(this);
        }
    }
}

// Usage — readable, no positional confusion
HttpRequest request = new HttpRequest.Builder("https://api.razorpay.com/v1/payments", "POST")
    .header("Authorization", "Bearer token123")
    .header("Content-Type", "application/json")
    .body("{\"amount\": 50000}")
    .timeout(5000)
    .retryCount(3)
    .build();
```

---

### Builder — Deep Follow-up Questions

**Q: Builder vs setters on a mutable object — what's the real advantage?**
> (1) **Immutability:** Builder produces an immutable object. Setters produce a mutable object that can be changed after creation — unsafe in multithreaded contexts and harder to reason about.
> (2) **Validation at build time:** You can do cross-field validation in `build()`. With setters, you'd need to validate every combination on every setter call.
> (3) **Readability:** Named parameters via fluent API. Compare `new User("John", null, null, 25, true, false)` vs `.name("John").age(25).active(true).build()`.

**Q: Is Builder thread-safe?**
> The builder itself is typically NOT thread-safe — it's meant to be used by a single thread to construct one object. The **built object** should be immutable and therefore inherently thread-safe (no synchronization needed if state never changes after construction).

**Q: How is Builder different from Abstract Factory?**
> Builder constructs **one complex object step by step** — the director controls the construction process. Abstract Factory creates **families of related objects** in one call — emphasis is on which objects to create together, not how to build one complex object.

**Q: When would you NOT use Builder?**
> For simple objects with 2-3 fields. Builder adds significant boilerplate (inner class, all the setters). For those cases, a static factory method or just a constructor is cleaner.

---

## 4. Abstract Factory

**Problem:** Create families of related objects without specifying concrete classes. Ensures objects from the same family are used together.
**Use cases:** UI themes (Light/Dark), cross-platform UI (Windows/Mac/Linux), database drivers (MySQL/PostgreSQL).

```java
// Abstract product interfaces
interface Button { void render(); void onClick(); }
interface TextField { void render(); String getValue(); }
interface Scrollbar { void render(); void scroll(int delta); }

// Family 1: Light theme products
class LightButton implements Button {
    public void render() { System.out.println("Rendering light button"); }
    public void onClick() { System.out.println("Light button clicked"); }
}
class LightTextField implements TextField {
    public void render() { System.out.println("Rendering light text field"); }
    public String getValue() { return ""; }
}

// Family 2: Dark theme products
class DarkButton implements Button {
    public void render() { System.out.println("Rendering dark button"); }
    public void onClick() { System.out.println("Dark button clicked"); }
}
class DarkTextField implements TextField {
    public void render() { System.out.println("Rendering dark text field"); }
    public String getValue() { return ""; }
}

// Abstract Factory
interface UIComponentFactory {
    Button createButton();
    TextField createTextField();
    Scrollbar createScrollbar();
}

// Concrete factories — guarantee same family
class LightThemeFactory implements UIComponentFactory {
    public Button createButton() { return new LightButton(); }
    public TextField createTextField() { return new LightTextField(); }
    public Scrollbar createScrollbar() { return new LightScrollbar(); }
}

class DarkThemeFactory implements UIComponentFactory {
    public Button createButton() { return new DarkButton(); }
    public TextField createTextField() { return new DarkTextField(); }
    public Scrollbar createScrollbar() { return new DarkScrollbar(); }
}

// Client — only uses abstractions
class UIScreen {
    private final Button button;
    private final TextField textField;

    UIScreen(UIComponentFactory factory) {  // inject factory
        this.button = factory.createButton();     // always consistent
        this.textField = factory.createTextField();
    }

    void render() {
        button.render();
        textField.render();
    }
}
```

---

### Abstract Factory — Deep Follow-up Questions

**Q: Factory Method vs Abstract Factory — one-line difference?**
> Factory Method creates **one product** (one factory method). Abstract Factory creates **a family of related products** (multiple factory methods, all products guaranteed to be compatible).

**Q: How do you add a new product type (e.g., Checkbox) to an existing Abstract Factory?**
> You must add `createCheckbox()` to the `UIComponentFactory` interface — this forces changes to all existing concrete factories. This violates OCP. This is a known trade-off: Abstract Factory is closed for new products but open for new families (new themes).

---

## 5. Prototype

**Problem:** Creating new objects is expensive (DB fetch, heavy initialization). Prefer cloning a template.
**Use cases:** Config objects loaded from DB, game characters with base stats, object caching.

```java
// Mark as Cloneable
public class GameCharacter implements Cloneable {
    private String name;
    private int health;
    private int attack;
    private List<String> inventory;  // ← IMPORTANT: need deep copy

    public GameCharacter(String name, int health, int attack) {
        this.name = name;
        this.health = health;
        this.attack = attack;
        this.inventory = new ArrayList<>();
    }

    // SHALLOW COPY — inventory list is SHARED (dangerous!)
    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();  // copies primitive fields; list reference is shared
    }

    // DEEP COPY — inventory list is also copied
    public GameCharacter deepClone() {
        GameCharacter clone = new GameCharacter(this.name, this.health, this.attack);
        clone.inventory = new ArrayList<>(this.inventory);  // new list with same elements
        return clone;
    }
}

// Prototype Registry
public class CharacterRegistry {
    private final Map<String, GameCharacter> templates = new HashMap<>();

    public void register(String key, GameCharacter prototype) {
        templates.put(key, prototype);
    }

    public GameCharacter getClone(String key) {
        GameCharacter template = templates.get(key);
        if (template == null) throw new IllegalArgumentException("Unknown template: " + key);
        return template.deepClone();  // always return a fresh copy
    }
}
```

---

### Prototype — Deep Follow-up Questions

**Q: Shallow copy vs Deep copy — when does it matter?**
> Shallow copy copies object references (both original and clone point to the same nested objects). If you modify a nested object in the clone, the original is affected. Deep copy creates new copies of all nested objects too. Rule: if your object has mutable fields that are objects (Lists, Maps, other custom objects), you need deep copy.

**Q: How is Prototype related to Object Pool pattern?**
> They're complementary. Prototype creates objects by cloning (avoids expensive construction). Object Pool reuses existing objects (avoids creation AND GC pressure). You'd use Prototype to pre-populate an Object Pool.

**Q: `Object.clone()` vs copy constructor vs custom `copy()` method — which to use?**
> `Object.clone()` is tricky — you must implement `Cloneable` (marker interface), handle `CloneNotSupportedException`, and it's a shallow copy by default. Most Java experts recommend **copy constructors** (`new Object(original)`) or a custom `deepClone()` method as clearer and safer alternatives.
