---
layout: post
title: "Pure Storage Interview: Skip List Data Structure Implementation"
date: 2025-11-16
categories: [Interview Preparation, Pure Storage, Technical Interview, Data Structures]
excerpt: "Complete Skip List implementation in C++: efficient ordered data structure with O(log n) average time complexity for search, insert, and delete operations. Includes detailed explanations and code."
---

## Problem Overview

**Pure Storage Interview: Skip List**

A Skip List is a probabilistic data structure that provides O(log n) average time complexity for search, insert, and delete operations. It's an alternative to balanced trees but simpler to implement.

### Problem Statement

Implement a Skip List with the following operations:
- `search(int num)`: Search for a value, return true if exists
- `insert(int num)`: Insert a value into the Skip List
- `erase(int num)`: Delete a value, return true if successful
- `getRandom()`: (Optional) Get a random element

### Why Skip List?

**Advantages:**
- **Simple implementation**: No complex rotations like AVL/Red-Black trees
- **Efficient operations**: O(log n) average time complexity
- **Dynamic**: Handles frequent insertions/deletions well
- **Range queries**: Easy to implement range-based operations

**Disadvantages:**
- **Space overhead**: Stores multiple pointers per node
- **Worst case**: O(n) time complexity (rare but possible)
- **Randomization**: Requires good random number generation

## Part 1: Skip List Structure

### Concept

A Skip List consists of multiple sorted linked lists (layers):
- **Level 0**: Contains all elements (base layer)
- **Level 1+**: Contains subsets of elements with "skip" pointers
- Each node appears in multiple layers with probability 1/2

### Visual Representation

```
Level 3:  HEAD -----------------------------------------> NULL
           |
Level 2:  HEAD --------> 7 -------------------------> NULL
           |             |
Level 1:  HEAD -> 3 -> 7 --------> 15 -> 19 -------> NULL
           |      |     |           |     |
Level 0:  HEAD->1->3->7->9->15->17->19->21->NULL
```

### Node Structure

```cpp
#include <vector>
#include <random>
#include <climits>
using namespace std;

class SkipListNode {
public:
    int val;
    vector<SkipListNode*> next;  // Pointers for each level
    
    SkipListNode(int value, int level) : val(value), next(level, nullptr) {}
};
```

## Part 2: Basic Skip List Implementation

### Class Definition

```cpp
class SkipList {
private:
    SkipListNode* head;
    int maxLevel;
    int currentLevel;
    random_device rd;
    mt19937 gen;
    uniform_real_distribution<> dis;
    
    // Generate random level (with probability 1/2 for each level)
    int randomLevel() {
        int level = 1;
        while (dis(gen) < 0.5 && level < maxLevel) {
            level++;
        }
        return level;
    }
    
public:
    SkipList(int maxL = 16) : maxLevel(maxL), currentLevel(1), gen(rd()), dis(0.0, 1.0) {
        // Create head node with maxLevel pointers
        head = new SkipListNode(INT_MIN, maxLevel);
    }
    
    ~SkipList() {
        SkipListNode* node = head;
        while (node) {
            SkipListNode* next = node->next[0];
            delete node;
            node = next;
        }
    }
};
```

## Part 3: Search Operation

### Algorithm

1. Start from the highest level of head
2. Move forward while next node's value < target
3. If next node's value >= target, go down one level
4. Repeat until level 0
5. Check if current node's next has target value

### Implementation

```cpp
bool search(int target) {
    SkipListNode* current = head;
    
    // Start from top level, work down
    for (int i = currentLevel - 1; i >= 0; i--) {
        // Move forward while next node's value < target
        while (current->next[i] && current->next[i]->val < target) {
            current = current->next[i];
        }
    }
    
    // Move to next node (should be >= target)
    current = current->next[0];
    
    // Check if found
    return current && current->val == target;
}
```

### Time Complexity

- **Average**: O(log n) - Each level skips about half the remaining elements
- **Worst case**: O(n) - If all nodes are at level 0 (extremely rare)

## Part 4: Insert Operation

### Algorithm

1. Find insertion position at each level (like search)
2. Generate random level for new node
3. Create new node with that level
4. Update pointers at each level

### Implementation

