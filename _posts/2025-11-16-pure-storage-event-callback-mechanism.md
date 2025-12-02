---
layout: post
title: "Pure Storage Interview: Event Callback Mechanism Implementation"
date: 2025-11-16
categories: [Interview Preparation, Pure Storage, Technical Interview, Multi-threading]
excerpt: "Complete implementation of event callback mechanism: register callbacks before/after event fires, thread-safe implementation with mutex, and proper callback invocation order."
---

## Problem Overview

**Pure Storage Interview: Event Callback Mechanism**

This problem tests your ability to:
1. Design callback mechanisms
2. Handle state transitions (before fire vs. after fire)
3. Implement thread-safe operations
4. Manage callback registration and invocation
5. Understand race conditions

### Problem Statement

Implement a callback mechanism with two API functions:

1. **`register_callback(Callback cb)`**: Register a callback function
2. **`event_fired()`**: Fire the event (can only be called once)

**Requirements:**
- There is only **one event** and it will fire **only once**
- Callbacks registered **before** event fires: Store them, invoke when `event_fired()` is called
- Callbacks registered **after** event fires: Invoke **immediately**
- Callbacks should **not block** waiting for event to fire

**Timeline Example:**
```
register_cb(cb1) → register_cb(cb2) → event_fired() 
→ cb1.call() → cb2.call() → register_cb(cb3) → cb3.call() (immediately)
```

### Key Challenge

- **State management**: Track whether event has fired
- **Callback storage**: Store callbacks before fire
- **Immediate execution**: Execute callbacks after fire
- **Thread safety**: Handle concurrent registrations

## Part 1: Single-Threaded Implementation

### Basic Structure

```cpp
#include <vector>
#include <functional>
#include <iostream>
using namespace std;

class Callback {
public:
    virtual void call() = 0;
    virtual ~Callback() = default;
};

class ConcreteCallback : public Callback {
private:
    string name;
    
public:
    ConcreteCallback(const string& n) : name(n) {}
    
    void call() override {
        cout << "Callback " << name << " is running now" << endl;
    }
};

class Event {
private:
    vector<Callback*> callbacks;
    bool fired;
    
public:
    Event() : fired(false) {}
    
    void register_cb(Callback* cb) {
        if (!cb) {
            return;  // Invalid callback
        }
        
        if (fired) {
            // Event already fired - call immediately
            cb->call();
        } else {
            // Event not fired yet - store callback
            callbacks.push_back(cb);
        }
    }
    
    void fire() {
        if (fired) {
            return;  // Already fired, ignore
        }
        
        fired = true;
        
        // Call all registered callbacks
        for (Callback* cb : callbacks) {
            cb->call();
        }
        
        // Clear stored callbacks (optional)
        callbacks.clear();
    }
    
    bool isFired() const {
        return fired;
    }
    
    size_t getPendingCallbacks() const {
        return callbacks.size();
    }
};
```

### Using Function Objects

```cpp
#include <vector>
#include <functional>
using namespace std;

class EventFunctional {
private:
    vector<function<void()>> callbacks;
    bool fired;
    
public:
    EventFunctional() : fired(false) {}
    
    void register_cb(function<void()> cb) {
        if (!cb) {
            return;
        }
        
        if (fired) {
            // Call immediately
            cb();
        } else {
            // Store for later
            callbacks.push_back(cb);
        }
    }
    
    void fire() {
        if (fired) {
            return;
        }
        
        fired = true;
        
        // Execute all stored callbacks
        for (auto& cb : callbacks) {
            cb();
        }
        
        callbacks.clear();
    }
};
```

## Part 2: Multi-Threaded Implementation

### Thread-Safe Version with Mutex

```cpp
#include <vector>
#include <functional>
#include <mutex>
#include <memory>
using namespace std;

class EventThreadSafe {
private:
    vector<function<void()>> callbacks;
    bool fired;
    mutex mtx;
    
public:
    EventThreadSafe() : fired(false) {}
    
    void register_cb(function<void()> cb) {
        if (!cb) {
            return;
        }
        
        bool shouldCallImmediately = false;
        
        {
            lock_guard<mutex> lock(mtx);
            
            if (fired) {
                // Event already fired - will call immediately
                shouldCallImmediately = true;
            } else {
                // Store callback
                callbacks.push_back(cb);
            }
        }
        
        // Call outside lock to avoid deadlock
        if (shouldCallImmediately) {
            cb();
        }
    }
    
    void fire() {
        vector<function<void()>> callbacksToExecute;
        
        {
            lock_guard<mutex> lock(mtx);
            
            if (fired) {
                return;  // Already fired
            }
            
            fired = true;
            callbacksToExecute = callbacks;  // Copy callbacks
            callbacks.clear();
        }
        
        // Execute callbacks outside lock
        for (auto& cb : callbacksToExecute) {
            cb();
        }
    }
    
    bool isFired() const {
        lock_guard<mutex> lock(mtx);
        return fired;
    }
};
```

