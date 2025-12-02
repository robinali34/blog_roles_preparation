---
layout: post
title: "Pure Storage Interview: Buddy Bit Map (Clear Bits & Set Bits)"
date: 2025-11-16
categories: [Interview Preparation, Pure Storage, Technical Interview, Algorithms]
excerpt: "Complete solution for Buddy Bit Map problem: implementing clearBits and setBits on a complete binary tree with parent-child relationship constraints. Includes DFS vs BFS discussion and space optimization techniques."
---

## Problem Overview

**Pure Storage Interview: Buddy Bit Map**

This is a classic Pure Storage interview problem that tests your understanding of:
1. Complete binary tree data structures
2. Range-based operations on bitmaps
3. Efficient parent-child relationship updates
4. Space and time complexity optimization
5. Virtual memory and paging considerations

### Problem Statement

A **Buddy Bit Map** is represented as a complete binary tree where:
- Each node can be **0** or **1**
- A node is **1** if and only if **ALL its children are 1**
- The tree is stored as a 2D array: `bits[level][index]`
  - Level 0 has 1 node
  - Level 1 has 2 nodes
  - Level 2 has 4 nodes
  - Level h has 2^h nodes

**Operations to implement:**
1. `clearBits(offset, length)`: Clear bits in the last level from `offset` to `offset+length-1`, updating parent nodes accordingly
2. `setBits(offset, length)`: Set bits in the last level from `offset` to `offset+length-1`, updating parent nodes accordingly

### Example Tree Structure

```
Level 0:           1
                  /   \
Level 1:         1     1
                / \   / \
Level 2:       1   1 1   1
              /|\ /|\ /|\ /|\
Level 3:     1 2 3 4 5 6 7 8
```

## Data Structure

```cpp
#include <vector>
#include <queue>
#include <iostream>
using namespace std;

class BuddyBitmap {
private:
    vector<vector<int>> bits;  // bits[level][index]
    int height;                // Height of the tree (0-indexed)
    int leafCount;             // Number of leaf nodes (2^height)
    
public:
    BuddyBitmap(int height) : height(height) {
        leafCount = 1 << height;  // 2^height
        
        // Initialize tree: level i has 2^i nodes
        for (int level = 0; level <= height; level++) {
            bits.push_back(vector<int>(1 << level, 1));  // All nodes start as 1
        }
    }
    
    // Get parent index
    int getParent(int index) {
        return index / 2;
    }
    
    // Get left child index
    int getLeftChild(int index) {
        return index * 2;
    }
    
    // Get right child index
    int getRightChild(int index) {
        return index * 2 + 1;
    }
    
    // Print tree (for debugging)
    void printTree() {
        for (int level = 0; level <= height; level++) {
            cout << "Level " << level << ": ";
            for (int i = 0; i < bits[level].size(); i++) {
                cout << bits[level][i] << " ";
            }
            cout << endl;
        }
    }
};
```

## Part 1: Clear Bits Implementation

### Approach 1: Naive (Inefficient)

Clear bits one by one and update parents for each:

```cpp
void clearBitsNaive(int offset, int length) {
    int level = height;  // Last level
    
    for (int i = offset; i < offset + length && i < leafCount; i++) {
        // Clear the bit
        bits[level][i] = 0;
        
        // Update all ancestors
        int currentIndex = i;
        int currentLevel = level;
        
        while (currentLevel > 0) {
            int parentIndex = getParent(currentIndex);
            currentLevel--;
            
            // Check if both children are 0
            int leftChild = getLeftChild(parentIndex);
            int rightChild = getRightChild(parentIndex);
            
            if (bits[currentLevel + 1][leftChild] == 0 && 
                bits[currentLevel + 1][rightChild] == 0) {
                bits[currentLevel][parentIndex] = 0;
            } else {
                bits[currentLevel][parentIndex] = 0;  // At least one child is 0
            }
            
            currentIndex = parentIndex;
        }
    }
}
```

**Problem:** Time complexity is O(height × length), which is inefficient.

### Approach 2: Range-Based (Optimal)

Process the range level by level, updating parents efficiently:

