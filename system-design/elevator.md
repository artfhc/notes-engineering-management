# Elevator System Design

## Goal

Design a system to control N elevators in a building with M floors. Users can call elevators from floors (hallway buttons) or select floors from inside elevators (car buttons).

---

## 1. Requirements

### Functional Requirements

- Multiple elevators
- Request elevator from a floor (up/down buttons)
- Select floor from inside the elevator
- Display elevator status: current floor, direction
- Handle concurrent requests
- Support maintenance mode or offline elevators (if time allows)

### Non-functional Requirements

- Low latency (fast response)
- High availability
- Scalable for high-rise buildings

---

## 2. Core Components

### a) Elevator

Each elevator has:

- `id`
- `currentFloor`
- `direction` (UP, DOWN, IDLE)
- `destinationQueue`: sorted list of floors to visit
- `status`: MOVING, IDLE, MAINTENANCE

### b) Request

- **External Request:** from a floor → `floorNumber`, `direction`
- **Internal Request:** from inside elevator → `floorNumber`

### c) Scheduler (Controller)

- Handles all external requests
- Assigns elevators to requests using an algorithm (e.g., nearest, idle, direction matching)

---

## 3. Classes (OOP Style)

```java
class Elevator {
    int id;
    int currentFloor;
    Direction direction;
    PriorityQueue<Integer> destinationQueue;
    Status status;

    void move();
    void addNewDestination(int floor);
    void step(); // one move
}

enum Direction { UP, DOWN, IDLE }
enum Status { MOVING, IDLE, MAINTENANCE }

class Request {
    int floor;
    Direction direction; // only for external
}

class ElevatorController {
    List<Elevator> elevators;

    void handleExternalRequest(Request request);
    void handleInternalRequest(int elevatorId, int floor);
}
```

---

## 4. Scheduling Strategy

### Simple Logic:

1. If any elevator is IDLE → assign it
2. Else, pick elevator that is:
   - Moving towards the requested floor
   - Has room in its queue

### Advanced:

- Add load balancing, estimated time to arrive, or elevator group zones

---

## 5. Concurrency Considerations

- Requests may come concurrently
- Use thread-safe queues
- Elevator state machine can be modeled with timers or threads

---

## 6. Scalability and Extensions

- Handle 100+ elevators: distribute control logic
- Predictive algorithm: pre-position idle elevators
- Maintenance mode, overload handling
- Logging and monitoring

---

## 7. Trade-offs & Challenges

- Real-time scheduling vs batch
- Centralized vs distributed control
- Handling edge cases: multiple requests at same time, emergency stop, etc.

---

## Deep Dive: Why Use PriorityQueue for destinationQueue?

### Purpose of destinationQueue

It stores the list of floors the elevator is scheduled to visit.

### Why a PriorityQueue?

**1. Optimal Floor Traversal**

Elevators generally move in one direction until all requests in that direction are served — this minimizes direction changes and improves efficiency. A PriorityQueue helps enforce that by:

- When moving up, store destinations in min-heap order (smallest floor first)
- When moving down, use max-heap (largest floor first)

So the elevator always goes to the next closest floor in its direction.

**2. What if we use a regular list?**

- You'd have to sort it manually or iterate the list each time to find the next optimal stop
- Inefficient and error-prone, especially if requests change frequently

**3. Dynamic Behavior**

Elevator requests can come in at any time. A PriorityQueue:

- Automatically reorders new floor requests
- Makes it easy to always fetch the next optimal floor with O(log n) insertion and O(1) peek

### Implementation Detail

You might keep two queues:

```java
PriorityQueue<Integer> upQueue = new PriorityQueue<>();
PriorityQueue<Integer> downQueue = new PriorityQueue<>(Collections.reverseOrder());
```

Switch between them based on direction.

### Summary

| Approach | With PriorityQueue | Without PriorityQueue |
|----------|-------------------|----------------------|
| **Traversal** | Efficient traversal order | Manual sorting or linear scan |
| **Dynamic Input** | Easier to manage dynamic input | Harder to maintain optimality |
| **Scalability** | Scales well | Error-prone in real-time logic |

---

## Alternative: Two Ordered Sets (or Sorted Lists)

Use two TreeSets (or SortedLists) to separate upward and downward requests:

### Structure:

```java
TreeSet<Integer> upStops = new TreeSet<>();
TreeSet<Integer> downStops = new TreeSet<>(Collections.reverseOrder());
Direction currentDirection;
```

### How it works:

**1. When elevator is going UP:**

- Use `upStops.first()` to get the next closest floor going up
- Keep serving upStops until empty
- Then switch to downStops

**2. When elevator is going DOWN:**

- Use `downStops.first()` (which gives the highest value first due to reverse order)
- Keep serving downStops until empty
- Then switch back to upStops

### Advantages

| Benefit | Description |
|---------|-------------|
| **Deterministic Order** | Floors are visited in the order that minimizes direction changes |
| **Fast Insert/Delete** | TreeSet provides log-time operations |
| **Direction-aware** | Separate queues help you reason clearly about movement logic |

### Why it's a good alternative

- `PriorityQueue` doesn't allow easy removal of arbitrary elements (e.g., cancel a request)
- `TreeSet` maintains unique values and makes it easy to iterate in order
- Clean direction handling: Up and down queues are isolated

---