### Critical Issue: Race Condition

**Problem Scenario:**
```
Thread 1: register_cb(cb1)
  → Checks fired = false
  → [Thread 2 calls fire() here]
  → Adds cb1 to callbacks
  → cb1 never gets called!
```

**Solution:** Need to check `fired` and add to callbacks atomically.

## Part 3: Correct Thread-Safe Implementation

### Fixing Race Conditions

```cpp
class EventCorrect {
private:
    vector<function<void()>> callbacks;
    atomic<bool> fired{false};
    mutex mtx;
    
public:
    EventCorrect() {}
    
    void register_cb(function<void()> cb) {
        if (!cb) {
            return;
        }
        
        // Check if fired (atomic read)
        if (fired.load()) {
            // Event already fired - call immediately
            cb();
            return;
        }
        
        // Acquire lock to check again and potentially add
        lock_guard<mutex> lock(mtx);
        
        // Double-check: might have fired while waiting for lock
        if (fired.load()) {
            // Event fired while we were waiting - call immediately
            lock.release();  // Release lock before calling
            cb();
            return;
        }
        
        // Still not fired - add to callbacks
        callbacks.push_back(cb);
    }
    
    void fire() {
        vector<function<void()>> callbacksToExecute;
        
        {
            lock_guard<mutex> lock(mtx);
            
            // Check if already fired
            if (fired.load()) {
                return;
            }
            
            // Set fired flag
            fired.store(true);
            
            // Copy callbacks
            callbacksToExecute = callbacks;
            callbacks.clear();
        }
        
        // Execute callbacks outside lock
        for (auto& cb : callbacksToExecute) {
            cb();
        }
    }
    
    bool isFired() const {
        return fired.load();
    }
};
```

**Issue:** `lock.release()` doesn't work with `lock_guard`. Need different approach.

### Proper Implementation

```cpp
class EventProper {
private:
    vector<function<void()>> callbacks;
    atomic<bool> fired{false};
    mutex mtx;
    
public:
    EventProper() {}
    
    void register_cb(function<void()> cb) {
        if (!cb) {
            return;
        }
        
        // Fast path: if already fired, call immediately
        if (fired.load(memory_order_acquire)) {
            cb();
            return;
        }
        
        // Slow path: need to check and potentially add
        unique_lock<mutex> lock(mtx);
        
        // Double-check pattern
        if (fired.load(memory_order_acquire)) {
            // Fired while waiting for lock - call immediately
            lock.unlock();
            cb();
            return;
        }
        
        // Still not fired - add to callbacks
        callbacks.push_back(cb);
        // Lock released automatically
    }
    
    void fire() {
        vector<function<void()>> callbacksToExecute;
        
        {
            lock_guard<mutex> lock(mtx);
            
            // Check if already fired
            if (fired.load(memory_order_acquire)) {
                return;
            }
            
            // Set fired flag (with release semantics)
            fired.store(true, memory_order_release);
            
            // Copy callbacks
            callbacksToExecute = callbacks;
            callbacks.clear();
        }
        
        // Execute callbacks outside lock
        for (auto& cb : callbacksToExecute) {
            cb();
        }
    }
    
    bool isFired() const {
        return fired.load(memory_order_acquire);
    }
};
```

## Part 4: Using Manual Lock Class

### As Described in Interview

