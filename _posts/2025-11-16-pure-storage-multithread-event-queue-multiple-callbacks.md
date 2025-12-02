---
layout: post
title: "Pure Storage Interview: Multi-thread Event Queue with Multiple Callbacks"
date: 2025-11-16
categories: [Interview Preparation, Pure Storage, Technical Interview, Multi-threading]
excerpt: "Complete implementation of multi-threaded event queue supporting multiple callbacks per event: thread-safe queue operations, callback registration, event processing, and concurrent access patterns."
---

## Problem Overview

**Pure Storage Interview: Multi-thread Event Queue with Multiple Callbacks**

This problem tests your ability to:
1. Design thread-safe event queue systems
2. Handle multiple callbacks per event
3. Manage concurrent event processing
4. Implement efficient producer-consumer patterns
5. Ensure callback execution order and fairness

### Problem Statement

Design and implement a multi-threaded event queue system where:
- Multiple events can be queued
- Each event can have multiple callbacks registered
- Multiple threads can register callbacks concurrently
- Multiple threads can process events concurrently
- Callbacks are executed when their associated event is processed

**Requirements:**
- Thread-safe operations
- Support multiple callbacks per event
- Efficient concurrent access
- Proper callback execution order

## Part 1: Basic Event Queue Structure

### Event and Callback Definitions

```cpp
#include <vector>
#include <functional>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <atomic>
#include <memory>
#include <iostream>
using namespace std;

struct Event {
    int id;
    string type;
    string data;
    
    Event(int i = 0, const string& t = "", const string& d = "") 
        : id(i), type(t), data(d) {}
};

class EventQueue {
private:
    struct QueuedEvent {
        Event event;
        vector<function<void(const Event&)>> callbacks;
        atomic<int> completedCallbacks{0};
        int totalCallbacks;
        
        QueuedEvent(const Event& e) : event(e), totalCallbacks(0) {}
    };
    
    queue<shared_ptr<QueuedEvent>> eventQueue;
    mutex queueMtx;
    condition_variable cv;
    atomic<bool> shutdown{false};
    
public:
    EventQueue() {}
    
    // Register callback for a specific event
    void registerCallback(int eventId, function<void(const Event&)> callback) {
        lock_guard<mutex> lock(queueMtx);
        
        // Find event in queue
        queue<shared_ptr<QueuedEvent>> tempQueue = eventQueue;
        while (!tempQueue.empty()) {
            auto queuedEvent = tempQueue.front();
            tempQueue.pop();
            
            if (queuedEvent->event.id == eventId) {
                queuedEvent->callbacks.push_back(callback);
                queuedEvent->totalCallbacks++;
                return;
            }
        }
        
        // Event not found - create new event
        auto newEvent = make_shared<QueuedEvent>(Event(eventId));
        newEvent->callbacks.push_back(callback);
        newEvent->totalCallbacks = 1;
        eventQueue.push(newEvent);
        cv.notify_one();
    }
    
    // Process next event
    bool processNextEvent() {
        shared_ptr<QueuedEvent> queuedEvent;
        
        {
            unique_lock<mutex> lock(queueMtx);
            
            cv.wait(lock, [this]() {
                return !eventQueue.empty() || shutdown.load();
            });
            
            if (shutdown.load() && eventQueue.empty()) {
                return false;
            }
            
            queuedEvent = eventQueue.front();
            eventQueue.pop();
        }
        
        // Execute all callbacks
        for (auto& callback : queuedEvent->callbacks) {
            callback(queuedEvent->event);
            queuedEvent->completedCallbacks++;
        }
        
        return true;
    }
    
    void shutdown() {
        shutdown.store(true);
        cv.notify_all();
    }
};
```

## Part 2: Improved Design with Event-Callback Mapping

### Better Data Structure

