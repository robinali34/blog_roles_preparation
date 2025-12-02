---
layout: post
title: "Pure Storage Interview: LRU Cache Implementation (LeetCode 146)"
date: 2025-11-16
categories: [Interview Preparation, Pure Storage, Technical Interview, Data Structures]
excerpt: "Complete LRU Cache implementation in C++: O(1) get and put operations using HashMap and Doubly Linked List. Includes detailed explanations, edge cases, and optimization strategies."
---

## Problem Overview

**Pure Storage Interview: LRU Cache (LeetCode 146)**

This is a classic system design and data structure problem that tests your ability to:
1. Design efficient data structures
2. Combine multiple data structures for optimal performance
3. Implement O(1) operations
4. Handle edge cases and memory management

### Problem Statement

Design and implement a data structure for a **Least Recently Used (LRU) cache**.

It should support the following operations:
- `LRUCache(int capacity)`: Initialize the cache with positive size `capacity`
- `int get(int key)`: Return the value of the key if it exists, otherwise return -1
- `void put(int key, int value)`: Update or insert the value
  - If the key already exists, update its value
  - If the key doesn't exist, insert the key-value pair
  - When the cache reaches its capacity, it should invalidate the **least recently used** item before inserting a new item

**Requirements:**
- Both `get` and `put` operations must run in **O(1) average time complexity**

### Example

```
Input:
["LRUCache", "put", "put", "get", "put", "get", "put", "get", "get", "get"]
[[2], [1, 1], [2, 2], [1], [3, 3], [2], [4, 4], [1], [3], [4]]

Output:
[null, null, null, 1, null, -1, null, -1, 3, 4]

Explanation:
LRUCache lRUCache = new LRUCache(2);
lRUCache.put(1, 1); // cache is {1=1}
lRUCache.put(2, 2); // cache is {1=1, 2=2}
lRUCache.get(1);    // return 1
lRUCache.put(3, 3); // LRU key was 2, evicts key 2, cache is {1=1, 3=3}
lRUCache.get(2);    // returns -1 (not found)
lRUCache.put(4, 4); // LRU key was 1, evicts key 1, cache is {4=4, 3=3}
lRUCache.get(1);    // returns -1 (not found)
lRUCache.get(3);    // returns 3
lRUCache.get(4);    // returns 4
```

## Part 1: Data Structure Design

### Why HashMap + Doubly Linked List?

**Requirements Analysis:**
1. **O(1) lookup**: Need HashMap for key → value mapping
2. **O(1) insertion**: Need efficient insertion
3. **O(1) deletion**: Need efficient deletion
4. **Track access order**: Need to know least recently used

**Solution:**
- **HashMap**: Maps `key → Node` for O(1) lookup
- **Doubly Linked List**: Maintains access order
  - Head: Most recently used
  - Tail: Least recently used

### Structure Design

```
HashMap: {key → Node pointer}
         |
         v
Doubly Linked List:
HEAD <-> [1, val1] <-> [2, val2] <-> [3, val3] <-> TAIL
         (most recent)                          (least recent)
```

## Part 2: Node Structure

### Doubly Linked List Node

```cpp
#include <unordered_map>
#include <iostream>
using namespace std;

class LRUCache {
private:
    struct Node {
        int key;
        int value;
        Node* prev;
        Node* next;
        
        Node(int k = 0, int v = 0) : key(k), value(v), prev(nullptr), next(nullptr) {}
    };
    
    int capacity;
    unordered_map<int, Node*> cache;
    Node* head;  // Dummy head (most recent)
    Node* tail;  // Dummy tail (least recent)
    
    // Helper functions
    void addNode(Node* node) {
        // Add node right after head
        node->prev = head;
        node->next = head->next;
        head->next->prev = node;
        head->next = node;
    }
    
    void removeNode(Node* node) {
        // Remove node from list
        Node* prev = node->prev;
        Node* next = node->next;
        prev->next = next;
        next->prev = prev;
    }
    
    void moveToHead(Node* node) {
        // Move node to head (mark as most recently used)
        removeNode(node);
        addNode(node);
    }
    
    Node* popTail() {
        // Remove and return the least recently used node
        Node* last = tail->prev;
        removeNode(last);
        return last;
    }
    
public:
    LRUCache(int cap) : capacity(cap) {
        // Create dummy head and tail
        head = new Node();
        tail = new Node();
        head->next = tail;
        tail->prev = head;
    }
    
    ~LRUCache() {
        // Clean up all nodes
        Node* current = head->next;
        while (current != tail) {
            Node* next = current->next;
            delete current;
            current = next;
        }
        delete head;
        delete tail;
    }
};
```

## Part 3: Complete Implementation

### Full LRU Cache Implementation

