---
layout: post
title: "Pure Storage Interview: MultiWorker Multi-threading Implementation"
date: 2025-11-16
categories: [Interview Preparation, Pure Storage, Technical Interview, Multi-threading]
excerpt: "Complete MultiWorker implementation: non-blocking multi-threaded work execution with callback completion. Includes thread-safe queue management, mutex synchronization, and proper callback handling."
---

## Problem Overview

**Pure Storage Interview: MultiWorker Multi-threading**

This problem tests your ability to:
1. Design multi-threaded systems
2. Implement non-blocking operations
3. Manage thread synchronization
4. Handle callbacks and completion tracking
5. Use thread-safe data structures

### Problem Statement

Given a `Worker` class with:
- `Worker(function<void()> work)`: Constructor takes a work function
- `void start(function<void()> callback)`: Starts work in background, calls callback when done

Implement a `MultiWorker` class that:
- Manages multiple `Worker` instances
- Has `start(function<void()> callback)` that is **non-blocking**
- Kicks off all work in background
- Calls callback when **all work is complete**

### Requirements

- **Non-blocking**: `start()` must return immediately
- **Background execution**: All work runs concurrently
- **Completion tracking**: Know when all work is done
- **Thread-safe**: Handle concurrent callback execution

## Part 1: Worker Class (Given)

### Base Worker Implementation

```cpp
#include <functional>
#include <thread>
#include <iostream>
using namespace std;

class Worker {
private:
    function<void()> work;
    
public:
    Worker(function<void()> w) : work(w) {}
    
    void start(function<void()> callback) {
        // Start work in a separate thread
        thread t([this, callback]() {
            // Execute the work
            work();
            
            // Call callback when done
            callback();
        });
        
        t.detach();  // Detach thread (non-blocking)
    }
};
```

**Note:** In real implementation, Worker might join threads or use thread pool. For this problem, we assume Worker handles its own threading.

## Part 2: Basic MultiWorker Implementation

### First Attempt (Incorrect)

```cpp
class MultiWorker {
private:
    vector<Worker> workers;
    
public:
    MultiWorker(vector<function<void()>> workFunctions) {
        for (auto& work : workFunctions) {
            workers.push_back(Worker(work));
        }
    }
    
    void start(function<void()> callback) {
        // Problem: How to know when all are done?
        for (auto& worker : workers) {
            worker.start([]() {
                // Individual callback - but we need to know when ALL are done
            });
        }
        // This returns immediately, but callback is never called!
    }
};
```

**Problem:** No way to track when all workers complete.

## Part 3: Thread-Safe Completion Tracking

### Using Mutex and Counter

```cpp
#include <mutex>
#include <atomic>
#include <vector>
#include <functional>
#include <thread>
using namespace std;

class MultiWorker {
private:
    vector<Worker> workers;
    mutex mtx;
    atomic<int> completedCount{0};
    int totalWorkers;
    function<void()> finalCallback;
    
    void onWorkerComplete() {
        lock_guard<mutex> lock(mtx);
        completedCount++;
        
        // Check if all workers are done
        if (completedCount == totalWorkers) {
            // All work complete - call final callback
            if (finalCallback) {
                finalCallback();
            }
        }
    }
    
public:
    MultiWorker(vector<function<void()>> workFunctions) {
        for (auto& work : workFunctions) {
            workers.push_back(Worker(work));
        }
        totalWorkers = workers.size();
    }
    
    void start(function<void()> callback) {
        finalCallback = callback;
        completedCount = 0;  // Reset counter
        
        // Start all workers with completion callback
        for (auto& worker : workers) {
            worker.start([this]() {
                this->onWorkerComplete();
            });
        }
        
        // Return immediately (non-blocking)
    }
};
```

**Issue:** `onWorkerComplete()` is called from worker threads, needs to be thread-safe.

## Part 4: Complete Thread-Safe Implementation

### Production-Ready Version

