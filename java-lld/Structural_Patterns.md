# Structural Patterns — Deep Dive

> Structural patterns answer: **How do you compose classes and objects into larger structures?**
> Key theme: flexibility, extensibility, and keeping components loosely coupled.

---

## Pattern Quick Reference

| Pattern | Problem it Solves | Key Signal |
|---|---|---|
| Adapter | Two incompatible interfaces | "legacy system", "third-party library" |
| Decorator | Add behavior dynamically | "add features at runtime", "avoid class explosion" |
| Proxy | Control/intercept object access | "lazy loading", "caching", "access control", "logging" |
| Facade | Simplify complex subsystem | "single entry point", "hide complexity" |
| Composite | Tree structures | "file/folder", "organization hierarchy" |
| Bridge | Two independent dimensions of change | "abstraction + implementation evolve separately" |
| Flyweight | Many similar objects (memory optimization) | "thousands of objects", "shared state" |

---

## 1. Adapter

**Problem:** Make two incompatible interfaces work together without modifying existing code.
**Mental model:** A power socket adapter. The plug is fixed; the socket is fixed. The adapter bridges them.

```java
// ─── TARGET: What your system expects ───
public interface PaymentProcessor {
    boolean processPayment(String customerId, double amount, String currency);
}

// ─── ADAPTEE: Legacy or third-party code (can't modify) ───
public class LegacyBillingSystem {
    public int chargeCustomer(int customerId, long amountInPaise) {
        // returns 0 for success, error code otherwise
        System.out.println("Legacy system charging: " + amountInPaise + " paise");
        return 0;
    }
}

// ─── ADAPTER: Makes the two compatible ───
public class LegacyBillingAdapter implements PaymentProcessor {
    private final LegacyBillingSystem legacySystem;

    public LegacyBillingAdapter(LegacyBillingSystem legacySystem) {
        this.legacySystem = legacySystem;
    }

    @Override
    public boolean processPayment(String customerId, double amount, String currency) {
        // Translate: String → int, rupees → paise, boolean return → int return
        int legacyId = Integer.parseInt(customerId);
        long amountInPaise = (long)(amount * 100);
        int result = legacySystem.chargeCustomer(legacyId, amountInPaise);
        return result == 0;  // translate int return to boolean
    }
}

// Client code uses only PaymentProcessor interface — unaware of legacy system
public class CheckoutService {
    private final PaymentProcessor processor;

    public CheckoutService(PaymentProcessor processor) { this.processor = processor; }

    public void checkout(String customerId, double amount) {
        if (!processor.processPayment(customerId, amount, "INR")) {
            throw new RuntimeException("Payment failed");
        }
    }
}
```

---

### Adapter — Deep Follow-up Questions

**Q: Adapter vs Facade — one-line difference?**
> **Adapter** changes the interface of ONE object to match what the client expects (compatibility bridge). **Facade** provides a SIMPLIFIED interface over a complex subsystem of MULTIPLE objects (complexity hiding). Adapter changes the "shape" of the interface; Facade reduces the surface area.

**Q: Object Adapter vs Class Adapter?**
> Object Adapter (shown above) uses composition — holds a reference to the Adaptee. **Class Adapter** uses multiple inheritance (only possible in languages that support it, like C++). Java doesn't support multiple class inheritance, so Java adapters are always Object Adapters.

**Q: Is Adapter a violation of OCP since the client code changes?**
> No. The client code doesn't change — it still uses `PaymentProcessor`. The adapter is a NEW class that wraps the old one. Existing code is untouched.

**Q: When would you NOT use Adapter?**
> If you control both sides (you own both the client and the adaptee code), prefer modifying one of them directly. Adapters add an indirection layer, which is justified only when you can't modify the source.

**Q: How do you handle thread safety in an Adapter?**
> If the underlying `Adaptee` is not thread-safe, the Adapter must either: (a) synchronize calls to the adaptee, or (b) create a new Adaptee instance per thread. Document this behavior clearly.

---

## 2. Decorator

**Problem:** Add behavior to objects at runtime without modifying the class. Avoids class explosion from combinatorial inheritance.
**Mental model:** Stack of transparent layers. Each adds something; the core object stays the same.

