# LLD: Parking Lot

> Frequency: Very High | Difficulty: Medium
> Tests: OOP modeling, State pattern, Factory pattern, concurrency

---

## Step 1 — Clarifying Questions (Ask These First)

- What types of vehicles? (Bike, Car, Truck)
- Multiple floors? Multiple entrances/exits?
- Pricing — flat rate vs per-hour vs per-vehicle-type?
- How is a slot assigned — nearest to entrance? Any preference?
- Do we need payment processing? Display boards?
- Concurrent entries (multi-threaded)?

---

## Step 2 — Core Requirements

1. Multiple floors, each with multiple slots of different types
2. Vehicle types: Bike, Car, Truck (each needs a specific slot size)
3. Issue a ticket on entry, process payment on exit
4. Slot assignment: nearest available slot
5. Thread-safe (multiple entry/exit simultaneously)

---

## Step 3 — Entities (Nouns)

```
ParkingLot   → singleton, top-level manager
Floor        → each floor has slots
ParkingSlot  → individual slot, has type and status
Vehicle      → Bike / Car / Truck (type hierarchy)
Ticket       → issued on entry, has entry time
Payment      → calculated on exit
PricingStrategy → strategy for calculating fee
```

---

## Step 4 — Class Diagram

```
ParkingLot (Singleton)
├── floors: List<Floor>               [Composition]
├── entryStrategy: SlotAssignStrategy [Strategy]
└── pricingStrategy: PricingStrategy  [Strategy]

Floor
├── floorNumber: int
└── slots: List<ParkingSlot>          [Composition]

ParkingSlot
├── slotId: String
├── slotType: SlotType (BIKE/CAR/TRUCK)
├── status: SlotStatus (AVAILABLE/OCCUPIED)
└── vehicle: Vehicle                  [Aggregation — vehicle exists independently]

Vehicle (abstract)
├── Car
├── Bike
└── Truck

Ticket
├── ticketId: String
├── vehicle: Vehicle
├── slot: ParkingSlot
└── entryTime: LocalDateTime

PricingStrategy (interface)
├── FlatRateStrategy
└── HourlyRateStrategy
```

---

## Step 5 — Full Java Code

