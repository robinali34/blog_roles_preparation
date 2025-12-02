---
layout: post
title: "Pure Storage Interview: Event Fire/Register Callbacks (Multi-threading)"
date: 2025-11-16
categories: [Interview Preparation, Pure Storage, Technical Interview, Multi-threading]
excerpt: "Complete solution for Event Fire/Register Callbacks multi-threading problem. Covers thread safety, deadlock prevention, race conditions, and various implementation approaches with detailed analysis."
---

## Problem Overview

**Pure Storage Interview: Event Fire/Register Callbacks**

This is a challenging multi-threading problem that tests your understanding of:
1. Thread synchronization and locks
2. Deadlock prevention
3. Race condition handling
4. Lock ordering and critical sections
5. Recursive callback scenarios

### Problem Statement

Implement an `EventFire` class with two methods:
1. **`register_cb(callback)`**: Register a callback
   - Before `fire()` is called: Store callbacks in a queue
   - After `fire()` is called: Execute callbacks immediately
2. **`fire()`**: Fire all registered events
   - Execute all previously registered callbacks
   - Set a flag so future `register_cb()` calls execute immediately
   - **Important**: `fire()` is called **only once** across all threads

### Requirements

- **Thread-safe**: Multiple threads can call `register_cb()` concurrently
- **Deadlock-free**: Callbacks might recursively call `register_cb()` or `fire()`
- **Correctness**: No callback should be lost or executed twice
- **Order preservation**: (Optional follow-up) Maintain registration order

## Part 1: Single-Threaded Solution

### Basic Implementation

```cpp
#include <queue>
#include <functional>
#include <iostream>
#include <mutex>
#include <thread>
#include <vector>
using namespace std;

class EventFire {
private:
    queue<function<void()>> eventQueue;
    bool isFired = false;
    
public:
    void register_cb(function<void()> cb) {
        if (!isFired) {
            eventQueue.push(cb);
        } else {
            cb();  // Execute immediately if already fired
        }
    }
    
    void fire() {
        // Execute all registered callbacks
        while (!eventQueue.empty()) {
            auto cb = eventQueue.front();
            eventQueue.pop();
            cb();
        }
        
        isFired = true;
    }
};
```

### Test Case

```cpp
void testSingleThread() {
    EventFire ef;
    
    ef.register_cb([]() { cout << "Callback 1" << endl; });
    ef.register_cb([]() { cout << "Callback 2" << endl; });
    ef.register_cb([]() { cout << "Callback 3" << endl; });
    
    ef.fire();  // Executes callbacks 1, 2, 3
    
    ef.register_cb([]() { cout << "Callback 4" << endl; });  // Executes immediately
    ef.register_cb([]() { cout << "Callback 5" << endl; });  // Executes immediately
}
```

## Part 2: Multi-Threaded Problems

### Race Conditions

**Problem 1: Lost Callbacks**
```cpp
// Thread 1: register_cb(cb1)
if (!isFired) {  // Check: isFired = false
    // [Thread 2 calls fire() here]
    eventQueue.push(cb1);  // cb1 might be lost!
}

// Thread 2: fire()
while (!eventQueue.isEmpty()) {
    // Process queue
}
isFired = true;  // Set flag
```

**Problem 2: Double Execution**
```cpp
// Thread 1: fire()
while (!eventQueue.isEmpty()) {
    auto cb = eventQueue.front();
    // [Thread 2 calls register_cb() here, sees isFired=false]
    eventQueue.pop();
    cb();  // Execute
}
isFired = true;  // Too late!
```

**Problem 3: Deadlock (Recursive Callbacks)**
```cpp
void fire() {
    lock.lock();
    while (!eventQueue.isEmpty()) {
        auto cb = eventQueue.front();
        eventQueue.pop();
        cb();  // If cb() calls register_cb(), deadlock!
    }
    lock.unlock();
}
```

## Part 3: Thread-Safe Solution (Version 2 - Correct)

### Key Insights

1. **Unlock before calling callbacks**: Prevents deadlock from recursive calls
2. **Set `isFired` early**: Prevents new registrations from being queued
3. **Re-acquire lock**: Need to check queue again after unlocking

