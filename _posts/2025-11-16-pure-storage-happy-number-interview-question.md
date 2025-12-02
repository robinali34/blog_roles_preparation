---
layout: post
title: "Pure Storage Interview: Happy Number Problem (LeetCode 202)"
date: 2025-11-16
categories: [Interview Preparation, Pure Storage, Technical Interview, Algorithms]
excerpt: "Complete solution for Happy Number problem: determine if a number is happy using HashSet and Floyd's cycle detection algorithm. Includes C++ implementations, complexity analysis, and common follow-up questions."
---

## Problem Overview

**LeetCode 202: Happy Number**

This is a classic interview problem that tests your ability to:
1. Detect cycles in sequences
2. Apply mathematical transformations
3. Optimize space complexity
4. Use cycle detection algorithms (Floyd's algorithm)

### Problem Statement

Write an algorithm to determine if a number `n` is "happy".

A **happy number** is a number defined by the following process:
- Starting with any positive integer, replace the number by the **sum of the squares of its digits**
- Repeat the process until the number equals 1 (where it will stay), or it **loops endlessly** in a cycle which does not include 1
- Those numbers for which this process **ends in 1** are happy numbers

**Example 1:**
```
Input: n = 19
Output: true
Explanation:
1² + 9² = 82
8² + 2² = 68
6² + 8² = 100
1² + 0² + 0² = 1
```

**Example 2:**
```
Input: n = 2
Output: false
Explanation:
2² = 4
4² = 16
1² + 6² = 37
3² + 7² = 58
5² + 8² = 89
8² + 9² = 145
1² + 4² + 5² = 42
4² + 2² = 20
2² + 0² = 4 (cycle detected!)
```

## Approach 1: Using HashSet to Detect Cycles

### Intuition

The key insight is that if we encounter a number we've seen before (and it's not 1), we're in a cycle and the number is not happy.

### Algorithm

1. Use a HashSet to store numbers we've seen
2. While `n != 1` and `n` is not in the set:
   - Add `n` to the set
   - Calculate the sum of squares of digits
3. Return `true` if `n == 1`, `false` otherwise

### Solution

```cpp
#include <unordered_set>
#include <iostream>
using namespace std;

class HappyNumber {
public:
    // Helper function: Calculate sum of squares of digits
    int sumOfSquares(int n) {
        int sum = 0;
        while (n > 0) {
            int digit = n % 10;
            sum += digit * digit;
            n /= 10;
        }
        return sum;
    }
    
    // Approach 1: Using HashSet
    bool isHappy(int n) {
        unordered_set<int> seen;
        
        while (n != 1 && seen.find(n) == seen.end()) {
            seen.insert(n);
            n = sumOfSquares(n);
        }
        
        return n == 1;
    }
};
```

### Complexity Analysis

- **Time Complexity:** O(log n) per iteration
  - Each iteration processes O(log n) digits
  - In worst case, we might iterate many times, but the cycle detection ensures we stop
  - Actually, the maximum number we can reach is bounded (for a 32-bit int, max is around 243)
  - So overall: O(log n) × number of iterations ≈ O(log n) × constant
  
- **Space Complexity:** O(log n) or O(1) depending on analysis
  - HashSet stores numbers we've seen
  - Maximum numbers in cycle is bounded (typically small, like 4 or 8)
  - So space is effectively O(1) for practical purposes

## Approach 2: Floyd's Cycle Detection (Optimal Space)

### Intuition

Instead of storing all seen numbers, we can use **Floyd's cycle detection algorithm** (tortoise and hare):
- Use two pointers: slow and fast
- Slow pointer moves one step at a time
- Fast pointer moves two steps at a time
- If there's a cycle, they will eventually meet
- If they meet at 1, the number is happy
- If they meet at any other number, it's not happy

### Algorithm

1. Initialize `slow = n`, `fast = sumOfSquares(n)`
2. While `fast != 1` and `slow != fast`:
   - `slow = sumOfSquares(slow)` (move one step)
   - `fast = sumOfSquares(sumOfSquares(fast))` (move two steps)
3. Return `true` if `fast == 1`, `false` otherwise

### Solution

