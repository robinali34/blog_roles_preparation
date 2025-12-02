---
layout: post
title: "Pure Storage Interview: Get Unique ID (Multi-threading Optimization)"
date: 2025-11-16
categories: [Interview Preparation, Pure Storage, Technical Interview, Multi-threading]
excerpt: "Complete solution for Get Unique ID problem: optimizing unique ID generation using buffering, producer-consumer pattern, and multi-threading to minimize expensive getIds() calls while handling latency spikes."
---

## Problem Overview

**Pure Storage Interview: Get Unique ID**

This is a multi-threading optimization problem that tests your ability to:
1. Optimize expensive operations through buffering
2. Handle latency spikes with background prefetching
3. Implement producer-consumer pattern
4. Balance memory usage vs. performance
5. Ensure thread safety

### Problem Statement

Given an expensive function:
```cpp
void getIds(int* buffer, int count);
```

This function fills `buffer` with `count` unique IDs. **Key constraint**: The function takes the same time whether you request 1 ID or 100 IDs (e.g., network latency is the bottleneck).

**Task**: Implement `getOneId()` that returns a single unique ID efficiently.

### Requirements

- **Minimize `getIds()` calls**: Since it's expensive, batch requests
- **Minimize memory usage**: Don't buffer too many IDs
- **Handle latency spikes**: Avoid blocking when buffer is empty
- **Thread-safe**: Multiple threads can call `getOneId()` concurrently
- **High throughput**: Serve requests quickly

## Part 1: Naive Solution (Inefficient)

### Simple Approach

```cpp
int getOneId() {
    int id;
    getIds(&id, 1);  // Expensive call for just 1 ID!
    return id;
}
```

**Problems:**
- **Very slow**: Calls expensive `getIds()` for each request
- **Low throughput**: If `getIds()` takes 0.1s, throughput = 10 ops/s
- **No optimization**: Doesn't leverage batch capability

**Throughput Analysis:**
- If `getIds()` latency = 0.1s
- Throughput = 1 / 0.1s = **10 ops/s**

## Part 2: Buffering Solution

### Basic Buffer Approach

```cpp
#include <queue>
#include <vector>
using namespace std;

class UniqueIdGenerator {
private:
    queue<int> buffer;
    static const int BUFFER_SIZE = 1000;  // Trade-off: memory vs calls
    
public:
    int getOneId() {
        // Refill buffer if empty
        if (buffer.empty()) {
            vector<int> ids(BUFFER_SIZE);
            getIds(ids.data(), BUFFER_SIZE);
            
            for (int id : ids) {
                buffer.push(id);
            }
        }
        
        // Return one ID from buffer
        int id = buffer.front();
        buffer.pop();
        return id;
    }
};
```

### Throughput Improvement

**Before**: 10 ops/s (calling `getIds()` for each request)

**After**: 
- Call `getIds()` once for 1000 IDs
- Serve 1000 requests from buffer
- If `getIds()` takes 0.1s, amortized latency = 0.1s / 1000 = 0.0001s per request
- Throughput ≈ **10,000 ops/s** (1000x improvement!)

**Trade-off:**
- **Memory**: Stores 1000 IDs in buffer
- **Latency spike**: First request after buffer empty waits 0.1s

## Part 3: Handling Latency Spikes

### Problem: Blocking on Empty Buffer

When buffer is empty, `getOneId()` blocks for 0.1s waiting for `getIds()` to complete.

### Solution: Background Prefetching

Use a background thread to prefetch IDs before buffer runs out.

