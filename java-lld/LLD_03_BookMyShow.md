# LLD: Movie Booking System (BookMyShow)

> Frequency: High | Difficulty: High
> Tests: Concurrency (seat booking), complex entity modeling, locking strategies

---

## Step 1 — Clarifying Questions

- Scope: Movie booking only, or also events/sports?
- Single city or multiple? Single screen per cinema?
- Seat selection: user chooses specific seats or system assigns?
- Payment: integrate payment gateway? Timeout on reserved seats?
- Cancellations and refunds?
- Concurrent bookings for same show?

---

## Step 2 — Core Requirements

1. List movies, shows, available seats
2. User selects show + seats
3. Seats are temporarily locked during payment (hold window)
4. On payment success → confirm booking, generate ticket
5. On payment failure/timeout → release seats
6. Handle concurrent bookings (two users trying same seat)

---

## Step 3 — Class Diagram

```
Movie
├── movieId, title, genre, duration

Cinema
├── name, city
└── screens: List<Screen>       [Composition]

Screen
├── screenId, name, totalSeats
└── shows: List<Show>           [Composition]

Show
├── showId
├── movie: Movie                [Aggregation]
├── startTime, endTime
└── seats: Map<String, Seat>    [Composition]

Seat
├── seatId (A1, B2...)
├── row, column
├── category: SeatCategory (NORMAL, PREMIUM, RECLINER)
├── status: SeatStatus (AVAILABLE, LOCKED, BOOKED)
└── lockedBy: String (userId who locked it)

Booking
├── bookingId
├── user: User
├── show: Show
├── seats: List<Seat>
├── status: BookingStatus (PENDING, CONFIRMED, CANCELLED)
└── totalPrice: double

BookingService (Core Orchestrator)
PricingService (Strategy)
```

---

## Step 4 — Full Java Code

