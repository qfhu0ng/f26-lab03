# RoomReserve Critique

Fill in each section. One section per milestone. Keep it short and specific. Point at
files and methods, not adjectives.

---

## Milestone 1: The design as it is

Describe the system as the code actually builds it.

**Data model.** What is a booking, in the code? What types hold it, and what has to stay
in agreement for a booking to make sense?

Ans: A booking is stored across two maps in InMemoryStore. slotsByRoomDate maps "room|date" to a List<long[]>. Each long[] stores the start and end times in minutes. bookerBySlot maps "room|date|start|end" to the user's name.
The room, date, start time, and end time in both maps must stay in sync. Each interval in slotsByRoomDate needs a matching user entry in bookerBySlot.

**Operations.** What can a caller do, and what goes in and out?

Ans: `RequestHandler` provides four operations. All parameters and return values are strings:

- `createBooking` receives a room, date, time interval, and user. It adds the booking and returns a success message or an error message.
- `cancelBooking` receives a room, date, and exact time interval. It removes the matching booking and returns a success message or an error message.
- `rescheduleBooking` receives a room, date, and the old and new intervals. It attempts to move the matching booking while keeping its original user, and returns a success message or an error message.
- `listBookings` receives a room and date. It returns the stored time intervals and users, or a message saying that no bookings exist.

**Structure.** What classes exist, what does each own, and who holds a reference to whom?

Ans: The system has four classes:

- `ReservationApp` is the entry point. Its main() method creates a local RequestHandler reference and calls its public methods.
- `RequestHandler` owns an InMemoryStore reference. It parses input, checks some requests, calls the store, and formats responses.
- `InMemoryStore` owns slotsByRoomDate and bookerBySlot. It stores, removes, and looks up booking data.
- `BookingPolicy` owns the business-hours, maximum-length, and overlap rules. No class holds a reference to it, so its rules are not used.

Reference path: `ReservationApp` -> `RequestHandler` -> `InMemoryStore`

**The no-double-booking invariant.** Where is it enforced? Name every place a check
happens, say what each one actually checks, and trace one reschedule request through the
code from the entry point to storage.

Ans: The relevant checks are:

- `RequestHandler.createBooking()` rejects overlapping intervals for the same room and date using `start < existingEnd && existingStart < end`. Adjacent intervals are allowed.
- `BookingPolicy.validate()` calls `overlaps()` with the same overlap condition, but this validation is never called by the request path.
- `InMemoryStore.addSlot()` rejects only an interval with exactly the same start and end times for the same room and date. It does not reject partial overlaps.

For a reschedule, `ReservationApp` calls `RequestHandler.rescheduleBooking()`. It parses the times, returning an error if parsing fails, and checks that the new end is after the new start. It gets the original user with `InMemoryStore.bookerFor()` and returns an error if the result is null. Otherwise, it removes the old booking with `removeSlot()` and attempts to insert the new one with `addSlot()`. It ignores both methods' return values and returns success. No overlap check occurs during rescheduling, so the no-double-booking invariant is not enforced across all operations.

---

## Milestone 2: Two design problems

Two problems. For each one, fill in all three parts.

### Problem 1

**The problem.** Name it, using the vocabulary from lecture (milestone 2 in the
handout names the three).

Ans: Representational gap. A booking is represented by a `long[]`, two maps, and repeated string keys. These types do not represent a complete booking directly.


**Where in the code.** File and method.

Ans: `InMemoryStore.java`: the `slotsByRoomDate` and `bookerBySlot` fields, and the `addSlot()`, `removeSlot()`, `slotsFor()`, and `bookerFor()` methods.


**What it makes expensive.** A concrete future change, or something that already goes
wrong today. What breaks first?

Ans: The current model has no data structure for a recurring series or the bookings that belong to it. Supporting an operation that updates or cancels a whole series would first require adding this relationship, then keeping the time and user records for every booking in sync.


### Problem 2

**The problem.**

Ans: Misplaced responsibility. Booking rules are split across `RequestHandler`, `BookingPolicy`, and `InMemoryStore`. The request path does not use `BookingPolicy`.

**Where in the code.**

Ans: `RequestHandler.java`: `createBooking()` and `rescheduleBooking()`; `BookingPolicy.java`: `validate()`; `InMemoryStore.java`: `addSlot()`.


**What it makes expensive.**

Ans: Adding per-building business hours would require checking several request paths and classes. Changing `BookingPolicy` alone would not change the running behavior. The current split also allows rescheduling to create overlapping bookings.

---

## Milestone 3: Two alternative decompositions

Two different ways to carve up this system. A different split of responsibility, not a
list of local code fixes. Read the handout's appendix before writing this section.

### Alternative A

**The decomposition.** What are the pieces, what does each own, and where do the rules
live?

Ans: A `Booking` class owns the room, date, interval, and user. `RequestHandler` only parses inputs and formats responses. `BookingService` owns the four operations and coordinates `BookingPolicy` with a `BookingRepository`. `BookingPolicy` owns all booking rules. `BookingRepository` defines storage operations, and `InMemoryBookingRepository` implements them.


**One tradeoff.** Something this option actually costs. "No real downside" is not a
tradeoff.

Ans: This design adds several classes and interfaces. Each operation must pass through more layers, which increases setup and coordination for a small system.


### Alternative B

**The decomposition.**

Ans: A `Booking` class owns one complete booking. A `RoomSchedule` owns all bookings for one room and date. It performs create, cancel, reschedule, and list operations and enforces all booking rules. A `ScheduleStore` maps each room and date to its `RoomSchedule`. `RequestHandler` parses inputs, gets the correct schedule, and formats responses.


**One tradeoff.**

Ans: `RoomSchedule` combines operations, rules, and booking data. It may become large when recurring bookings, cross-midnight bookings, and per-building rules are added.


### Preference

Which one, and under what conditions? Say what the choice depends on, and what would
make you pick the other one instead.

I prefer Alternative A if the planned features continue to grow. Separate policy and repository boundaries allow booking rules and storage to change independently. I would choose Alternative B if RoomReserve stays a small in-memory service with few fixed rules, because it uses fewer components and keeps each room's changes together.
