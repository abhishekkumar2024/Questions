# Low-Level System Design (LLD) Guide

LLD = object-oriented design of a single system: classes, relationships, interfaces, patterns, and working code. Interviewers check whether you can model a problem cleanly and extend it without rewriting.

---

## Part A: The LLD Interview Framework (45 min)

| Step | Time | What to do |
|---|---|---|
| 1. Clarify requirements | 5 min | Ask: actors, core actions, constraints, scale of the single machine, concurrency |
| 2. Identify entities | 3 min | Nouns become classes; verbs become methods |
| 3. Define relationships | 3 min | Composition vs inheritance; ownership; cardinality |
| 4. Class diagram + APIs | 7 min | Public methods, key fields, enums |
| 5. Core code | 20 min | Write the main flow; keep it compiling in your head |
| 6. Extensibility + edge cases | 7 min | Show how a new rule plugs in; concurrency; invalid input |

**Clarifying questions to always ask:**
- What is in scope? (e.g., "just booking, or also payments?")
- Single-threaded or concurrent access?
- What must be extensible? (new vehicle types, new pricing rules)
- What should happen on invalid input? (exceptions vs return codes)
- Does it need persistence, or in-memory only?

**Say out loud:** "I'll start with the core entities, then the main flow, then handle extensibility and concurrency."

---

## Part B: Principles and Patterns

### B1. SOLID (one line each, with a quick example)
- **S: Single Responsibility.** A class has one reason to change. Example: separate `Parker` (who parks) from `FeeCalculator` (how much to charge).
- **O: Open/Closed.** Open for extension, closed for modification. Example: add a new discount by adding a class that implements `DiscountStrategy`, not by editing an `if` chain.
- **L: Liskov Substitution.** Subtypes must work wherever the base type works. Example: a `Square` that extends `Rectangle` and breaks `setWidth` violates this.
- **I: Interface Segregation.** Small, focused interfaces. Example: `Printable` and `Scannable` instead of one fat `Machine` interface.
- **D: Dependency Inversion.** Depend on abstractions. Example: `OrderService` takes a `PaymentGateway` interface, not `StripeClient`.

### B2. Composition over inheritance
Prefer "has-a" over "is-a" unless the is-a relationship is stable. Composition makes behavior swappable at runtime.

### B3. Design patterns you must know (with the signal that tells you to use them)

| Pattern | Use when | Example |
|---|---|---|
| **Strategy** | Multiple interchangeable algorithms | Pricing rules, rate-limit algorithms, parking fee |
| **State** | Object behavior changes with its state | Vending machine, elevator, order lifecycle |
| **Observer** | One event notifies many listeners | Notification service, stock price updates |
| **Factory / Factory Method** | Creation depends on type or config | Creating vehicles, notification channels |
| **Builder** | Many optional parameters | Building a complex request or a meal |
| **Singleton** | Exactly one instance (use sparingly) | Config, logger; prefer dependency injection |
| **Decorator** | Add behavior dynamically | Coffee with add-ons, logging/compression wrappers |
| **Command** | Encapsulate a request, enable undo | Text editor undo, job queue |
| **Chain of Responsibility** | Pass a request along handlers | Logger levels, middleware, approval chains |
| **Template Method** | Fixed algorithm with customizable steps | Report generation, game turn |
| **Iterator** | Traverse a collection without exposing its internals | Playlist, tree traversal |
| **Facade** | Simple interface over a complex subsystem | `OrderFacade` over inventory, payment, shipping |

### B4. Concurrency in LLD (a common follow-up)
- **Shared state:** protect with a lock (`threading.Lock` in Python, `synchronized` or `ReentrantLock` in Java).
- **Check-then-act race:** `if seat.is_free(): seat.book()` is a race. Make the check and the act atomic under one lock.
- **Lock granularity:** lock per resource (per seat, per account), not globally, unless simplicity matters more.
- **Deadlock:** acquire multiple locks in a consistent global order (e.g., by ID).
- **Thread-safe collections:** `ConcurrentHashMap` (Java), `queue.Queue` (Python).