```cpp
#include <unordered_map>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <atomic>

class MultiCallbackEventQueue {
private:
    struct EventData {
        Event event;
        vector<function<void(const Event&)>> callbacks;
        bool processed;
        
        EventData(const Event& e) : event(e), processed(false) {}
    };
    
    queue<int> eventIdQueue;  // Queue of event IDs
    unordered_map<int, shared_ptr<EventData>> events;  // Event ID -> EventData
    mutex mtx;
    condition_variable cv;
    atomic<bool> shutdown{false};
    
public:
    MultiCallbackEventQueue() {}
    
    // Add event to queue
    void addEvent(const Event& event) {
        lock_guard<mutex> lock(mtx);
        
        if (events.find(event.id) == events.end()) {
            events[event.id] = make_shared<EventData>(event);
            eventIdQueue.push(event.id);
            cv.notify_one();
        }
    }
    
    // Register callback for event
    void registerCallback(int eventId, function<void(const Event&)> callback) {
        lock_guard<mutex> lock(mtx);
        
        if (events.find(eventId) == events.end()) {
            // Event doesn't exist - create it
            events[eventId] = make_shared<EventData>(Event(eventId));
            eventIdQueue.push(eventId);
        }
        
        events[eventId]->callbacks.push_back(callback);
        cv.notify_one();
    }
    
    // Process next event
    optional<Event> processNextEvent() {
        shared_ptr<EventData> eventData;
        
        {
            unique_lock<mutex> lock(mtx);
            
            cv.wait(lock, [this]() {
                return !eventIdQueue.empty() || shutdown.load();
            });
            
            if (shutdown.load() && eventIdQueue.empty()) {
                return nullopt;
            }
            
            int eventId = eventIdQueue.front();
            eventIdQueue.pop();
            eventData = events[eventId];
        }
        
        // Execute callbacks outside lock
        if (!eventData->processed) {
            eventData->processed = true;
            for (auto& callback : eventData->callbacks) {
                callback(eventData->event);
            }
        }
        
        return eventData->event;
    }
    
    void shutdown() {
        shutdown.store(true);
        cv.notify_all();
    }
};
```

## Part 3: Complete Thread-Safe Implementation

### Production-Ready Version

```cpp
#include <queue>
#include <unordered_map>
#include <vector>
#include <functional>
#include <mutex>
#include <condition_variable>
#include <atomic>
#include <memory>
#include <optional>
#include <thread>
#include <iostream>
using namespace std;

class MultiCallbackEventQueue {
private:
    struct EventWithCallbacks {
        Event event;
        vector<function<void(const Event&)>> callbacks;
        atomic<bool> processed{false};
        mutex callbackMtx;  // Protect callbacks vector
        
        EventWithCallbacks(const Event& e) : event(e) {}
    };
    
    queue<int> eventQueue;  // Queue of event IDs
    unordered_map<int, shared_ptr<EventWithCallbacks>> eventMap;
    mutex queueMtx;
    condition_variable cv;
    atomic<bool> shutdown{false};
    
    // Remove processed events (optional cleanup)
    void cleanupProcessed() {
        lock_guard<mutex> lock(queueMtx);
        auto it = eventMap.begin();
        while (it != eventMap.end()) {
            if (it->second->processed.load()) {
                it = eventMap.erase(it);
            } else {
                ++it;
            }
        }
    }
    
public:
    MultiCallbackEventQueue() {}
    
    ~MultiCallbackEventQueue() {
        shutdown();
    }
    
    // Add event to queue
    void addEvent(const Event& event) {
        lock_guard<mutex> lock(queueMtx);
        
        if (eventMap.find(event.id) == eventMap.end()) {
            eventMap[event.id] = make_shared<EventWithCallbacks>(event);
            eventQueue.push(event.id);
            cv.notify_one();
        }
    }
    
    // Register callback for event (can be called multiple times)
    void registerCallback(int eventId, function<void(const Event&)> callback) {
        if (!callback) {
            return;
        }
        
        shared_ptr<EventWithCallbacks> eventData;
        bool needsNotification = false;
        
        {
            lock_guard<mutex> lock(queueMtx);
            
            if (eventMap.find(eventId) == eventMap.end()) {
                // Create new event
                eventMap[eventId] = make_shared<EventWithCallbacks>(Event(eventId));
                eventQueue.push(eventId);
                needsNotification = true;
            }
            
            eventData = eventMap[eventId];
        }
        
        // Add callback (might be adding while processing)
        {
            lock_guard<mutex> lock(eventData->callbackMtx);
            
            // Check if already processed
            if (eventData->processed.load()) {
                // Event already processed - call immediately
                lock.unlock();
                callback(eventData->event);
                return;
            }
            
            eventData->callbacks.push_back(callback);
        }
        
        if (needsNotification) {
            cv.notify_one();
        }
    }
    
    // Process next event
    optional<Event> processNextEvent() {
        shared_ptr<EventWithCallbacks> eventData;
        
        {
            unique_lock<mutex> lock(queueMtx);
            
            cv.wait(lock, [this]() {
                return !eventQueue.empty() || shutdown.load();
            });
            
            if (shutdown.load() && eventQueue.empty()) {
                return nullopt;
            }
            
            int eventId = eventQueue.front();
            eventQueue.pop();
            eventData = eventMap[eventId];
        }
        
        // Check if already processed
        bool expected = false;
        if (!eventData->processed.compare_exchange_strong(expected, true)) {
            // Already processed by another thread
            return eventData->event;
        }
        
        // Execute callbacks
        vector<function<void(const Event&)>> callbacksToExecute;
        
        {
            lock_guard<mutex> lock(eventData->callbackMtx);
            callbacksToExecute = eventData->callbacks;
        }
        
        // Execute outside lock
        for (auto& callback : callbacksToExecute) {
            try {
                callback(eventData->event);
            } catch (const exception& e) {
                cerr << "Callback error: " << e.what() << endl;
            }
        }
        
        return eventData->event;
    }
    
    // Get queue size
    size_t getQueueSize() const {
        lock_guard<mutex> lock(queueMtx);
        return eventQueue.size();
    }
    
    // Get callbacks count for event
    size_t getCallbackCount(int eventId) const {
        lock_guard<mutex> lock(queueMtx);
        auto it = eventMap.find(eventId);
        if (it == eventMap.end()) {
            return 0;
        }
        
        lock_guard<mutex> callbackLock(it->second->callbackMtx);
        return it->second->callbacks.size();
    }
    
    void shutdown() {
        shutdown.store(true);
        cv.notify_all();
    }
    
    bool isShutdown() const {
        return shutdown.load();
    }
};
```

