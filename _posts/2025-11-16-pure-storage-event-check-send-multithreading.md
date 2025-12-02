---
layout: post
title: "Pure Storage Interview: Event Checking and Sending (Multi-threading)"
date: 2025-11-16
categories: [Interview Preparation, Pure Storage, Technical Interview, Multi-threading]
excerpt: "Complete implementation of thread-safe event system: check events and send events with producer-consumer pattern. Includes C++ implementations with mutex, condition variables, and queue management."
---

## Problem Overview

**Pure Storage Interview: Event Checking and Sending**

This problem tests your ability to:
1. Design thread-safe event systems
2. Implement producer-consumer pattern
3. Handle concurrent event checking and sending
4. Manage event queues efficiently
5. Ensure thread synchronization

### Problem Statement

Design and implement a thread-safe event system with two operations:

1. **`sendEvent(Event event)`**: Send an event to the system
2. **`checkEvent()`**: Check/retrieve an event from the system

**Requirements:**
- Multiple threads can call `sendEvent()` concurrently
- Multiple threads can call `checkEvent()` concurrently
- Thread-safe: No race conditions or data corruption
- Efficient: Handle high-frequency events
- Blocking/non-blocking: `checkEvent()` may block if no events available

### Use Cases

- Event-driven systems
- Message queues
- Producer-consumer scenarios
- Logging systems
- Notification systems

## Part 1: Basic Thread-Safe Event Queue

### Simple Implementation with Mutex

```cpp
#include <queue>
#include <mutex>
#include <condition_variable>
#include <memory>
#include <iostream>
using namespace std;

struct Event {
    int id;
    string data;
    Event(int i = 0, const string& d = "") : id(i), data(d) {}
};

class EventSystem {
private:
    queue<Event> eventQueue;
    mutex mtx;
    condition_variable cv;
    
public:
    // Send event (producer)
    void sendEvent(const Event& event) {
        lock_guard<mutex> lock(mtx);
        eventQueue.push(event);
        cv.notify_one();  // Notify one waiting thread
    }
    
    // Check event (consumer) - blocking
    Event checkEvent() {
        unique_lock<mutex> lock(mtx);
        
        // Wait until queue is not empty
        cv.wait(lock, [this]() { return !eventQueue.empty(); });
        
        Event event = eventQueue.front();
        eventQueue.pop();
        
        return event;
    }
    
    // Check event - non-blocking (returns false if no event)
    bool checkEvent(Event& event) {
        lock_guard<mutex> lock(mtx);
        
        if (eventQueue.empty()) {
            return false;
        }
        
        event = eventQueue.front();
        eventQueue.pop();
        return true;
    }
    
    // Get queue size
    size_t size() const {
        lock_guard<mutex> lock(mtx);
        return eventQueue.size();
    }
    
    bool empty() const {
        lock_guard<mutex> lock(mtx);
        return eventQueue.empty();
    }
};
```

## Part 2: Enhanced Event System with Timeout

### Non-Blocking with Timeout Support

```cpp
#include <queue>
#include <mutex>
#include <condition_variable>
#include <chrono>
#include <optional>
using namespace std;
using namespace chrono;

class EventSystemTimeout {
private:
    queue<Event> eventQueue;
    mutable mutex mtx;
    condition_variable cv;
    atomic<bool> shutdown{false};
    
public:
    // Send event
    void sendEvent(const Event& event) {
        lock_guard<mutex> lock(mtx);
        eventQueue.push(event);
        cv.notify_one();
    }
    
    // Check event with timeout
    optional<Event> checkEvent(milliseconds timeout) {
        unique_lock<mutex> lock(mtx);
        
        // Wait with timeout
        bool notified = cv.wait_for(lock, timeout, [this]() {
            return !eventQueue.empty() || shutdown.load();
        });
        
        if (shutdown.load()) {
            return nullopt;
        }
        
        if (!notified || eventQueue.empty()) {
            return nullopt;  // Timeout
        }
        
        Event event = eventQueue.front();
        eventQueue.pop();
        return event;
    }
    
    // Check event - blocking
    optional<Event> checkEvent() {
        unique_lock<mutex> lock(mtx);
        
        cv.wait(lock, [this]() {
            return !eventQueue.empty() || shutdown.load();
        });
        
        if (shutdown.load() && eventQueue.empty()) {
            return nullopt;
        }
        
        Event event = eventQueue.front();
        eventQueue.pop();
        return event;
    }
    
    // Shutdown event system
    void shutdown() {
        shutdown.store(true);
        cv.notify_all();
    }
    
    size_t size() const {
        lock_guard<mutex> lock(mtx);
        return eventQueue.size();
    }
};
```