```cpp
void clearBits(int offset, int length) {
    if (offset < 0 || offset >= leafCount || length <= 0) {
        return;
    }
    
    int end = min(offset + length - 1, leafCount - 1);
    int level = height;
    
    // Clear bits in the last level
    for (int i = offset; i <= end; i++) {
        bits[level][i] = 0;
    }
    
    // Update parents level by level
    int start = offset;
    end = min(offset + length - 1, leafCount - 1);
    
    for (int currentLevel = height - 1; currentLevel >= 0; currentLevel--) {
        // Convert to parent level indices
        start = start / 2;
        end = end / 2;
        
        // Update all parents in this range
        for (int i = start; i <= end; i++) {
            int leftChild = getLeftChild(i);
            int rightChild = getRightChild(i);
            
            // Parent is 1 only if both children are 1
            if (bits[currentLevel + 1][leftChild] == 1 && 
                bits[currentLevel + 1][rightChild] == 1) {
                bits[currentLevel][i] = 1;
            } else {
                bits[currentLevel][i] = 0;
            }
        }
    }
}
```

**Time Complexity:** O(height + length)
- O(length) to clear leaf level
- O(height) to update each level (range shrinks by ~half each level)

**Space Complexity:** O(1) extra space

### Optimized Range-Based Solution

```cpp
void clearBitsOptimized(int offset, int length) {
    if (offset < 0 || offset >= leafCount || length <= 0) {
        return;
    }
    
    int end = min(offset + length - 1, leafCount - 1);
    
    // Clear bits in the last level
    for (int i = offset; i <= end; i++) {
        bits[height][i] = 0;
    }
    
    // Update parents bottom-up
    int start = offset;
    end = min(offset + length - 1, leafCount - 1);
    
    for (int level = height - 1; level >= 0; level--) {
        // Map to parent level: each parent covers 2 children
        start = start / 2;
        end = end / 2;
        
        // Update parent nodes in range [start, end]
        for (int i = start; i <= end; i++) {
            int leftIdx = i * 2;
            int rightIdx = i * 2 + 1;
            
            // Parent = 1 only if both children = 1
            bits[level][i] = (bits[level + 1][leftIdx] == 1 && 
                             bits[level + 1][rightIdx] == 1) ? 1 : 0;
        }
    }
}
```

## Part 2: Set Bits Implementation

Set bits is more complex because we need to check if setting a bit makes both children 1, which would require setting the parent to 1.

### Approach

```cpp
void setBits(int offset, int length) {
    if (offset < 0 || offset >= leafCount || length <= 0) {
        return;
    }
    
    int end = min(offset + length - 1, leafCount - 1);
    
    // Set bits in the last level
    for (int i = offset; i <= end; i++) {
        bits[height][i] = 1;
    }
    
    // Update parents bottom-up
    int start = offset;
    end = min(offset + length - 1, leafCount - 1);
    
    for (int level = height - 1; level >= 0; level--) {
        // Map to parent level
        start = start / 2;
        end = end / 2;
        
        // Update parent nodes in range [start, end]
        for (int i = start; i <= end; i++) {
            int leftIdx = i * 2;
            int rightIdx = i * 2 + 1;
            
            // Parent = 1 only if both children = 1
            bits[level][i] = (bits[level + 1][leftIdx] == 1 && 
                             bits[level + 1][rightIdx] == 1) ? 1 : 0;
        }
    }
}
```

**Key Difference from clearBits:**
- For `clearBits`: If any child is 0, parent becomes 0
- For `setBits`: Parent becomes 1 only if **both** children are 1

The logic is actually the same - we check both children and set parent accordingly.

## Complete Solution