## Part 4: Worker Thread Pool Implementation

### Multiple Worker Threads Processing Events

```cpp
#include <vector>
#include <thread>
#include <atomic>

class EventQueueWithWorkers {
private:
    MultiCallbackEventQueue eventQueue;
    vector<thread> workerThreads;
    atomic<bool> running{false};
    int numWorkers;
    
    void workerThread() {
        while (running.load() || eventQueue.getQueueSize() > 0) {
            auto event = eventQueue.processNextEvent();
            if (!event.has_value() && !running.load()) {
                break;
            }
        }
    }
    
public:
    EventQueueWithWorkers(int workers = 4) : numWorkers(workers) {}
    
    ~EventQueueWithWorkers() {
        stop();
    }
    
    void start() {
        if (running.load()) {
            return;
        }
        
        running.store(true);
        
        for (int i = 0; i < numWorkers; i++) {
            workerThreads.emplace_back(&EventQueueWithWorkers::workerThread, this);
        }
    }
    
    void stop() {
        running.store(false);
        eventQueue.shutdown();
        
        for (auto& t : workerThreads) {
            if (t.joinable()) {
                t.join();
            }
        }
        
        workerThreads.clear();
    }
    
    void addEvent(const Event& event) {
        eventQueue.addEvent(event);
    }
    
    void registerCallback(int eventId, function<void(const Event&)> callback) {
        eventQueue.registerCallback(eventId, callback);
    }
    
    size_t getQueueSize() const {
        return eventQueue.getQueueSize();
    }
};
```

## Part 5: Advanced: Priority and Ordering

### Priority-Based Event Processing