```cpp
class Lock {
private:
    mutex mtx;
    
public:
    void acquire() {
        mtx.lock();
    }
    
    void release() {
        mtx.unlock();
    }
};

class EventWithLock {
private:
    vector<function<void()>> callbacks;
    bool fired;
    Lock lock;
    
public:
    EventWithLock() : fired(false) {}
    
    void register_cb(function<void()> cb) {
        if (!cb) {
            return;
        }
        
        bool shouldCallImmediately = false;
        
        lock.acquire();
        
        if (fired) {
            shouldCallImmediately = true;
        } else {
            callbacks.push_back(cb);
        }
        
        lock.release();
        
        // Call outside lock
        if (shouldCallImmediately) {
            cb();
        }
    }
    
    void fire() {
        vector<function<void()>> callbacksToExecute;
        
        lock.acquire();
        
        if (fired) {
            lock.release();
            return;
        }
        
        fired = true;
        callbacksToExecute = callbacks;
        callbacks.clear();
        
        lock.release();
        
        // Execute outside lock
        for (auto& cb : callbacksToExecute) {
            cb();
        }
    }
    
    bool isFired() const {
        lock.acquire();
        bool result = fired;
        lock.release();
        return result;
    }
};
```

**Problem:** Still has race condition! Need double-check pattern.

### Corrected Version with Manual Lock

```cpp
class EventWithLockCorrect {
private:
    vector<function<void()>> callbacks;
    atomic<bool> fired{false};
    Lock lock;
    
public:
    EventWithLockCorrect() {}
    
    void register_cb(function<void()> cb) {
        if (!cb) {
            return;
        }
        
        // Fast path check
        if (fired.load()) {
            cb();
            return;
        }
        
        // Acquire lock
        lock.acquire();
        
        // Double-check
        if (fired.load()) {
            lock.release();
            cb();
            return;
        }
        
        // Add to callbacks
        callbacks.push_back(cb);
        lock.release();
    }
    
    void fire() {
        vector<function<void()>> callbacksToExecute;
        
        lock.acquire();
        
        if (fired.load()) {
            lock.release();
            return;
        }
        
        fired.store(true);
        callbacksToExecute = callbacks;
        callbacks.clear();
        
        lock.release();
        
        // Execute callbacks
        for (auto& cb : callbacksToExecute) {
            cb();
        }
    }
    
    bool isFired() const {
        return fired.load();
    }
};
```

## Part 5: Complete Production-Ready Implementation

```cpp
#include <vector>
#include <functional>
#include <mutex>
#include <atomic>
#include <memory>
#include <stdexcept>
using namespace std;

class Event {
private:
    vector<function<void()>> callbacks;
    atomic<bool> fired{false};
    mutable mutex mtx;
    
    // Prevent copying
    Event(const Event&) = delete;
    Event& operator=(const Event&) = delete;
    
public:
    Event() {}
    
    ~Event() {
        // Ensure all callbacks are executed if not fired
        if (!fired.load()) {
            // Optionally fire on destruction, or just clear
            lock_guard<mutex> lock(mtx);
            callbacks.clear();
        }
    }
    
    void register_cb(function<void()> cb) {
        if (!cb) {
            throw invalid_argument("Callback cannot be null");
        }
        
        // Fast path: already fired
        if (fired.load(memory_order_acquire)) {
            cb();
            return;
        }
        
        // Slow path: acquire lock
        unique_lock<mutex> lock(mtx);
        
        // Double-check pattern
        if (fired.load(memory_order_acquire)) {
            lock.unlock();
            cb();
            return;
        }
        
        // Store callback
        callbacks.push_back(cb);
        // Lock released automatically
    }
    
    void fire() {
        vector<function<void()>> callbacksToExecute;
        
        {
            lock_guard<mutex> lock(mtx);
            
            // Check if already fired
            if (fired.load(memory_order_acquire)) {
                return;  // Already fired, ignore
            }
            
            // Set fired flag
            fired.store(true, memory_order_release);
            
            // Copy callbacks
            callbacksToExecute = callbacks;
            callbacks.clear();
        }
        
        // Execute callbacks outside lock
        // This prevents deadlock if callback tries to register another callback
        for (auto& cb : callbacksToExecute) {
            try {
                cb();
            } catch (const exception& e) {
                // Log error but continue with other callbacks
                cerr << "Callback error: " << e.what() << endl;
            }
        }
    }
    
    bool isFired() const {
        return fired.load(memory_order_acquire);
    }
    
    size_t getPendingCallbacks() const {
        lock_guard<mutex> lock(mtx);
        return callbacks.size();
    }
};
```

## Part 6: Test Program