### Implementation

```cpp
#include <mutex>
#include <queue>
#include <functional>
#include <iostream>
using namespace std;

class EventFire {
private:
    queue<function<void()>> eventQueue;
    bool isFired = false;
    mutex mtx;
    
public:
    void register_cb(function<void()> cb) {
        lock_guard<mutex> lock(mtx);
        
        if (!isFired) {
            eventQueue.push(cb);
            // Lock automatically released here
        } else {
            // Unlock before calling callback to prevent deadlock
            mtx.unlock();
            cb();
            mtx.lock();
        }
    }
    
    void fire() {
        lock_guard<mutex> lock(mtx);
        
        // Set flag FIRST to prevent new registrations
        isFired = true;
        
        // Process queue: unlock before each callback
        while (!eventQueue.empty()) {
            auto cb = eventQueue.front();
            eventQueue.pop();
            
            // CRITICAL: Unlock before calling callback
            mtx.unlock();
            cb();  // Callback might recursively call register_cb()
            mtx.lock();
        }
        
        // Lock automatically released here
    }
};
```

### Why This Works

1. **`isFired = true` first**: Prevents race condition where new callbacks are registered after `fire()` starts
2. **Unlock before `cb()`**: Prevents deadlock if callback recursively calls `register_cb()` or `fire()`
3. **Re-acquire lock**: Ensures thread-safe queue access

### Using Manual Lock/Unlock (More Explicit)

```cpp
class EventFire {
private:
    queue<function<void()>> eventQueue;
    bool isFired = false;
    mutex mtx;
    
public:
    void register_cb(function<void()> cb) {
        mtx.lock();
        
        if (!isFired) {
            eventQueue.push(cb);
            mtx.unlock();
        } else {
            mtx.unlock();
            cb();  // Execute outside lock
        }
    }
    
    void fire() {
        mtx.lock();
        
        // Set flag FIRST
        isFired = true;
        
        // Process queue
        while (!eventQueue.empty()) {
            auto cb = eventQueue.front();
            eventQueue.pop();
            
            // Unlock before callback
            mtx.unlock();
            cb();
            mtx.lock();
        }
        
        mtx.unlock();
    }
};
```

## Part 4: Common Mistakes and Analysis

### Version 3: Intern's "Optimization" (WRONG)

```cpp
void fire() {
    mtx.lock();
    isFired = true;
    mtx.unlock();  // Unlock too early!
    
    while (!eventQueue.empty()) {  // Race condition!
        auto cb = eventQueue.front();
        eventQueue.pop();
        cb();
    }
}
```

**Problems:**
1. **Race condition**: Between `unlock()` and checking `eventQueue.empty()`, another thread might modify the queue
2. **Lost callbacks**: Thread might register callback after `isFired=true` but before queue processing
3. **Double execution**: Callback might be executed twice if queue is modified concurrently

**Scenario:**
```
Thread 1 (fire): lock() -> isFired=true -> unlock()
Thread 2 (register): lock() -> sees isFired=true -> unlock() -> cb() executes
Thread 1 (fire): checks queue.empty() -> processes queue -> cb() executes again!
```

### Version 4: Unlock Before Setting Flag (WRONG)

```cpp
void fire() {
    mtx.lock();
    mtx.unlock();  // No protection!
    isFired = true;  // Race condition!
    
    while (!eventQueue.empty()) {
        auto cb = eventQueue.front();
        eventQueue.pop();
        cb();
    }
}
```

**Problems:**
1. **No synchronization**: `isFired` is set without lock protection
2. **Race condition**: Threads might see inconsistent state
3. **Lost callbacks**: Callbacks registered between unlock and setting flag might be lost

### Version 5: Barrier Pattern (CORRECT)

```cpp
void fire() {
    isFired = true;  // Set flag first (barrier)
    
    mtx.lock();  // Acquire lock
    mtx.unlock();  // Release immediately (barrier check)
    
    while (!eventQueue.empty()) {
        auto cb = eventQueue.front();
        eventQueue.pop();
        cb();
    }
}
```