```cpp
#include <vector>
#include <queue>
#include <iostream>
#include <algorithm>
using namespace std;

class BuddyBitmap {
private:
    vector<vector<int>> bits;
    int height;
    int leafCount;
    
public:
    BuddyBitmap(int h) : height(h) {
        leafCount = 1 << height;
        
        // Initialize: level i has 2^i nodes, all set to 1
        for (int level = 0; level <= height; level++) {
            bits.push_back(vector<int>(1 << level, 1));
        }
    }
    
    // Clear bits from offset to offset+length-1
    void clearBits(int offset, int length) {
        if (offset < 0 || offset >= leafCount || length <= 0) {
            return;
        }
        
        int end = min(offset + length - 1, leafCount - 1);
        
        // Clear leaf level
        for (int i = offset; i <= end; i++) {
            bits[height][i] = 0;
        }
        
        // Update parents bottom-up
        int start = offset;
        end = min(offset + length - 1, leafCount - 1);
        
        for (int level = height - 1; level >= 0; level--) {
            start = start / 2;
            end = end / 2;
            
            for (int i = start; i <= end; i++) {
                int leftIdx = i * 2;
                int rightIdx = i * 2 + 1;
                
                // Parent = 1 only if both children = 1
                bits[level][i] = (bits[level + 1][leftIdx] == 1 && 
                                 bits[level + 1][rightIdx] == 1) ? 1 : 0;
            }
        }
    }
    
    // Set bits from offset to offset+length-1
    void setBits(int offset, int length) {
        if (offset < 0 || offset >= leafCount || length <= 0) {
            return;
        }
        
        int end = min(offset + length - 1, leafCount - 1);
        
        // Set leaf level
        for (int i = offset; i <= end; i++) {
            bits[height][i] = 1;
        }
        
        // Update parents bottom-up
        int start = offset;
        end = min(offset + length - 1, leafCount - 1);
        
        for (int level = height - 1; level >= 0; level--) {
            start = start / 2;
            end = end / 2;
            
            for (int i = start; i <= end; i++) {
                int leftIdx = i * 2;
                int rightIdx = i * 2 + 1;
                
                // Parent = 1 only if both children = 1
                bits[level][i] = (bits[level + 1][leftIdx] == 1 && 
                                 bits[level + 1][rightIdx] == 1) ? 1 : 0;
            }
        }
    }
    
    // Print tree for visualization
    void printTree() {
        for (int level = 0; level <= height; level++) {
            cout << "Level " << level << ": ";
            for (int i = 0; i < bits[level].size(); i++) {
                cout << bits[level][i] << " ";
            }
            cout << endl;
        }
        cout << endl;
    }
    
    // Get bit value at leaf level
    int getBit(int offset) {
        if (offset < 0 || offset >= leafCount) return -1;
        return bits[height][offset];
    }
};

// Test program
int main() {
    BuddyBitmap bm(3);  // Height 3, 8 leaf nodes
    
    cout << "Initial tree:" << endl;
    bm.printTree();
    
    cout << "After clearBits(2, 3):" << endl;
    bm.clearBits(2, 3);  // Clear bits 2, 3, 4
    bm.printTree();
    
    cout << "After setBits(1, 2):" << endl;
    bm.setBits(1, 2);  // Set bits 1, 2
    bm.printTree();
    
    return 0;
}
```

## BFS vs DFS Discussion

### BFS Implementation (Using Queue)

```cpp
void clearBitsBFS(int offset, int length) {
    if (offset < 0 || offset >= leafCount || length <= 0) {
        return;
    }
    
    int end = min(offset + length - 1, leafCount - 1);
    
    // Clear leaf level
    for (int i = offset; i <= end; i++) {
        bits[height][i] = 0;
    }
    
    // BFS: Process level by level
    queue<pair<int, int>> q;  // {level, start_index}
    q.push({height, offset});
    q.push({height, end});
    
    // Process each level
    for (int level = height - 1; level >= 0; level--) {
        int levelStart = offset / (1 << (height - level));
        int levelEnd = end / (1 << (height - level));
        
        // Update all nodes in range
        for (int i = levelStart; i <= levelEnd; i++) {
            int leftIdx = i * 2;
            int rightIdx = i * 2 + 1;
            
            bits[level][i] = (bits[level + 1][leftIdx] == 1 && 
                             bits[level + 1][rightIdx] == 1) ? 1 : 0;
        }
    }
}
```

### Space Complexity Analysis

**BFS with Queue:**
- Worst case: Queue stores indices for one level
- For level h: 2^h nodes
- **Space: O(2^height)** in worst case

**Optimization:**
- Instead of storing all indices, reset start/end for each level
- **Space: O(1)** extra space