```cpp
#include <unordered_map>
#include <iostream>
using namespace std;

class LRUCache {
private:
    struct Node {
        int key;
        int value;
        Node* prev;
        Node* next;
        
        Node(int k = 0, int v = 0) : key(k), value(v), prev(nullptr), next(nullptr) {}
    };
    
    int capacity;
    unordered_map<int, Node*> cache;
    Node* head;  // Dummy head
    Node* tail;  // Dummy tail
    
    // Add node right after head
    void addNode(Node* node) {
        node->prev = head;
        node->next = head->next;
        head->next->prev = node;
        head->next = node;
    }
    
    // Remove node from list
    void removeNode(Node* node) {
        Node* prev = node->prev;
        Node* next = node->next;
        prev->next = next;
        next->prev = prev;
    }
    
    // Move node to head (mark as most recently used)
    void moveToHead(Node* node) {
        removeNode(node);
        addNode(node);
    }
    
    // Remove and return the least recently used node
    Node* popTail() {
        Node* last = tail->prev;
        removeNode(last);
        return last;
    }
    
public:
    LRUCache(int cap) : capacity(cap) {
        head = new Node();
        tail = new Node();
        head->next = tail;
        tail->prev = head;
    }
    
    ~LRUCache() {
        Node* current = head->next;
        while (current != tail) {
            Node* next = current->next;
            delete current;
            current = next;
        }
        delete head;
        delete tail;
    }
    
    int get(int key) {
        // Check if key exists
        if (cache.find(key) == cache.end()) {
            return -1;
        }
        
        // Get node and move to head
        Node* node = cache[key];
        moveToHead(node);
        
        return node->value;
    }
    
    void put(int key, int value) {
        // Check if key already exists
        if (cache.find(key) != cache.end()) {
            // Update existing node
            Node* node = cache[key];
            node->value = value;
            moveToHead(node);
        } else {
            // Create new node
            Node* newNode = new Node(key, value);
            
            // Check if cache is full
            if (cache.size() >= capacity) {
                // Remove least recently used
                Node* tail = popTail();
                cache.erase(tail->key);
                delete tail;
            }
            
            // Add new node
            addNode(newNode);
            cache[key] = newNode;
        }
    }
    
    // Helper: Print cache state (for debugging)
    void printCache() {
        cout << "Cache (capacity=" << capacity << ", size=" << cache.size() << "): ";
        Node* current = head->next;
        while (current != tail) {
            cout << "[" << current->key << "=" << current->value << "] ";
            current = current->next;
        }
        cout << endl;
    }
};
```

## Part 4: Alternative Implementation (Using STL list)

### Simpler Implementation with std::list

```cpp
#include <unordered_map>
#include <list>
#include <utility>
using namespace std;

class LRUCacheSTL {
private:
    int capacity;
    list<pair<int, int>> items;  // List of (key, value) pairs
    unordered_map<int, list<pair<int, int>>::iterator> cache;
    
public:
    LRUCacheSTL(int cap) : capacity(cap) {}
    
    int get(int key) {
        // Check if key exists
        if (cache.find(key) == cache.end()) {
            return -1;
        }
        
        // Move to front (most recently used)
        auto it = cache[key];
        int value = it->second;
        items.erase(it);
        items.push_front({key, value});
        cache[key] = items.begin();
        
        return value;
    }
    
    void put(int key, int value) {
        // Check if key already exists
        if (cache.find(key) != cache.end()) {
            // Update existing
            auto it = cache[key];
            items.erase(it);
            items.push_front({key, value});
            cache[key] = items.begin();
        } else {
            // Check if cache is full
            if (items.size() >= capacity) {
                // Remove least recently used (back of list)
                int lruKey = items.back().first;
                items.pop_back();
                cache.erase(lruKey);
            }
            
            // Add new item
            items.push_front({key, value});
            cache[key] = items.begin();
        }
    }
};
```

**Pros:**
- Simpler code
- Less memory management
- Uses standard library

**Cons:**
- Slightly less efficient (list operations)
- Less control over implementation

## Part 5: Edge Cases and Error Handling

### Enhanced Version with Edge Cases

