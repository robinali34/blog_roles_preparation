---
layout: post
title: "Pure Storage Interview: Collatz Conjecture Count Function"
date: 2025-11-16
categories: [Interview Preparation, Pure Storage, Technical Interview, Algorithms, Optimization]
excerpt: "Complete solution for counting Collatz sequence steps: basic implementation, memoization optimization, and handling memory constraints. Includes C++ implementations and optimization strategies."
---

## Problem Overview

**Pure Storage Interview: Collatz Conjecture Count**

This problem tests your ability to:
1. Implement recursive/iterative algorithms
2. Optimize using memoization
3. Handle memory constraints
4. Trade off between time and space complexity

### Problem Statement

Given a function:
```cpp
int next(int n) {
    if (n % 2 == 0) return n / 2;
    return n * 3 + 1;
}
```

This function is part of the **Collatz conjecture**: for any positive integer, repeatedly applying `next()` will eventually converge to 1.

**Task:** Implement `count(n)` that returns how many calls to `next()` are needed to reach 1.

**Follow-up:** Optimize the solution and handle memory constraints.

## Part 1: Basic Implementation

### Naive Approach

```cpp
#include <iostream>
using namespace std;

class CollatzCounter {
public:
    int next(int n) {
        if (n % 2 == 0) return n / 2;
        return n * 3 + 1;
    }
    
    int count(int n) {
        int steps = 0;
        int current = n;
        
        while (current != 1) {
            current = next(current);
            steps++;
            
            // Safety check for infinite loop (shouldn't happen per conjecture)
            if (steps > 1000000) {
                return -1;  // Error: too many steps
            }
        }
        
        return steps;
    }
};
```

### Recursive Version

```cpp
class CollatzCounterRecursive {
public:
    int next(int n) {
        if (n % 2 == 0) return n / 2;
        return n * 3 + 1;
    }
    
    int count(int n) {
        if (n == 1) return 0;
        return 1 + count(next(n));
    }
};
```

**Time Complexity:** O(k) where k is the number of steps to reach 1  
**Space Complexity:** O(k) for recursive version, O(1) for iterative

**Problem:** For large n, this recalculates the same values multiple times.

## Part 2: Memoization Optimization

### Using HashMap

```cpp
#include <unordered_map>

class CollatzCounterMemoized {
private:
    unordered_map<int, int> memo;
    
public:
    int next(int n) {
        if (n % 2 == 0) return n / 2;
        return n * 3 + 1;
    }
    
    int count(int n) {
        // Base case
        if (n == 1) return 0;
        
        // Check memoization
        if (memo.find(n) != memo.end()) {
            return memo[n];
        }
        
        // Calculate and memoize
        int result = 1 + count(next(n));
        memo[n] = result;
        
        return result;
    }
    
    // Get memoization statistics
    size_t getMemoSize() const {
        return memo.size();
    }
    
    void clearMemo() {
        memo.clear();
    }
};
```

### Iterative Version with Memoization

```cpp
class CollatzCounterIterativeMemo {
private:
    unordered_map<int, int> memo;
    
public:
    int next(int n) {
        if (n % 2 == 0) return n / 2;
        return n * 3 + 1;
    }
    
    int count(int n) {
        if (n == 1) return 0;
        
        // Check if already computed
        if (memo.find(n) != memo.end()) {
            return memo[n];
        }
        
        // Build sequence and memoize
        vector<int> sequence;
        int current = n;
        
        // Collect sequence until we hit a memoized value or reach 1
        while (current != 1 && memo.find(current) == memo.end()) {
            sequence.push_back(current);
            current = next(current);
        }
        
        // Get base count (either from memo or 0 for 1)
        int baseCount = (current == 1) ? 0 : memo[current];
        
        // Backfill memoization
        for (int i = sequence.size() - 1; i >= 0; i--) {
            baseCount++;
            memo[sequence[i]] = baseCount;
        }
        
        return memo[n];
    }
};
```