**Why This Works:**
- **Barrier pattern**: Setting `isFired = true` before acquiring lock creates a barrier
- **Threads wait**: If `register_cb()` holds the lock, `fire()` will wait
- **No race**: Once lock is acquired (even briefly), we know no other thread is modifying state
- **Correctness**: All callbacks registered before `isFired=true` are guaranteed to be in queue

**However**, this version still has issues:
- Queue access is not protected during processing
- Better to use Version 2 with proper lock management

## Part 5: Complete Thread-Safe Solution

### Final Correct Implementation

```cpp
#include <mutex>
#include <queue>
#include <functional>
#include <memory>
#include <atomic>
using namespace std;

class EventFire {
private:
    queue<function<void()>> eventQueue;
    atomic<bool> isFired{false};  // Atomic for read optimization
    mutex mtx;
    
public:
    void register_cb(function<void()> cb) {
        // Fast path: if already fired, execute immediately
        if (isFired.load(memory_order_acquire)) {
            cb();
            return;
        }
        
        // Slow path: need to check and possibly queue
        lock_guard<mutex> lock(mtx);
        
        // Double-check: might have been fired while waiting for lock
        if (isFired.load(memory_order_acquire)) {
            // Unlock before callback
            mtx.unlock();
            cb();
            return;
        }
        
        eventQueue.push(cb);
        // Lock released automatically
    }
    
    void fire() {
        // Set flag first (barrier)
        isFired.store(true, memory_order_release);
        
        // Process queue with proper lock management
        unique_lock<mutex> lock(mtx);
        
        while (!eventQueue.empty()) {
            auto cb = eventQueue.front();
            eventQueue.pop();
            
            // Unlock before callback to prevent deadlock
            lock.unlock();
            cb();
            lock.lock();
        }
        
        // Lock released automatically
    }
};
```

### Simplified Version (Easier to Understand)

```cpp
class EventFire {
private:
    queue<function<void()>> eventQueue;
    bool isFired = false;
    mutex mtx;
    
public:
    void register_cb(function<void()> cb) {
        unique_lock<mutex> lock(mtx);
        
        if (!isFired) {
            eventQueue.push(cb);
            lock.unlock();
        } else {
            lock.unlock();
            cb();  // Execute outside lock
        }
    }
    
    void fire() {
        unique_lock<mutex> lock(mtx);
        
        // Set flag FIRST
        isFired = true;
        
        // Process all queued callbacks
        while (!eventQueue.empty()) {
            auto cb = eventQueue.front();
            eventQueue.pop();
            
            // CRITICAL: Unlock before callback
            lock.unlock();
            cb();  // Callback might be recursive
            lock.lock();
        }
        
        // Lock released automatically
    }
};
```

## Part 6: Walkthrough of Critical Scenarios

### Scenario 1: Normal Flow

```
Thread 1: register_cb(cb1)
  -> lock() -> queue.push(cb1) -> unlock()
  
Thread 2: register_cb(cb2)
  -> lock() -> queue.push(cb2) -> unlock()
  
Thread 3: fire()
  -> lock() -> isFired=true -> process queue
    -> unlock() -> cb1() -> lock()
    -> unlock() -> cb2() -> lock()
  -> unlock()
```

### Scenario 2: Race Condition Prevention

```
Thread 1: fire() starts
  -> lock() -> isFired=true
  
Thread 2: register_cb(cb3) arrives
  -> waits for lock...
  
Thread 1: processes queue -> unlock() -> cb1()
  
Thread 2: acquires lock -> sees isFired=true
  -> unlock() -> cb3() executes immediately
  
Thread 1: lock() -> continues processing
```

### Scenario 3: Recursive Callback (Deadlock Prevention)

```cpp
void recursiveCallback() {
    EventFire* ef = getInstance();
    ef->register_cb([]() { cout << "Nested callback" << endl; });
}

void fire() {
    lock();
    isFired = true;
    while (!queue.empty()) {
        auto cb = queue.front();
        queue.pop();
        unlock();  // CRITICAL: unlock before callback
        cb();  // This might call register_cb() recursively
        lock();  // Re-acquire lock
    }
    unlock();
}
```

**Without unlock before callback**: Deadlock!
**With unlock before callback**: Works correctly!