```cpp
class HappyNumber {
public:
    int sumOfSquares(int n) {
        int sum = 0;
        while (n > 0) {
            int digit = n % 10;
            sum += digit * digit;
            n /= 10;
        }
        return sum;
    }
    
    // Approach 2: Floyd's Cycle Detection (Optimal Space)
    bool isHappyFloyd(int n) {
        int slow = n;
        int fast = sumOfSquares(n);
        
        while (fast != 1 && slow != fast) {
            slow = sumOfSquares(slow);           // Move one step
            fast = sumOfSquares(sumOfSquares(fast)); // Move two steps
        }
        
        return fast == 1;
    }
};
```

### Why This Works

**Mathematical Proof:**
- If the number is happy, both pointers will eventually reach 1 and stay there
- If there's a cycle (not containing 1), the fast pointer will eventually "lap" the slow pointer
- When they meet, if it's at 1, the number is happy; otherwise, it's not

**Example with n = 19:**
```
slow = 19, fast = 82
slow = 82, fast = 68
slow = 68, fast = 100
slow = 100, fast = 1
slow = 1, fast = 1
They meet at 1 → happy number!
```

**Example with n = 2:**
```
slow = 2, fast = 4
slow = 4, fast = 16
slow = 16, fast = 37
slow = 37, fast = 58
slow = 58, fast = 89
slow = 89, fast = 145
slow = 145, fast = 42
slow = 42, fast = 20
slow = 20, fast = 4
slow = 4, fast = 16
... eventually they meet at 4 (not 1) → not happy!
```

### Complexity Analysis

- **Time Complexity:** O(log n)
  - Same as HashSet approach
  - Fast pointer moves twice as fast, but still O(log n) per step
  
- **Space Complexity:** O(1)
  - Only uses two integer variables
  - **This is the key advantage over HashSet approach!**

## Complete Solution with Both Approaches

```cpp
#include <unordered_set>
#include <iostream>
#include <vector>
using namespace std;

class HappyNumber {
public:
    // Helper: Calculate sum of squares of digits
    int sumOfSquares(int n) {
        int sum = 0;
        while (n > 0) {
            int digit = n % 10;
            sum += digit * digit;
            n /= 10;
        }
        return sum;
    }
    
    // Approach 1: HashSet (Easy to understand)
    bool isHappyHashSet(int n) {
        unordered_set<int> seen;
        
        while (n != 1 && seen.find(n) == seen.end()) {
            seen.insert(n);
            n = sumOfSquares(n);
        }
        
        return n == 1;
    }
    
    // Approach 2: Floyd's Cycle Detection (Optimal Space)
    bool isHappyFloyd(int n) {
        int slow = n;
        int fast = sumOfSquares(n);
        
        while (fast != 1 && slow != fast) {
            slow = sumOfSquares(slow);
            fast = sumOfSquares(sumOfSquares(fast));
        }
        
        return fast == 1;
    }
    
    // Wrapper function (using optimal approach)
    bool isHappy(int n) {
        return isHappyFloyd(n);
    }
    
    // Helper: Print the sequence for debugging
    void printSequence(int n) {
        cout << "Sequence for " << n << ": ";
        unordered_set<int> seen;
        
        while (n != 1 && seen.find(n) == seen.end()) {
            cout << n << " -> ";
            seen.insert(n);
            n = sumOfSquares(n);
        }
        cout << n << endl;
    }
};

// Test program
int main() {
    HappyNumber hn;
    
    // Test cases
    vector<int> testCases = {19, 2, 7, 1, 4};
    
    for (int n : testCases) {
        bool result = hn.isHappy(n);
        cout << n << " is " << (result ? "happy" : "not happy") << endl;
        hn.printSequence(n);
        cout << endl;
    }
    
    return 0;
}
```

## Understanding the Cycle

### Why Do Cycles Exist?

For any number, the sum of squares of digits will always be:
- **Bounded**: For a k-digit number, maximum sum is k × 9² = 81k
- For example, a 3-digit number max sum is 243
- This means we'll eventually hit a number we've seen before (pigeonhole principle)

### Common Cycles