```cpp
#include <queue>
#include <functional>

struct PriorityEvent {
    Event event;
    int priority;
    int sequence;  // For FIFO within same priority
    
    bool operator<(const PriorityEvent& other) const {
        if (priority != other.priority) {
            return priority < other.priority;  // Higher priority first
        }
        return sequence > other.sequence;  // Earlier sequence first
    }
};

class PriorityEventQueue {
private:
    priority_queue<PriorityEvent> eventQueue;
    unordered_map<int, shared_ptr<EventWithCallbacks>> eventMap;
    mutex mtx;
    condition_variable cv;
    atomic<int> sequenceCounter{0};
    atomic<bool> shutdown{false};
    
public:
    void addEvent(const Event& event, int priority = 0) {
        lock_guard<mutex> lock(mtx);
        
        int seq = sequenceCounter++;
        PriorityEvent pEvent;
        pEvent.event = event;
        pEvent.priority = priority;
        pEvent.sequence = seq;
        
        eventQueue.push(pEvent);
        
        if (eventMap.find(event.id) == eventMap.end()) {
            eventMap[event.id] = make_shared<EventWithCallbacks>(event);
        }
        
        cv.notify_one();
    }
    
    void registerCallback(int eventId, function<void(const Event&)> callback) {
        lock_guard<mutex> lock(mtx);
        
        if (eventMap.find(eventId) == eventMap.end()) {
            eventMap[eventId] = make_shared<EventWithCallbacks>(Event(eventId));
        }
        
        eventMap[eventId]->callbacks.push_back(callback);
    }
    
    optional<Event> processNextEvent() {
        PriorityEvent pEvent;
        
        {
            unique_lock<mutex> lock(mtx);
            
            cv.wait(lock, [this]() {
                return !eventQueue.empty() || shutdown.load();
            });
            
            if (shutdown.load() && eventQueue.empty()) {
                return nullopt;
            }
            
            pEvent = eventQueue.top();
            eventQueue.pop();
        }
        
        // Execute callbacks
        auto it = eventMap.find(pEvent.event.id);
        if (it != eventMap.end()) {
            for (auto& callback : it->second->callbacks) {
                callback(pEvent.event);
            }
        }
        
        return pEvent.event;
    }
    
    void shutdown() {
        shutdown.store(true);
        cv.notify_all();
    }
};
```

## Part 6: Complete Test Program