**Time Complexity:** O(k) amortized, where k is unique values encountered  
**Space Complexity:** O(k) for memoization table

**Improvement:** Avoids recalculating paths that have been computed before.

## Part 3: Memory Constraints Problem

### Problem: HashMap Too Large

For very large n or many queries, the memoization table can grow very large.

### Solution 1: LRU Cache (Limited Memory)

```cpp
#include <list>
#include <unordered_map>

class LRUCache {
private:
    struct Node {
        int key;
        int value;
    };
    
    list<Node> cache;
    unordered_map<int, list<Node>::iterator> map;
    int capacity;
    
public:
    LRUCache(int cap) : capacity(cap) {}
    
    int get(int key) {
        if (map.find(key) == map.end()) {
            return -1;  // Not found
        }
        
        // Move to front (most recently used)
        cache.splice(cache.begin(), cache, map[key]);
        return map[key]->value;
    }
    
    void put(int key, int value) {
        if (map.find(key) != map.end()) {
            // Update existing
            map[key]->value = value;
            cache.splice(cache.begin(), cache, map[key]);
            return;
        }
        
        // Add new
        if (cache.size() >= capacity) {
            // Remove least recently used
            int lruKey = cache.back().key;
            cache.pop_back();
            map.erase(lruKey);
        }
        
        cache.push_front({key, value});
        map[key] = cache.begin();
    }
    
    bool contains(int key) {
        return map.find(key) != map.end();
    }
};

class CollatzCounterLRU {
private:
    LRUCache cache;
    
public:
    CollatzCounterLRU(int cacheSize = 10000) : cache(cacheSize) {}
    
    int next(int n) {
        if (n % 2 == 0) return n / 2;
        return n * 3 + 1;
    }
    
    int count(int n) {
        if (n == 1) return 0;
        
        // Check cache
        int cached = cache.get(n);
        if (cached != -1) {
            return cached;
        }
        
        // Calculate
        int result = 1 + count(next(n));
        
        // Store in cache
        cache.put(n, result);
        
        return result;
    }
};
```

### Solution 2: Bloom Filter + Selective Memoization

```cpp
#include <bitset>
#include <functional>

class BloomFilter {
private:
    bitset<100000> bits;
    hash<int> hash1, hash2, hash3;
    
public:
    void insert(int n) {
        bits[hash1(n) % 100000] = true;
        bits[hash2(n) % 100000] = true;
        bits[hash3(n) % 100000] = true;
    }
    
    bool mightContain(int n) {
        return bits[hash1(n) % 100000] && 
               bits[hash2(n) % 100000] && 
               bits[hash3(n) % 100000];
    }
};

class CollatzCounterBloom {
private:
    unordered_map<int, int> memo;
    BloomFilter bloom;
    int maxMemoSize;
    
public:
    CollatzCounterBloom(int maxSize = 50000) : maxMemoSize(maxSize) {}
    
    int next(int n) {
        if (n % 2 == 0) return n / 2;
        return n * 3 + 1;
    }
    
    int count(int n) {
        if (n == 1) return 0;
        
        // Check memoization if available
        if (memo.size() < maxMemoSize) {
            if (memo.find(n) != memo.end()) {
                return memo[n];
            }
        } else {
            // Use bloom filter to check if we've seen it
            if (bloom.mightContain(n)) {
                // Might be in memo, check
                if (memo.find(n) != memo.end()) {
                    return memo[n];
                }
            }
        }
        
        // Calculate
        int result = 1 + count(next(n));
        
        // Store if we have space
        if (memo.size() < maxMemoSize) {
            memo[n] = result;
            bloom.insert(n);
        }
        
        return result;
    }
};
```

### Solution 3: Disk-Based Memoization

