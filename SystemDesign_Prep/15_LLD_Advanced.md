# 15. Advanced LLD: More Problems and Deeper Techniques

Read after file 02. Same method: requirements → entities → patterns → code → extensions → concurrency.

---

## Problem A: Chess

**Requirements:** two players, 8x8 board, all standard piece moves, check, checkmate, stalemate, castling, en passant, promotion, undo.

**Entities:** `Board`, `Piece` (abstract; `Rook, Bishop, Queen, King, Knight, Pawn` subclasses), `Move`, `Player`, `Game`, `GameState`.

**Patterns:** Polymorphism for piece moves (each piece implements `candidate_moves`); Command for moves (enables undo); State for game status.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass(frozen=True)
class Move:
    src: tuple[int, int]
    dst: tuple[int, int]
    promotion: str | None = None

class Piece(ABC):
    def __init__(self, color: str):
        self.color = color
        self.moved = False

    @abstractmethod
    def candidate_moves(self, board, pos):   # pseudo-legal, ignores check
        ...

class Rook(Piece):
    def candidate_moves(self, board, pos):
        return board.slide(pos, [(1,0),(-1,0),(0,1),(0,-1)], self.color)

class Board:
    def __init__(self):
        self.grid: list[list[Piece | None]] = [[None]*8 for _ in range(8)]
        self.history: list[tuple[Move, object]] = []   # for undo

    def slide(self, pos, directions, color):
        """Walk each direction until edge or blocked; include capture square."""
        out = []
        r0, c0 = pos
        for dr, dc in directions:
            r, c = r0 + dr, c0 + dc
            while 0 <= r < 8 and 0 <= c < 8:
                target = self.grid[r][c]
                if target is None:
                    out.append(((r0,c0),(r,c)))
                else:
                    if target.color != color:
                        out.append(((r0,c0),(r,c)))
                    break
                r, c = r + dr, c + dc
        return out

    def in_check(self, color) -> bool:
        """King is attacked by any opposing pseudo-legal move."""
        king = next(((r,c) for r in range(8) for c in range(8)
                     if isinstance(self.grid[r][c], King) and self.grid[r][c].color == color), None)
        for r in range(8):
            for c in range(8):
                p = self.grid[r][c]
                if p and p.color != color:
                    if any(dst == king for _, dst in p.candidate_moves(self, (r,c))):
                        return True
        return False

    def make(self, move: Move):
        (r0,c0),(r1,c1) = move.src, move.dst
        captured = self.grid[r1][c1]
        self.history.append((move, captured))
        self.grid[r1][c1], self.grid[r0][c0] = self.grid[r0][c0], None

    def undo(self):
        move, captured = self.history.pop()
        (r0,c0),(r1,c1) = move.src, move.dst
        self.grid[r0][c0], self.grid[r1][c1] = self.grid[r1][c1], captured

class Game:
    def legal_moves(self, board: Board, color: str):
        """Filter pseudo-legal moves: a move is legal only if it doesn't leave own king in check."""
        legal = []
        for r in range(8):
            for c in range(8):
                p = board.grid[r][c]
                if p and p.color == color:
                    for src, dst in p.candidate_moves(board, (r,c)):
                        board.make(Move(src, dst))
                        ok = not board.in_check(color)
                        board.undo()
                        if ok:
                            legal.append(Move(src, dst))
        return legal
    # Checkmate = in_check and no legal moves; stalemate = not in_check and no legal moves.
```
**Key points to say:** pseudo-legal vs legal generation (check filter by make/undo); castling and en passant need extra state (`moved` flags, last double-step pawn); repetition and 50-move rules as extensions; performance is not the focus in LLD.

---

## Problem B: Library Management

**Requirements:** books with multiple copies; members borrow up to N books for D days; reservations queue when no copy; fines for late returns; search by title/author/ISBN.

**Entities:** `Book` (metadata), `BookCopy` (barcode, status), `Member`, `Loan`, `Reservation`, `Fine`, `Catalog` (search index).

**Patterns:** Strategy for fine calculation (flat, per-day, tiered); Observer for notifying the next reservation when a copy returns; Factory for member types with different limits.

**Core flow:**
1. `borrow(member, copy)`: check member limit and outstanding fines; set copy `ON_LOAN`; create `Loan(due=now+D)`.
2. `return_copy(copy)`: close loan; compute fine via strategy; if reservation queue non-empty, mark copy `RESERVED_FOR(next)` and notify.
3. `reserve(member, book)`: enqueue; if a copy is available, reserve immediately.

**Concurrency:** two members borrowing the last copy: `borrow` does a check-and-set on copy status under a lock (or conditional update). Reservation queue is a FIFO per book, guarded the same way.

**Extension points:** renewals (limit count), different loan periods per member type, digital books (separate `Asset` hierarchy with license limits).

---

## Problem C: Food Delivery (Swiggy/Zomato-style)

**Requirements:** restaurants have menus; customers place orders; restaurant accepts; delivery partner assigned; order status tracked; payment; ratings.

**Entities:** `Restaurant`, `MenuItem`, `Cart`, `Order` (state machine), `DeliveryPartner`, `Assignment`, `Payment`.

**Patterns:** State (`PLACED → ACCEPTED → PREPARING → READY → PICKED_UP → DELIVERED`, plus `CANCELLED`); Strategy for partner assignment (nearest, least-loaded, batch); Observer for status updates to customer/restaurant/partner; Factory or Strategy for discount rules.

**Key logic:**
- Price calculation: sum items → apply discount strategies in defined order → taxes → fees. Keep the calculation pure (no DB) so it can be tested.
- Assignment: candidate partners within radius; a partner receives one offer at a time with timeout; on reject, move to next. Same conditional-assignment trick as Case 8 in the HLD walkthroughs.
- Cancellation rules depend on state (State pattern: cancellable before `PREPARING` only).

**Concurrency:** order state transitions are guarded by a version or lock; partner offers are exclusive.

---

## Problem D: Meeting Room Scheduler

**Requirements:** rooms with capacity and features (projector); book a time slot; no overlaps; search rooms for a time window and capacity; recurring meetings (optional).

**Entities:** `Room`, `Booking(room, start, end, owner)`, `Scheduler`.

**Core structure:** per room, keep bookings sorted by start time (interval list). Overlap check: `new.start < existing.end and existing.start < new.end`. Binary search for insertion point → O(log n) check plus neighbor checks.

```python
import bisect
from dataclasses import dataclass