```java
// ─── ENUMS ───
public enum SlotType { BIKE, CAR, TRUCK }
public enum SlotStatus { AVAILABLE, OCCUPIED }
public enum VehicleType { BIKE, CAR, TRUCK }

// ─── VEHICLE HIERARCHY ───
public abstract class Vehicle {
    private final String licensePlate;
    private final VehicleType vehicleType;

    protected Vehicle(String licensePlate, VehicleType vehicleType) {
        this.licensePlate = licensePlate;
        this.vehicleType = vehicleType;
    }

    public String getLicensePlate() { return licensePlate; }
    public VehicleType getVehicleType() { return vehicleType; }
    public abstract SlotType getRequiredSlotType();
}

public class Car extends Vehicle {
    public Car(String plate) { super(plate, VehicleType.CAR); }
    @Override public SlotType getRequiredSlotType() { return SlotType.CAR; }
}

public class Bike extends Vehicle {
    public Bike(String plate) { super(plate, VehicleType.BIKE); }
    @Override public SlotType getRequiredSlotType() { return SlotType.BIKE; }
}

public class Truck extends Vehicle {
    public Truck(String plate) { super(plate, VehicleType.TRUCK); }
    @Override public SlotType getRequiredSlotType() { return SlotType.TRUCK; }
}

// ─── PARKING SLOT ───
public class ParkingSlot {
    private final String slotId;
    private final SlotType slotType;
    private volatile SlotStatus status;  // volatile for visibility
    private Vehicle parkedVehicle;

    public ParkingSlot(String slotId, SlotType slotType) {
        this.slotId = slotId;
        this.slotType = slotType;
        this.status = SlotStatus.AVAILABLE;
    }

    public String getSlotId() { return slotId; }
    public SlotType getSlotType() { return slotType; }
    public SlotStatus getStatus() { return status; }
    public Vehicle getParkedVehicle() { return parkedVehicle; }

    // Synchronized to prevent two threads assigning same slot
    public synchronized boolean assignVehicle(Vehicle vehicle) {
        if (status == SlotStatus.OCCUPIED) return false;
        this.parkedVehicle = vehicle;
        this.status = SlotStatus.OCCUPIED;
        return true;
    }

    public synchronized void removeVehicle() {
        this.parkedVehicle = null;
        this.status = SlotStatus.AVAILABLE;
    }
}

// ─── FLOOR ───
public class Floor {
    private final int floorNumber;
    private final List<ParkingSlot> slots;

    public Floor(int floorNumber, List<ParkingSlot> slots) {
        this.floorNumber = floorNumber;
        this.slots = new ArrayList<>(slots);
    }

    public int getFloorNumber() { return floorNumber; }

    // Find first available slot of given type
    public Optional<ParkingSlot> findAvailableSlot(SlotType type) {
        return slots.stream()
            .filter(s -> s.getSlotType() == type && s.getStatus() == SlotStatus.AVAILABLE)
            .findFirst();
    }

    public int getAvailableCount(SlotType type) {
        return (int) slots.stream()
            .filter(s -> s.getSlotType() == type && s.getStatus() == SlotStatus.AVAILABLE)
            .count();
    }
}

// ─── TICKET ───
public class Ticket {
    private final String ticketId;
    private final Vehicle vehicle;
    private final ParkingSlot slot;
    private final LocalDateTime entryTime;

    public Ticket(String ticketId, Vehicle vehicle, ParkingSlot slot) {
        this.ticketId = ticketId;
        this.vehicle = vehicle;
        this.slot = slot;
        this.entryTime = LocalDateTime.now();
    }

    public String getTicketId() { return ticketId; }
    public Vehicle getVehicle() { return vehicle; }
    public ParkingSlot getSlot() { return slot; }
    public LocalDateTime getEntryTime() { return entryTime; }
}

// ─── PRICING STRATEGY ───
public interface PricingStrategy {
    double calculateFee(Ticket ticket, LocalDateTime exitTime);
}

public class HourlyRateStrategy implements PricingStrategy {
    private final Map<VehicleType, Double> rates;

    public HourlyRateStrategy() {
        rates = new EnumMap<>(VehicleType.class);
        rates.put(VehicleType.BIKE, 20.0);
        rates.put(VehicleType.CAR, 50.0);
        rates.put(VehicleType.TRUCK, 100.0);
    }

    @Override
    public double calculateFee(Ticket ticket, LocalDateTime exitTime) {
        long minutes = ChronoUnit.MINUTES.between(ticket.getEntryTime(), exitTime);
        long hours = (long) Math.ceil(minutes / 60.0);  // round up to next hour
        hours = Math.max(1, hours);  // minimum 1 hour
        double ratePerHour = rates.get(ticket.getVehicle().getVehicleType());
        return hours * ratePerHour;
    }
}

// ─── PARKING LOT (Singleton) ───
public class ParkingLot {
    private static volatile ParkingLot instance;

    private final String name;
    private final List<Floor> floors;
    private final PricingStrategy pricingStrategy;
    private final Map<String, Ticket> activeTickets = new ConcurrentHashMap<>();
    private final AtomicInteger ticketCounter = new AtomicInteger(0);

    private ParkingLot(String name, List<Floor> floors, PricingStrategy pricingStrategy) {
        this.name = name;
        this.floors = Collections.unmodifiableList(floors);
        this.pricingStrategy = pricingStrategy;
    }

    public static ParkingLot getInstance(String name, List<Floor> floors, PricingStrategy strategy) {
        if (instance == null) {
            synchronized (ParkingLot.class) {
                if (instance == null) {
                    instance = new ParkingLot(name, floors, strategy);
                }
            }
        }
        return instance;
    }

    // ─── PARK: find slot, assign, issue ticket ───
    public Ticket park(Vehicle vehicle) {
        SlotType required = vehicle.getRequiredSlotType();

        // Find available slot across floors (first-fit strategy)
        for (Floor floor : floors) {
            Optional<ParkingSlot> slotOpt = floor.findAvailableSlot(required);
            if (slotOpt.isPresent()) {
                ParkingSlot slot = slotOpt.get();
                // assignVehicle is synchronized — prevents double-booking
                if (slot.assignVehicle(vehicle)) {
                    String ticketId = "TKT-" + ticketCounter.incrementAndGet();
                    Ticket ticket = new Ticket(ticketId, vehicle, slot);
                    activeTickets.put(ticketId, ticket);
                    System.out.println("Parked " + vehicle.getLicensePlate() +
                        " at Floor " + floor.getFloorNumber() + ", Slot " + slot.getSlotId());
                    return ticket;
                }
                // assignVehicle returned false → another thread grabbed it, try next
            }
        }
        throw new RuntimeException("Parking lot full for " + required);
    }

    // ─── UNPARK: process payment, free slot ───
    public double unpark(String ticketId) {
        Ticket ticket = activeTickets.remove(ticketId);
        if (ticket == null) throw new IllegalArgumentException("Invalid ticket: " + ticketId);

        LocalDateTime exitTime = LocalDateTime.now();
        double fee = pricingStrategy.calculateFee(ticket, exitTime);

        ticket.getSlot().removeVehicle();  // synchronized inside slot
        System.out.println("Unparked " + ticket.getVehicle().getLicensePlate() +
            " | Duration: " + ChronoUnit.MINUTES.between(ticket.getEntryTime(), exitTime) +
            " min | Fee: ₹" + fee);
        return fee;
    }

    public void displayStatus() {
        System.out.println("=== " + name + " Status ===");
        for (Floor floor : floors) {
            System.out.println("Floor " + floor.getFloorNumber() + " — " +
                "Bikes: " + floor.getAvailableCount(SlotType.BIKE) +
                " Cars: " + floor.getAvailableCount(SlotType.CAR) +
                " Trucks: " + floor.getAvailableCount(SlotType.TRUCK));
        }
    }
}
```