```java
// ─── COMPONENT interface ───
public interface DataSource {
    void writeData(String data);
    String readData();
}

// ─── CONCRETE COMPONENT: Core behavior ───
public class FileDataSource implements DataSource {
    private final String filename;

    public FileDataSource(String filename) { this.filename = filename; }

    @Override
    public void writeData(String data) {
        System.out.println("Writing to file: " + data);
        // actual file write
    }

    @Override
    public String readData() {
        System.out.println("Reading from file");
        return "raw data";
    }
}

// ─── BASE DECORATOR: wraps a DataSource ───
public abstract class DataSourceDecorator implements DataSource {
    protected final DataSource wrapped;  // reference to the component

    public DataSourceDecorator(DataSource source) { this.wrapped = source; }

    @Override
    public void writeData(String data) { wrapped.writeData(data); }  // delegate by default

    @Override
    public String readData() { return wrapped.readData(); }
}

// ─── CONCRETE DECORATOR 1: Encryption ───
public class EncryptionDecorator extends DataSourceDecorator {
    public EncryptionDecorator(DataSource source) { super(source); }

    @Override
    public void writeData(String data) {
        String encrypted = encrypt(data);
        wrapped.writeData(encrypted);  // write encrypted version
    }

    @Override
    public String readData() {
        String encrypted = wrapped.readData();
        return decrypt(encrypted);  // decrypt on read
    }

    private String encrypt(String data) { return "ENC[" + data + "]"; }
    private String decrypt(String data) { return data.replace("ENC[", "").replace("]", ""); }
}

// ─── CONCRETE DECORATOR 2: Compression ───
public class CompressionDecorator extends DataSourceDecorator {
    public CompressionDecorator(DataSource source) { super(source); }

    @Override
    public void writeData(String data) {
        String compressed = compress(data);
        wrapped.writeData(compressed);
    }

    @Override
    public String readData() {
        return decompress(wrapped.readData());
    }

    private String compress(String data) { return "ZIP[" + data + "]"; }
    private String decompress(String data) { return data.replace("ZIP[", "").replace("]", ""); }
}

// Usage — compose behaviors at runtime
DataSource source = new FileDataSource("data.txt");

// Write: compress → encrypt → write to file
DataSource layered = new EncryptionDecorator(
                         new CompressionDecorator(
                             source));

layered.writeData("sensitive data");
// Output: write(ENC[ZIP[sensitive data]])
```

---

### Decorator — Deep Follow-up Questions

**Q: Decorator vs Inheritance — when do you prefer Decorator?**
> Use Decorator when you have **combinatorial features** that can be mixed. With 5 features (Encryption, Compression, Caching, Logging, Validation), inheritance would need up to 2^5 = 32 classes to cover all combinations. Decorator needs only 5 decorator classes — combine them at runtime as needed.
>
> Also use Decorator when you can't subclass (final class) or when you need to add behavior to an instance rather than the whole class.

**Q: What is the order of decorators important for?**
> Yes, critically. `Encrypt(Compress(data))` and `Compress(Encrypt(data))` are different. In the example: write order is Compress first, then Encrypt (inner → outer). Read order is Decrypt first, then Decompress (reversal). Always draw the stack when discussing this in interviews.

**Q: Is Decorator thread-safe?**
> The Decorator itself (the wrapping logic) is typically stateless, so thread safety depends on the wrapped object. If `FileDataSource.writeData()` isn't synchronized, wrapping it with decorators doesn't make it thread-safe. You'd add a `SynchronizationDecorator` or use `synchronized` methods.

**Q: Java's `java.io` uses Decorator extensively — can you explain?**
> Yes. `InputStream` is the component. `FileInputStream` is the concrete component. `BufferedInputStream`, `DataInputStream`, `GZIPInputStream` are all decorators:
> ```java
> InputStream in = new BufferedInputStream(
>                      new GZIPInputStream(
>                          new FileInputStream("data.gz")));
> ```

**Q: Decorator vs Proxy — they look the same structurally. What's the difference?**
> **Structure:** Identical — both wrap an object implementing the same interface.
> **Intent:** Different. Decorator **adds behavior** (encryption, logging, caching). Proxy **controls access** to the object (lazy loading, access restriction, remote invocation). A Proxy often manages the lifecycle of the wrapped object; Decorator usually receives it from outside.

---

## 3. Proxy

**Problem:** Control access to an object. Add a surrogate that intercepts calls for: lazy loading, caching, security, logging, remote access.

