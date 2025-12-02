---
layout: post
title: "Pure Storage Interview: O(1) Set Data Structure Implementation"
date: 2025-11-16
categories: [Interview Preparation, Pure Storage, Technical Interview, Data Structures]
excerpt: "Complete O(1) Set implementation: efficient set data structure with O(1) add, remove, lookup, clear operations and O(k) iterate, where k is the number of elements. Uses two arrays for optimal performance."
---

## Problem Overview

**Pure Storage Interview: O(1) Set**

Design a Set data structure for integers in range [0, N] with the following operations:
- `add(int x)`: Add element x - **O(1)**
- `remove(int x)`: Remove element x - **O(1)**
- `lookup(int x)`: Check if x exists - **O(1)**
- `clear()`: Remove all elements - **O(1)**
- `iterate(function<void(int)>)`: Iterate over all elements - **O(k)** where k is number of elements

### Constraints

- Elements are in range [0, N]
- No duplicates allowed
- Cannot use built-in HashMap/Set
- Must use arrays (int[])
- Cannot use garbage collection for `clear()` (cannot just allocate new array)

## Part 1: Approach 1 - Random Access Array (Index Array)

### Concept

Use a single array where `arr[i]` indicates whether value `i` is in the set.

```cpp
class SetApproach1 {
private:
    vector<int> indexArray;  // indexArray[i] = 1 if i is in set, 0 otherwise
    int maxValue;
    
public:
    SetApproach1(int n) : maxValue(n) {
        indexArray.resize(n + 1, 0);
    }
    
    void add(int x) {
        if (x < 0 || x > maxValue) return;
        indexArray[x] = 1;  // O(1)
    }
    
    void remove(int x) {
        if (x < 0 || x > maxValue) return;
        indexArray[x] = 0;  // O(1)
    }
    
    bool lookup(int x) {
        if (x < 0 || x > maxValue) return false;
        return indexArray[x] == 1;  // O(1)
    }
    
    void clear() {
        // Need to set all to 0 - O(N)
        for (int i = 0; i <= maxValue; i++) {
            indexArray[i] = 0;
        }
    }
    
    void iterate(function<void(int)> f) {
        // Need to check all N elements - O(N)
        for (int i = 0; i <= maxValue; i++) {
            if (indexArray[i] == 1) {
                f(i);
            }
        }
    }
};
```

### Complexity Analysis

| Operation | Time Complexity |
|-----------|----------------|
| `add` | O(1) ✅ |
| `remove` | O(1) ✅ |
| `lookup` | O(1) ✅ |
| `clear` | O(N) ❌ |
| `iterate` | O(N) ❌ |

**Problem**: `clear()` and `iterate()` are O(N), not optimal.

## Part 2: Approach 2 - Sequential Array (Element Array)

### Concept

Use a single array to store elements sequentially, maintaining order.

```cpp
class SetApproach2 {
private:
    vector<int> elementArray;  // Stores elements in order
    int maxValue;
    
public:
    SetApproach2(int n) : maxValue(n) {
        elementArray.reserve(n + 1);
    }
    
    void add(int x) {
        if (x < 0 || x > maxValue) return;
        // Check if already exists - O(k)
        for (int val : elementArray) {
            if (val == x) return;
        }
        elementArray.push_back(x);  // O(1) amortized
    }
    
    void remove(int x) {
        // Find and remove - O(k)
        for (auto it = elementArray.begin(); it != elementArray.end(); it++) {
            if (*it == x) {
                elementArray.erase(it);
                return;
            }
        }
    }
    
    bool lookup(int x) {
        // Linear search - O(k)
        for (int val : elementArray) {
            if (val == x) return true;
        }
        return false;
    }
    
    void clear() {
        elementArray.clear();  // O(1) - just reset size
    }
    
    void iterate(function<void(int)> f) {
        // Iterate only valid elements - O(k)
        for (int val : elementArray) {
            f(val);
        }
    }
};
```

### Complexity Analysis

| Operation | Time Complexity |
|-----------|----------------|
| `add` | O(k) ❌ (need to check duplicates) |
| `remove` | O(k) ❌ |
| `lookup` | O(k) ❌ |
| `clear` | O(1) ✅ |
| `iterate` | O(k) ✅ |

**Problem**: `add()`, `remove()`, and `lookup()` are O(k), not O(1).

## Part 3: Optimal Solution - Combined Approach

### Key Insight

Combine both approaches:
- **Index Array**: Maps value → position in element array (for O(1) lookup)
- **Element Array**: Stores elements sequentially (for O(k) iterate and O(1) clear)