```java
// ─── ENUMS ───
public enum SeatCategory { NORMAL, PREMIUM, RECLINER }
public enum SeatStatus { AVAILABLE, LOCKED, BOOKED }
public enum BookingStatus { PENDING, CONFIRMED, CANCELLED, EXPIRED }

// ─── SEAT — most critical class for concurrency ───
public class Seat {
    private final String seatId;
    private final int row;
    private final int column;
    private final SeatCategory category;

    private volatile SeatStatus status;
    private String lockedByUserId;
    private LocalDateTime lockExpiry;

    private static final int LOCK_DURATION_MINUTES = 10;

    public Seat(String seatId, int row, int column, SeatCategory category) {
        this.seatId = seatId;
        this.row = row;
        this.column = column;
        this.category = category;
        this.status = SeatStatus.AVAILABLE;
    }

    public String getSeatId() { return seatId; }
    public SeatCategory getCategory() { return category; }
    public SeatStatus getStatus() { return status; }

    // Synchronized: prevents two users locking same seat
    public synchronized boolean lock(String userId) {
        // Auto-expire stale locks
        if (status == SeatStatus.LOCKED && LocalDateTime.now().isAfter(lockExpiry)) {
            status = SeatStatus.AVAILABLE;
            lockedByUserId = null;
        }

        if (status != SeatStatus.AVAILABLE) return false;

        this.status = SeatStatus.LOCKED;
        this.lockedByUserId = userId;
        this.lockExpiry = LocalDateTime.now().plusMinutes(LOCK_DURATION_MINUTES);
        return true;
    }

    public synchronized boolean confirm(String userId) {
        if (status != SeatStatus.LOCKED || !userId.equals(lockedByUserId)) return false;
        if (LocalDateTime.now().isAfter(lockExpiry)) {
            status = SeatStatus.AVAILABLE;
            return false;
        }
        this.status = SeatStatus.BOOKED;
        return true;
    }

    public synchronized void release(String userId) {
        if (status == SeatStatus.LOCKED && userId.equals(lockedByUserId)) {
            this.status = SeatStatus.AVAILABLE;
            this.lockedByUserId = null;
        }
    }

    public boolean isAvailable() {
        if (status == SeatStatus.LOCKED && LocalDateTime.now().isAfter(lockExpiry)) {
            return true;  // optimistic — full check happens in lock()
        }
        return status == SeatStatus.AVAILABLE;
    }
}

// ─── MOVIE ───
public class Movie {
    private final String movieId;
    private final String title;
    private final String genre;
    private final int durationMinutes;

    public Movie(String movieId, String title, String genre, int durationMinutes) {
        this.movieId = movieId;
        this.title = title;
        this.genre = genre;
        this.durationMinutes = durationMinutes;
    }

    public String getMovieId() { return movieId; }
    public String getTitle() { return title; }
    public String getGenre() { return genre; }
}

// ─── SHOW ───
public class Show {
    private final String showId;
    private final Movie movie;
    private final LocalDateTime startTime;
    private final Map<String, Seat> seats;  // seatId → Seat

    public Show(String showId, Movie movie, LocalDateTime startTime, List<Seat> seatList) {
        this.showId = showId;
        this.movie = movie;
        this.startTime = startTime;
        this.seats = new ConcurrentHashMap<>();
        seatList.forEach(s -> seats.put(s.getSeatId(), s));
    }

    public String getShowId() { return showId; }
    public Movie getMovie() { return movie; }
    public LocalDateTime getStartTime() { return startTime; }

    public List<Seat> getAvailableSeats() {
        return seats.values().stream()
            .filter(Seat::isAvailable)
            .collect(Collectors.toList());
    }

    public Seat getSeat(String seatId) { return seats.get(seatId); }

    // Atomic: lock ALL requested seats or none (all-or-nothing)
    public List<Seat> lockSeats(String userId, List<String> seatIds) {
        List<Seat> lockedSeats = new ArrayList<>();
        List<Seat> failedSeats = new ArrayList<>();

        for (String seatId : seatIds) {
            Seat seat = seats.get(seatId);
            if (seat == null) {
                failedSeats.add(null);
                break;
            }
            if (seat.lock(userId)) {
                lockedSeats.add(seat);
            } else {
                failedSeats.add(seat);
                break;
            }
        }

        if (!failedSeats.isEmpty()) {
            // Rollback — release all seats locked so far
            lockedSeats.forEach(s -> s.release(userId));
            throw new SeatUnavailableException("Seat(s) unavailable: " + failedSeats);
        }

        return lockedSeats;
    }

    public void confirmSeats(String userId, List<String> seatIds) {
        for (String seatId : seatIds) {
            Seat seat = seats.get(seatId);
            if (!seat.confirm(userId)) {
                throw new BookingException("Failed to confirm seat: " + seatId);
            }
        }
    }

    public void releaseSeats(String userId, List<String> seatIds) {
        for (String seatId : seatIds) {
            Seat seat = seats.get(seatId);
            if (seat != null) seat.release(userId);
        }
    }
}

// ─── PRICING SERVICE ───
public interface PricingService {
    double calculatePrice(Seat seat);
}

public class DefaultPricingService implements PricingService {
    private final Map<SeatCategory, Double> basePrices;

    public DefaultPricingService() {
        basePrices = new EnumMap<>(SeatCategory.class);
        basePrices.put(SeatCategory.NORMAL, 200.0);
        basePrices.put(SeatCategory.PREMIUM, 350.0);
        basePrices.put(SeatCategory.RECLINER, 500.0);
    }

    @Override
    public double calculatePrice(Seat seat) {
        return basePrices.getOrDefault(seat.getCategory(), 200.0);
    }
}

// ─── BOOKING ───
public class Booking {
    private final String bookingId;
    private final String userId;
    private final Show show;
    private final List<Seat> seats;
    private BookingStatus status;
    private final double totalPrice;
    private final LocalDateTime createdAt;

    public Booking(String bookingId, String userId, Show show,
                   List<Seat> seats, double totalPrice) {
        this.bookingId = bookingId;
        this.userId = userId;
        this.show = show;
        this.seats = Collections.unmodifiableList(new ArrayList<>(seats));
        this.totalPrice = totalPrice;
        this.status = BookingStatus.PENDING;
        this.createdAt = LocalDateTime.now();
    }

    public String getBookingId() { return bookingId; }
    public String getUserId() { return userId; }
    public BookingStatus getStatus() { return status; }
    public double getTotalPrice() { return totalPrice; }
    public List<Seat> getSeats() { return seats; }
    public Show getShow() { return show; }

    public synchronized void confirm() { this.status = BookingStatus.CONFIRMED; }
    public synchronized void cancel() { this.status = BookingStatus.CANCELLED; }
    public synchronized void expire() { this.status = BookingStatus.EXPIRED; }
}

// ─── BOOKING SERVICE ───
public class BookingService {
    private final PricingService pricingService;
    private final Map<String, Booking> bookings = new ConcurrentHashMap<>();
    private final AtomicInteger bookingCounter = new AtomicInteger(0);

    public BookingService(PricingService pricingService) {
        this.pricingService = pricingService;
    }

    // Step 1: Lock seats and create pending booking
    public Booking initiateBooking(String userId, Show show, List<String> seatIds) {
        // All-or-nothing seat lock
        List<Seat> lockedSeats = show.lockSeats(userId, seatIds);

        double totalPrice = lockedSeats.stream()
            .mapToDouble(pricingService::calculatePrice)
            .sum();

        String bookingId = "BKG-" + bookingCounter.incrementAndGet();
        Booking booking = new Booking(bookingId, userId, show, lockedSeats, totalPrice);
        bookings.put(bookingId, booking);

        System.out.println("Booking initiated: " + bookingId + " | ₹" + totalPrice + " | 10 min to pay");
        return booking;
    }

    // Step 2: Payment success → confirm
    public void confirmBooking(String bookingId, String paymentId) {
        Booking booking = getBookingOrThrow(bookingId);
        List<String> seatIds = booking.getSeats().stream()
            .map(Seat::getSeatId).collect(Collectors.toList());

        booking.getShow().confirmSeats(booking.getUserId(), seatIds);
        booking.confirm();
        System.out.println("Booking confirmed: " + bookingId + " | Payment: " + paymentId);
    }

    // Step 3: Payment failure or timeout → release
    public void cancelBooking(String bookingId) {
        Booking booking = getBookingOrThrow(bookingId);
        List<String> seatIds = booking.getSeats().stream()
            .map(Seat::getSeatId).collect(Collectors.toList());

        booking.getShow().releaseSeats(booking.getUserId(), seatIds);
        booking.cancel();
        System.out.println("Booking cancelled: " + bookingId);
    }

    private Booking getBookingOrThrow(String bookingId) {
        Booking booking = bookings.get(bookingId);
        if (booking == null) throw new IllegalArgumentException("Booking not found: " + bookingId);
        return booking;
    }
}
```