## Part 3: Complete Producer-Consumer Implementation

### Full-Featured Event System

```cpp
#include <queue>
#include <mutex>
#include <condition_variable>
#include <atomic>
#include <chrono>
#include <optional>
#include <vector>
#include <thread>
#include <iostream>
using namespace std;
using namespace chrono;

struct Event {
    int id;
    string type;
    string data;
    system_clock::time_point timestamp;
    
    Event(int i = 0, const string& t = "", const string& d = "") 
        : id(i), type(t), data(d), timestamp(system_clock::now()) {}
};

class EventSystem {
private:
    queue<Event> eventQueue;
    mutable mutex mtx;
    condition_variable cv;
    atomic<bool> shutdown{false};
    atomic<size_t> totalSent{0};
    atomic<size_t> totalChecked{0};
    size_t maxQueueSize;
    
public:
    EventSystem(size_t maxSize = 10000) : maxQueueSize(maxSize) {}
    
    // Send event (producer)
    bool sendEvent(const Event& event) {
        unique_lock<mutex> lock(mtx);
        
        // Wait if queue is full
        cv.wait(lock, [this]() {
            return eventQueue.size() < maxQueueSize || shutdown.load();
        });
        
        if (shutdown.load()) {
            return false;
        }
        
        eventQueue.push(event);
        totalSent++;
        cv.notify_one();  // Notify one consumer
        
        return true;
    }
    
    // Check event - blocking (consumer)
    optional<Event> checkEvent() {
        unique_lock<mutex> lock(mtx);
        
        // Wait until event available
        cv.wait(lock, [this]() {
            return !eventQueue.empty() || shutdown.load();
        });
        
        if (shutdown.load() && eventQueue.empty()) {
            return nullopt;
        }
        
        Event event = eventQueue.front();
        eventQueue.pop();
        totalChecked++;
        
        return event;
    }
    
    // Check event - non-blocking
    optional<Event> tryCheckEvent() {
        lock_guard<mutex> lock(mtx);
        
        if (eventQueue.empty()) {
            return nullopt;
        }
        
        Event event = eventQueue.front();
        eventQueue.pop();
        totalChecked++;
        
        return event;
    }
    
    // Check event with timeout
    optional<Event> checkEvent(milliseconds timeout) {
        unique_lock<mutex> lock(mtx);
        
        bool notified = cv.wait_for(lock, timeout, [this]() {
            return !eventQueue.empty() || shutdown.load();
        });
        
        if (shutdown.load() && eventQueue.empty()) {
            return nullopt;
        }
        
        if (!notified || eventQueue.empty()) {
            return nullopt;  // Timeout
        }
        
        Event event = eventQueue.front();
        eventQueue.pop();
        totalChecked++;
        
        return event;
    }
    
    // Shutdown
    void shutdown() {
        shutdown.store(true);
        cv.notify_all();
    }
    
    // Statistics
    size_t size() const {
        lock_guard<mutex> lock(mtx);
        return eventQueue.size();
    }
    
    bool empty() const {
        lock_guard<mutex> lock(mtx);
        return eventQueue.empty();
    }
    
    size_t getTotalSent() const {
        return totalSent.load();
    }
    
    size_t getTotalChecked() const {
        return totalChecked.load();
    }
    
    bool isShutdown() const {
        return shutdown.load();
    }
};
```

## Part 4: Multiple Consumers Pattern

### Fair Distribution Among Consumers