@dataclass(frozen=True, order=True)
class Booking:
    start: int
    end: int
    owner: str = ""

class Room:
    def __init__(self, room_id: str, capacity: int):
        self.id, self.capacity = room_id, capacity
        self._bookings: list[Booking] = []           # sorted by start

    def is_free(self, start: int, end: int) -> bool:
        i = bisect.bisect_left(self._bookings, Booking(start, start))
        if i < len(self._bookings) and self._bookings[i].start < end:
            return False
        if i > 0 and self._bookings[i-1].end > start:
            return False
        return True

    def book(self, start: int, end: int, owner: str) -> bool:
        if start >= end or not self.is_free(start, end):
            return False
        bisect.insort(self._bookings, Booking(start, end, owner))
        return True
```
**Extensions:** recurring bookings (generate instances, check all), time zones (store UTC), locking per room for concurrency, search via index on capacity and features.

---

## Problem E: Snake and Ladder

**Requirements:** N players, board size configurable, dice (configurable count), snakes and ladders, winner determination.

**Entities:** `Board` (jump map: start → end), `Dice` (Strategy: fair, loaded, multiple dice), `Player`, `Game`.

**Key logic:** `new_pos = pos + roll`; if `new_pos` in jump map, apply jump; if `> N`, stay. Turn order with a queue. Winner: first to reach exactly N (or ≥ N per rule — state the rule).

**Extensions:** Dice strategy; undo; game-over state; persistence of game for replay.

---

## Problem F: Multi-Level Cache (LLD + Concurrency)

**Requirements:** L1 in-process LRU, L2 shared (interface), loader on miss; write-through or write-behind policy; thread-safe; TTL per entry.

**Patterns:** Strategy (write policy), Decorator (adds TTL or metrics around a cache), Template Method (get flow: check L1 → L2 → loader → populate).

**Concurrency point:** single-flight loading — concurrent misses for the same key wait for one loader call instead of N loader calls. Implementation: per-key `Future` stored in a map under lock; first caller creates it and runs the loader; others wait on it.

```python
import threading
from concurrent.futures import Future

class SingleFlight:
    def __init__(self):
        self._lock = threading.Lock()
        self._inflight: dict[str, Future] = {}

    def do(self, key, loader):
        with self._lock:
            fut = self._inflight.get(key)
            leader = fut is None
            if leader:
                fut = Future()
                self._inflight[key] = fut
        if leader:
            try:
                fut.set_result(loader())
            except Exception as e:
                fut.set_exception(e)
            finally:
                with self._lock:
                    self._inflight.pop(key, None)
        return fut.result()
```

---

## Advanced LLD Techniques (concepts to name)

- **Immutability:** prefer immutable value objects (`frozen dataclass`, `record`) for Money, Money-like amounts, Move, Booking. Reduces concurrency bugs.
- **Invariants:** enforce in constructors and methods, not in callers. Example: `Money` cannot be negative unless allowed; `Booking.start < end`.
- **Dependency injection:** pass clocks, repositories, and strategies into constructors. Makes tests deterministic (inject a fake clock).
- **Testing strategy:** unit tests per strategy; state machine transition table tests; concurrency test with many threads asserting invariants (no double booking).
- **Error handling:** domain exceptions (`SeatUnavailable`, `InvalidTransition`) instead of boolean returns when the caller must react; document which you chose.
- **Extensibility checklist:** new type of X → add a class, not an `if`; new rule → new strategy; new state → new state class and transition table entry.
- **Java mapping:** interfaces for strategies, `enum` with abstract methods for state transitions, `ConcurrentHashMap.computeIfAbsent` for single-flight-like maps, `ReentrantLock` with `tryLock` for timeouts, `record` for immutable values.

---

## Practice Set (after file 02 problems)
1. Chess (pieces and legal move filter).
2. Library management (state + concurrency).
3. Food delivery (state machine + assignment).
4. Meeting room scheduler (intervals + bisect).
5. Snake and ladder (simple, fast warm-up).
6. Multi-level cache with single-flight.
Write each from a blank file within a timer (45 min), then compare.