```cpp
#include <vector>
#include <functional>
#include <mutex>
#include <atomic>
#include <memory>
#include <iostream>
using namespace std;

class Worker {
private:
    function<void()> work;
    
public:
    Worker(function<void()> w) : work(w) {}
    
    void start(function<void()> callback) {
        thread t([this, callback]() {
            try {
                work();
            } catch (const exception& e) {
                cerr << "Worker error: " << e.what() << endl;
            }
            callback();
        });
        t.detach();
    }
};

class MultiWorker {
private:
    vector<Worker> workers;
    mutex mtx;
    atomic<int> completedCount{0};
    int totalWorkers;
    function<void()> finalCallback;
    bool started;
    
    void onWorkerComplete() {
        lock_guard<mutex> lock(mtx);
        int current = ++completedCount;
        
        // Check if all workers are done
        if (current == totalWorkers && finalCallback) {
            finalCallback();
        }
    }
    
public:
    MultiWorker(vector<function<void()>> workFunctions) 
        : totalWorkers(workFunctions.size()), started(false) {
        for (auto& work : workFunctions) {
            workers.emplace_back(work);
        }
    }
    
    void start(function<void()> callback) {
        lock_guard<mutex> lock(mtx);
        
        if (started) {
            throw runtime_error("MultiWorker already started");
        }
        
        finalCallback = callback;
        completedCount = 0;
        started = true;
        
        // Start all workers
        for (auto& worker : workers) {
            worker.start([this]() {
                this->onWorkerComplete();
            });
        }
        
        // Return immediately (non-blocking)
    }
    
    // Reset for reuse (optional)
    void reset() {
        lock_guard<mutex> lock(mtx);
        started = false;
        completedCount = 0;
        finalCallback = nullptr;
    }
    
    int getTotalWorkers() const {
        return totalWorkers;
    }
    
    int getCompletedCount() const {
        return completedCount.load();
    }
};
```

## Part 5: Alternative Implementation with Queue

### Queue-Based Approach (As Described)

```cpp
#include <queue>
#include <mutex>
#include <atomic>
#include <vector>
#include <functional>
#include <thread>
using namespace std;

class MultiWorkerQueue {
private:
    queue<Worker> workerQueue;
    mutex queueMtx;
    atomic<int> activeWorkers{0};
    function<void()> finalCallback;
    int totalWorkers;
    
    void onWorkerComplete() {
        lock_guard<mutex> lock(queueMtx);
        
        // Decrement active workers
        activeWorkers--;
        
        // Check if queue is empty (all work done)
        if (activeWorkers == 0 && workerQueue.empty()) {
            // All work complete
            if (finalCallback) {
                finalCallback();
            }
        }
    }
    
public:
    MultiWorkerQueue(vector<function<void()>> workFunctions) 
        : totalWorkers(workFunctions.size()) {
        for (auto& work : workFunctions) {
            workerQueue.push(Worker(work));
        }
    }
    
    void start(function<void()> callback) {
        lock_guard<mutex> lock(queueMtx);
        
        finalCallback = callback;
        activeWorkers = totalWorkers;
        
        // Execute all workers
        while (!workerQueue.empty()) {
            Worker worker = workerQueue.front();
            workerQueue.pop();
            
            // Start worker with completion callback
            worker.start([this]() {
                this->onWorkerComplete();
            });
        }
        
        // Return immediately (non-blocking)
    }
};
```

## Part 6: Enhanced Implementation with Error Handling

### Complete Robust Version