```cpp
#include <queue>
#include <vector>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <atomic>
using namespace std;

class UniqueIdGenerator {
private:
    queue<int> buffer;
    mutex mtx;
    condition_variable cv;
    atomic<bool> running{true};
    thread prefetchThread;
    
    static const int BUFFER_SIZE = 1000;
    static const int REFILL_THRESHOLD = 200;  // Refill when below this
    
    void prefetchWorker() {
        while (running) {
            unique_lock<mutex> lock(mtx);
            
            // Wait until buffer needs refill
            cv.wait(lock, [this]() {
                return !running || buffer.size() < REFILL_THRESHOLD;
            });
            
            if (!running) break;
            
            // Release lock during expensive operation
            lock.unlock();
            
            // Prefetch IDs
            vector<int> ids(BUFFER_SIZE);
            getIds(ids.data(), BUFFER_SIZE);
            
            // Add to buffer
            lock.lock();
            for (int id : ids) {
                buffer.push(id);
            }
            cv.notify_all();  // Notify waiting threads
        }
    }
    
public:
    UniqueIdGenerator() {
        // Initial fill
        vector<int> ids(BUFFER_SIZE);
        getIds(ids.data(), BUFFER_SIZE);
        for (int id : ids) {
            buffer.push(id);
        }
        
        // Start background thread
        prefetchThread = thread(&UniqueIdGenerator::prefetchWorker, this);
    }
    
    ~UniqueIdGenerator() {
        running = false;
        cv.notify_all();
        if (prefetchThread.joinable()) {
            prefetchThread.join();
        }
    }
    
    int getOneId() {
        unique_lock<mutex> lock(mtx);
        
        // Wait if buffer is empty (shouldn't happen with prefetching)
        cv.wait(lock, [this]() { return !buffer.empty(); });
        
        int id = buffer.front();
        buffer.pop();
        
        // Trigger refill if below threshold
        if (buffer.size() < REFILL_THRESHOLD) {
            cv.notify_one();
        }
        
        return id;
    }
};
```

### How It Works

1. **Initial fill**: Buffer starts with 1000 IDs
2. **Background thread**: Monitors buffer size
3. **Refill trigger**: When buffer < 200, start prefetching
4. **Non-blocking**: `getOneId()` never waits for `getIds()` (prefetching happens in background)

**Latency**: Near-zero (served from buffer)
**Throughput**: High (no blocking)

## Part 4: Producer-Consumer Pattern (Complete Solution)

### Thread-Safe Implementation

```cpp
#include <queue>
#include <vector>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <atomic>
#include <iostream>
using namespace std;

// External function (given)
void getIds(int* buffer, int count) {
    // Simulate expensive operation (network call, etc.)
    // In real implementation, this would fetch from remote server
    static int nextId = 1;
    for (int i = 0; i < count; i++) {
        buffer[i] = nextId++;
    }
    // Simulate network latency
    this_thread::sleep_for(chrono::milliseconds(100));
}

class UniqueIdGenerator {
private:
    queue<int> buffer;
    mutex mtx;
    condition_variable cv;
    atomic<bool> running{true};
    thread producerThread;
    
    static const int BUFFER_SIZE = 1000;
    static const int REFILL_THRESHOLD = 200;
    
    void producer() {
        while (running) {
            unique_lock<mutex> lock(mtx);
            
            // Wait until buffer needs refill
            cv.wait(lock, [this]() {
                return !running || buffer.size() < REFILL_THRESHOLD;
            });
            
            if (!running) break;
            
            // Release lock before expensive operation
            lock.unlock();
            
            // Expensive operation: fetch IDs
            vector<int> ids(BUFFER_SIZE);
            getIds(ids.data(), BUFFER_SIZE);
            
            // Re-acquire lock to update buffer
            lock.lock();
            for (int id : ids) {
                buffer.push(id);
            }
            
            // Notify consumers
            cv.notify_all();
        }
    }
    
public:
    UniqueIdGenerator() {
        // Initial fill
        vector<int> ids(BUFFER_SIZE);
        getIds(ids.data(), BUFFER_SIZE);
        for (int id : ids) {
            buffer.push(id);
        }
        
        // Start producer thread
        producerThread = thread(&UniqueIdGenerator::producer, this);
    }
    
    ~UniqueIdGenerator() {
        running = false;
        cv.notify_all();
        if (producerThread.joinable()) {
            producerThread.join();
        }
    }
    
    int getOneId() {
        unique_lock<mutex> lock(mtx);
        
        // Wait if buffer is empty
        cv.wait(lock, [this]() { return !buffer.empty(); });
        
        int id = buffer.front();
        buffer.pop();
        
        // Trigger producer if needed
        if (buffer.size() < REFILL_THRESHOLD) {
            cv.notify_one();
        }
        
        return id;
    }
    
    // Get current buffer size (for monitoring)
    size_t getBufferSize() {
        lock_guard<mutex> lock(mtx);
        return buffer.size();
    }
};
```