### B5. Common modeling mistakes
- God class that does everything.
- Using strings where enums belong (`status = "ACTVE"`).
- Public mutable fields that let anyone change state.
- Business logic in the controller or in the data class.
- `if type == ...` chains that should be polymorphism or Strategy.

---

## Part C: LLD Problems

Each problem gives: requirements, entities, key patterns, core code, and extension points. Code is in Python for readability; translate to Java or Go if your interview requires it.

---

### Problem 1: Parking Lot

**Requirements:** multi-floor parking lot; vehicle types (bike, car, truck) with matching spot sizes; issue ticket on entry; calculate fee on exit (hourly, different rates by type); multiple entry/exit gates; count of free spots by type.

**Entities:** `ParkingLot`, `Floor`, `ParkingSpot` (with type), `Vehicle` (with type), `Ticket`, `FeeStrategy`, `Gate`.

**Patterns:** Strategy (fee calculation), Factory (spot allocation by vehicle size).

```python
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum
from math import ceil
import threading

class VehicleType(Enum):
    BIKE = 1
    CAR = 2
    TRUCK = 3

class SpotType(Enum):
    SMALL = 1
    MEDIUM = 2
    LARGE = 3

# Vehicle type -> compatible spot type
COMPATIBLE = {
    VehicleType.BIKE: [SpotType.SMALL, SpotType.MEDIUM, SpotType.LARGE],
    VehicleType.CAR: [SpotType.MEDIUM, SpotType.LARGE],
    VehicleType.TRUCK: [SpotType.LARGE],
}

@dataclass
class Vehicle:
    plate: str
    vtype: VehicleType

@dataclass
class ParkingSpot:
    spot_id: str
    spot_type: SpotType
    vehicle: "Vehicle | None" = None

    @property
    def free(self) -> bool:
        return self.vehicle is None

@dataclass
class Ticket:
    ticket_id: str
    vehicle: Vehicle
    spot: ParkingSpot
    entry_time: datetime

class FeeStrategy:
    RATE_PER_HOUR = {VehicleType.BIKE: 10, VehicleType.CAR: 20, VehicleType.TRUCK: 40}
    def fee(self, ticket: Ticket, exit_time: datetime) -> int:
        hours = max(1, ceil((exit_time - ticket.entry_time).total_seconds() / 3600))
        return hours * self.RATE_PER_HOUR[ticket.vehicle.vtype]

class ParkingLot:
    def __init__(self, spots: list[ParkingSpot], fee_strategy: FeeStrategy):
        self._spots = spots
        self._active: dict[str, Ticket] = {}   # ticket_id -> Ticket
        self._fee = fee_strategy
        self._lock = threading.Lock()
        self._counter = 0

    def park(self, vehicle: Vehicle, now: datetime) -> Ticket | None:
        with self._lock:                       # check-and-allocate must be atomic
            for spot in self._spots:
                if spot.free and spot.spot_type in COMPATIBLE[vehicle.vtype]:
                    spot.vehicle = vehicle
                    self._counter += 1
                    t = Ticket(f"T{self._counter}", vehicle, spot, now)
                    self._active[t.ticket_id] = t
                    return t
            return None                        # lot full for this type

    def unpark(self, ticket_id: str, now: datetime) -> int:
        with self._lock:
            t = self._active.pop(ticket_id)    # raises KeyError on invalid ticket
            t.spot.vehicle = None
            return self._fee.fee(t, now)

    def free_count(self, spot_type: SpotType) -> int:
        return sum(1 for s in self._spots if s.free and s.spot_type == spot_type)
```

**Extension points:**
- New pricing (weekend, EV charging): add a `FeeStrategy` subclass.
- Spot allocation policy (nearest to entrance): replace the loop with a `SpotAllocator` strategy.
- Multiple floors: group spots by floor, keep a per-type free-list for O(1) allocation.

---

### Problem 2: LRU Cache

**Requirements:** `get(key)` and `put(key, value)` in O(1); evict least recently used on capacity overflow.

**Key idea:** hash map for O(1) lookup + doubly linked list for O(1) reordering.