```cpp
#include <vector>
#include <functional>
#include <mutex>
#include <atomic>
#include <exception>
#include <stdexcept>
#include <iostream>
using namespace std;

class Worker {
private:
    function<void()> work;
    
public:
    Worker(function<void()> w) : work(w) {
        if (!w) {
            throw invalid_argument("Work function cannot be null");
        }
    }
    
    void start(function<void()> callback) {
        if (!callback) {
            throw invalid_argument("Callback function cannot be null");
        }
        
        thread t([this, callback]() {
            try {
                work();
            } catch (const exception& e) {
                cerr << "Worker execution error: " << e.what() << endl;
            } catch (...) {
                cerr << "Worker execution error: Unknown exception" << endl;
            }
            
            // Always call callback, even on error
            try {
                callback();
            } catch (const exception& e) {
                cerr << "Callback error: " << e.what() << endl;
            }
        });
        
        t.detach();
    }
};

class MultiWorker {
private:
    vector<Worker> workers;
    mutex mtx;
    atomic<int> completedCount{0};
    atomic<int> errorCount{0};
    int totalWorkers;
    function<void()> finalCallback;
    bool started;
    bool callbackCalled;
    
    void onWorkerComplete(bool hadError = false) {
        lock_guard<mutex> lock(mtx);
        
        if (hadError) {
            errorCount++;
        }
        
        int current = ++completedCount;
        
        // Check if all workers are done
        if (current == totalWorkers && !callbackCalled) {
            callbackCalled = true;
            if (finalCallback) {
                // Call callback in separate thread to avoid blocking
                thread t([this]() {
                    try {
                        finalCallback();
                    } catch (const exception& e) {
                        cerr << "Final callback error: " << e.what() << endl;
                    }
                });
                t.detach();
            }
        }
    }
    
public:
    MultiWorker(vector<function<void()>> workFunctions) 
        : totalWorkers(workFunctions.size()), started(false), callbackCalled(false) {
        if (workFunctions.empty()) {
            throw invalid_argument("Work functions cannot be empty");
        }
        
        for (auto& work : workFunctions) {
            workers.emplace_back(work);
        }
    }
    
    void start(function<void()> callback) {
        if (!callback) {
            throw invalid_argument("Callback cannot be null");
        }
        
        lock_guard<mutex> lock(mtx);
        
        if (started) {
            throw runtime_error("MultiWorker already started");
        }
        
        finalCallback = callback;
        completedCount = 0;
        errorCount = 0;
        callbackCalled = false;
        started = true;
        
        // Start all workers
        for (auto& worker : workers) {
            worker.start([this]() {
                this->onWorkerComplete();
            });
        }
        
        // Return immediately (non-blocking)
    }
    
    // Wait for completion (blocking version - optional)
    void waitForCompletion() {
        while (completedCount < totalWorkers) {
            this_thread::sleep_for(chrono::milliseconds(10));
        }
    }
    
    // Get statistics
    int getCompletedCount() const {
        return completedCount.load();
    }
    
    int getErrorCount() const {
        return errorCount.load();
    }
    
    bool isComplete() const {
        return completedCount.load() == totalWorkers;
    }
    
    // Reset for reuse
    void reset() {
        lock_guard<mutex> lock(mtx);
        started = false;
        callbackCalled = false;
        completedCount = 0;
        errorCount = 0;
        finalCallback = nullptr;
    }
};
```

## Part 7: Test Program