```cpp
void insert(int num) {
    // Track nodes that need to be updated at each level
    vector<SkipListNode*> update(maxLevel, nullptr);
    SkipListNode* current = head;
    
    // Find insertion position for each level
    for (int i = currentLevel - 1; i >= 0; i--) {
        while (current->next[i] && current->next[i]->val < num) {
            current = current->next[i];
        }
        update[i] = current;  // Last node before insertion point
    }
    
    // Check if already exists
    current = current->next[0];
    if (current && current->val == num) {
        return;  // Already exists
    }
    
    // Generate random level for new node
    int newLevel = randomLevel();
    
    // Update max level if needed
    if (newLevel > currentLevel) {
        for (int i = currentLevel; i < newLevel; i++) {
            update[i] = head;
        }
        currentLevel = newLevel;
    }
    
    // Create new node
    SkipListNode* newNode = new SkipListNode(num, newLevel);
    
    // Update pointers at each level
    for (int i = 0; i < newLevel; i++) {
        newNode->next[i] = update[i]->next[i];
        update[i]->next[i] = newNode;
    }
}
```

### Example: Inserting 9

```
Before:
Level 1: HEAD -> 3 -> 7 -> 15 -> NULL
Level 0: HEAD->1->3->7->15->17->NULL

After inserting 9 (random level = 1):
Level 1: HEAD -> 3 -> 7 -> 9 -> 15 -> NULL
Level 0: HEAD->1->3->7->9->15->17->NULL
```

## Part 5: Delete Operation

### Algorithm

1. Find node to delete (like search)
2. Update pointers at each level to skip the node
3. Delete the node
4. Update max level if needed

### Implementation

```cpp
bool erase(int num) {
    vector<SkipListNode*> update(maxLevel, nullptr);
    SkipListNode* current = head;
    
    // Find node to delete
    for (int i = currentLevel - 1; i >= 0; i--) {
        while (current->next[i] && current->next[i]->val < num) {
            current = current->next[i];
        }
        update[i] = current;
    }
    
    // Check if node exists
    current = current->next[0];
    if (!current || current->val != num) {
        return false;  // Not found
    }
    
    // Update pointers at each level
    for (int i = 0; i < currentLevel; i++) {
        if (update[i]->next[i] != current) {
            break;  // No more levels contain this node
        }
        update[i]->next[i] = current->next[i];
    }
    
    // Update max level
    while (currentLevel > 1 && head->next[currentLevel - 1] == nullptr) {
        currentLevel--;
    }
    
    delete current;
    return true;
}
```

## Part 6: Complete Implementation