```python
class _Node:
    __slots__ = ("key", "val", "prev", "next")
    def __init__(self, key=None, val=None):
        self.key, self.val = key, val
        self.prev = self.next = None

class LRUCache:
    def __init__(self, capacity: int):
        self.cap = capacity
        self.map: dict[int, _Node] = {}
        # sentinels: head.next is most recent, tail.prev is least recent
        self.head, self.tail = _Node(), _Node()
        self.head.next, self.tail.prev = self.tail, self.head

    def _remove(self, node: _Node) -> None:
        node.prev.next, node.next.prev = node.next, node.prev

    def _add_front(self, node: _Node) -> None:
        node.next, node.prev = self.head.next, self.head
        self.head.next.prev = node
        self.head.next = node

    def get(self, key: int) -> int:
        if key not in self.map:
            return -1
        node = self.map[key]
        self._remove(node)
        self._add_front(node)
        return node.val

    def put(self, key: int, value: int) -> None:
        if key in self.map:
            node = self.map[key]
            node.val = value
            self._remove(node)
            self._add_front(node)
            return
        if len(self.map) == self.cap:
            lru = self.tail.prev              # evict least recent
            self._remove(lru)
            del self.map[lru.key]
        node = _Node(key, value)
        self.map[key] = node
        self._add_front(node)
```

**Follow-ups:** thread safety (wrap with lock), TTL (store expiry time in node), generic types, LFU variant (frequency buckets).

---

### Problem 3: Vending Machine (State Pattern)

**Requirements:** select product; insert coins; dispense; return change; cancel. Behavior depends on state (no coin, has coin, dispensing, out of stock).

**Pattern:** State. Each state class handles the same actions differently. No big `if state == ...` chain.

```python
from abc import ABC, abstractmethod

class VendingMachine:
    def __init__(self, inventory: dict[str, tuple[int, int]]):  # code -> (price, qty)
        self.inventory = inventory
        self.balance = 0
        self.selected: str | None = None
        self.state: "State" = IdleState()

    def set_state(self, s: "State") -> None:
        self.state = s

    def insert_coin(self, amount: int): self.state.insert_coin(self, amount)
    def select(self, code: str): self.state.select(self, code)
    def dispense(self): self.state.dispense(self)
    def cancel(self): self.state.cancel(self)

class State(ABC):
    @abstractmethod
    def insert_coin(self, m, amount): ...
    @abstractmethod
    def select(self, m, code): ...
    @abstractmethod
    def dispense(self, m): ...
    @abstractmethod
    def cancel(self, m): ...

class IdleState(State):
    def insert_coin(self, m, amount):
        m.balance += amount
        m.set_state(HasMoneyState())
    def select(self, m, code): print("Insert money first")
    def dispense(self, m): print("Select a product first")
    def cancel(self, m): print("Nothing to cancel")

class HasMoneyState(State):
    def insert_coin(self, m, amount): m.balance += amount
    def select(self, m, code):
        price, qty = m.inventory.get(code, (None, 0))
        if price is None or qty == 0:
            print("Unavailable"); return
        if m.balance < price:
            print("Insufficient balance"); return
        m.selected = code
        m.set_state(DispenseState())
    def dispense(self, m): print("Select a product first")
    def cancel(self, m):
        print(f"Refund {m.balance}"); m.balance = 0; m.set_state(IdleState())

class DispenseState(State):
    def insert_coin(self, m, amount): print("Please wait")
    def select(self, m, code): print("Already dispensing")
    def dispense(self, m):
        price, qty = m.inventory[m.selected]
        m.inventory[m.selected] = (price, qty - 1)
        m.balance -= price
        print(f"Dispensed {m.selected}, change {m.balance}")
        m.balance, m.selected = 0, None
        m.set_state(IdleState())
    def cancel(self, m): print("Cannot cancel while dispensing")
```

**Follow-ups:** out-of-stock state, coin validation, persistence of state, admin refill flow.

---

### Problem 4: Rate Limiter (LLD version with Strategy)

**Requirements:** pluggable algorithm per API; per-key limits; thread-safe.