```cpp
class LRUCache {
private:
    struct Node {
        int key;
        int value;
        Node* prev;
        Node* next;
        
        Node(int k = 0, int v = 0) : key(k), value(v), prev(nullptr), next(nullptr) {}
    };
    
    int capacity;
    unordered_map<int, Node*> cache;
    Node* head;
    Node* tail;
    
    void addNode(Node* node) {
        node->prev = head;
        node->next = head->next;
        head->next->prev = node;
        head->next = node;
    }
    
    void removeNode(Node* node) {
        Node* prev = node->prev;
        Node* next = node->next;
        prev->next = next;
        next->prev = prev;
    }
    
    void moveToHead(Node* node) {
        removeNode(node);
        addNode(node);
    }
    
    Node* popTail() {
        Node* last = tail->prev;
        removeNode(last);
        return last;
    }
    
public:
    LRUCache(int cap) {
        if (cap <= 0) {
            throw invalid_argument("Capacity must be positive");
        }
        capacity = cap;
        head = new Node();
        tail = new Node();
        head->next = tail;
        tail->prev = head;
    }
    
    ~LRUCache() {
        Node* current = head->next;
        while (current != tail) {
            Node* next = current->next;
            delete current;
            current = next;
        }
        delete head;
        delete tail;
    }
    
    // Copy constructor (delete to prevent copying)
    LRUCache(const LRUCache&) = delete;
    LRUCache& operator=(const LRUCache&) = delete;
    
    int get(int key) {
        if (cache.find(key) == cache.end()) {
            return -1;
        }
        
        Node* node = cache[key];
        moveToHead(node);
        return node->value;
    }
    
    void put(int key, int value) {
        if (cache.find(key) != cache.end()) {
            // Update existing
            Node* node = cache[key];
            node->value = value;
            moveToHead(node);
        } else {
            // Create new node
            Node* newNode = new Node(key, value);
            
            if (cache.size() >= capacity) {
                // Remove LRU
                Node* tail = popTail();
                cache.erase(tail->key);
                delete tail;
            }
            
            addNode(newNode);
            cache[key] = newNode;
        }
    }
    
    // Additional utility functions
    bool contains(int key) const {
        return cache.find(key) != cache.end();
    }
    
    size_t size() const {
        return cache.size();
    }
    
    bool empty() const {
        return cache.empty();
    }
    
    void clear() {
        Node* current = head->next;
        while (current != tail) {
            Node* next = current->next;
            cache.erase(current->key);
            delete current;
            current = next;
        }
        head->next = tail;
        tail->prev = head;
    }
};
```

## Part 6: Test Program

```cpp
#include <iostream>
#include <cassert>
#include <vector>

void testLRUCache() {
    cout << "=== Testing LRU Cache ===" << endl;
    
    // Test 1: Basic operations
    LRUCache cache(2);
    
    cache.put(1, 1);
    cache.put(2, 2);
    assert(cache.get(1) == 1);  // returns 1
    
    cache.put(3, 3);  // evicts key 2
    assert(cache.get(2) == -1);  // returns -1 (not found)
    
    cache.put(4, 4);  // evicts key 1
    assert(cache.get(1) == -1);  // returns -1 (not found)
    assert(cache.get(3) == 3);   // returns 3
    assert(cache.get(4) == 4);   // returns 4
    
    cout << "Test 1 passed!" << endl;
    
    // Test 2: Update existing key
    LRUCache cache2(2);
    cache2.put(1, 1);
    cache2.put(2, 2);
    cache2.put(1, 10);  // Update existing
    assert(cache2.get(1) == 10);
    assert(cache2.get(2) == 2);
    
    cout << "Test 2 passed!" << endl;
    
    // Test 3: Capacity 1
    LRUCache cache3(1);
    cache3.put(1, 1);
    assert(cache3.get(1) == 1);
    cache3.put(2, 2);  // Evicts 1
    assert(cache3.get(1) == -1);
    assert(cache3.get(2) == 2);
    
    cout << "Test 3 passed!" << endl;
    
    // Test 4: Get updates access order
    LRUCache cache4(2);
    cache4.put(1, 1);
    cache4.put(2, 2);
    cache4.get(1);  // Make 1 most recently used
    cache4.put(3, 3);  // Should evict 2, not 1
    assert(cache4.get(1) == 1);
    assert(cache4.get(2) == -1);
    assert(cache4.get(3) == 3);
    
    cout << "Test 4 passed!" << endl;
    
    cout << "\nAll tests passed!" << endl;
}

void testPerformance() {
    cout << "\n=== Performance Test ===" << endl;
    
    LRUCache cache(1000);
    
    // Insert 1000 items
    for (int i = 0; i < 1000; i++) {
        cache.put(i, i * 2);
    }
    
    // Access all items
    for (int i = 0; i < 1000; i++) {
        assert(cache.get(i) == i * 2);
    }
    
    // Insert more items (should evict old ones)
    for (int i = 1000; i < 2000; i++) {
        cache.put(i, i * 2);
    }
    
    // Check that old items are evicted
    assert(cache.get(0) == -1);
    assert(cache.get(999) == -1);
    assert(cache.get(1000) == 2000);
    assert(cache.get(1999) == 3998);
    
    cout << "Performance test passed!" << endl;
}

int main() {
    testLRUCache();
    testPerformance();
    return 0;
}
```

## Part 7: Complexity Analysis

### Time Complexity