```cpp
#include <iostream>
#include <vector>
#include <random>
#include <climits>
#include <algorithm>
using namespace std;

class SkipListNode {
public:
    int val;
    vector<SkipListNode*> next;
    
    SkipListNode(int value, int level) : val(value), next(level, nullptr) {}
};

class SkipList {
private:
    SkipListNode* head;
    int maxLevel;
    int currentLevel;
    random_device rd;
    mt19937 gen;
    uniform_real_distribution<> dis;
    
    int randomLevel() {
        int level = 1;
        while (dis(gen) < 0.5 && level < maxLevel) {
            level++;
        }
        return level;
    }
    
public:
    SkipList(int maxL = 16) : maxLevel(maxL), currentLevel(1), gen(rd()), dis(0.0, 1.0) {
        head = new SkipListNode(INT_MIN, maxLevel);
    }
    
    ~SkipList() {
        SkipListNode* node = head;
        while (node) {
            SkipListNode* next = node->next[0];
            delete node;
            node = next;
        }
    }
    
    bool search(int target) {
        SkipListNode* current = head;
        
        for (int i = currentLevel - 1; i >= 0; i--) {
            while (current->next[i] && current->next[i]->val < target) {
                current = current->next[i];
            }
        }
        
        current = current->next[0];
        return current && current->val == target;
    }
    
    void insert(int num) {
        vector<SkipListNode*> update(maxLevel, nullptr);
        SkipListNode* current = head;
        
        for (int i = currentLevel - 1; i >= 0; i--) {
            while (current->next[i] && current->next[i]->val < num) {
                current = current->next[i];
            }
            update[i] = current;
        }
        
        current = current->next[0];
        if (current && current->val == num) {
            return;  // Already exists
        }
        
        int newLevel = randomLevel();
        if (newLevel > currentLevel) {
            for (int i = currentLevel; i < newLevel; i++) {
                update[i] = head;
            }
            currentLevel = newLevel;
        }
        
        SkipListNode* newNode = new SkipListNode(num, newLevel);
        for (int i = 0; i < newLevel; i++) {
            newNode->next[i] = update[i]->next[i];
            update[i]->next[i] = newNode;
        }
    }
    
    bool erase(int num) {
        vector<SkipListNode*> update(maxLevel, nullptr);
        SkipListNode* current = head;
        
        for (int i = currentLevel - 1; i >= 0; i--) {
            while (current->next[i] && current->next[i]->val < num) {
                current = current->next[i];
            }
            update[i] = current;
        }
        
        current = current->next[0];
        if (!current || current->val != num) {
            return false;
        }
        
        for (int i = 0; i < currentLevel; i++) {
            if (update[i]->next[i] != current) {
                break;
            }
            update[i]->next[i] = current->next[i];
        }
        
        while (currentLevel > 1 && head->next[currentLevel - 1] == nullptr) {
            currentLevel--;
        }
    
        delete current;
        return true;
    }
    
    // Helper: Print Skip List (for debugging)
    void print() {
        for (int i = currentLevel - 1; i >= 0; i--) {
            cout << "Level " << i << ": ";
            SkipListNode* node = head->next[i];
            while (node) {
                cout << node->val << " -> ";
                node = node->next[i];
            }
            cout << "NULL" << endl;
        }
    }
    
    // Get all elements in sorted order
    vector<int> getAll() {
        vector<int> result;
        SkipListNode* node = head->next[0];
        while (node) {
            result.push_back(node->val);
            node = node->next[0];
        }
        return result;
    }
};
```

## Part 7: Advanced Operations

### Range Query

```cpp
vector<int> rangeQuery(int start, int end) {
    vector<int> result;
    SkipListNode* current = head;
    
    // Find start position
    for (int i = currentLevel - 1; i >= 0; i--) {
        while (current->next[i] && current->next[i]->val < start) {
            current = current->next[i];
        }
    }
    
    // Traverse from start to end
    current = current->next[0];
    while (current && current->val <= end) {
        result.push_back(current->val);
        current = current->next[0];
    }
    
    return result;
}
```

### Get Minimum/Maximum

```cpp
int getMin() {
    if (!head->next[0]) {
        return INT_MIN;  // Empty
    }
    return head->next[0]->val;
}

int getMax() {
    SkipListNode* current = head;
    
    // Go to highest level
    for (int i = currentLevel - 1; i >= 0; i--) {
        while (current->next[i]) {
            current = current->next[i];
        }
    }
    
    return current->val;
}
```

### Get Size

```cpp
int size() {
    int count = 0;
    SkipListNode* node = head->next[0];
    while (node) {
        count++;
        node = node->next[0];
    }
    return count;
}
```

## Part 8: Time and Space Complexity

### Time Complexity

| Operation | Average | Worst Case |
|-----------|---------|------------|
| **Search** | O(log n) | O(n) |
| **Insert** | O(log n) | O(n) |
| **Delete** | O(log n) | O(n) |
| **Range Query** | O(log n + k) | O(n) |

Where `k` is the number of elements in the range.

### Space Complexity

- **Average**: O(n) - Each node appears in ~2 levels on average
- **Worst case**: O(n log n) - If all nodes are at max level (extremely rare)

**Expected number of levels**: log₂(n)
**Expected nodes per level**: n / 2^level

## Part 9: Comparison with Other Data Structures

| Data Structure | Search | Insert | Delete | Space | Complexity |
|----------------|--------|--------|--------|-------|------------|
| **Skip List** | O(log n) | O(log n) | O(log n) | O(n) | Simple |
| **Balanced BST** | O(log n) | O(log n) | O(log n) | O(n) | Complex |
| **Hash Table** | O(1) | O(1) | O(1) | O(n) | No order |
| **Sorted Array** | O(log n) | O(n) | O(n) | O(n) | Static |

### When to Use Skip List?

✅ **Use Skip List when:**
- Need ordered data structure
- Frequent insertions/deletions
- Want simpler implementation than balanced trees
- Need range queries
- Concurrent access (can be made lock-free)