```cpp
#include <iostream>
#include <thread>
#include <chrono>
#include <vector>
#include <atomic>
#include <cassert>

void testBasicMultiCallback() {
    cout << "=== Testing Basic Multi-Callback ===" << endl;
    
    MultiCallbackEventQueue queue;
    atomic<int> callbackCount{0};
    
    // Add event
    Event event(1, "test", "data");
    queue.addEvent(event);
    
    // Register multiple callbacks
    queue.registerCallback(1, [&callbackCount](const Event& e) {
        callbackCount++;
        cout << "Callback 1: Event " << e.id << endl;
    });
    
    queue.registerCallback(1, [&callbackCount](const Event& e) {
        callbackCount++;
        cout << "Callback 2: Event " << e.id << endl;
    });
    
    queue.registerCallback(1, [&callbackCount](const Event& e) {
        callbackCount++;
        cout << "Callback 3: Event " << e.id << endl;
    });
    
    // Process event
    auto processed = queue.processNextEvent();
    assert(processed.has_value());
    assert(processed->id == 1);
    assert(callbackCount == 3);
    
    cout << "Basic multi-callback test passed!" << endl;
}

void testConcurrentRegistration() {
    cout << "\n=== Testing Concurrent Registration ===" << endl;
    
    MultiCallbackEventQueue queue;
    atomic<int> totalCallbacks{0};
    const int NUM_THREADS = 10;
    const int CALLBACKS_PER_THREAD = 5;
    
    // Add event first
    queue.addEvent(Event(1, "concurrent", "test"));
    
    vector<thread> threads;
    
    // Multiple threads register callbacks
    for (int i = 0; i < NUM_THREADS; i++) {
        threads.emplace_back([&queue, &totalCallbacks, i]() {
            for (int j = 0; j < CALLBACKS_PER_THREAD; j++) {
                queue.registerCallback(1, [&totalCallbacks, i, j](const Event& e) {
                    totalCallbacks++;
                });
            }
        });
    }
    
    // Wait a bit for registrations
    this_thread::sleep_for(chrono::milliseconds(100));
    
    // Process event
    queue.processNextEvent();
    
    for (auto& t : threads) {
        t.join();
    }
    
    int expected = NUM_THREADS * CALLBACKS_PER_THREAD;
    cout << "Expected: " << expected << ", Actual: " << totalCallbacks << endl;
    assert(totalCallbacks >= expected);
    
    cout << "Concurrent registration test passed!" << endl;
}

void testMultipleEvents() {
    cout << "\n=== Testing Multiple Events ===" << endl;
    
    MultiCallbackEventQueue queue;
    atomic<int> event1Callbacks{0};
    atomic<int> event2Callbacks{0};
    
    // Add events
    queue.addEvent(Event(1, "event1", "data1"));
    queue.addEvent(Event(2, "event2", "data2"));
    
    // Register callbacks for event 1
    queue.registerCallback(1, [&event1Callbacks](const Event& e) {
        event1Callbacks++;
    });
    queue.registerCallback(1, [&event1Callbacks](const Event& e) {
        event1Callbacks++;
    });
    
    // Register callbacks for event 2
    queue.registerCallback(2, [&event2Callbacks](const Event& e) {
        event2Callbacks++;
    });
    queue.registerCallback(2, [&event2Callbacks](const Event& e) {
        event2Callbacks++;
    });
    queue.registerCallback(2, [&event2Callbacks](const Event& e) {
        event2Callbacks++;
    });
    
    // Process events
    queue.processNextEvent();
    queue.processNextEvent();
    
    assert(event1Callbacks == 2);
    assert(event2Callbacks == 3);
    
    cout << "Multiple events test passed!" << endl;
}

void testWorkerThreads() {
    cout << "\n=== Testing Worker Threads ===" << endl;
    
    EventQueueWithWorkers queue(4);  // 4 worker threads
    queue.start();
    
    atomic<int> totalCallbacks{0};
    const int NUM_EVENTS = 20;
    const int CALLBACKS_PER_EVENT = 3;
    
    // Add events
    for (int i = 1; i <= NUM_EVENTS; i++) {
        queue.addEvent(Event(i, "event", "data"));
        
        // Register callbacks
        for (int j = 0; j < CALLBACKS_PER_EVENT; j++) {
            queue.registerCallback(i, [&totalCallbacks](const Event& e) {
                totalCallbacks++;
            });
        }
    }
    
    // Wait for processing
    while (queue.getQueueSize() > 0) {
        this_thread::sleep_for(chrono::milliseconds(10));
    }
    
    this_thread::sleep_for(chrono::milliseconds(100));  // Allow callbacks to complete
    
    int expected = NUM_EVENTS * CALLBACKS_PER_EVENT;
    cout << "Expected: " << expected << ", Actual: " << totalCallbacks << endl;
    assert(totalCallbacks == expected);
    
    queue.stop();
    
    cout << "Worker threads test passed!" << endl;
}

int main() {
    testBasicMultiCallback();
    testConcurrentRegistration();
    testMultipleEvents();
    testWorkerThreads();
    
    cout << "\nAll tests passed!" << endl;
    return 0;
}
```

## Part 7: Design Considerations

### Data Structure Choices

**Option 1: Queue + Map**
- Queue: Maintains processing order
- Map: Fast lookup for callback registration
- **Pros**: Efficient, supports multiple callbacks
- **Cons**: More complex

**Option 2: Single Queue**
- Queue of events with embedded callbacks
- **Pros**: Simple
- **Cons**: Harder to add callbacks to existing events

**Option 3: Separate Queues**
- One queue per event type
- **Pros**: Better isolation
- **Cons**: More overhead

### Thread Safety Strategies

1. **Single Mutex**: Protect entire queue
   - Simple but may cause contention
   
2. **Multiple Mutexes**: Separate locks for queue and callbacks
   - Better concurrency
   - More complex

3. **Lock-Free**: Use atomic operations
   - Best performance
   - Most complex

### Callback Execution Order

**Options:**
1. **Registration order**: Execute in order registered
2. **Priority order**: Execute by priority
3. **Parallel execution**: Execute concurrently
4. **Sequential execution**: One at a time

## Part 8: Performance Optimization

### Batch Processing