### Data Structure

```cpp
class O1Set {
private:
    vector<int> indexMap;      // indexMap[value] = position in elementArray
    vector<int> elementArray;  // Stores actual elements
    int size;                  // Current number of elements
    int maxValue;
    static const int NOT_IN_SET = -1;
    
public:
    O1Set(int n) : maxValue(n), size(0) {
        indexMap.resize(n + 1, NOT_IN_SET);
        elementArray.reserve(n + 1);
    }
};
```

### How It Works

**Example with N = 10:**
```
After add(3), add(7), add(1):

indexMap:    [0] [1] [2] [3] [4] [5] [6] [7] [8] [9] [10]
             -1   2  -1   0  -1  -1  -1   1  -1  -1  -1
             
elementArray: [3, 7, 1]
              index: 0  1  2
              
size = 3
```

**Relationship:**
- `indexMap[3] = 0` means value 3 is at position 0 in elementArray
- `elementArray[0] = 3` confirms this
- `indexMap[7] = 1` means value 7 is at position 1
- `elementArray[1] = 7` confirms this

## Part 4: Implementation - Add Operation

### Algorithm

1. Check if element already exists using `indexMap`
2. If not exists, add to end of `elementArray`
3. Update `indexMap[value] = position`

```cpp
void add(int x) {
    if (x < 0 || x > maxValue) return;
    
    // Check if already exists - O(1)
    if (indexMap[x] != NOT_IN_SET) {
        return;  // Already in set
    }
    
    // Add to element array
    int position = size;
    elementArray.push_back(x);
    indexMap[x] = position;
    size++;
}
```

**Time Complexity**: O(1)

## Part 5: Implementation - Remove Operation

### Algorithm

1. Check if element exists using `indexMap`
2. If exists, swap with last element in `elementArray`
3. Update `indexMap` for swapped element
4. Remove last element and update `indexMap[x] = NOT_IN_SET`

```cpp
void remove(int x) {
    if (x < 0 || x > maxValue) return;
    
    // Check if exists - O(1)
    int position = indexMap[x];
    if (position == NOT_IN_SET) {
        return;  // Not in set
    }
    
    // Swap with last element
    int lastElement = elementArray[size - 1];
    elementArray[position] = lastElement;
    indexMap[lastElement] = position;
    
    // Remove last element
    elementArray.pop_back();
    indexMap[x] = NOT_IN_SET;
    size--;
}
```

**Key Point**: Swapping with last element allows O(1) removal!

**Time Complexity**: O(1)

### Example

```
Before remove(3):
indexMap:    [3]=0, [7]=1, [1]=2
elementArray: [3, 7, 1]
size = 3

After remove(3):
- Swap elementArray[0] with elementArray[2]
- elementArray: [1, 7]
- Update indexMap[1] = 0
- Set indexMap[3] = NOT_IN_SET
- size = 2
```

## Part 6: Implementation - Lookup Operation

### Algorithm

Simply check `indexMap[x]` - if not `NOT_IN_SET`, element exists.

```cpp
bool lookup(int x) {
    if (x < 0 || x > maxValue) return false;
    return indexMap[x] != NOT_IN_SET;
}
```

**Time Complexity**: O(1)

## Part 7: Implementation - Clear Operation

### Algorithm

Cannot clear arrays (would be O(N)). Instead, reset size counter.

```cpp
void clear() {
    // Reset size - O(1)
    // Don't clear arrays (would be O(N))
    // Just mark all as invalid by resetting size
    size = 0;
    
    // Note: We don't need to reset indexMap because:
    // - New adds will overwrite old values
    // - Lookup checks size implicitly through indexMap
    // But for correctness, we should reset indexMap entries
    // However, that would be O(N)...
}
```

### Problem: Index Map Cleanup

If we don't reset `indexMap`, old values remain. But resetting is O(N).

### Solution: Version Counter

Use a version counter to invalidate old entries:

```cpp
class O1Set {
private:
    vector<int> indexMap;
    vector<int> elementArray;
    vector<int> versionMap;  // versionMap[value] = version when added
    int size;
    int maxValue;
    int currentVersion;
    static const int NOT_IN_SET = -1;
    
public:
    O1Set(int n) : maxValue(n), size(0), currentVersion(0) {
        indexMap.resize(n + 1, NOT_IN_SET);
        versionMap.resize(n + 1, -1);
        elementArray.reserve(n + 1);
    }
    
    void clear() {
        size = 0;
        currentVersion++;  // Invalidate all entries - O(1)!
    }
    
    bool lookup(int x) {
        if (x < 0 || x > maxValue) return false;
        int pos = indexMap[x];
        if (pos == NOT_IN_SET) return false;
        // Check if version matches
        return versionMap[x] == currentVersion && pos < size;
    }
    
    void add(int x) {
        if (x < 0 || x > maxValue) return;
        if (lookup(x)) return;
        
        int position = size;
        elementArray.push_back(x);
        indexMap[x] = position;
        versionMap[x] = currentVersion;
        size++;
    }
    
    void remove(int x) {
        if (x < 0 || x > maxValue) return;
        int position = indexMap[x];
        if (position == NOT_IN_SET || versionMap[x] != currentVersion) {
            return;
        }
        
        int lastElement = elementArray[size - 1];
        elementArray[position] = lastElement;
        indexMap[lastElement] = position;
        versionMap[lastElement] = currentVersion;
        
        elementArray.pop_back();
        indexMap[x] = NOT_IN_SET;
        size--;
    }
};
```

**Alternative**: Reset indexMap entries lazily (only when needed), but version counter is cleaner.

## Part 8: Implementation - Iterate Operation

### Algorithm

Simply iterate over `elementArray` up to `size`.

```cpp
void iterate(function<void(int)> f) {
    for (int i = 0; i < size; i++) {
        f(elementArray[i]);
    }
}
```

**Time Complexity**: O(k) where k is number of elements ✅

## Part 9: Complete Implementation

```cpp
#include <vector>
#include <functional>
#include <iostream>
#include <algorithm>
using namespace std;

class O1Set {
private:
    vector<int> indexMap;      // indexMap[value] = position in elementArray
    vector<int> elementArray;  // Stores elements sequentially
    vector<int> versionMap;    // versionMap[value] = version when added
    int size;                  // Current number of elements
    int maxValue;
    int currentVersion;
    static const int NOT_IN_SET = -1;
    
public:
    O1Set(int n) : maxValue(n), size(0), currentVersion(0) {
        indexMap.resize(n + 1, NOT_IN_SET);
        versionMap.resize(n + 1, -1);
        elementArray.reserve(n + 1);
    }
    
    void add(int x) {
        if (x < 0 || x > maxValue) return;
        if (lookup(x)) return;  // Already exists
        
        int position = size;
        elementArray.push_back(x);
        indexMap[x] = position;
        versionMap[x] = currentVersion;
        size++;
    }
    
    void remove(int x) {
        if (x < 0 || x > maxValue) return;
        int position = indexMap[x];
        
        // Check if exists and version matches
        if (position == NOT_IN_SET || 
            versionMap[x] != currentVersion || 
            position >= size) {
            return;
        }
        
        // Swap with last element
        int lastElement = elementArray[size - 1];
        elementArray[position] = lastElement;
        indexMap[lastElement] = position;
        versionMap[lastElement] = currentVersion;
        
        // Remove last element
        elementArray.pop_back();
        indexMap[x] = NOT_IN_SET;
        size--;
    }
    
    bool lookup(int x) {
        if (x < 0 || x > maxValue) return false;
        int pos = indexMap[x];
        if (pos == NOT_IN_SET) return false;
        
        // Check version and bounds
        return versionMap[x] == currentVersion && 
               pos < size && 
               elementArray[pos] == x;
    }
    
    void clear() {
        // O(1) clear using version counter
        size = 0;
        currentVersion++;
    }
    
    void iterate(function<void(int)> f) {
        // O(k) where k is number of elements
        for (int i = 0; i < size; i++) {
            f(elementArray[i]);
        }
    }
    
    int getSize() const {
        return size;
    }
    
    bool isEmpty() const {
        return size == 0;
    }
    
    // Helper: Print for debugging
    void print() {
        cout << "Set contents: ";
        iterate([](int x) { cout << x << " "; });
        cout << endl;
    }
};
```

## Part 10: Common Bugs and Edge Cases

### Bug 1: Not Checking Bounds in Remove

```cpp
// WRONG:
void remove(int x) {
    int position = indexMap[x];
    elementArray[position] = elementArray[size - 1];  // Bug if x not in set!
}

// CORRECT:
void remove(int x) {
    int position = indexMap[x];
    if (position == NOT_IN_SET || position >= size) return;  // Check first!
    // ... rest of code
}
```

### Bug 2: Not Updating IndexMap for Swapped Element