The most common cycle for unhappy numbers is:
```
4 → 16 → 37 → 58 → 89 → 145 → 42 → 20 → 4
```

Once we detect any number in this cycle, we know the number is not happy.

## Follow-up Questions

### Follow-up 1: Find All Happy Numbers in a Range

```cpp
vector<int> findHappyNumbers(int start, int end) {
    vector<int> result;
    HappyNumber hn;
    
    for (int i = start; i <= end; i++) {
        if (hn.isHappy(i)) {
            result.push_back(i);
        }
    }
    
    return result;
}
```

### Follow-up 2: Count Happy Numbers Up to N

```cpp
int countHappyNumbers(int n) {
    int count = 0;
    HappyNumber hn;
    
    for (int i = 1; i <= n; i++) {
        if (hn.isHappy(i)) {
            count++;
        }
    }
    
    return count;
}
```

### Follow-up 3: Find the Next Happy Number

```cpp
int nextHappyNumber(int n) {
    HappyNumber hn;
    int current = n + 1;
    
    while (!hn.isHappy(current)) {
        current++;
    }
    
    return current;
}
```

### Follow-up 4: Optimize for Multiple Queries

If we need to check many numbers, we can use memoization:

```cpp
class HappyNumberMemoized {
private:
    unordered_map<int, bool> memo;
    
public:
    int sumOfSquares(int n) {
        int sum = 0;
        while (n > 0) {
            int digit = n % 10;
            sum += digit * digit;
            n /= 10;
        }
        return sum;
    }
    
    bool isHappy(int n) {
        if (memo.find(n) != memo.end()) {
            return memo[n];
        }
        
        unordered_set<int> seen;
        int current = n;
        
        while (current != 1 && seen.find(current) == seen.end()) {
            seen.insert(current);
            current = sumOfSquares(current);
            
            // Check memoization
            if (memo.find(current) != memo.end()) {
                bool result = memo[current];
                // Update memo for all numbers in current path
                for (int num : seen) {
                    memo[num] = result;
                }
                return result;
            }
        }
        
        bool result = (current == 1);
        // Memoize all numbers in the path
        for (int num : seen) {
            memo[num] = result;
        }
        memo[n] = result;
        
        return result;
    }
};
```

## Edge Cases

1. **n = 1**: Should return `true` (already happy)
2. **n = 0**: Should return `false` (not a positive integer, but handle gracefully)
3. **Large numbers**: Works fine due to bounded sum of squares
4. **Negative numbers**: Typically not considered, but can handle by taking absolute value

## Comparison: HashSet vs Floyd's Algorithm

| Aspect | HashSet Approach | Floyd's Algorithm |
|--------|-----------------|-------------------|
| **Time Complexity** | O(log n) | O(log n) |
| **Space Complexity** | O(log n) or O(1) | O(1) |
| **Readability** | Easier to understand | Requires cycle detection knowledge |
| **Memory Usage** | Stores all seen numbers | Only 2 variables |
| **Best For** | Learning, debugging | Production, space-constrained |

## Key Insights

1. **Cycle Detection**: The core challenge is detecting cycles efficiently
2. **Bounded Values**: Sum of squares is always bounded, ensuring termination
3. **Space Optimization**: Floyd's algorithm achieves O(1) space
4. **Mathematical Property**: All unhappy numbers eventually enter the cycle: 4 → 16 → 37 → 58 → 89 → 145 → 42 → 20 → 4

## Interview Tips

1. **Start with HashSet**: Easier to explain and implement
2. **Mention optimization**: Discuss Floyd's algorithm for O(1) space
3. **Handle edge cases**: n = 1, n = 0, negative numbers
4. **Explain complexity**: Why it's O(log n) not O(n)
5. **Discuss follow-ups**: Be ready for range queries, counting, etc.

## Summary

The Happy Number problem is an excellent example of:
- **Cycle detection** in sequences
- **Space-time trade-offs** (HashSet vs Floyd's)
- **Mathematical transformations** (sum of squares)
- **Optimization techniques** (memoization for multiple queries)

**Recommended Approach**: Use Floyd's cycle detection for optimal O(1) space complexity, but be ready to explain both approaches.