```cpp
class FairEventSystem {
private:
    queue<Event> eventQueue;
    mutable mutex mtx;
    condition_variable cv;
    atomic<bool> shutdown{false};
    atomic<int> activeConsumers{0};
    
public:
    // Send event
    void sendEvent(const Event& event) {
        lock_guard<mutex> lock(mtx);
        eventQueue.push(event);
        cv.notify_all();  // Notify all consumers
    }
    
    // Check event - blocking
    optional<Event> checkEvent(int consumerId) {
        unique_lock<mutex> lock(mtx);
        activeConsumers++;
        
        // Wait for event
        cv.wait(lock, [this]() {
            return !eventQueue.empty() || shutdown.load();
        });
        
        activeConsumers--;
        
        if (shutdown.load() && eventQueue.empty()) {
            return nullopt;
        }
        
        if (eventQueue.empty()) {
            return nullopt;
        }
        
        Event event = eventQueue.front();
        eventQueue.pop();
        
        // Notify other consumers if more events available
        if (!eventQueue.empty()) {
            cv.notify_one();
        }
        
        return event;
    }
    
    void shutdown() {
        shutdown.store(true);
        cv.notify_all();
    }
    
    size_t size() const {
        lock_guard<mutex> lock(mtx);
        return eventQueue.size();
    }
    
    int getActiveConsumers() const {
        return activeConsumers.load();
    }
};
```

## Part 5: Priority Event System

### Events with Priority Levels

```cpp
#include <queue>
#include <vector>
#include <algorithm>

struct PriorityEvent {
    Event event;
    int priority;  // Higher priority = processed first
    
    PriorityEvent(const Event& e, int p) : event(e), priority(p) {}
    
    bool operator<(const PriorityEvent& other) const {
        return priority < other.priority;  // Lower priority value = higher priority
    }
};

class PriorityEventSystem {
private:
    priority_queue<PriorityEvent> eventQueue;
    mutable mutex mtx;
    condition_variable cv;
    atomic<bool> shutdown{false};
    
public:
    void sendEvent(const Event& event, int priority = 0) {
        lock_guard<mutex> lock(mtx);
        eventQueue.push(PriorityEvent(event, priority));
        cv.notify_one();
    }
    
    optional<Event> checkEvent() {
        unique_lock<mutex> lock(mtx);
        
        cv.wait(lock, [this]() {
            return !eventQueue.empty() || shutdown.load();
        });
        
        if (shutdown.load() && eventQueue.empty()) {
            return nullopt;
        }
        
        PriorityEvent pEvent = eventQueue.top();
        eventQueue.pop();
        
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
#include <vector>
#include <thread>
#include <chrono>
#include <atomic>
#include <cassert>

void testBasicEventSystem() {
    cout << "=== Testing Basic Event System ===" << endl;
    
    EventSystem eventSystem;
    atomic<int> receivedCount{0};
    
    // Producer thread
    thread producer([&eventSystem]() {
        for (int i = 0; i < 10; i++) {
            Event event(i, "test", "data" + to_string(i));
            eventSystem.sendEvent(event);
            this_thread::sleep_for(chrono::milliseconds(10));
        }
    });
    
    // Consumer thread
    thread consumer([&eventSystem, &receivedCount]() {
        for (int i = 0; i < 10; i++) {
            auto event = eventSystem.checkEvent();
            if (event.has_value()) {
                receivedCount++;
                cout << "Received event: " << event->id << endl;
            }
        }
    });
    
    producer.join();
    consumer.join();
    
    assert(receivedCount == 10);
    cout << "Basic test passed!" << endl;
}

void testMultipleProducersConsumers() {
    cout << "\n=== Testing Multiple Producers/Consumers ===" << endl;
    
    EventSystem eventSystem;
    atomic<int> sentCount{0};
    atomic<int> receivedCount{0};
    const int NUM_PRODUCERS = 5;
    const int NUM_CONSUMERS = 3;
    const int EVENTS_PER_PRODUCER = 20;
    
    vector<thread> producers;
    vector<thread> consumers;
    
    // Start producers
    for (int i = 0; i < NUM_PRODUCERS; i++) {
        producers.emplace_back([&eventSystem, &sentCount, i]() {
            for (int j = 0; j < EVENTS_PER_PRODUCER; j++) {
                Event event(i * EVENTS_PER_PRODUCER + j, "producer" + to_string(i), "data");
                eventSystem.sendEvent(event);
                sentCount++;
            }
        });
    }
    
    // Start consumers
    for (int i = 0; i < NUM_CONSUMERS; i++) {
        consumers.emplace_back([&eventSystem, &receivedCount]() {
            while (receivedCount < NUM_PRODUCERS * EVENTS_PER_PRODUCER) {
                auto event = eventSystem.checkEvent();
                if (event.has_value()) {
                    receivedCount++;
                }
            }
        });
    }
    
    // Wait for all producers
    for (auto& t : producers) {
        t.join();
    }
    
    // Wait for all consumers
    for (auto& t : consumers) {
        t.join();
    }
    
    assert(sentCount == NUM_PRODUCERS * EVENTS_PER_PRODUCER);
    assert(receivedCount == NUM_PRODUCERS * EVENTS_PER_PRODUCER);
    
    cout << "Sent: " << sentCount << ", Received: " << receivedCount << endl;
    cout << "Multiple producers/consumers test passed!" << endl;
}

void testTimeout() {
    cout << "\n=== Testing Timeout ===" << endl;
    
    EventSystemTimeout eventSystem;
    
    // Try to check event with timeout (should timeout)
    auto event1 = eventSystem.checkEvent(chrono::milliseconds(100));
    assert(!event1.has_value());
    cout << "Timeout test 1 passed: No event available" << endl;
    
    // Send event
    Event e(1, "test", "data");
    eventSystem.sendEvent(e);
    
    // Check event (should succeed)
    auto event2 = eventSystem.checkEvent(chrono::milliseconds(100));
    assert(event2.has_value());
    assert(event2->id == 1);
    cout << "Timeout test 2 passed: Event received" << endl;
}

void testConcurrency() {
    cout << "\n=== Testing High Concurrency ===" << endl;
    
    EventSystem eventSystem;
    const int NUM_THREADS = 20;
    const int EVENTS_PER_THREAD = 100;
    atomic<int> totalReceived{0};
    
    vector<thread> threads;
    
    // Create mixed producer/consumer threads
    for (int i = 0; i < NUM_THREADS; i++) {
        if (i % 2 == 0) {
            // Producer thread
            threads.emplace_back([&eventSystem, i]() {
                for (int j = 0; j < EVENTS_PER_THREAD; j++) {
                    Event event(i * EVENTS_PER_THREAD + j, "thread" + to_string(i), "data");
                    eventSystem.sendEvent(event);
                }
            });
        } else {
            // Consumer thread
            threads.emplace_back([&eventSystem, &totalReceived]() {
                for (int j = 0; j < EVENTS_PER_THREAD; j++) {
                    auto event = eventSystem.checkEvent();
                    if (event.has_value()) {
                        totalReceived++;
                    }
                }
            });
        }
    }
    
    for (auto& t : threads) {
        t.join();
    }
    
    cout << "Total received: " << totalReceived << endl;
    cout << "Concurrency test passed!" << endl;
}

int main() {
    testBasicEventSystem();
    testMultipleProducersConsumers();
    testTimeout();
    testConcurrency();
    
    cout << "\nAll tests passed!" << endl;
    return 0;
}
```