---

## Step 6 — Design Decisions

| Decision | Choice | Why |
|---|---|---|
| ParkingLot | Singleton | Only one lot, one source of truth |
| Slot assignment in `assignVehicle` | synchronized | Prevents two threads booking same slot |
| `activeTickets` | ConcurrentHashMap | Thread-safe map for concurrent park/unpark |
| `ticketCounter` | AtomicInteger | Lock-free atomic increment for ticket IDs |
| Pricing | Strategy pattern | Different pricing models swappable without changing ParkingLot |
| Vehicle types | Inheritance | Clear IS-A relationship, `getRequiredSlotType()` polymorphically correct |

---

## Step 7 — Concurrency Follow-up Questions

**Q: Two threads try to park a Car at the same time and only one slot is left. What happens?**
> Both threads call `floor.findAvailableSlot(CAR)` — both see the same available slot. Then both call `slot.assignVehicle(vehicle)`. Since `assignVehicle` is `synchronized`, only one thread's `synchronized` block runs at a time.
>
> Thread 1 enters, sees `AVAILABLE`, assigns, returns `true`. Thread 1 gets the ticket.
> Thread 2 enters (after Thread 1 exits), sees `OCCUPIED`, returns `false`. Thread 2 continues to the next floor. If no slots remain, exception is thrown.

**Q: Why is `assignVehicle` synchronized on the slot object and not on `ParkingLot`?**
> Synchronizing on `ParkingLot` (or `this`) would be too coarse — it would block ALL parking operations when only ONE slot is being contested. Synchronizing on each `ParkingSlot` instance allows concurrent parking in different slots. This is called **striped locking** (finer granularity = better throughput).

**Q: Is `floor.findAvailableSlot()` + `slot.assignVehicle()` a check-then-act issue?**
> Yes, exactly! Between `findAvailableSlot()` (check) and `assignVehicle()` (act), another thread can grab the slot. That's why `assignVehicle()` returns `boolean` — if another thread grabbed it, we retry (`if (slot.assignVehicle(vehicle))`). This is the optimistic concurrency approach.

**Q: `activeTickets` is a `ConcurrentHashMap`. Is `park()` still fully thread-safe?**
> `ConcurrentHashMap.put()` is thread-safe. However, the composite `park()` operation (find slot → assign → create ticket → put in map) is not an atomic unit. Two partial failures are possible:
> (1) Slot assigned but ticket not added to map (exception between) → slot is stuck as OCCUPIED. Fix: use try-finally to remove vehicle if ticket creation fails.
> (2) Acceptable for our design — each step is independently consistent.

**Q: How would you add a waitlist feature (park when slot available)?**
> Use `BlockingQueue<ParkingRequest>` per vehicle type. When lot is full, add request to queue. When a slot is freed in `unpark()`, poll the queue and assign the waiting vehicle. Use a background thread or call in `unpark()` directly.

**Q: Could you use `StampedLock` instead of `synchronized` for better read performance?**
> Yes. `StampedLock` supports optimistic reads — you read WITHOUT a lock, then validate that no write happened. If validation fails, fall back to a read lock. For `ParkingSlot.getStatus()` (called frequently to find available slots), optimistic reading would be faster than synchronization.
