---
layout: post
title: "Pure Storage Interview: Sort Colors Variant - Minimize Swaps"
date: 2025-11-16
categories: [Interview Preparation, Pure Storage, Technical Interview, Algorithms]
excerpt: "Pure Storage variation of Sort Colors problem: minimize swap operations to move D blocks to front, with follow-up to maximize G blocks at end. Storage-level optimization focused on reducing swap count rather than iterations."
---

## Problem Overview

**Pure Storage Interview: Sort Colors Variant**

This is a variation of the classic "Sort Colors" problem (LeetCode 75), but with a critical difference: **minimize swap operations** rather than just sorting efficiently.

### Problem Context

In a storage system, file blocks are marked with three types:
- **D** (like 0 in Sort Colors)
- **F** (like 1 in Sort Colors)  
- **G** (like 2 in Sort Colors)

The goal is to reorganize blocks with **minimum swap operations**, which is crucial at the storage level where swap operations are expensive.

### Problem Statement

**Part 1:** Move all D blocks to the front with minimum swaps. The relative order of F and G blocks doesn't matter.

**Part 2:** Move all D blocks to the front while **maximizing** the number of G blocks at the end, **without increasing** the total swap count from Part 1.

## Understanding Swap Minimization

### Why Swaps Matter More Than Iterations?

In storage systems:
- **Swap operations** involve physical disk I/O operations
- Each swap may require reading from one location and writing to another
- **Iterations** are just pointer movements in memory
- Minimizing swaps directly reduces I/O overhead

### Key Insight

Unlike the standard Dutch National Flag algorithm which minimizes iterations, we need to:
1. Identify which elements need to be moved
2. Find optimal swap pairs to minimize total swaps
3. Use a greedy approach: swap D with non-D elements that are furthest from their target positions

## Part 1: Move All D to Front with Minimum Swaps

### Approach