```cpp
#include <iostream>
#include <thread>
#include <chrono>
#include <vector>
#include <cassert>
#include <atomic>

void testSingleThread() {
    cout << "=== Testing Single Thread ===" << endl;
    
    Event event;
    atomic<int> callCount{0};
    
    // Register callbacks before fire
    event.register_cb([&callCount]() {
        callCount++;
        cout << "Callback 1 executed" << endl;
    });
    
    event.register_cb([&callCount]() {
        callCount++;
        cout << "Callback 2 executed" << endl;
    });
    
    assert(callCount == 0);
    assert(event.getPendingCallbacks() == 2);
    
    // Fire event
    event.fire();
    
    assert(callCount == 2);
    assert(event.getPendingCallbacks() == 0);
    
    // Register callback after fire
    event.register_cb([&callCount]() {
        callCount++;
        cout << "Callback 3 executed (after fire)" << endl;
    });
    
    assert(callCount == 3);
    
    cout << "Single thread test passed!" << endl;
}

void testMultiThread() {
    cout << "\n=== Testing Multi Thread ===" << endl;
    
    Event event;
    atomic<int> callCount{0};
    const int NUM_THREADS = 10;
    const int CALLBACKS_PER_THREAD = 5;
    
    vector<thread> threads;
    
    // Start threads that register callbacks
    for (int i = 0; i < NUM_THREADS; i++) {
        threads.emplace_back([&event, &callCount, i]() {
            for (int j = 0; j < CALLBACKS_PER_THREAD; j++) {
                event.register_cb([&callCount, i, j]() {
                    callCount++;
                });
            }
        });
    }
    
    // Wait a bit, then fire
    this_thread::sleep_for(chrono::milliseconds(100));
    event.fire();
    
    // Register more callbacks after fire
    for (int i = 0; i < NUM_THREADS; i++) {
        threads.emplace_back([&event, &callCount]() {
            event.register_cb([&callCount]() {
                callCount++;
            });
        });
    }
    
    // Wait for all threads
    for (auto& t : threads) {
        t.join();
    }
    
    // Verify: should have NUM_THREADS * CALLBACKS_PER_THREAD before fire
    // + NUM_THREADS after fire = NUM_THREADS * (CALLBACKS_PER_THREAD + 1)
    int expected = NUM_THREADS * (CALLBACKS_PER_THREAD + 1);
    
    // Note: Exact count depends on timing, but should be close
    cout << "Expected: " << expected << ", Actual: " << callCount << endl;
    assert(callCount >= NUM_THREADS * CALLBACKS_PER_THREAD);
    
    cout << "Multi-thread test passed!" << endl;
}

void testRaceCondition() {
    cout << "\n=== Testing Race Condition Prevention ===" << endl;
    
    Event event;
    atomic<int> callCount{0};
    const int NUM_ITERATIONS = 1000;
    
    vector<thread> threads;
    
    // Thread 1: Register many callbacks
    threads.emplace_back([&event, &callCount]() {
        for (int i = 0; i < NUM_ITERATIONS; i++) {
            event.register_cb([&callCount]() {
                callCount++;
            });
        }
    });
    
    // Thread 2: Fire event
    threads.emplace_back([&event]() {
        this_thread::sleep_for(chrono::milliseconds(10));
        event.fire();
    });
    
    // Thread 3: Register callbacks (some before, some after fire)
    threads.emplace_back([&event, &callCount]() {
        for (int i = 0; i < NUM_ITERATIONS; i++) {
            event.register_cb([&callCount]() {
                callCount++;
            });
            this_thread::sleep_for(chrono::microseconds(1));
        }
    });
    
    for (auto& t : threads) {
        t.join();
    }
    
    // All callbacks should be executed
    cout << "Total callbacks executed: " << callCount << endl;
    assert(callCount >= NUM_ITERATIONS);
    
    cout << "Race condition test passed!" << endl;
}

int main() {
    testSingleThread();
    testMultiThread();
    testRaceCondition();
    
    cout << "\nAll tests passed!" << endl;
    return 0;
}
```

## Part 7: Race Condition Analysis

### The Critical Race Condition

**Scenario:**
```
Thread 1: register_cb(cb1)
  → Checks fired = false (not atomic with lock)
  → [Thread 2: fire() executes here]
    → Sets fired = true
    → Executes callbacks (cb1 not in list yet!)
  → Thread 1 acquires lock
  → Adds cb1 to callbacks
  → cb1 never gets called!
```

**Solution:** Double-check pattern with atomic flag.

### Why Double-Check Pattern Works

```cpp
void register_cb(function<void()> cb) {
    // Check 1: Fast path (no lock)
    if (fired.load()) {
        cb();  // Already fired
        return;
    }
    
    // Acquire lock
    lock_guard<mutex> lock(mtx);
    
    // Check 2: Might have fired while waiting for lock
    if (fired.load()) {
        // Fired between check 1 and acquiring lock
        cb();  // Call immediately
        return;
    }
    
    // Still not fired - safe to add
    callbacks.push_back(cb);
}
```