```python
import time, threading
from abc import ABC, abstractmethod

class RateLimitStrategy(ABC):
    @abstractmethod
    def allow(self, key: str) -> bool: ...

class TokenBucket(RateLimitStrategy):
    def __init__(self, capacity: int, refill_per_sec: float):
        self.cap, self.rate = capacity, refill_per_sec
        self.state: dict[str, tuple[float, float]] = {}   # key -> (tokens, last_ts)
        self.lock = threading.Lock()

    def allow(self, key: str) -> bool:
        now = time.monotonic()
        with self.lock:
            tokens, last = self.state.get(key, (self.cap, now))
            tokens = min(self.cap, tokens + (now - last) * self.rate)
            if tokens >= 1:
                self.state[key] = (tokens - 1, now)
                return True
            self.state[key] = (tokens, now)
            return False

class FixedWindow(RateLimitStrategy):
    def __init__(self, limit: int, window_sec: int):
        self.limit, self.window = limit, window_sec
        self.counts: dict[tuple[str, int], int] = {}
        self.lock = threading.Lock()

    def allow(self, key: str) -> bool:
        bucket = (key, int(time.time() // self.window))
        with self.lock:
            self.counts[bucket] = self.counts.get(bucket, 0) + 1
            return self.counts[bucket] <= self.limit
```

**Follow-ups:** cleanup of stale keys (LRU or TTL), distributed version with Redis Lua, per-tier limits via a config map.

---

### Problem 5: Elevator System

**Requirements:** multiple elevators; floor requests from inside and outside; move toward requested floors efficiently; door open/close.

**Entities:** `Elevator` (direction, current floor, stop set), `Controller` (assigns requests), `Request`.
**Patterns:** State (IDLE, MOVING_UP, MOVING_DOWN, DOORS_OPEN), Strategy (scheduling: nearest elevator, SCAN/LOOK algorithm).

**Core ideas:**
- Each elevator keeps two sorted sets of stops: `up_stops` and `down_stops`.
- While moving up, serve stops above the current floor; when none remain, reverse (LOOK algorithm).
- Controller assigns an external request to the elevator with the lowest cost: distance plus direction mismatch penalty.
- Use a lock per elevator; the controller holds a lock only while assigning.

**Skeleton:**
```python
class Elevator:
    def __init__(self, eid: int):
        self.id, self.floor, self.direction = eid, 0, 0    # 0 idle, 1 up, -1 down
        self.stops: set[int] = set()

    def add_stop(self, floor: int) -> None:
        self.stops.add(floor)

    def cost(self, floor: int, direction: int) -> int:
        """Lower is better: distance plus penalty if moving the wrong way."""
        dist = abs(self.floor - floor)
        if self.direction == 0:
            return dist                       # idle: just distance
        ahead = (self.direction == 1 and floor >= self.floor) or \
                (self.direction == -1 and floor <= self.floor)
        if ahead and direction == self.direction:
            return dist                       # already heading that way
        return dist + 10                      # wrong way: penalty (tunable)

    def step(self) -> None:
        """Advance one floor toward the next stop (called each tick)."""
        if not self.stops:
            self.direction = 0
            return
        if self.direction == 0:
            target = min(self.stops, key=lambda f: abs(f - self.floor))
            self.direction = 1 if target > self.floor else -1 if target < self.floor else 0
        if self.direction == 1 and any(f > self.floor for f in self.stops):
            self.floor += 1
        elif self.direction == -1 and any(f < self.floor for f in self.stops):
            self.floor -= 1
        else:
            self.direction = -self.direction  # reverse when nothing ahead (LOOK)
        self.stops.discard(self.floor)        # open doors, serve stop
```

**Follow-ups:** weight/capacity limits, emergency mode, priority requests, what happens if a door is blocked.

---

### Problem 6: Logger (Chain of Responsibility)

**Requirements:** log levels (DEBUG, INFO, WARN, ERROR); multiple sinks (console, file, remote); each sink handles levels at or above its threshold; easy to add sinks.

```python
from abc import ABC, abstractmethod
from enum import IntEnum

class Level(IntEnum):
    DEBUG = 1
    INFO = 2
    WARN = 3
    ERROR = 4

class Sink(ABC):
    def __init__(self, min_level: Level, next_sink: "Sink | None" = None):
        self.min_level, self.next = min_level, next_sink

    def log(self, level: Level, msg: str) -> None:
        if level >= self.min_level:
            self.write(msg)
        if self.next:
            self.next.log(level, msg)

    @abstractmethod
    def write(self, msg: str) -> None: ...

class ConsoleSink(Sink):
    def write(self, msg: str) -> None: print("[console]", msg)

class FileSink(Sink):
    def write(self, msg: str) -> None:
        with open("app.log", "a") as f: f.write(msg + "\n")
```