## Part 7: Performance Optimization

### Lock-Free Queue (Advanced)

```cpp
#include <atomic>
#include <memory>

template<typename T>
class LockFreeQueue {
private:
    struct Node {
        atomic<T*> data;
        atomic<Node*> next;
        
        Node() : data(nullptr), next(nullptr) {}
    };
    
    atomic<Node*> head;
    atomic<Node*> tail;
    
public:
    LockFreeQueue() {
        Node* dummy = new Node();
        head.store(dummy);
        tail.store(dummy);
    }
    
    void enqueue(T item) {
        Node* new_node = new Node();
        T* data = new T(item);
        new_node->data.store(data);
        
        Node* prev_tail = tail.exchange(new_node);
        prev_tail->next.store(new_node);
    }
    
    bool dequeue(T& result) {
        Node* head_node = head.load();
        Node* next = head_node->next.load();
        
        if (next == nullptr) {
            return false;  // Empty
        }
        
        T* data = next->data.load();
        if (data == nullptr) {
            return false;
        }
        
        result = *data;
        head.store(next);
        delete head_node;
        delete data;
        
        return true;
    }
};

class LockFreeEventSystem {
private:
    LockFreeQueue<Event> eventQueue;
    condition_variable cv;
    mutex cvMtx;
    atomic<bool> shutdown{false};
    
public:
    void sendEvent(const Event& event) {
        eventQueue.enqueue(event);
        cv.notify_one();
    }
    
    optional<Event> checkEvent() {
        Event event;
        if (eventQueue.dequeue(event)) {
            return event;
        }
        
        unique_lock<mutex> lock(cvMtx);
        cv.wait(lock, [this, &event]() {
            return eventQueue.dequeue(event) || shutdown.load();
        });
        
        if (shutdown.load()) {
            return nullopt;
        }
        
        return event;
    }
    
    void shutdown() {
        shutdown.store(true);
        cv.notify_all();
    }
};
```