```cpp
#include <iostream>
#include <chrono>
#include <thread>
#include <vector>
#include <cassert>

void testMultiWorker() {
    cout << "=== Testing MultiWorker ===" << endl;
    
    // Test 1: Basic functionality
    {
        atomic<bool> callbackCalled{false};
        
        vector<function<void()>> workFunctions = {
            []() { 
                this_thread::sleep_for(chrono::milliseconds(100));
                cout << "Work 1 completed" << endl;
            },
            []() { 
                this_thread::sleep_for(chrono::milliseconds(150));
                cout << "Work 2 completed" << endl;
            },
            []() { 
                this_thread::sleep_for(chrono::milliseconds(200));
                cout << "Work 3 completed" << endl;
            }
        };
        
        MultiWorker multiWorker(workFunctions);
        
        multiWorker.start([&callbackCalled]() {
            cout << "All work completed!" << endl;
            callbackCalled = true;
        });
        
        // Verify non-blocking
        assert(!callbackCalled);
        cout << "start() returned immediately (non-blocking)" << endl;
        
        // Wait for completion
        multiWorker.waitForCompletion();
        assert(callbackCalled);
        assert(multiWorker.isComplete());
        
        cout << "Test 1 passed!" << endl;
    }
    
    // Test 2: Empty work functions
    {
        vector<function<void()>> workFunctions = {
            []() {},
            []() {},
            []() {}
        };
        
        MultiWorker multiWorker(workFunctions);
        atomic<bool> done{false};
        
        multiWorker.start([&done]() {
            done = true;
        });
        
        multiWorker.waitForCompletion();
        assert(done);
        cout << "Test 2 passed: Empty work functions" << endl;
    }
    
    // Test 3: Single worker
    {
        vector<function<void()>> workFunctions = {
            []() {
                this_thread::sleep_for(chrono::milliseconds(50));
            }
        };
        
        MultiWorker multiWorker(workFunctions);
        atomic<bool> done{false};
        
        multiWorker.start([&done]() {
            done = true;
        });
        
        multiWorker.waitForCompletion();
        assert(done);
        cout << "Test 3 passed: Single worker" << endl;
    }
    
    // Test 4: Many workers
    {
        vector<function<void()>> workFunctions;
        for (int i = 0; i < 100; i++) {
            workFunctions.push_back([i]() {
                this_thread::sleep_for(chrono::milliseconds(10));
            });
        }
        
        MultiWorker multiWorker(workFunctions);
        atomic<int> completionCount{0};
        
        multiWorker.start([&completionCount]() {
            completionCount++;
        });
        
        multiWorker.waitForCompletion();
        assert(completionCount == 1);  // Callback called exactly once
        assert(multiWorker.isComplete());
        
        cout << "Test 4 passed: Many workers (100)" << endl;
    }
    
    cout << "\nAll tests passed!" << endl;
}

void testConcurrency() {
    cout << "\n=== Testing Concurrency ===" << endl;
    
    vector<function<void()>> workFunctions;
    for (int i = 0; i < 50; i++) {
        workFunctions.push_back([i]() {
            // Simulate work
            this_thread::sleep_for(chrono::milliseconds(rand() % 100));
        });
    }
    
    MultiWorker multiWorker(workFunctions);
    atomic<bool> callbackCalled{false};
    
    auto start = chrono::high_resolution_clock::now();
    
    multiWorker.start([&callbackCalled]() {
        callbackCalled = true;
    });
    
    multiWorker.waitForCompletion();
    
    auto end = chrono::high_resolution_clock::now();
    auto duration = chrono::duration_cast<chrono::milliseconds>(end - start);
    
    assert(callbackCalled);
    cout << "50 workers completed in " << duration.count() << " ms" << endl;
    cout << "Concurrency test passed!" << endl;
}

int main() {
    testMultiWorker();
    testConcurrency();
    return 0;
}
```

## Part 8: Design Considerations

### Thread Safety Analysis

**Critical Sections:**
1. **`onWorkerComplete()`**: Multiple threads call this simultaneously
   - **Solution**: Use mutex to protect shared state
   - **Check**: Must be atomic to prevent race conditions

2. **Counter increment**: `completedCount++`
   - **Solution**: Use `atomic<int>` or mutex protection
   - **Why**: Multiple threads increment simultaneously

3. **Callback invocation**: Only call once
   - **Solution**: Use flag (`callbackCalled`) to prevent multiple calls
   - **Why**: Race condition if two workers complete simultaneously

### Memory Management

**Potential Issues:**
1. **Worker destruction**: Workers might outlive MultiWorker
   - **Solution**: Ensure callbacks don't capture invalid pointers
   - **Use**: `shared_ptr` or ensure MultiWorker lifetime

2. **Callback lifetime**: Callback might be destroyed
   - **Solution**: Copy callback or use `shared_ptr`

### Performance Considerations

1. **Mutex contention**: High contention on completion callback
   - **Mitigation**: Minimize critical section
   - **Alternative**: Use lock-free atomic operations

2. **Thread creation**: Creating many threads
   - **Consideration**: Thread pool might be better for many workers
   - **Trade-off**: Simplicity vs. performance

## Part 9: Alternative: Lock-Free Implementation

### Using Only Atomic Operations