| Operation | Time Complexity | Explanation |
|-----------|----------------|-------------|
| **get(key)** | O(1) | HashMap lookup + list operations |
| **put(key, value)** | O(1) | HashMap operations + list operations |
| **removeNode** | O(1) | Direct pointer manipulation |
| **moveToHead** | O(1) | Two O(1) operations |

### Space Complexity

- **O(capacity)**: 
  - HashMap: O(capacity) entries
  - Doubly Linked List: O(capacity) nodes
  - Total: O(capacity)

## Part 8: Optimization Strategies

### Strategy 1: Memory Pool

Reduce allocation overhead:

```cpp
class LRUCachePool {
private:
    struct Node {
        int key;
        int value;
        Node* prev;
        Node* next;
    };
    
    vector<Node> nodePool;
    int poolIndex;
    
    Node* allocateNode(int key, int value) {
        if (poolIndex < nodePool.size()) {
            Node* node = &nodePool[poolIndex++];
            node->key = key;
            node->value = value;
            return node;
        }
        return new Node(key, value);
    }
    
    // ... rest of implementation
};
```

### Strategy 2: Lock-Free for Concurrent Access

```cpp
#include <atomic>
#include <mutex>

class ThreadSafeLRUCache {
private:
    LRUCache cache;
    mutable mutex mtx;
    
public:
    ThreadSafeLRUCache(int cap) : cache(cap) {}
    
    int get(int key) {
        lock_guard<mutex> lock(mtx);
        return cache.get(key);
    }
    
    void put(int key, int value) {
        lock_guard<mutex> lock(mtx);
        cache.put(key, value);
    }
};
```

## Part 9: Interview Discussion Points

### Key Questions and Answers

**Q1: Why HashMap + Doubly Linked List?**

**A:**
- **HashMap**: O(1) key lookup
- **Doubly Linked List**: O(1) insertion/deletion at any position
- **Combined**: O(1) for all operations

**Q2: Why not use array or vector?**

**A:**
- Array: O(n) to find and remove LRU element
- Vector: O(n) to remove element from middle
- Doubly Linked List: O(1) to remove any node

**Q3: Why doubly linked list, not singly?**

**A:**
- Need to remove node given only the node pointer
- Singly linked list: Need previous node (O(n) to find)
- Doubly linked list: Direct access to previous (O(1))

**Q4: What if capacity is very large?**

**A:**
- Memory usage: O(capacity)
- Consider: Disk-based cache, distributed cache
- Alternative: Time-based expiration

**Q5: How to handle thread safety?**

**A:**
- Add mutex for thread-safe operations
- Consider: Read-write locks for better concurrency
- Alternative: Lock-free data structures

## Part 10: Real-World Applications

### Use Cases

1. **CPU Cache**: L1/L2/L3 cache replacement
2. **Web Browsers**: Back/forward button history
3. **Operating Systems**: Page replacement algorithms
4. **Databases**: Buffer pool management
5. **CDN**: Content caching

### Variations

1. **LFU Cache**: Least Frequently Used
2. **LRU-K**: Consider K most recent accesses
3. **ARC**: Adaptive Replacement Cache
4. **2Q**: Two-queue algorithm

## Key Takeaways

### Design Principles

1. **Combine data structures**: HashMap + Doubly Linked List
2. **Dummy nodes**: Simplify edge case handling
3. **O(1) operations**: All operations must be constant time
4. **Memory management**: Proper cleanup in destructor

### Implementation Tips

1. **Use dummy head/tail**: Simplifies insertion/deletion
2. **Separate helper functions**: `addNode`, `removeNode`, `moveToHead`
3. **Update both structures**: HashMap and list must stay in sync
4. **Handle edge cases**: Empty cache, capacity 1, etc.

### Interview Tips

1. **Start with design**: Explain HashMap + Doubly Linked List choice
2. **Draw diagrams**: Show data structure layout
3. **Handle edge cases**: Capacity 1, empty cache, etc.
4. **Discuss trade-offs**: Memory vs. performance
5. **Mention alternatives**: LFU, time-based expiration

## Summary

LRU Cache is a fundamental data structure problem that demonstrates:

- **Data structure design**: Combining HashMap and Doubly Linked List
- **O(1) operations**: Efficient get and put operations
- **Memory management**: Proper cleanup and resource handling
- **System design**: Understanding caching strategies

**Key implementation:**
- **HashMap**: O(1) key lookup
- **Doubly Linked List**: O(1) insertion/deletion
- **Dummy nodes**: Simplify operations
- **Time complexity**: O(1) for all operations
- **Space complexity**: O(capacity)

Understanding LRU Cache is essential for:
- System design interviews
- Performance optimization
- Memory management
- Caching strategies

This knowledge is particularly valuable for storage companies like Pure Storage, where efficient caching is critical for performance.