## Part 8: Design Patterns and Best Practices

### Producer-Consumer Pattern

**Key Components:**
1. **Shared Queue**: Thread-safe data structure
2. **Mutex**: Protects queue operations
3. **Condition Variable**: Efficient waiting/notification
4. **Atomic Counters**: Track statistics without locks

### Thread Safety Considerations

1. **Mutex Protection**: All queue operations must be protected
2. **Condition Variable**: Efficient blocking instead of busy-waiting
3. **Atomic Operations**: For counters and flags
4. **RAII**: Use lock_guard/unique_lock for exception safety

### Performance Tips

1. **Minimize Critical Section**: Keep lock hold time short
2. **Batch Operations**: Process multiple events at once if possible
3. **Lock-Free Structures**: For high-performance scenarios
4. **Separate Locks**: Different locks for different operations if needed

## Part 9: Interview Discussion Points

### Key Questions and Answers

**Q1: Why use condition variable instead of busy-waiting?**

**A:**
- **Efficiency**: Condition variable blocks thread until notified
- **CPU usage**: Busy-waiting wastes CPU cycles
- **Scalability**: Better for many waiting threads

**Q2: Why notify_one() vs notify_all()?**

**A:**
- **notify_one()**: Wakes one thread (more efficient)
- **notify_all()**: Wakes all threads (for priority/fairness)
- **Trade-off**: Efficiency vs. fairness

**Q3: How to handle queue overflow?**

**A:**
- **Blocking**: Wait until space available
- **Reject**: Return error if queue full
- **Drop**: Drop oldest/newest events
- **Resize**: Dynamically grow queue

**Q4: What if consumer is slow?**

**A:**
- **Multiple consumers**: Distribute load
- **Batching**: Process multiple events together
- **Priority**: Process important events first
- **Backpressure**: Slow down producers

**Q5: How to ensure event ordering?**

**A:**
- **Single consumer**: Process sequentially
- **Sequence numbers**: Track order
- **Priority queue**: Order by priority
- **Partitioning**: Separate queues per type

## Key Takeaways

### Design Principles

1. **Thread Safety**: Protect all shared data
2. **Efficient Waiting**: Use condition variables
3. **Fairness**: Consider multiple consumers
4. **Error Handling**: Handle edge cases gracefully

### Implementation Patterns

1. **Mutex + Condition Variable**: Standard producer-consumer
2. **Atomic Operations**: For counters and flags
3. **RAII**: Automatic lock management
4. **Optional Return**: Handle no-event case

### Common Pitfalls

1. **Deadlock**: Holding locks too long
2. **Race Conditions**: Unprotected shared state
3. **Lost Notifications**: Missing notify calls
4. **Memory Leaks**: Not cleaning up events

### Interview Tips

1. **Start simple**: Basic mutex + queue
2. **Add features**: Timeout, priority, etc.
3. **Discuss trade-offs**: Blocking vs. non-blocking
4. **Handle edge cases**: Empty queue, shutdown, errors
5. **Optimize**: Lock-free, batching, etc.

## Summary

Event checking and sending implementation demonstrates:

- **Producer-Consumer Pattern**: Classic concurrent design
- **Thread Synchronization**: Mutex and condition variables
- **Efficient Waiting**: Blocking without busy-waiting
- **Scalability**: Handle multiple producers/consumers
- **Robustness**: Error handling and edge cases

**Key implementation:**
- **Thread-safe queue**: Mutex-protected operations
- **Condition variable**: Efficient blocking/waking
- **Atomic counters**: Lock-free statistics
- **Multiple modes**: Blocking, non-blocking, timeout

Understanding these concepts is essential for:
- System design interviews
- Concurrent programming
- Event-driven systems
- Message queue design

This knowledge is particularly valuable for storage companies like Pure Storage, where efficient event handling and concurrent operations are critical for performance.