```cpp
#include <fstream>
#include <sstream>

class DiskMemoization {
private:
    string filename;
    unordered_map<int, int> memoryCache;  // Small in-memory cache
    int cacheSize;
    
    string getKey(int n) {
        return to_string(n);
    }
    
    void saveToDisk(int n, int value) {
        ofstream file(filename, ios::app);
        if (file.is_open()) {
            file << n << " " << value << "\n";
            file.close();
        }
    }
    
    int loadFromDisk(int n) {
        ifstream file(filename);
        string line;
        
        while (getline(file, line)) {
            istringstream iss(line);
            int key, value;
            if (iss >> key >> value) {
                if (key == n) {
                    return value;
                }
            }
        }
        
        return -1;  // Not found
    }
    
public:
    DiskMemoization(string file = "collatz_memo.txt", int cache = 1000) 
        : filename(file), cacheSize(cache) {}
    
    int next(int n) {
        if (n % 2 == 0) return n / 2;
        return n * 3 + 1;
    }
    
    int count(int n) {
        if (n == 1) return 0;
        
        // Check memory cache first
        if (memoryCache.find(n) != memoryCache.end()) {
            return memoryCache[n];
        }
        
        // Check disk
        int diskValue = loadFromDisk(n);
        if (diskValue != -1) {
            // Add to memory cache if space available
            if (memoryCache.size() < cacheSize) {
                memoryCache[n] = diskValue;
            }
            return diskValue;
        }
        
        // Calculate
        int result = 1 + count(next(n));
        
        // Store in memory cache
        if (memoryCache.size() < cacheSize) {
            memoryCache[n] = result;
        } else {
            // Save to disk
            saveToDisk(n, result);
        }
        
        return result;
    }
};
```

**Note:** Disk-based solution is slow but handles unlimited memory constraints.

## Part 4: Optimized Solutions

### Solution 1: Path Compression

Only memoize values along the path, not all intermediate values:

```cpp
class CollatzCounterPathCompression {
private:
    unordered_map<int, int> memo;
    
public:
    int next(int n) {
        if (n % 2 == 0) return n / 2;
        return n * 3 + 1;
    }
    
    int count(int n) {
        if (n == 1) return 0;
        
        if (memo.find(n) != memo.end()) {
            return memo[n];
        }
        
        // Only memoize every k-th value to save memory
        int result = 1 + count(next(n));
        
        // Selective memoization: only store if n is "interesting"
        // e.g., store powers of 2, or values divisible by certain numbers
        if (n % 100 == 0 || (n & (n - 1)) == 0) {  // Powers of 2
            memo[n] = result;
        }
        
        return result;
    }
};
```

### Solution 2: Two-Level Caching

```cpp
class CollatzCounterTwoLevel {
private:
    // Level 1: Small values (always in memory)
    unordered_map<int, int> smallCache;
    
    // Level 2: Large values (LRU cache)
    LRUCache largeCache;
    
    static const int SMALL_THRESHOLD = 1000000;
    
public:
    CollatzCounterTwoLevel(int largeCacheSize = 10000) 
        : largeCache(largeCacheSize) {}
    
    int next(int n) {
        if (n % 2 == 0) return n / 2;
        return n * 3 + 1;
    }
    
    int count(int n) {
        if (n == 1) return 0;
        
        // Check appropriate cache
        if (n < SMALL_THRESHOLD) {
            if (smallCache.find(n) != smallCache.end()) {
                return smallCache[n];
            }
        } else {
            int cached = largeCache.get(n);
            if (cached != -1) {
                return cached;
            }
        }
        
        // Calculate
        int result = 1 + count(next(n));
        
        // Store in appropriate cache
        if (n < SMALL_THRESHOLD) {
            smallCache[n] = result;
        } else {
            largeCache.put(n, result);
        }
        
        return result;
    }
};
```

## Part 5: Complete Optimized Solution