**Note:** this is Chain of Responsibility where every handler may act and the chain continues. A "first handler wins" version is also common; say which you chose and why.

**Follow-ups:** async logging with a queue and background thread, formatters (Decorator), log rotation.

---

### Problem 7: Splitwise (Expense Sharing)

**Requirements:** add users and group expenses; split equally, by exact amount, or by percentage; show balances; simplify debts.

**Entities:** `User`, `Expense` (payer, amount, splits), `SplitStrategy` (Strategy), `BalanceSheet` (net balances per user pair).

**Key algorithm (debt simplification):**
1. Compute net balance per user: `paid - owed`.
2. Split into creditors (positive) and debtors (negative).
3. Greedily match largest debtor with largest creditor; transfer `min(|debt|, credit)`.
4. Repeat until all zero. This minimizes the number of transactions in practice (exact minimum is NP-hard in general).

```python
import heapq
from collections import defaultdict

def simplify(balances: dict[str, int]) -> list[tuple[str, str, int]]:
    """balances: user -> net (positive = is owed money). Amounts in paise (ints)."""
    creditors = [(-v, u) for u, v in balances.items() if v > 0]   # max-heap via negation
    debtors = [(v, u) for u, v in balances.items() if v < 0]      # v is negative
    heapq.heapify(creditors); heapq.heapify(debtors)
    txns = []
    while creditors and debtors:
        c_neg, c = heapq.heappop(creditors)
        d_val, d = heapq.heappop(debtors)
        c_amt, d_amt = -c_neg, -d_val
        pay = min(c_amt, d_amt)
        txns.append((d, c, pay))
        if c_amt > pay: heapq.heappush(creditors, (-(c_amt - pay), c))
        if d_amt > pay: heapq.heappush(debtors, (-(d_amt - pay), d))
    return txns
```

**Follow-ups:** use integers for money (avoid float rounding), rounding remainders for equal splits, validation (splits sum to total).

---

### Problem 8: Movie Ticket Booking (Concurrency Focus)

**Requirements:** shows, seats; user holds seats for a limited time; confirm or release; no double booking under concurrent requests.

**Key ideas:**
- Seat states: `AVAILABLE → HELD (with expiry) → BOOKED`. A background job releases expired holds.
- Atomic transition: `hold(seat, user)` must check and set state under a lock or with a DB conditional update (`UPDATE seats SET status='HELD' WHERE id=? AND status='AVAILABLE'`).
- Lock per show, or per seat for finer granularity. Acquire multiple seat locks in sorted order to avoid deadlock.

```python
import threading, time

class Seat:
    def __init__(self, sid: str):
        self.sid, self.status, self.holder, self.expires = sid, "AVAILABLE", None, 0.0

class Show:
    def __init__(self, seats: list[str], hold_sec: int = 300):
        self.seats = {s: Seat(s) for s in seats}
        self.lock = threading.Lock()
        self.hold_sec = hold_sec

    def _expire(self, now: float) -> None:
        for s in self.seats.values():
            if s.status == "HELD" and s.expires < now:
                s.status, s.holder = "AVAILABLE", None

    def hold(self, user: str, seat_ids: list[str]) -> bool:
        now = time.monotonic()
        with self.lock:                        # all-or-nothing for the group
            self._expire(now)
            if any(self.seats[s].status != "AVAILABLE" for s in seat_ids):
                return False
            for s in seat_ids:
                self.seats[s].status, self.seats[s].holder = "HELD", user
                self.seats[s].expires = now + self.hold_sec
            return True

    def confirm(self, user: str, seat_ids: list[str]) -> bool:
        now = time.monotonic()
        with self.lock:
            ok = all(self.seats[s].status == "HELD" and
                     self.seats[s].holder == user and
                     self.seats[s].expires >= now for s in seat_ids)
            if ok:
                for s in seat_ids:
                    self.seats[s].status = "BOOKED"
            return ok
```