## Part 7: Follow-up: Order Preservation

### Problem

If multiple threads register callbacks concurrently, we might want to preserve registration order.

### Solution: Two-Queue Approach

```cpp
class EventFireOrdered {
private:
    queue<function<void()>> queue1;
    queue<function<void()>> queue2;
    queue<function<void()>>* activeQueue = &queue1;
    bool isFired = false;
    mutex mtx;
    
public:
    void register_cb(function<void()> cb) {
        lock_guard<mutex> lock(mtx);
        
        if (!isFired) {
            activeQueue->push(cb);
        } else {
            mtx.unlock();
            cb();
            mtx.lock();
        }
    }
    
    void fire() {
        unique_lock<mutex> lock(mtx);
        
        isFired = true;
        
        // Swap queues atomically
        queue<function<void()>>* processingQueue = activeQueue;
        activeQueue = (activeQueue == &queue1) ? &queue2 : &queue1;
        
        // Process the queue we swapped out
        while (!processingQueue->empty()) {
            auto cb = processingQueue->front();
            processingQueue->pop();
            
            lock.unlock();
            cb();
            lock.lock();
        }
        
        lock.unlock();
    }
};
```

## Part 8: Lock-Free Approach (Advanced)

### Using Atomic Operations

```cpp
#include <atomic>
#include <vector>
#include <memory>

class EventFireLockFree {
private:
    vector<function<void()>> callbacks;
    atomic<bool> isFired{false};
    atomic<int> registrationPhase{0};
    
public:
    void register_cb(function<void()> cb) {
        // Fast path
        if (isFired.load()) {
            cb();
            return;
        }
        
        // Use atomic operations for lock-free registration
        // (Simplified - full lock-free implementation is complex)
        // In practice, might still need some synchronization
    }
};
```

**Note**: Full lock-free implementation is complex and may not be necessary for most use cases.

## Part 9: Complete Test Program

```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <chrono>

void testMultiThread() {
    EventFire ef;
    vector<thread> threads;
    
    // Register callbacks from multiple threads
    for (int i = 0; i < 5; i++) {
        threads.emplace_back([&ef, i]() {
            ef.register_cb([i]() {
                cout << "Callback " << i << " from thread " 
                     << this_thread::get_id() << endl;
            });
        });
    }
    
    // Wait for registrations
    for (auto& t : threads) {
        t.join();
    }
    
    // Fire events
    thread fireThread([&ef]() {
        ef.fire();
    });
    
    // Register more callbacks after fire
    threads.clear();
    for (int i = 5; i < 10; i++) {
        threads.emplace_back([&ef, i]() {
            ef.register_cb([i]() {
                cout << "Immediate callback " << i << endl;
            });
        });
    }
    
    fireThread.join();
    for (auto& t : threads) {
        t.join();
    }
}

int main() {
    testMultiThread();
    return 0;
}
```

## Key Takeaways

### Critical Points

1. **Set `isFired = true` FIRST**: Prevents race conditions
2. **Unlock before callbacks**: Prevents deadlock from recursive calls
3. **Re-acquire lock**: Ensure thread-safe queue access
4. **Double-check pattern**: Check `isFired` again after acquiring lock

### Common Mistakes

1. ❌ Unlocking before setting `isFired`
2. ❌ Holding lock during callback execution
3. ❌ Not re-acquiring lock after callback
4. ❌ Processing queue without lock protection

### Interview Tips

1. **Start simple**: Single-threaded solution first
2. **Identify problems**: Discuss race conditions and deadlocks
3. **Explain each lock**: Why it's needed, when to release
4. **Walk through scenarios**: Show understanding of edge cases
5. **Consider optimizations**: Fast path, atomic operations, etc.

## Summary

This problem tests deep understanding of:
- **Thread synchronization**: Proper lock usage
- **Deadlock prevention**: Unlocking before external calls
- **Race conditions**: Order of operations matters
- **Critical sections**: Minimizing lock hold time
- **Recursive scenarios**: Callbacks calling back into the system

The key insight is: **Always unlock before calling external code (callbacks) to prevent deadlock**, and **set flags early to prevent race conditions**.