```cpp
#include <iostream>
#include <unordered_map>
#include <vector>
#include <algorithm>
using namespace std;

class CollatzCounter {
private:
    unordered_map<int, int> memo;
    int maxMemoSize;
    bool useMemoization;
    
public:
    CollatzCounter(int maxSize = 100000, bool useMemo = true) 
        : maxMemoSize(maxSize), useMemoization(useMemo) {}
    
    int next(int n) {
        if (n % 2 == 0) return n / 2;
        return n * 3 + 1;
    }
    
    // Basic count without memoization
    int countBasic(int n) {
        int steps = 0;
        int current = n;
        
        while (current != 1) {
            current = next(current);
            steps++;
            
            // Safety check
            if (steps > 10000000) {
                return -1;  // Error
            }
        }
        
        return steps;
    }
    
    // Count with memoization
    int count(int n) {
        if (n == 1) return 0;
        
        if (!useMemoization) {
            return countBasic(n);
        }
        
        // Check memoization
        if (memo.find(n) != memo.end()) {
            return memo[n];
        }
        
        // If memo is full, use basic approach for this branch
        if (memo.size() >= maxMemoSize) {
            return countBasic(n);
        }
        
        // Calculate and memoize
        int result = 1 + count(next(n));
        memo[n] = result;
        
        return result;
    }
    
    // Iterative version with memoization (more memory efficient)
    int countIterative(int n) {
        if (n == 1) return 0;
        
        if (memo.find(n) != memo.end()) {
            return memo[n];
        }
        
        // Build sequence
        vector<int> sequence;
        int current = n;
        
        // Collect until we hit memoized value or 1
        while (current != 1 && memo.find(current) == memo.end()) {
            sequence.push_back(current);
            current = next(current);
        }
        
        // Get base count
        int baseCount = (current == 1) ? 0 : memo[current];
        
        // Backfill memoization
        for (int i = sequence.size() - 1; i >= 0; i--) {
            baseCount++;
            if (memo.size() < maxMemoSize) {
                memo[sequence[i]] = baseCount;
            }
        }
        
        return baseCount;
    }
    
    // Get statistics
    size_t getMemoSize() const {
        return memo.size();
    }
    
    void clearMemo() {
        memo.clear();
    }
    
    // Batch processing: count multiple numbers efficiently
    vector<int> countBatch(const vector<int>& numbers) {
        vector<int> results;
        for (int n : numbers) {
            results.push_back(count(n));
        }
        return results;
    }
};
```

## Part 6: Performance Analysis

### Time Complexity

| Approach | Time Complexity | Space Complexity |
|----------|----------------|------------------|
| **Basic** | O(k) | O(1) |
| **Memoized** | O(k) amortized | O(k) |
| **LRU Cache** | O(k) amortized | O(capacity) |
| **Disk-based** | O(k × disk I/O) | O(cache_size) |

Where k is the number of steps to reach 1.

### Memory Usage Analysis

For Collatz sequences:
- **Typical sequence length**: ~20-30 steps for numbers < 10^6
- **Peak value**: Can be much larger than starting value
- **Memoization benefit**: Significant for repeated queries

### Optimization Strategies

1. **Full Memoization**: Best for repeated queries, worst for memory
2. **LRU Cache**: Good balance, limited memory
3. **Selective Memoization**: Store only "important" values
4. **Two-level Cache**: Combine small and large value strategies

## Part 7: Test Program