❌ **Don't use Skip List when:**
- Need guaranteed O(log n) worst case
- Memory is extremely constrained
- Only need hash table operations

## Part 10: Test Program

```cpp
#include <iostream>
#include <vector>
#include <cassert>

void testSkipList() {
    SkipList sl;
    
    // Test insert
    cout << "=== Testing Insert ===" << endl;
    sl.insert(3);
    sl.insert(1);
    sl.insert(7);
    sl.insert(5);
    sl.insert(9);
    sl.print();
    
    // Test search
    cout << "\n=== Testing Search ===" << endl;
    assert(sl.search(5) == true);
    assert(sl.search(3) == true);
    assert(sl.search(10) == false);
    cout << "Search tests passed!" << endl;
    
    // Test delete
    cout << "\n=== Testing Delete ===" << endl;
    assert(sl.erase(5) == true);
    assert(sl.erase(5) == false);  // Already deleted
    assert(sl.search(5) == false);
    sl.print();
    
    // Test range query
    cout << "\n=== Testing Range Query ===" << endl;
    vector<int> range = sl.rangeQuery(3, 9);
    for (int val : range) {
        cout << val << " ";
    }
    cout << endl;
    
    // Test getAll
    cout << "\n=== All Elements ===" << endl;
    vector<int> all = sl.getAll();
    for (int val : all) {
        cout << val << " ";
    }
    cout << endl;
    
    cout << "\nAll tests passed!" << endl;
}

int main() {
    testSkipList();
    return 0;
}
```

## Part 11: Optimization Techniques

### 1. Deterministic Skip List

Instead of random levels, use deterministic structure:

```cpp
int deterministicLevel(int num) {
    // Use bit manipulation to determine level
    int level = 1;
    while ((num & (1 << (level - 1))) == 0 && level < maxLevel) {
        level++;
    }
    return level;
}
```

### 2. Memory Pool

Reduce allocation overhead:

```cpp
class MemoryPool {
private:
    vector<SkipListNode*> pool;
    int poolSize;
    
public:
    SkipListNode* allocate(int val, int level) {
        if (pool.empty()) {
            return new SkipListNode(val, level);
        }
        SkipListNode* node = pool.back();
        pool.pop_back();
        node->val = val;
        node->next.resize(level, nullptr);
        return node;
    }
    
    void deallocate(SkipListNode* node) {
        pool.push_back(node);
    }
};
```

### 3. Lock-Free Skip List

For concurrent access:

```cpp
#include <atomic>

class LockFreeSkipListNode {
public:
    int val;
    vector<atomic<LockFreeSkipListNode*>> next;
    
    LockFreeSkipListNode(int value, int level) 
        : val(value), next(level) {
        for (int i = 0; i < level; i++) {
            next[i] = nullptr;
        }
    }
};
```

## Key Takeaways

### Skip List Fundamentals

1. **Structure**: Multiple sorted linked lists
2. **Randomization**: Each node appears in level i with probability 1/2^i
3. **Search**: Start from top, go down when needed
4. **Insert/Delete**: Update pointers at all levels

### Critical Points

1. **Random level generation**: Crucial for performance
2. **Pointer updates**: Must update all levels correctly
3. **Head node**: Special node with INT_MIN value
4. **Level management**: Track current max level

### Interview Tips

1. **Start simple**: Explain concept first
2. **Draw diagrams**: Visual representation helps
3. **Handle edge cases**: Empty list, duplicate values
4. **Discuss trade-offs**: Space vs. time, simplicity vs. guarantees
5. **Mention optimizations**: Memory pool, lock-free, etc.

## Summary

Skip List is an elegant probabilistic data structure:

- **Average performance**: O(log n) for all operations
- **Simplicity**: Easier than balanced trees
- **Dynamic**: Handles frequent updates well
- **Flexibility**: Easy to extend with range queries

**Use cases:**
- Database indexes
- Priority queues
- Range queries
- Concurrent data structures

Understanding Skip Lists demonstrates:
- Knowledge of probabilistic data structures
- Ability to balance simplicity and performance
- Understanding of trade-offs in system design

This is valuable for storage systems companies like Pure Storage, where efficient data structures are crucial.