```java
// ─── SUBJECT interface ───
public interface Image {
    void display();
    int getWidth();
    int getHeight();
}

// ─── REAL SUBJECT: expensive to create ───
public class HighResolutionImage implements Image {
    private final String filename;
    private byte[] imageData;

    public HighResolutionImage(String filename) {
        this.filename = filename;
        loadFromDisk();  // expensive! reads entire file
    }

    private void loadFromDisk() {
        System.out.println("Loading high-res image from disk: " + filename);
        // imagine reading 50MB file
        this.imageData = new byte[50 * 1024 * 1024];
    }

    @Override
    public void display() { System.out.println("Displaying image: " + filename); }

    @Override
    public int getWidth() { return 4000; }

    @Override
    public int getHeight() { return 3000; }
}

// ─── VIRTUAL PROXY: lazy loading + caching ───
public class ImageProxy implements Image {
    private final String filename;
    private HighResolutionImage realImage;  // null until first use

    public ImageProxy(String filename) {
        this.filename = filename;
        // No loading here! Proxy is cheap to create.
    }

    @Override
    public void display() {
        if (realImage == null) {
            realImage = new HighResolutionImage(filename);  // lazy load on first use
        }
        realImage.display();
    }

    @Override
    public int getWidth() {
        // Can answer some queries without loading: return thumbnail dimensions
        if (realImage == null) return 100;  // placeholder width
        return realImage.getWidth();
    }

    @Override
    public int getHeight() {
        if (realImage == null) return 75;
        return realImage.getHeight();
    }
}

// ─── PROTECTION PROXY: access control ───
public class SecureImageProxy implements Image {
    private final Image realImage;
    private final User currentUser;

    public SecureImageProxy(Image image, User user) {
        this.realImage = image;
        this.currentUser = user;
    }

    @Override
    public void display() {
        if (!currentUser.hasPermission("VIEW_IMAGE")) {
            throw new SecurityException("Access denied for: " + currentUser.getName());
        }
        realImage.display();
    }

    @Override
    public int getWidth() { return realImage.getWidth(); }

    @Override
    public int getHeight() { return realImage.getHeight(); }
}
```

**Caching Proxy Example (very common in interviews):**
```java
public class CachedUserRepository implements UserRepository {
    private final UserRepository dbRepository;
    private final Map<Long, User> cache = new ConcurrentHashMap<>();

    public CachedUserRepository(UserRepository dbRepository) {
        this.dbRepository = dbRepository;
    }

    @Override
    public User findById(Long id) {
        return cache.computeIfAbsent(id, dbRepository::findById);  // load if absent
    }

    @Override
    public void save(User user) {
        dbRepository.save(user);
        cache.put(user.getId(), user);  // keep cache consistent
    }

    @Override
    public void delete(Long id) {
        dbRepository.delete(id);
        cache.remove(id);  // invalidate cache
    }
}
```

---

### Proxy — Deep Follow-up Questions

**Q: Four types of Proxy — explain each?**

| Type | What it does | Example |
|---|---|---|
| Virtual Proxy | Lazy loading — delays expensive initialization | Image thumbnail vs full image |
| Protection Proxy | Access control — checks permissions before delegating | Admin vs regular user access |
| Caching Proxy | Stores results, returns cached for repeated calls | `CachedUserRepository` above |
| Remote Proxy | Represents object in another address space | RMI stub, gRPC client |

**Q: Is the Caching Proxy thread-safe? What issues can arise?**
> `ConcurrentHashMap` makes individual operations atomic, but `computeIfAbsent` is only atomically safe for the map. If the underlying `dbRepository.findById()` is called concurrently for the same key, it might be called twice (acceptable for reads). For writes, you need careful cache invalidation to avoid stale reads.

**Q: How does Spring AOP relate to the Proxy pattern?**
> Spring's `@Transactional`, `@Cacheable`, `@Secured` all work via **dynamic proxies** (either JDK dynamic proxy or CGLIB). When you call a `@Transactional` method, you're actually calling a proxy that starts/commits the transaction, then delegates to your actual method.

**Q: Proxy vs Decorator — they have identical structure. How do you tell them apart in code review?**
> Look at the **intent**: Is the wrapper adding new behavior for the caller (Decorator), or is it controlling/mediating access to the real object (Proxy)? A Proxy often creates or manages the lifecycle of the wrapped object. A Decorator receives it from outside.

---

## 4. Facade

**Problem:** Provide a simple, unified interface to a complex subsystem. Clients don't need to know about internal components.

```java
// Complex subsystem classes
class VideoConverter { void convert(String file, String format) { } }
class AudioDecoder { void decode(String file) { } }
class BitrateReader { byte[] read(String file, String format) { return null; } }
class CodecFactory { Object extract(byte[] bytes, String format) { return null; } }

// Facade: simple interface hiding all complexity
public class VideoConversionFacade {
    private final VideoConverter converter = new VideoConverter();
    private final AudioDecoder decoder = new AudioDecoder();
    private final BitrateReader reader = new BitrateReader();
    private final CodecFactory codecFactory = new CodecFactory();

    // Client calls ONE method. Internally orchestrates 4 subsystems.
    public File convertVideo(String filename, String targetFormat) {
        decoder.decode(filename);
        byte[] buffer = reader.read(filename, "ogg");
        Object sourceCodec = codecFactory.extract(buffer, "ogg");
        converter.convert(filename, targetFormat);
        return new File(filename.replace(".ogg", "." + targetFormat));
    }
}
```