## Part 5: Optimization: Exponential Back-off

### Adaptive Buffer Size

Instead of fixed buffer size, adapt based on demand:

```cpp
class AdaptiveUniqueIdGenerator {
private:
    queue<int> buffer;
    mutex mtx;
    condition_variable cv;
    atomic<bool> running{true};
    thread producerThread;
    
    int currentBufferSize = 1000;  // Start with 1000
    static const int MIN_BUFFER_SIZE = 500;
    static const int MAX_BUFFER_SIZE = 10000;
    static const int REFILL_THRESHOLD_RATIO = 5;  // Refill at 1/5 of buffer
    
    void producer() {
        while (running) {
            unique_lock<mutex> lock(mtx);
            
            int threshold = currentBufferSize / REFILL_THRESHOLD_RATIO;
            cv.wait(lock, [this, threshold]() {
                return !running || buffer.size() < threshold;
            });
            
            if (!running) break;
            
            // Adjust buffer size based on consumption rate
            int bufferSize = currentBufferSize;
            lock.unlock();
            
            // Fetch IDs
            vector<int> ids(bufferSize);
            getIds(ids.data(), bufferSize);
            
            lock.lock();
            for (int id : ids) {
                buffer.push(id);
            }
            
            // Adaptive sizing: if buffer was empty, increase size
            // If buffer never got low, decrease size
            if (buffer.size() < threshold * 2) {
                currentBufferSize = min(currentBufferSize * 2, MAX_BUFFER_SIZE);
            } else {
                currentBufferSize = max(currentBufferSize / 2, MIN_BUFFER_SIZE);
            }
            
            cv.notify_all();
        }
    }
    
public:
    AdaptiveUniqueIdGenerator() {
        vector<int> ids(currentBufferSize);
        getIds(ids.data(), currentBufferSize);
        for (int id : ids) {
            buffer.push(id);
        }
        producerThread = thread(&AdaptiveUniqueIdGenerator::producer, this);
    }
    
    ~AdaptiveUniqueIdGenerator() {
        running = false;
        cv.notify_all();
        if (producerThread.joinable()) {
            producerThread.join();
        }
    }
    
    int getOneId() {
        unique_lock<mutex> lock(mtx);
        cv.wait(lock, [this]() { return !buffer.empty(); });
        
        int id = buffer.front();
        buffer.pop();
        
        int threshold = currentBufferSize / REFILL_THRESHOLD_RATIO;
        if (buffer.size() < threshold) {
            cv.notify_one();
        }
        
        return id;
    }
};
```

## Part 6: Double Buffer Pattern (Advanced)

### Eliminate Lock Contention

Use double buffering to reduce lock contention:

```cpp
#include <atomic>
#include <array>

class DoubleBufferUniqueIdGenerator {
private:
    array<queue<int>, 2> buffers;
    atomic<int> activeBuffer{0};  // 0 or 1
    mutex mtx;
    condition_variable cv;
    atomic<bool> running{true};
    thread producerThread;
    
    static const int BUFFER_SIZE = 1000;
    
    void producer() {
        while (running) {
            unique_lock<mutex> lock(mtx);
            
            int current = activeBuffer.load();
            int next = 1 - current;
            
            // Wait until active buffer needs refill
            cv.wait(lock, [this, current]() {
                return !running || buffers[current].size() < BUFFER_SIZE / 5;
            });
            
            if (!running) break;
            
            lock.unlock();
            
            // Fill next buffer
            vector<int> ids(BUFFER_SIZE);
            getIds(ids.data(), BUFFER_SIZE);
            
            lock.lock();
            for (int id : ids) {
                buffers[next].push(id);
            }
            
            // Switch buffers atomically
            activeBuffer.store(next);
            cv.notify_all();
        }
    }
    
public:
    DoubleBufferUniqueIdGenerator() {
        // Initial fill
        vector<int> ids(BUFFER_SIZE);
        getIds(ids.data(), BUFFER_SIZE);
        for (int id : ids) {
            buffers[0].push(id);
        }
        
        producerThread = thread(&DoubleBufferUniqueIdGenerator::producer, this);
    }
    
    ~DoubleBufferUniqueIdGenerator() {
        running = false;
        cv.notify_all();
        if (producerThread.joinable()) {
            producerThread.join();
        }
    }
    
    int getOneId() {
        unique_lock<mutex> lock(mtx);
        
        int current = activeBuffer.load();
        cv.wait(lock, [this, current]() {
            return !buffers[current].empty();
        });
        
        int id = buffers[current].front();
        buffers[current].pop();
        
        if (buffers[current].size() < BUFFER_SIZE / 5) {
            cv.notify_one();
        }
        
        return id;
    }
};
```

## Part 7: Complete Solution with Error Handling

```cpp
#include <queue>
#include <vector>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <atomic>
#include <stdexcept>
#include <chrono>
using namespace std;

class UniqueIdGenerator {
private:
    queue<int> buffer;
    mutex mtx;
    condition_variable cv;
    atomic<bool> running{true};
    thread producerThread;
    atomic<int> totalGenerated{0};
    atomic<int> totalServed{0};
    
    static const int BUFFER_SIZE = 1000;
    static const int REFILL_THRESHOLD = 200;
    static const int MAX_WAIT_MS = 5000;  // Max wait time
    
    void producer() {
        while (running) {
            unique_lock<mutex> lock(mtx);
            
            cv.wait(lock, [this]() {
                return !running || buffer.size() < REFILL_THRESHOLD;
            });
            
            if (!running) break;
            
            lock.unlock();
            
            try {
                // Fetch IDs
                vector<int> ids(BUFFER_SIZE);
                getIds(ids.data(), BUFFER_SIZE);
                
                lock.lock();
                for (int id : ids) {
                    buffer.push(id);
                }
                totalGenerated += BUFFER_SIZE;
                cv.notify_all();
            } catch (const exception& e) {
                // Handle error (log, retry, etc.)
                cerr << "Error fetching IDs: " << e.what() << endl;
            }
        }
    }
    
public:
    UniqueIdGenerator() {
        // Initial fill
        try {
            vector<int> ids(BUFFER_SIZE);
            getIds(ids.data(), BUFFER_SIZE);
            for (int id : ids) {
                buffer.push(id);
            }
            totalGenerated = BUFFER_SIZE;
        } catch (const exception& e) {
            throw runtime_error("Failed to initialize ID generator: " + string(e.what()));
        }
        
        producerThread = thread(&UniqueIdGenerator::producer, this);
    }
    
    ~UniqueIdGenerator() {
        running = false;
        cv.notify_all();
        if (producerThread.joinable()) {
            producerThread.join();
        }
    }
    
    int getOneId() {
        unique_lock<mutex> lock(mtx);
        
        // Wait with timeout
        if (!cv.wait_for(lock, chrono::milliseconds(MAX_WAIT_MS), 
                        [this]() { return !buffer.empty(); })) {
            throw runtime_error("Timeout waiting for ID");
        }
        
        int id = buffer.front();
        buffer.pop();
        totalServed++;
        
        // Trigger refill if needed
        if (buffer.size() < REFILL_THRESHOLD) {
            cv.notify_one();
        }
        
        return id;
    }
    
    // Statistics
    struct Stats {
        int bufferSize;
        int totalGenerated;
        int totalServed;
    };
    
    Stats getStats() {
        lock_guard<mutex> lock(mtx);
        return {
            static_cast<int>(buffer.size()),
            totalGenerated.load(),
            totalServed.load()
        };
    }
};
```