```cpp
class LockFreeMultiWorker {
private:
    vector<Worker> workers;
    atomic<int> completedCount{0};
    int totalWorkers;
    function<void()> finalCallback;
    atomic<bool> callbackCalled{false};
    
    void onWorkerComplete() {
        int current = ++completedCount;
        
        // Check if all done (lock-free)
        if (current == totalWorkers) {
            // Use compare-and-swap to ensure callback called only once
            bool expected = false;
            if (callbackCalled.compare_exchange_strong(expected, true)) {
                if (finalCallback) {
                    finalCallback();
                }
            }
        }
    }
    
public:
    LockFreeMultiWorker(vector<function<void()>> workFunctions) 
        : totalWorkers(workFunctions.size()) {
        for (auto& work : workFunctions) {
            workers.emplace_back(work);
        }
    }
    
    void start(function<void()> callback) {
        finalCallback = callback;
        completedCount = 0;
        callbackCalled = false;
        
        for (auto& worker : workers) {
            worker.start([this]() {
                this->onWorkerComplete();
            });
        }
    }
};
```

**Advantages:**
- No mutex contention
- Better performance under high concurrency

**Disadvantages:**
- More complex
- Requires careful atomic operation ordering

## Part 10: Interview Discussion Points

### Key Questions and Answers

**Q1: Why use mutex for completion tracking?**

**A:**
- Multiple worker threads call `onWorkerComplete()` simultaneously
- Need to atomically check and update completion state
- Mutex ensures thread-safe access to shared state

**Q2: Why atomic counter instead of regular int?**

**A:**
- `completedCount++` is not atomic (read-modify-write)
- Multiple threads incrementing simultaneously causes race conditions
- Atomic operations ensure correct counting

**Q3: How to ensure callback is called only once?**

**A:**
- Use a flag (`callbackCalled`) to track if callback was invoked
- Check flag before calling callback
- Use mutex or atomic compare-and-swap

**Q4: What if a worker throws an exception?**

**A:**
- Wrap work execution in try-catch
- Still call completion callback (work is "done")
- Track error count separately if needed

**Q5: How to handle very large number of workers?**

**A:**
- **Thread pool**: Reuse threads instead of creating many
- **Batching**: Process workers in batches
- **Async I/O**: Use async operations instead of threads

**Q6: Can we reuse MultiWorker?**

**A:**
- Add `reset()` method to reset state
- Ensure all previous work is complete
- Reset counters and flags

## Key Takeaways

### Design Principles

1. **Non-blocking operations**: Return immediately, work in background
2. **Thread safety**: Protect shared state with mutex/atomic
3. **Completion tracking**: Know when all work is done
4. **Error handling**: Handle exceptions gracefully

### Implementation Patterns

1. **Counter pattern**: Track completed workers
2. **Callback pattern**: Notify when all complete
3. **Mutex protection**: Thread-safe state updates
4. **Atomic operations**: Lock-free counting

### Common Pitfalls

1. **Race conditions**: Multiple threads updating counter
2. **Double callback**: Callback called multiple times
3. **Memory issues**: Callbacks capturing invalid pointers
4. **Blocking operations**: Accidentally blocking in start()

### Interview Tips

1. **Start simple**: Basic counter + mutex approach
2. **Discuss trade-offs**: Mutex vs. atomic, blocking vs. non-blocking
3. **Handle edge cases**: Empty workers, single worker, errors
4. **Explain design**: Why mutex, why atomic, why non-blocking
5. **Consider optimizations**: Lock-free, thread pools, batching

## Summary

MultiWorker implementation demonstrates:

- **Multi-threading design**: Coordinating multiple concurrent tasks
- **Non-blocking APIs**: Background execution patterns
- **Thread synchronization**: Mutex and atomic operations
- **Callback management**: Completion notification
- **Error handling**: Robust concurrent systems

**Key implementation:**
- **Counter tracking**: Atomic counter for completion
- **Mutex protection**: Thread-safe state updates
- **Callback invocation**: Called once when all complete
- **Non-blocking start**: Returns immediately

Understanding these concepts is essential for:
- System design interviews
- Concurrent programming
- Thread synchronization
- Async operation design

This knowledge is particularly valuable for storage companies like Pure Storage, where efficient concurrent operations and non-blocking APIs are critical for performance.