---

## Step 5 — Design Decisions

| Decision | Choice | Reason |
|---|---|---|
| Seat locking | `synchronized` per Seat | Fine-grained — concurrent users don't block each other unless contesting same seat |
| All-or-nothing locking | Lock in order, rollback on failure | Avoids partial booking (3 of 4 seats booked) |
| `ConcurrentHashMap` for seats | Instead of synchronized HashMap | Better read throughput for `getAvailableSeats()` |
| Lock timeout | 10-minute expiry in seat | Releases locked seats if payment takes too long |
| Pending booking state | `PENDING` before payment | Enables payment gateway integration; seats reserved during payment |

---

## Step 6 — Concurrency Follow-up Questions

**Q: Two users select seats A1 and A2 at the same time. User1 wants A1+A2, User2 wants A2+A3. Can they deadlock?**
> Classic deadlock scenario!
> - User1 locks A1, then tries to lock A2
> - User2 locks A2, then tries to lock A3
> - No deadlock here because they're not waiting on each other (different final seats).
>
> But consider: User1 wants {A1, A2}, User2 wants {A2, A1}:
> - User1 locks A1, waits for A2
> - User2 locks A2, waits for A1 → DEADLOCK
>
> **Fix:** Always lock seats in a consistent order (e.g., alphabetically by seatId). `lockSeats()` should sort `seatIds` before locking. This breaks the circular dependency.
>
> ```java
> public List<Seat> lockSeats(String userId, List<String> seatIds) {
>     List<String> sortedIds = new ArrayList<>(seatIds);
>     Collections.sort(sortedIds);  // consistent order prevents deadlock
>     // ... rest of locking logic
> }
> ```

**Q: What if User1's payment takes 15 minutes and the lock was only for 10 minutes?**
> The seat's `lock()` checks lock expiry and auto-releases stale locks. If payment succeeds after expiry, `confirmSeats()` calls `seat.confirm()` which checks `LocalDateTime.now().isAfter(lockExpiry)` and returns `false`. The booking service would get a `BookingException` and should refund the payment.
>
> To prevent this: Start a background timer when a booking is initiated. At 10 minutes, automatically cancel the booking and release seats.

**Q: `show.lockSeats()` locks seats one by one. What's the issue and how do you fix it?**
> Between locking A1 and A2, a concurrent user could lock A2. User1 then fails, rolls back A1, and the user needs to retry. This is optimistic concurrency with retry.
>
> Better approach for high-contention shows: Use a `synchronized` block over the entire `lockSeats` operation, or use a show-level `ReentrantLock`. This reduces throughput but eliminates the retry loop.
>
> For most shows (not Coldplay concert), optimistic per-seat locking is better because lock contention is rare.

**Q: How do you handle the "9:55 PM show starts in 5 minutes, no more bookings" rule?**
> Add a `cutoffTime` to Show (e.g., 30 minutes before start). In `BookingService.initiateBooking()`, check:
> ```java
> if (LocalDateTime.now().isAfter(show.getStartTime().minusMinutes(30))) {
>     throw new BookingCutoffException("Booking window closed for this show");
> }
> ```

**Q: How would you scale this to handle millions of concurrent bookings (e.g., IPL final)?**
> Single machine: Fine-grained locks (per-seat) is already good.
> Distributed: Use **distributed locks** (Redis SETNX with expiry) instead of Java `synchronized`. Each seat's lock is a Redis key `seat:{seatId}:lock` with 10-minute TTL. Only one node can acquire it. Use Lua scripts for atomic check-and-set.