```cpp
class BatchEventQueue {
private:
    MultiCallbackEventQueue baseQueue;
    vector<Event> batch;
    mutex batchMtx;
    size_t batchSize;
    
public:
    BatchEventQueue(size_t size = 10) : batchSize(size) {}
    
    void addEvent(const Event& event) {
        lock_guard<mutex> lock(batchMtx);
        batch.push_back(event);
        
        if (batch.size() >= batchSize) {
            flushBatch();
        }
    }
    
    void flushBatch() {
        for (const auto& event : batch) {
            baseQueue.addEvent(event);
        }
        batch.clear();
    }
};
```

### Lock-Free Optimizations

```cpp
#include <atomic>

class LockFreeEventQueue {
private:
    struct Node {
        atomic<Node*> next;
        Event event;
        vector<function<void(const Event&)>> callbacks;
    };
    
    atomic<Node*> head;
    atomic<Node*> tail;
    
public:
    LockFreeEventQueue() {
        Node* dummy = new Node();
        head.store(dummy);
        tail.store(dummy);
    }
    
    void enqueue(const Event& event) {
        Node* new_node = new Node();
        new_node->event = event;
        
        Node* prev_tail = tail.exchange(new_node);
        prev_tail->next.store(new_node);
    }
    
    bool dequeue(Event& event) {
        Node* head_node = head.load();
        Node* next = head_node->next.load();
        
        if (next == nullptr) {
            return false;
        }
        
        event = next->event;
        head.store(next);
        delete head_node;
        
        return true;
    }
};
```

## Part 9: Interview Discussion Points

### Key Questions and Answers

**Q1: How to handle callback registration after event is processed?**

**A:**
- Check if event is already processed
- If processed: call callback immediately
- If not: add to callback list
- Use atomic flag to check processing state

**Q2: What if multiple threads process the same event?**

**A:**
- Use atomic compare-and-swap to ensure only one thread processes
- Or use mutex to protect processing
- Mark event as processed atomically

**Q3: How to ensure all callbacks are executed?**

**A:**
- Lock callback list during registration
- Check processing state before adding
- Execute callbacks atomically
- Handle callbacks registered during processing

**Q4: Performance bottlenecks?**

**A:**
- **Single mutex**: High contention
- **Callback execution in lock**: Blocks other operations
- **Large callback lists**: Copy overhead

**Solutions:**
- Separate locks for queue and callbacks
- Execute callbacks outside locks
- Use lock-free structures

**Q5: How to handle callback errors?**

**A:**
- Wrap callback execution in try-catch
- Continue with other callbacks
- Log errors but don't crash
- Optionally track failed callbacks

## Key Takeaways

### Design Principles

1. **Separation of concerns**: Queue management vs. callback execution
2. **Thread safety**: Protect shared data structures
3. **Efficient locking**: Minimize lock contention
4. **Error handling**: Robust callback execution

### Implementation Patterns

1. **Queue + Map**: Efficient lookup and ordering
2. **Double-check pattern**: Handle race conditions
3. **Execute outside lock**: Prevent deadlock
4. **Atomic flags**: Track processing state

### Common Pitfalls

1. **Race conditions**: Registering callbacks during processing
2. **Deadlock**: Calling callbacks while holding locks
3. **Lost callbacks**: Not checking processing state
4. **Performance**: Holding locks during callback execution

### Interview Tips

1. **Start simple**: Basic queue + callbacks
2. **Add thread safety**: Mutex protection
3. **Handle edge cases**: Registration during processing
4. **Optimize**: Separate locks, execute outside lock
5. **Discuss trade-offs**: Performance vs. complexity

## Summary

Multi-thread event queue with multiple callbacks demonstrates:

- **Event queue design**: Efficient event storage and processing
- **Multiple callbacks**: Support for multiple handlers per event
- **Thread synchronization**: Safe concurrent access
- **Callback management**: Registration and execution
- **Performance optimization**: Minimizing lock contention

**Key implementation:**
- **Queue + Map structure**: Efficient event management
- **Thread-safe operations**: Mutex protection
- **Callback execution**: Outside locks to prevent deadlock
- **Processing state**: Atomic flags for race condition prevention

Understanding these concepts is essential for:
- System design interviews
- Event-driven architectures
- Concurrent systems
- Message queue design

This knowledge is particularly valuable for storage companies like Pure Storage, where efficient event processing and concurrent callback handling are critical for system performance.