## Part 8: Performance Analysis

### Throughput Comparison

| Approach | Throughput | Latency | Memory |
|----------|-----------|---------|--------|
| **Naive** | 10 ops/s | 0.1s | O(1) |
| **Buffer (1000)** | ~10,000 ops/s | 0.0001s (amortized) | O(1000) |
| **Prefetching** | ~10,000 ops/s | <0.001s | O(1000) |
| **Double Buffer** | ~10,000 ops/s | <0.001s | O(2000) |

### Key Metrics

1. **Throughput**: Requests per second
   - Naive: Limited by `getIds()` latency
   - Buffered: Amortized over batch size

2. **Latency**: Time to get one ID
   - Naive: 0.1s (blocking)
   - Buffered: Near-zero (served from buffer)

3. **Memory**: Buffer size
   - Trade-off: More memory = fewer `getIds()` calls

4. **Latency Spike**: Worst-case wait time
   - Without prefetching: 0.1s (when buffer empty)
   - With prefetching: Near-zero (background refill)

## Part 9: Test Program

```cpp
#include <iostream>
#include <vector>
#include <thread>
#include <chrono>

void testMultiThread() {
    UniqueIdGenerator gen;
    const int NUM_THREADS = 10;
    const int REQUESTS_PER_THREAD = 100;
    
    vector<thread> threads;
    atomic<int> totalIds{0};
    
    auto start = chrono::high_resolution_clock::now();
    
    for (int i = 0; i < NUM_THREADS; i++) {
        threads.emplace_back([&gen, &totalIds, REQUESTS_PER_THREAD]() {
            for (int j = 0; j < REQUESTS_PER_THREAD; j++) {
                int id = gen.getOneId();
                totalIds++;
            }
        });
    }
    
    for (auto& t : threads) {
        t.join();
    }
    
    auto end = chrono::high_resolution_clock::now();
    auto duration = chrono::duration_cast<chrono::milliseconds>(end - start);
    
    cout << "Total IDs: " << totalIds << endl;
    cout << "Time: " << duration.count() << " ms" << endl;
    cout << "Throughput: " << (totalIds * 1000.0 / duration.count()) << " ops/s" << endl;
    
    auto stats = gen.getStats();
    cout << "Buffer size: " << stats.bufferSize << endl;
    cout << "Total generated: " << stats.totalGenerated << endl;
}

int main() {
    testMultiThread();
    return 0;
}
```

## Key Takeaways

### Design Principles

1. **Batch Operations**: Minimize expensive calls by batching
2. **Prefetching**: Use background threads to avoid latency spikes
3. **Thread Safety**: Proper synchronization for concurrent access
4. **Memory Trade-offs**: Balance buffer size vs. memory usage
5. **Adaptive Sizing**: Adjust buffer size based on demand

### Critical Points

1. **Unlock before expensive operations**: Don't hold lock during `getIds()`
2. **Condition variables**: Efficient waiting and notification
3. **Producer-consumer pattern**: Separate fetching from serving
4. **Threshold-based refill**: Trigger refill before buffer empties

### Interview Tips

1. **Start simple**: Naive solution first
2. **Identify bottlenecks**: Expensive `getIds()` calls
3. **Optimize incrementally**: Buffer → Prefetching → Multi-threading
4. **Discuss trade-offs**: Memory vs. latency vs. throughput
5. **Consider edge cases**: Buffer empty, errors, shutdown

## Summary

This problem demonstrates:
- **Batching optimization**: Reduce expensive calls
- **Producer-consumer pattern**: Background prefetching
- **Thread synchronization**: Proper lock management
- **Performance trade-offs**: Memory vs. latency vs. throughput
- **System design**: Handling latency spikes and high throughput

The key insight is: **Use background prefetching to eliminate latency spikes while maintaining high throughput through batching**.