```cpp
// WRONG:
void remove(int x) {
    int position = indexMap[x];
    elementArray[position] = elementArray[size - 1];
    indexMap[x] = NOT_IN_SET;
    // Forgot to update indexMap[lastElement]!
}

// CORRECT:
void remove(int x) {
    int position = indexMap[x];
    int lastElement = elementArray[size - 1];
    elementArray[position] = lastElement;
    indexMap[lastElement] = position;  // Must update!
    indexMap[x] = NOT_IN_SET;
}
```

### Bug 3: Clear Without Version Counter

```cpp
// WRONG: O(N) clear
void clear() {
    for (int i = 0; i <= maxValue; i++) {
        indexMap[i] = NOT_IN_SET;  // O(N)!
    }
    size = 0;
}

// CORRECT: O(1) clear with version
void clear() {
    size = 0;
    currentVersion++;  // O(1)!
}
```

### Edge Cases

1. **Duplicate adds**: Should be ignored
2. **Remove non-existent**: Should do nothing
3. **Out of range**: Should be rejected
4. **Clear empty set**: Should work correctly
5. **Iterate empty set**: Should do nothing

## Part 11: Test Program

```cpp
#include <iostream>
#include <cassert>
#include <vector>

void testO1Set() {
    O1Set set(10);
    
    // Test add
    cout << "=== Testing Add ===" << endl;
    set.add(3);
    set.add(7);
    set.add(1);
    set.add(3);  // Duplicate - should be ignored
    assert(set.getSize() == 3);
    set.print();
    
    // Test lookup
    cout << "\n=== Testing Lookup ===" << endl;
    assert(set.lookup(3) == true);
    assert(set.lookup(7) == true);
    assert(set.lookup(5) == false);
    cout << "Lookup tests passed!" << endl;
    
    // Test remove
    cout << "\n=== Testing Remove ===" << endl;
    set.remove(3);
    assert(set.lookup(3) == false);
    assert(set.getSize() == 2);
    set.print();
    
    // Test iterate
    cout << "\n=== Testing Iterate ===" << endl;
    vector<int> collected;
    set.iterate([&collected](int x) {
        collected.push_back(x);
    });
    assert(collected.size() == 2);
    cout << "Iterate test passed!" << endl;
    
    // Test clear
    cout << "\n=== Testing Clear ===" << endl;
    set.clear();
    assert(set.isEmpty() == true);
    assert(set.lookup(7) == false);
    set.add(5);
    assert(set.lookup(5) == true);
    cout << "Clear test passed!" << endl;
    
    cout << "\nAll tests passed!" << endl;
}

int main() {
    testO1Set();
    return 0;
}
```

## Part 12: Complexity Analysis

### Final Complexity

| Operation | Time Complexity | Space Complexity |
|-----------|----------------|------------------|
| `add` | O(1) ✅ | O(1) |
| `remove` | O(1) ✅ | O(1) |
| `lookup` | O(1) ✅ | O(1) |
| `clear` | O(1) ✅ | O(1) |
| `iterate` | O(k) ✅ | O(1) |

Where k is the number of elements in the set.

### Space Complexity

- **Index Map**: O(N)
- **Element Array**: O(N) worst case
- **Version Map**: O(N)
- **Total**: O(N)

## Key Takeaways

### Design Principles

1. **Two arrays**: Index map for O(1) lookup, element array for O(k) iterate
2. **Swap trick**: Remove by swapping with last element
3. **Version counter**: O(1) clear without resetting arrays
4. **Bounds checking**: Always validate indices

### Critical Points

1. **Remove operation**: Must update indexMap for swapped element
2. **Clear operation**: Use version counter, not array reset
3. **Lookup validation**: Check version and bounds
4. **Duplicate handling**: Check before adding

### Interview Tips

1. **Start with naive approaches**: Show understanding of trade-offs
2. **Identify bottlenecks**: Which operations are slow?
3. **Combine solutions**: Use best parts of each approach
4. **Handle edge cases**: Bounds, duplicates, empty set
5. **Discuss space**: O(N) space for O(1) operations

## Summary

This problem demonstrates:

- **Data structure design**: Combining multiple structures for optimal performance
- **Trade-off analysis**: Space vs. time complexity
- **Clever optimizations**: Swap trick, version counter
- **Edge case handling**: Bounds checking, duplicate prevention

**Key insight**: Use two arrays - one for O(1) lookup (index map) and one for O(k) iteration (element array), with a version counter for O(1) clear.

This is a classic interview problem that tests:
- Understanding of array-based data structures
- Ability to optimize operations
- Attention to detail (bugs in remove/clear)
- System design thinking (space-time trade-offs)

Perfect for storage systems companies like Pure Storage where efficient data structures are crucial!