## Concurrency Handling

Concurrency handling is critical in elevator system design because multiple events can happen simultaneously: button presses, elevator movements, state updates, etc.

### Key Concurrency Challenges

| Scenario | Example |
|----------|---------|
| Multiple users pressing floor buttons at the same time | Several people call elevators from different floors |
| Multiple elevators changing state | One is moving up, another becomes idle |
| Requests added while elevator is en route | Someone presses a button inside while it's already moving |
| Updating UI while state is changing | UI displays direction, floor number, door status |

### Best Practices for Concurrency Handling

**1. Use Locks or Synchronization**

Protect shared resources like `destinationQueue`, elevator state, etc.

Java: `synchronized`, `ReentrantLock`

```java
synchronized (elevator) {
    elevator.addNewDestination(5);
}
```

**2. Immutable Request Objects**

Make Request objects immutable to avoid corruption due to shared references.

**3. Thread per Elevator (or Scheduled Executor)**

Each elevator can be run as a separate thread or scheduled task to simulate movement:

```java
Runnable elevatorWorker = () -> {
    while (true) {
        elevator.step();
        sleep(1 second);
    }
};
```

**4. Thread-Safe Collections**

Use concurrent data structures like:

- `ConcurrentLinkedQueue` for incoming requests
- `ConcurrentHashMap` for mapping elevator states

**5. Request Dispatcher (Central Controller)**

The controller can run on a dedicated thread to:

- Collect external requests
- Assign them to elevators
- Prevent race conditions with atomic operations or synchronized blocks

**6. State Machines**

Elevator behavior can be modeled as a state machine:

```
IDLE → MOVING_UP → DOORS_OPEN → IDLE
```

Each state transition is atomic and protected from concurrent interference.

**7. Events and Message Queues (Advanced)**

Use an event-driven or actor-based system:

- Elevator receives messages: `MoveToFloor`, `OpenDoor`
- Requests are queued and processed sequentially

Tools:
- Java: `BlockingQueue`
- Akka (Scala/Java): Actor model

### Example Java Snippet with Synchronization

```java
class Elevator {
    private final Lock lock = new ReentrantLock();
    private final TreeSet<Integer> upStops = new TreeSet<>();

    public void addStop(int floor) {
        lock.lock();
        try {
            upStops.add(floor);
        } finally {
            lock.unlock();
        }
    }

    public void move() {
        lock.lock();
        try {
            // move to next floor, update state
        } finally {
            lock.unlock();
        }
    }
}
```

### Summary

| Strategy | Purpose |
|----------|---------|
| Locks / synchronized | Protect shared state (floor queues, direction, etc.) |
| Threads or Timers | Simulate independent elevator behavior |
| Concurrent collections | Thread-safe request handling |
| Message queues or actor model | Decouple logic and ensure safe async communication |

---

## Scalability and Extensibility

Scalability and extensibility are critical when evolving from a basic elevator system to a large, production-grade system (think: airports, hospitals, or smart buildings).

### Scalability: How to Scale as Load or Size Grows

**1. More Elevators and Floors**

- **Problem:** Centralized scheduling becomes a bottleneck
- **Solution:**
  - **Zoning:** Assign elevators to floor zones (e.g., low/mid/high)
  - **Distributed controllers:** One controller per zone, with a coordinator
  - Partition request queues per elevator or floor zone

**2. Load Balancing**

Handle burst traffic during peak hours (e.g., 9AM at office lobby)

Techniques:
- Predictive pre-positioning (e.g., move idle elevators to busy floors)
- Time-based scheduling optimization
- Elevator grouping: bank elevators together and let only one respond per group

**3. Horizontal Scaling**

If requests/updates become too frequent:

- Use message queues (e.g., Kafka, RabbitMQ) between button systems and controllers
- Microservices: Separate floor request handler, assignment engine, state monitor

### Extensibility: How to Add More Features Easily

**1. Pluggable Scheduling Algorithms**

Keep scheduler as a strategy or interface:

```java
interface SchedulingStrategy {
    Elevator assignElevator(Request request, List<Elevator> elevators);
}
```

Plug in algorithms like:
- Nearest elevator
- Load-balancing scheduler
- Predictive ML-based scheduler

**2. Maintenance Mode**

- Add a `MAINTENANCE` state
- Exclude such elevators from assignment
- Automatically notify admin or backend system

**3. Security and Access Control**

- VIP elevators, service-only floors
- Floor access based on keycard or app

**4. IoT/Real-Time Monitoring**

Add hooks to send elevator telemetry to a backend:

- Floor position
- Door state
- Usage stats
- Health metrics

Dashboard for monitoring via WebSocket or polling APIs

**5. Mobile App Integration**

Users can:
- Call elevator from phone
- Set destination floor in advance
- Elevator pre-positioning via app prediction

**6. Energy Optimization**

- Group similar requests to minimize total travel
- Auto-sleep idle elevators

### Summary Table

| Area | Techniques/Ideas |
|------|------------------|
| **Scalability** | Zoning, distributed controllers, microservices |
| **Load Handling** | Predictive prepositioning, elevator groups, async messaging |
| **Extensibility** | Modular scheduler, maintenance mode, app integration |
| **Monitoring** | Telemetry APIs, dashboards, alerts |
| **Smart Features** | ML-based dispatching, energy-efficient routing |