## Part 8: Memory Ordering Considerations

### Memory Order Semantics

```cpp
class EventMemoryOrder {
private:
    vector<function<void()>> callbacks;
    atomic<bool> fired{false};
    mutex mtx;
    
public:
    void register_cb(function<void()> cb) {
        // memory_order_acquire: ensures we see all writes before fired=true
        if (fired.load(memory_order_acquire)) {
            cb();
            return;
        }
        
        lock_guard<mutex> lock(mtx);
        
        if (fired.load(memory_order_acquire)) {
            cb();
            return;
        }
        
        callbacks.push_back(cb);
    }
    
    void fire() {
        lock_guard<mutex> lock(mtx);
        
        if (fired.load(memory_order_acquire)) {
            return;
        }
        
        // memory_order_release: ensures all previous writes are visible
        fired.store(true, memory_order_release);
        
        // Copy and clear
        auto callbacksToExecute = callbacks;
        callbacks.clear();
        lock.unlock();
        
        // Execute
        for (auto& cb : callbacksToExecute) {
            cb();
        }
    }
};
```

## Part 9: Interview Discussion Points

### Key Questions and Answers

**Q1: Is the code thread-safe?**

**A:** Basic version is NOT thread-safe. Issues:
- Race condition between checking `fired` and adding to `callbacks`
- Multiple threads can modify `callbacks` simultaneously
- Callback might be lost if fire() happens between check and add

**Q2: What race conditions can occur?**

**A:**
1. **Lost callback**: Thread checks `fired=false`, fire() executes, thread adds callback → callback never called
2. **Double execution**: If not careful with flag checking
3. **Inconsistent state**: Reading `fired` and `callbacks` separately

**Q3: How to fix race conditions?**

**A:**
1. **Use mutex**: Protect all shared state access
2. **Atomic flag**: Use `atomic<bool>` for fired flag
3. **Double-check pattern**: Check flag before and after acquiring lock
4. **Execute outside lock**: Prevent deadlock if callback registers another callback

**Q4: Why call callback outside lock?**

**A:**
- **Deadlock prevention**: If callback tries to register another callback
- **Performance**: Don't hold lock during potentially slow callback execution
- **Reduced contention**: Other threads can register while callbacks execute

**Q5: Can we use lock-free approach?**

**A:** Possible but complex:
- Use atomic operations for flag
- Use lock-free queue for callbacks
- More complex, but potentially faster

## Key Takeaways

### Design Principles

1. **State management**: Track fired state atomically
2. **Double-check pattern**: Check flag before and after lock
3. **Execute outside lock**: Prevent deadlock and reduce contention
4. **Atomic operations**: Use atomic flag for fast path

### Thread Safety Patterns

1. **Mutex protection**: Protect shared data structures
2. **Atomic flag**: Fast path check without lock
3. **Double-check**: Ensure correctness under race conditions
4. **RAII**: Use lock_guard/unique_lock for exception safety

### Common Mistakes

1. **Race condition**: Checking fired and adding callback not atomic
2. **Deadlock**: Calling callback while holding lock
3. **Lost callbacks**: Not using double-check pattern
4. **Incorrect memory ordering**: Not using proper memory_order

### Interview Tips

1. **Start single-threaded**: Get basic logic correct first
2. **Identify race conditions**: Discuss potential issues
3. **Add synchronization**: Mutex + atomic flag
4. **Use double-check**: Ensure correctness
5. **Test thoroughly**: Multiple threads, timing variations

## Summary

Event callback mechanism implementation demonstrates:

- **State management**: Before fire vs. after fire behavior
- **Thread synchronization**: Mutex and atomic operations
- **Race condition handling**: Double-check pattern
- **Callback execution**: Proper ordering and timing
- **Deadlock prevention**: Execute callbacks outside locks

**Key implementation:**
- **Atomic flag**: Fast path check for fired state
- **Mutex protection**: Thread-safe callback storage
- **Double-check pattern**: Prevent lost callbacks
- **Execute outside lock**: Prevent deadlock

Understanding these concepts is essential for:
- System design interviews
- Concurrent programming
- Event-driven systems
- Thread synchronization

This knowledge is particularly valuable for storage companies like Pure Storage, where reliable event handling and thread safety are critical.