**Q: Facade vs Adapter?**
> Facade wraps a **subsystem** (multiple classes) with a **simplified** interface. Adapter wraps **one object** with a **compatible** interface. Facade is about simplification; Adapter is about compatibility.

**Q: Does Facade violate DIP?**
> Yes, typically. The Facade creates its subsystem objects directly (tight coupling). For testability, inject subsystems via constructor. But Facade's main goal is simplicity — if you inject all 4 subsystems, you lose simplicity. Acceptable trade-off for integration/infrastructure code.

---

## 5. Composite

**Problem:** Treat individual objects and groups of objects uniformly. Perfect for tree-like hierarchies.

```java
// ─── COMPONENT interface ───
public interface FileSystemNode {
    String getName();
    long getSize();
    void print(String indent);
}

// ─── LEAF: has no children ───
public class File implements FileSystemNode {
    private final String name;
    private final long size;

    public File(String name, long size) { this.name = name; this.size = size; }

    @Override public String getName() { return name; }
    @Override public long getSize() { return size; }
    @Override public void print(String indent) {
        System.out.println(indent + "📄 " + name + " (" + size + " bytes)");
    }
}

// ─── COMPOSITE: can contain children ───
public class Folder implements FileSystemNode {
    private final String name;
    private final List<FileSystemNode> children = new ArrayList<>();

    public Folder(String name) { this.name = name; }

    public void add(FileSystemNode node) { children.add(node); }
    public void remove(FileSystemNode node) { children.remove(node); }

    @Override public String getName() { return name; }

    @Override
    public long getSize() {
        return children.stream().mapToLong(FileSystemNode::getSize).sum();  // recursive
    }

    @Override
    public void print(String indent) {
        System.out.println(indent + "📁 " + name);
        for (FileSystemNode child : children) {
            child.print(indent + "  ");  // recursive
        }
    }
}

// Usage — treat files and folders uniformly
Folder root = new Folder("root");
root.add(new File("readme.txt", 1024));
Folder src = new Folder("src");
src.add(new File("Main.java", 4096));
src.add(new File("Utils.java", 2048));
root.add(src);

root.print("");    // prints entire tree
root.getSize();    // recursively sums all files
```

**Q: When NOT to use Composite?**
> When all nodes are truly equal (no leaf/branch distinction). Also when operations don't make sense for leaves (e.g., `addChild()` on a File). Be careful about the interface — some operations belong only on Composite. Using a `throw UnsupportedOperationException()` on leaves for child-management methods is acceptable.

---

## 6. Bridge

**Problem:** Two dimensions of variation — separate them so each can grow independently. Avoids class explosion.
**Mental model:** A TV remote (abstraction) can control different TV brands (implementation) independently.

```java
// ─── IMPLEMENTATION side (can vary) ───
public interface MessageSender {
    void sendMessage(String recipient, String content);
}

class EmailSender implements MessageSender {
    public void sendMessage(String to, String content) {
        System.out.println("Email to " + to + ": " + content);
    }
}

class SMSSender implements MessageSender {
    public void sendMessage(String to, String content) {
        System.out.println("SMS to " + to + ": " + content);
    }
}

// ─── ABSTRACTION side (can vary independently) ───
public abstract class Notification {
    protected MessageSender sender;  // ← the bridge

    public Notification(MessageSender sender) { this.sender = sender; }

    public abstract void notify(String recipient, String message);
}

class UrgentNotification extends Notification {
    public UrgentNotification(MessageSender sender) { super(sender); }

    @Override
    public void notify(String recipient, String message) {
        sender.sendMessage(recipient, "URGENT: " + message);  // adds urgency prefix
    }
}

class ScheduledNotification extends Notification {
    private final LocalDateTime scheduledTime;

    public ScheduledNotification(MessageSender sender, LocalDateTime time) {
        super(sender);
        this.scheduledTime = time;
    }

    @Override
    public void notify(String recipient, String message) {
        System.out.println("Scheduled at: " + scheduledTime);
        sender.sendMessage(recipient, message);
    }
}

// 2 notification types × 2 senders = 4 combinations without any class explosion
Notification n1 = new UrgentNotification(new EmailSender());
Notification n2 = new UrgentNotification(new SMSSender());
Notification n3 = new ScheduledNotification(new EmailSender(), LocalDateTime.now().plusHours(2));
```

**Q: Bridge vs Strategy?**
> Structurally similar — both inject a varying implementation. **Bridge** is a structural pattern: separates abstraction from implementation as a design-time decision, both hierarchies can grow. **Strategy** is a behavioral pattern: focused on swapping algorithms at runtime; only one "strategy" dimension varies. Bridge solves "class explosion from two dimensions"; Strategy solves "pluggable algorithms".