**Follow-ups:** payment timeout, distributed version with Redis locks (with TTL and fencing tokens), DB row locking vs optimistic versioning.

---

### Problem 9: Observer-Based Notification Service

**Requirements:** publishers emit events (order shipped, price drop); subscribers choose channels (email, SMS, push); add channels without changing publishers.

```python
from abc import ABC, abstractmethod

class Subscriber(ABC):
    @abstractmethod
    def notify(self, event: dict) -> None: ...

class EventBus:
    def __init__(self):
        self._subs: dict[str, list[Subscriber]] = {}

    def subscribe(self, topic: str, sub: Subscriber) -> None:
        self._subs.setdefault(topic, []).append(sub)

    def publish(self, topic: str, event: dict) -> None:
        for sub in self._subs.get(topic, []):
            try:
                sub.notify(event)
            except Exception as e:      # one bad subscriber must not block the others
                print(f"subscriber failed: {e}")

class EmailSender(Subscriber):
    def notify(self, event: dict) -> None: print("email:", event)

class SmsSender(Subscriber):
    def notify(self, event: dict) -> None: print("sms:", event)

bus = EventBus()
bus.subscribe("order.shipped", EmailSender())
bus.subscribe("order.shipped", SmsSender())
bus.publish("order.shipped", {"order_id": 42})
```

**Follow-ups:** async delivery with a queue per subscriber, retries, unsubscribe, ordering guarantees.

---

### Problem 10: Tic-Tac-Toe (Extensible Board Game)

**Requirements:** NxN board; two players; win detection; draw; easy to add players or AI.

**Design:** `Board` (grid + win check), `Player` (symbol + strategy), `Game` (turn loop). AI is a `MoveStrategy` (Strategy pattern).

```python
class Board:
    def __init__(self, n: int = 3):
        self.n = n
        self.grid = [[None] * n for _ in range(n)]
        self.moves = 0

    def place(self, r: int, c: int, sym: str) -> bool:
        if not (0 <= r < self.n and 0 <= c < self.n) or self.grid[r][c] is not None:
            return False
        self.grid[r][c] = sym
        self.moves += 1
        return True

    def winner_at(self, r: int, c: int) -> bool:
        """Only check lines through the last move: O(n), not O(n^2)."""
        s = self.grid[r][c]
        n = self.n
        row = all(self.grid[r][j] == s for j in range(n))
        col = all(self.grid[i][c] == s for i in range(n))
        diag = r == c and all(self.grid[i][i] == s for i in range(n))
        anti = r + c == n - 1 and all(self.grid[i][n - 1 - i] == s for i in range(n))
        return row or col or diag or anti

    def is_full(self) -> bool:
        return self.moves == self.n * self.n
```

**Follow-ups:** undo (Command pattern), AI with minimax, saving game state, a Connect Four variant with a different win check.

---

## Part D: Quick Templates

**Class diagram shorthand you can write on a whiteboard:**
```
ParkingLot 1 ── * ParkingSpot
ParkingSpot  ── 0..1 Vehicle
ParkingLot ──> FeeStrategy (interface)
  ├─ HourlyFee
  └─ WeekendFee
```

**Final checklist before you say "done":**
- [ ] Each class has one responsibility.
- [ ] No `if type ==` chains for behavior that could be a strategy.
- [ ] Shared state is locked; check-and-act is atomic.
- [ ] Invalid inputs are handled (raise or return an error, and say which).
- [ ] You showed where a new feature plugs in without editing existing code.

---

## Part E: Practice Order
1. Parking lot (warm-up for entities and Strategy).
2. LRU cache (data structures + coding).
3. Vending machine (State).
4. Rate limiter (Strategy + thread safety).
5. Logger (Chain of Responsibility).
6. Notification (Observer).
7. Splitwise (algorithms in design).
8. Elevator (State + scheduling).
9. Movie booking (concurrency).
10. Tic-tac-toe (clean extensible code, under time pressure).

**Redo the code from scratch after 3 days, without looking.** That's where the learning sticks.