```cpp
#include <iostream>
#include <chrono>
#include <vector>

void testCollatzCounter() {
    CollatzCounter counter(100000, true);
    
    cout << "=== Testing Collatz Counter ===" << endl;
    
    // Test 1: Small numbers
    vector<int> testCases = {1, 2, 3, 4, 5, 10, 27, 100, 1000};
    
    for (int n : testCases) {
        int steps = counter.count(n);
        cout << "count(" << n << ") = " << steps << endl;
    }
    
    // Test 2: Performance comparison
    cout << "\n=== Performance Test ===" << endl;
    
    CollatzCounter basicCounter(0, false);  // No memoization
    CollatzCounter memoCounter(100000, true);  // With memoization
    
    vector<int> largeNumbers = {1000000, 2000000, 3000000, 4000000, 5000000};
    
    // Basic
    auto start = chrono::high_resolution_clock::now();
    for (int n : largeNumbers) {
        basicCounter.count(n);
    }
    auto end = chrono::high_resolution_clock::now();
    auto basicTime = chrono::duration_cast<chrono::milliseconds>(end - start);
    
    // Memoized
    start = chrono::high_resolution_clock::now();
    for (int n : largeNumbers) {
        memoCounter.count(n);
    }
    end = chrono::high_resolution_clock::now();
    auto memoTime = chrono::duration_cast<chrono::milliseconds>(end - start);
    
    cout << "Basic time: " << basicTime.count() << " ms" << endl;
    cout << "Memoized time: " << memoTime.count() << " ms" << endl;
    cout << "Speedup: " << (double)basicTime.count() / memoTime.count() << "x" << endl;
    cout << "Memo size: " << memoCounter.getMemoSize() << " entries" << endl;
    
    // Test 3: Memory-constrained scenario
    cout << "\n=== Memory-Constrained Test ===" << endl;
    CollatzCounterLRU lruCounter(1000);  // Small cache
    
    for (int n : largeNumbers) {
        int steps = lruCounter.count(n);
        cout << "count(" << n << ") = " << steps << " (LRU cache)" << endl;
    }
}

int main() {
    testCollatzCounter();
    return 0;
}
```

## Part 8: Interview Discussion Points

### Key Questions and Answers

**Q1: How to optimize count(n)?**

**A:** Use memoization (HashMap):
- Store computed values to avoid recalculation
- Reduces time from O(k) to O(1) for repeated queries
- Space-time trade-off

**Q2: What if memory is not enough?**

**A:** Several strategies:
1. **LRU Cache**: Limit cache size, evict least recently used
2. **Selective Memoization**: Only store "important" values
3. **Two-level Cache**: Small values in memory, large values in LRU
4. **Disk-based**: Store on disk, keep small in-memory cache
5. **Bloom Filter**: Probabilistic check before expensive lookup

**Q3: Which values to memoize?**

**A:** Strategies:
- **All values**: Best performance, most memory
- **Powers of 2**: Common in sequences
- **Every k-th value**: Balance memory and performance
- **Small values only**: Most queries are for small numbers

**Q4: Time/Space complexity?**

**A:**
- **Time**: O(k) without memoization, O(1) amortized with memoization
- **Space**: O(1) basic, O(k) full memoization, O(capacity) with LRU

**Q5: When to use each approach?**

**A:**
- **Basic**: Single query, memory-constrained
- **Full Memoization**: Many repeated queries, memory available
- **LRU Cache**: Many queries, limited memory
- **Disk-based**: Very large datasets, persistent storage needed

## Key Takeaways

### Optimization Strategies

1. **Memoization**: Dramatically improves repeated query performance
2. **LRU Cache**: Good balance for memory-constrained scenarios
3. **Selective Storage**: Store only valuable entries
4. **Multi-level Caching**: Combine different strategies

### Memory Management

1. **Set limits**: Define maximum cache size
2. **Eviction policies**: LRU, LFU, or time-based
3. **Tiered storage**: Memory + disk for large datasets
4. **Probabilistic structures**: Bloom filters for space efficiency

### Interview Tips

1. **Start simple**: Basic implementation first
2. **Add optimization**: Memoization for repeated queries
3. **Discuss trade-offs**: Memory vs. performance
4. **Handle constraints**: LRU, disk, or selective memoization
5. **Measure performance**: Compare different approaches

## Summary

This problem demonstrates:

- **Algorithm optimization**: Memoization pattern
- **Memory management**: Handling constraints
- **System design**: Multi-level caching strategies
- **Trade-off analysis**: Time vs. space complexity

**Key insights:**
- Memoization provides huge speedup for repeated queries
- Memory constraints require smart caching strategies
- LRU cache balances performance and memory
- Disk-based solution handles unlimited scale

Understanding these concepts is valuable for:
- System design interviews
- Performance optimization
- Memory-constrained systems
- Caching strategies

This is particularly relevant for storage companies like Pure Storage, where efficient memory usage and caching are critical.