**Strategy:** Use two pointers approach:
- `left`: Points to the first position that should contain D (but currently doesn't)
- `right`: Points to the last position, scanning backwards for D blocks

**Algorithm:**
1. Find the rightmost D that's not in its correct position
2. Find the leftmost non-D that should be swapped
3. Swap them
4. Repeat until all D's are at the front

### Solution 1: Two-Pointer Approach

```cpp
#include <vector>
#include <string>
#include <iostream>
using namespace std;

class ColorSorter {
public:
    // Part 1: Move all D to front with minimum swaps
    int moveDToFront(vector<char>& blocks) {
        int n = blocks.size();
        int swapCount = 0;
        int left = 0;  // First position that should have D
        
        // Find all D's and count them
        int dCount = 0;
        for (char c : blocks) {
            if (c == 'D') dCount++;
        }
        
        // Two-pointer approach
        int right = n - 1;
        
        while (left < dCount) {
            // If current position already has D, move left pointer
            if (blocks[left] == 'D') {
                left++;
                continue;
            }
            
            // Find rightmost D that's not in correct position
            while (right >= dCount && blocks[right] != 'D') {
                right--;
            }
            
            // If we found a D to swap
            if (right >= dCount && blocks[right] == 'D') {
                swap(blocks[left], blocks[right]);
                swapCount++;
                left++;
                right--;
            } else {
                // No more D's to move
                break;
            }
        }
        
        return swapCount;
    }
};
```

### Solution 2: Optimal Greedy Approach

The above solution works, but we can optimize further by being smarter about which elements to swap:

```cpp
int moveDToFrontOptimal(vector<char>& blocks) {
    int n = blocks.size();
    int swapCount = 0;
    
    // Count D's
    int dCount = 0;
    for (char c : blocks) {
        if (c == 'D') dCount++;
    }
    
    // Two pointers: left for first non-D position, right scanning for D
    int left = 0;
    int right = n - 1;
    
    while (left < dCount && right >= dCount) {
        // Skip D's already in correct position
        while (left < dCount && blocks[left] == 'D') {
            left++;
        }
        
        // Skip non-D's already in correct position (right side)
        while (right >= dCount && blocks[right] != 'D') {
            right--;
        }
        
        // Swap if both pointers are valid
        if (left < dCount && right >= dCount) {
            swap(blocks[left], blocks[right]);
            swapCount++;
            left++;
            right--;
        }
    }
    
    return swapCount;
}
```

### Explanation

**Example:** `['F', 'D', 'G', 'D', 'F', 'G']`

1. `dCount = 2` (positions 1 and 3 have D)
2. Target: First 2 positions should be D
3. `left = 0` (has 'F', should be 'D')
4. `right = 5` (has 'G', not D)
5. Move `right` backwards: `right = 4` (has 'F'), `right = 3` (has 'D' ✓)
6. Swap `blocks[0]` ('F') with `blocks[3]` ('D')
7. Result: `['D', 'D', 'G', 'F', 'F', 'G']`
8. `left = 1` (has 'D' ✓), `left = 2` (has 'G', but `left >= dCount`, done)
9. **Total swaps: 1**

**Time Complexity:** O(n)  
**Space Complexity:** O(1)

## Part 2: Move D to Front + Maximize G at End

### Problem Statement

Move all D blocks to the front while **maximizing** the number of G blocks at the end, **without increasing** the swap count from Part 1.

**Key Constraint:** We can only "incidentally" move G to the end while moving D to front. We cannot do additional swaps just to move G.

### Approach

**Strategy:** When swapping D with a non-D element, prefer swapping with F over G when possible, because:
- F can stay in the middle (we don't care about F/G order)
- G should ideally end up at the back
- If we swap D with G, G moves forward (away from end)
- If we swap D with F, G naturally stays/ends up at the back

**Algorithm:**
1. Use the same two-pointer approach as Part 1
2. When choosing which non-D element to swap with D, prefer F over G
3. This maximizes G's ending up at the end without extra swaps

### Solution

```cpp
// Part 2: Move D to front while maximizing G at end
int moveDToFrontMaximizeG(vector<char>& blocks) {
    int n = blocks.size();
    int swapCount = 0;
    
    // Count D's
    int dCount = 0;
    for (char c : blocks) {
        if (c == 'D') dCount++;
    }
    
    // Three regions: [0, dCount) should be D, [dCount, n) can be F or G
    // Strategy: When swapping D with non-D, prefer F over G
    // This leaves more G's at the end naturally
    
    int left = 0;
    int rightF = n - 1;  // Prefer to swap D with F
    int rightG = n - 1;  // Fallback: swap D with G if no F available
    
    while (left < dCount) {
        // Skip D's already in correct position
        if (blocks[left] == 'D') {
            left++;
            continue;
        }
        
        // Try to find F first (prefer swapping D with F)
        while (rightF >= dCount && (rightF < left || blocks[rightF] != 'F')) {
            rightF--;
        }
        
        // If found F, swap with it
        if (rightF >= dCount && blocks[rightF] == 'F') {
            swap(blocks[left], blocks[rightF]);
            swapCount++;
            left++;
            rightF--;
            continue;
        }
        
        // If no F found, look for G
        while (rightG >= dCount && (rightG < left || blocks[rightG] != 'G')) {
            rightG--;
        }
        
        // Swap with G if found
        if (rightG >= dCount && blocks[rightG] == 'G') {
            swap(blocks[left], blocks[rightG]);
            swapCount++;
            left++;
            rightG--;
        } else {
            // No more non-D elements to swap
            break;
        }
    }
    
    return swapCount;
}
```

### More Elegant Solution

Here's a cleaner approach that's easier to understand:

```cpp
int moveDToFrontMaximizeG(vector<char>& blocks) {
    int n = blocks.size();
    int swapCount = 0;
    
    // Count D's
    int dCount = 0;
    for (char c : blocks) {
        if (c == 'D') dCount++;
    }
    
    int left = 0;
    int right = n - 1;
    
    // First pass: Move D to front, prefer swapping with F
    while (left < dCount && right >= dCount) {
        // Skip D's already in correct position
        if (blocks[left] == 'D') {
            left++;
            continue;
        }
        
        // Find rightmost F first (prefer F over G)
        int fPos = right;
        while (fPos >= dCount && blocks[fPos] != 'F') {
            fPos--;
        }
        
        // If found F, swap with it
        if (fPos >= dCount && blocks[fPos] == 'F') {
            swap(blocks[left], blocks[fPos]);
            swapCount++;
            left++;
            right = fPos - 1;
            continue;
        }
        
        // If no F, find rightmost G
        int gPos = right;
        while (gPos >= dCount && blocks[gPos] != 'G') {
            gPos--;
        }
        
        // Swap with G if found
        if (gPos >= dCount && blocks[gPos] == 'G') {
            swap(blocks[left], blocks[gPos]);
            swapCount++;
            left++;
            right = gPos - 1;
        } else {
            // No more non-D elements
            break;
        }
    }
    
    return swapCount;
}
```

### Explanation

**Example:** `['F', 'D', 'G', 'D', 'F', 'G']`

**Part 1 result:** `['D', 'D', 'G', 'F', 'F', 'G']` (1 swap)

**Part 2 with optimization:**
1. `left = 0` (has 'F', should be 'D')
2. Look for F first: `right = 5` ('G'), `right = 4` ('F' ✓)
3. Swap `blocks[0]` ('F') with `blocks[4]` ('F')
4. Result: `['F', 'D', 'G', 'D', 'F', 'G']` - wait, this doesn't help!

Actually, let me reconsider. The key insight is:
- We need to swap a D (from right side) with a non-D (from left side)
- When we have a choice between F and G on the left side, we should swap D with F
- This leaves G's in place, maximizing G's at the end

Let me revise the algorithm:

```cpp
int moveDToFrontMaximizeGRevised(vector<char>& blocks) {
    int n = blocks.size();
    int swapCount = 0;
    
    // Count D's
    int dCount = 0;
    for (char c : blocks) {
        if (c == 'D') dCount++;
    }
    
    // Find D's that are not in front positions
    // Find non-D's that are in front positions
    // When swapping, prefer swapping D (from back) with F (from front) over G (from front)
    
    int left = 0;
    int right = n - 1;
    
    while (left < dCount && right >= dCount) {
        // Skip D's already in correct position
        if (blocks[left] == 'D') {
            left++;
            continue;
        }
        
        // Find rightmost D
        while (right >= dCount && blocks[right] != 'D') {
            right--;
        }
        
        if (right < dCount) break;
        
        // Now we have: blocks[left] is non-D, blocks[right] is D
        // If left has F, prefer it (G stays in place, maximizing G at end)
        // If left has G, we have to swap it (no choice)
        
        swap(blocks[left], blocks[right]);
        swapCount++;
        left++;
        right--;
    }
    
    // The optimization happens naturally: when we swap D with F,
    // G's that were already towards the end stay there
    // We don't need extra logic because we're scanning from right to left
    
    return swapCount;
}
```

Actually, the real optimization is more subtle. Let me think about this more carefully:

**Key Insight for Part 2:**
- We want to maximize G at the end
- When we swap D (from right) with a non-D (from left), we're moving that non-D to the right
- If we swap D with F, F moves right (doesn't matter)
- If we swap D with G, G moves right (away from end - bad!)
- So we should **prefer swapping D with F** when we have a choice

But in the two-pointer approach, we're scanning from right to left for D's, and from left to right for non-D's. The choice of which non-D to swap with depends on what's on the left.

Actually, I think the correct approach is:
1. Find all D's that need to move to front
2. Find all non-D's in front positions
3. When pairing them for swaps, prefer pairing D with F over pairing D with G

Let me write the correct solution:

```cpp
int moveDToFrontMaximizeGCorrect(vector<char>& blocks) {
    int n = blocks.size();
    int swapCount = 0;
    
    // Count D's
    int dCount = 0;
    for (char c : blocks) {
        if (c == 'D') dCount++;
    }
    
    // Strategy: When we need to swap a D (from back) with a non-D (from front),
    // prefer swapping with F over G, because:
    // - Swapping D with F: F moves to back (doesn't matter)
    // - Swapping D with G: G moves forward (away from end - bad!)
    
    int left = 0;
    int right = n - 1;
    
    // First, identify positions that need D
    while (left < dCount) {
        if (blocks[left] == 'D') {
            left++;
            continue;
        }
        
        // Find rightmost D
        while (right >= dCount && blocks[right] != 'D') {
            right--;
        }
        
        if (right < dCount) break;
        
        // Now: blocks[left] is non-D (F or G), blocks[right] is D
        // If blocks[left] is F, swap (preferred)
        // If blocks[left] is G, we still need to swap, but try to minimize
        
        // Actually, we can't avoid swapping if left has G
        // The optimization is: when we have multiple D's to swap,
        // prioritize swapping with F positions over G positions
        
        swap(blocks[left], blocks[right]);
        swapCount++;
        left++;
        right--;
    }
    
    return swapCount;
}
```

Wait, I think I'm overcomplicating this. Let me re-read the problem statement:

"Move all D blocks to the front while maximizing the number of G blocks at the end, without increasing swap count."

The key is: **when we have a choice** of which element to swap D with, we should prefer F over G. But in a simple two-pointer approach, we don't really have a choice - we just swap whatever is at the left with whatever D we find on the right.

The optimization comes from being smarter about **which D to swap with which non-D**. Let me write a solution that considers this:

```cpp
int moveDToFrontMaximizeGFinal(vector<char>& blocks) {
    int n = blocks.size();
    int swapCount = 0;
    
    // Count D's
    int dCount = 0;
    for (char c : blocks) {
        if (c == 'D') dCount++;
    }
    
    // Collect positions that need D (non-D in [0, dCount))
    vector<int> needD;  // positions that should have D but don't
    for (int i = 0; i < dCount; i++) {
        if (blocks[i] != 'D') {
            needD.push_back(i);
        }
    }
    
    // Collect D's that are in wrong positions ([dCount, n))
    vector<int> haveD;  // positions that have D but shouldn't (yet)
    for (int i = dCount; i < n; i++) {
        if (blocks[i] == 'D') {
            haveD.push_back(i);
        }
    }
    
    // Now pair them optimally: prefer pairing D with F over G
    int dIdx = 0;
    for (int pos : needD) {
        if (dIdx >= haveD.size()) break;
        
        // If current position has F, it's good to swap (F moves back)
        // If current position has G, we still need to swap, but it's less ideal
        
        // Actually, we can't really choose - we have to swap
        // The "maximizing G" happens naturally because:
        // - We swap D's from right with non-D's from left
        // - G's that are already towards the right stay there
        // - We don't move G's unnecessarily
        
        swap(blocks[pos], blocks[haveD[dIdx]]);
        swapCount++;
        dIdx++;
    }
    
    return swapCount;
}
```

Actually, I realize the issue: the optimization in Part 2 is subtle. Since we're doing the same number of swaps, the question is: **can we arrange which D swaps with which non-D** to maximize G at the end?

The answer is: when we have multiple non-D's in front positions, we should swap D with F first, leaving G's in place. But since we're scanning left to right, we encounter them in order.

Let me provide a solution that explicitly handles this:

```cpp
int moveDToFrontMaximizeGExplicit(vector<char>& blocks) {
    int n = blocks.size();
    int swapCount = 0;
    
    // Count D's
    int dCount = 0;
    for (char c : blocks) {
        if (c == 'D') dCount++;
    }
    
    // Strategy: Process F positions first, then G positions
    // This maximizes G's staying towards the end
    
    int left = 0;
    int right = n - 1;
    
    // First pass: Swap D with F positions
    int leftF = 0;
    while (leftF < dCount && right >= dCount) {
        // Find next F position in front
        while (leftF < dCount && blocks[leftF] != 'F') {
            leftF++;
        }
        
        if (leftF >= dCount) break;
        
        // Find rightmost D
        while (right >= dCount && blocks[right] != 'D') {
            right--;
        }
        
        if (right < dCount) break;
        
        swap(blocks[leftF], blocks[right]);
        swapCount++;
        leftF++;
        right--;
    }
    
    // Second pass: Swap remaining D with G positions
    int leftG = 0;
    while (leftG < dCount && right >= dCount) {
        // Find next G position in front
        while (leftG < dCount && blocks[leftG] != 'G') {
            leftG++;
        }
        
        if (leftG >= dCount) break;
        
        // Find rightmost D
        while (right >= dCount && blocks[right] != 'D') {
            right--;
        }
        
        if (right < dCount) break;
        
        swap(blocks[leftG], blocks[right]);
        swapCount++;
        leftG++;
        right--;
    }
    
    return swapCount;
}
```

## Complete Solution with Test Cases

```cpp
#include <vector>
#include <string>
#include <iostream>
#include <algorithm>
using namespace std;

class StorageBlockSorter {
public:
    // Part 1: Move all D to front with minimum swaps
    int moveDToFront(vector<char>& blocks) {
        int n = blocks.size();
        int swapCount = 0;
        
        // Count D's
        int dCount = 0;
        for (char c : blocks) {
            if (c == 'D') dCount++;
        }
        
        int left = 0;
        int right = n - 1;
        
        while (left < dCount && right >= dCount) {
            // Skip D's already in correct position
            if (blocks[left] == 'D') {
                left++;
                continue;
            }
            
            // Find rightmost D
            while (right >= dCount && blocks[right] != 'D') {
                right--;
            }
            
            if (right >= dCount) {
                swap(blocks[left], blocks[right]);
                swapCount++;
                left++;
                right--;
            } else {
                break;
            }
        }
        
        return swapCount;
    }
    
    // Part 2: Move D to front while maximizing G at end
    int moveDToFrontMaximizeG(vector<char>& blocks) {
        int n = blocks.size();
        int swapCount = 0;
        
        // Count D's
        int dCount = 0;
        for (char c : blocks) {
            if (c == 'D') dCount++;
        }
        
        // Strategy: Process F positions first, then G positions
        // This leaves more G's towards the end
        
        int right = n - 1;
        
        // First pass: Swap D with F positions (preferred)
        for (int leftF = 0; leftF < dCount && right >= dCount; leftF++) {
            if (blocks[leftF] == 'F') {
                // Find rightmost D
                while (right >= dCount && blocks[right] != 'D') {
                    right--;
                }
                
                if (right >= dCount) {
                    swap(blocks[leftF], blocks[right]);
                    swapCount++;
                    right--;
                } else {
                    break;
                }
            }
        }
        
        // Second pass: Swap remaining D with G positions
        for (int leftG = 0; leftG < dCount && right >= dCount; leftG++) {
            if (blocks[leftG] == 'G') {
                // Find rightmost D
                while (right >= dCount && blocks[right] != 'D') {
                    right--;
                }
                
                if (right >= dCount) {
                    swap(blocks[leftG], blocks[right]);
                    swapCount++;
                    right--;
                } else {
                    break;
                }
            }
        }
        
        return swapCount;
    }
    
    // Helper: Print blocks
    void printBlocks(const vector<char>& blocks) {
        for (char c : blocks) {
            cout << c << " ";
        }
        cout << endl;
    }
};

// Test cases
int main() {
    StorageBlockSorter sorter;
    
    // Test Case 1
    vector<char> blocks1 = {'F', 'D', 'G', 'D', 'F', 'G'};
    cout << "Test 1:\n";
    cout << "Original: ";
    sorter.printBlocks(blocks1);
    
    int swaps1 = sorter.moveDToFront(blocks1);
    cout << "After Part 1 (" << swaps1 << " swaps): ";
    sorter.printBlocks(blocks1);
    
    // Reset for Part 2
    blocks1 = {'F', 'D', 'G', 'D', 'F', 'G'};
    int swaps2 = sorter.moveDToFrontMaximizeG(blocks1);
    cout << "After Part 2 (" << swaps2 << " swaps): ";
    sorter.printBlocks(blocks1);
    cout << endl;
    
    // Test Case 2
    vector<char> blocks2 = {'G', 'F', 'D', 'G', 'D', 'F'};
    cout << "Test 2:\n";
    cout << "Original: ";
    sorter.printBlocks(blocks2);
    
    int swaps3 = sorter.moveDToFront(blocks2);
    cout << "After Part 1 (" << swaps3 << " swaps): ";
    sorter.printBlocks(blocks2);
    
    blocks2 = {'G', 'F', 'D', 'G', 'D', 'F'};
    int swaps4 = sorter.moveDToFrontMaximizeG(blocks2);
    cout << "After Part 2 (" << swaps4 << " swaps): ";
    sorter.printBlocks(blocks2);
    
    return 0;
}
```

## Key Insights

### Part 1: Minimize Swaps

1. **Two-pointer technique**: Use left pointer for positions that need D, right pointer scanning backwards for D's
2. **Greedy approach**: Always swap the leftmost non-D with the rightmost D
3. **Optimal**: This gives the minimum number of swaps needed

### Part 2: Maximize G at End

1. **Prefer F over G**: When swapping D with a non-D, prefer F because:
   - Swapping D with F: F moves to back (doesn't matter for G)
   - Swapping D with G: G moves forward (away from end - bad!)
2. **Two-pass approach**: 
   - First pass: Swap D with all F positions
   - Second pass: Swap remaining D with G positions
3. **Same swap count**: We still do the same number of swaps, but we maximize G's staying at the end

## Complexity Analysis

- **Time Complexity:** O(n) - Single pass through the array
- **Space Complexity:** O(1) - Only using a few variables

## Comparison with Standard Sort Colors

| Aspect | Standard Sort Colors | Pure Storage Variant |
|--------|---------------------|---------------------|
| Goal | Minimize iterations | Minimize swaps |
| Approach | Dutch National Flag | Two-pointer with swap optimization |
| Focus | Time complexity | I/O operations |
| Optimization | O(n) single pass | O(n) with swap preference |

## Summary

This problem demonstrates:
1. **Storage-level optimization**: Minimizing expensive I/O operations (swaps)
2. **Greedy algorithms**: Choosing optimal swap pairs
3. **Problem variations**: Adapting classic algorithms to specific constraints
4. **Trade-off analysis**: Understanding when to optimize for different metrics

The key takeaway is that in storage systems, **swap operations are expensive**, so minimizing them is more important than minimizing iterations or comparisons.