```cpp
void clearBitsOptimizedBFS(int offset, int length) {
    if (offset < 0 || offset >= leafCount || length <= 0) {
        return;
    }
    
    int end = min(offset + length - 1, leafCount - 1);
    
    // Clear leaf level
    for (int i = offset; i <= end; i++) {
        bits[height][i] = 0;
    }
    
    // BFS: Process level by level, reset start/end each time
    for (int level = height - 1; level >= 0; level--) {
        // Reset range for this level (no queue needed!)
        int start = offset / (1 << (height - level));
        int levelEnd = end / (1 << (height - level));
        
        // Update all nodes in range
        for (int i = start; i <= levelEnd; i++) {
            int leftIdx = i * 2;
            int rightIdx = i * 2 + 1;
            
            bits[level][i] = (bits[level + 1][leftIdx] == 1 && 
                             bits[level + 1][rightIdx] == 1) ? 1 : 0;
        }
    }
}
```

### DFS vs BFS: Virtual Memory Considerations

**DFS Issues:**
1. **Page in/Page out cost**: When tree is too large for RAM, DFS causes frequent page faults
   - DFS traverses deep paths, accessing non-contiguous memory
   - High cost of loading pages from disk
2. **Spatial locality**: DFS doesn't exploit cache line locality well
   - Cache loads consecutive blocks together
   - DFS jumps around, missing cache benefits
   - **However**: This cost is minor compared to paging

**BFS Advantages:**
1. **Better spatial locality**: Processes level by level (consecutive memory)
2. **Fewer page faults**: Accesses memory in order
3. **Cache-friendly**: Exploits cache line loading

**Key Insight:** For large trees that don't fit in RAM, **paging costs dominate**, making BFS more efficient than DFS.

## Time Complexity Analysis

### Why O(height + length) not O(height × length)?

**Naive approach:** O(height × length)
- For each of `length` bits, update `height` levels
- Total: length × height operations

**Range-based approach:** O(height + length)
- Clear leaf level: O(length)
- Update each level: O(range_size)
- Range shrinks by ~half each level: length + length/2 + length/4 + ... ≈ 2×length
- But actually, we process at most `height` levels with shrinking ranges
- **Key**: We don't process each bit individually at each level
- We process the **range** at each level, which is O(1) per level after the first

**More precise analysis:**
- Leaf level: O(length)
- Each parent level: O(range_size), where range_size ≤ length/2^(level)
- Sum: length + length/2 + length/4 + ... ≤ 2×length
- Plus height levels: O(height + length)

## Bonus: Search Bits (Find Consecutive Empty Slots)

Find `n` consecutive empty (0) slots:

```cpp
int searchBits(int n) {
    if (n <= 0 || n > leafCount) {
        return -1;
    }
    
    int consecutive = 0;
    int start = -1;
    
    for (int i = 0; i < leafCount; i++) {
        if (bits[height][i] == 0) {
            if (consecutive == 0) {
                start = i;
            }
            consecutive++;
            
            if (consecutive >= n) {
                return start;
            }
        } else {
            consecutive = 0;
            start = -1;
        }
    }
    
    return -1;  // Not found
}
```

**Optimized with early termination:**

```cpp
int searchBitsOptimized(int n) {
    if (n <= 0 || n > leafCount) {
        return -1;
    }
    
    int consecutive = 0;
    int start = -1;
    
    for (int i = 0; i < leafCount; i++) {
        if (bits[height][i] == 0) {
            if (consecutive == 0) {
                start = i;
            }
            consecutive++;
            
            if (consecutive >= n) {
                return start;
            }
        } else {
            consecutive = 0;
        }
    }
    
    return -1;
}
```

## Summary

### Key Takeaways

1. **Range-based operations**: Process ranges level by level, not bit by bit
2. **Time complexity**: O(height + length), not O(height × length)
3. **Space optimization**: Reset start/end for each level instead of using queue
4. **BFS vs DFS**: BFS is better for large trees due to better memory locality
5. **Parent update rule**: Parent = 1 if and only if both children = 1

### Interview Tips

1. **Clarify the problem**: Ask about data structure representation
2. **Start with naive**: Show understanding, then optimize
3. **Discuss trade-offs**: BFS vs DFS, space vs time
4. **Consider edge cases**: Invalid offsets, lengths, boundaries
5. **Explain complexity**: Why O(height + length) not O(height × length)

This problem tests your ability to:
- Work with tree data structures
- Optimize range operations
- Understand memory hierarchy and paging
- Balance time and space complexity

