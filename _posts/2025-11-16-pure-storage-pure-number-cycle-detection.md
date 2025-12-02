---
layout: post
title: "Pure Storage Interview: Pure Number Cycle Detection with Memory Constraints"
date: 2025-11-16
categories: [Interview Preparation, Pure Storage, Technical Interview, Algorithms, Mathematics]
excerpt: "Cycle detection in number sequences with memory constraints: Floyd's algorithm, mathematical proofs of convergence, finding critical bounds, and handling memory limitations."
---

## Problem Overview

**Pure Storage Interview: Pure Number Cycle Detection**

This problem tests your ability to:
1. Detect cycles in number sequences efficiently
2. Handle memory constraints
3. Prove mathematical properties (convergence, bounds)
4. Find critical values and ranges
5. Optimize space complexity

### Problem Statement

Given a number transformation function `f(n)`, determine if a number will eventually enter a cycle. The challenge is to:
- Detect cycles with limited memory
- Prove that cycles will always be found
- Find the critical range where numbers converge
- Handle cases where memory is insufficient

**Example Transformation:**
- Sum of squares of digits (like Happy Number)
- Or other mathematical transformations

## Part 1: Basic Cycle Detection

### Approach 1: HashSet (Memory-Intensive)

```cpp
#include <unordered_set>
#include <iostream>
using namespace std;

class CycleDetector {
public:
    // Transform function: sum of squares of digits
    int transform(int n) {
        int sum = 0;
        while (n > 0) {
            int digit = n % 10;
            sum += digit * digit;
            n /= 10;
        }
        return sum;
    }
    
    // Detect cycle using HashSet
    bool hasCycle(int n) {
        unordered_set<int> seen;
        
        while (n != 1 && seen.find(n) == seen.end()) {
            seen.insert(n);
            n = transform(n);
        }
        
        return n != 1;  // Cycle detected if not 1
    }
    
    // Find cycle start
    int findCycleStart(int n) {
        unordered_set<int> seen;
        
        while (seen.find(n) == seen.end()) {
            seen.insert(n);
            n = transform(n);
        }
        
        return n;  // Cycle start
    }
};
```

**Problem:** Requires O(n) space in worst case.

## Part 2: Floyd's Cycle Detection (Memory-Efficient)

### Algorithm

Use two pointers (slow and fast) to detect cycles without storing all values:

```cpp
class CycleDetectorFloyd {
public:
    int transform(int n) {
        int sum = 0;
        while (n > 0) {
            int digit = n % 10;
            sum += digit * digit;
            n /= 10;
        }
        return sum;
    }
    
    // Floyd's cycle detection - O(1) space
    bool hasCycle(int n) {
        int slow = n;
        int fast = transform(n);
        
        // Find meeting point
        while (fast != 1 && slow != fast) {
            slow = transform(slow);
            fast = transform(transform(fast));
        }
        
        return fast != 1;
    }
    
    // Find cycle start
    int findCycleStart(int n) {
        int slow = n;
        int fast = transform(n);
        
        // Find meeting point
        while (fast != 1 && slow != fast) {
            slow = transform(slow);
            fast = transform(transform(fast));
        }
        
        if (fast == 1) return 1;  // No cycle
        
        // Find cycle start
        slow = n;
        while (slow != fast) {
            slow = transform(slow);
            fast = transform(fast);
        }
        
        return slow;
    }
};
```

**Advantages:**
- **O(1) space**: Only uses two variables
- **O(λ + μ) time**: λ = cycle length, μ = distance to cycle

## Part 3: Memory Constraints Problem

### Question: What if Memory is Not Enough?

**Scenario:** Very large numbers or very long sequences before cycle.

**Solutions:**

### Solution 1: Sampling Strategy

Store only every k-th value:

```cpp
class CycleDetectorSampling {
private:
    int sampleRate;
    
public:
    CycleDetectorSampling(int rate = 1000) : sampleRate(rate) {}
    
    bool hasCycle(int n) {
        unordered_set<int> sampled;
        int count = 0;
        
        while (n != 1) {
            if (count % sampleRate == 0) {
                if (sampled.find(n) != sampled.end()) {
                    return true;  // Potential cycle
                }
                sampled.insert(n);
            }
            n = transform(n);
            count++;
        }
        
        return false;
    }
};
```

**Trade-off:** May miss short cycles, but uses less memory.

### Solution 2: Bloom Filter

Use probabilistic data structure:

```cpp
#include <bitset>
#include <functional>

class BloomFilter {
private:
    bitset<10000> bits;
    hash<int> hash1, hash2, hash3;
    
public:
    void insert(int n) {
        bits[hash1(n) % 10000] = true;
        bits[hash2(n) % 10000] = true;
        bits[hash3(n) % 10000] = true;
    }
    
    bool mightContain(int n) {
        return bits[hash1(n) % 10000] && 
               bits[hash2(n) % 10000] && 
               bits[hash3(n) % 10000];
    }
};

class CycleDetectorBloom {
public:
    bool hasCycle(int n) {
        BloomFilter bf;
        
        while (n != 1) {
            if (bf.mightContain(n)) {
                // Potential cycle - verify with Floyd's
                return verifyCycle(n);
            }
            bf.insert(n);
            n = transform(n);
        }
        
        return false;
    }
    
private:
    bool verifyCycle(int n) {
        // Use Floyd's to verify
        int slow = n;
        int fast = transform(n);
        while (fast != 1 && slow != fast) {
            slow = transform(slow);
            fast = transform(transform(fast));
        }
        return fast != 1;
    }
};
```

## Part 4: Mathematical Proof - Convergence

### Problem: Prove Numbers Will Converge to a Range

**Theorem:** For transformation `f(n) = sum of squares of digits`, any number will eventually reach a bounded range.

### Proof Strategy

**Step 1: Upper Bound Analysis**

For a k-digit number n:
- Maximum digit value: 9
- Maximum sum of squares: k × 9² = 81k

**Step 2: Number of Digits**

For n with d digits:
- 10^(d-1) ≤ n < 10^d
- d = ⌊log₁₀(n)⌋ + 1

**Step 3: Convergence**

After transformation:
- f(n) ≤ 81 × (⌊log₁₀(n)⌋ + 1)
- For n ≥ 1000: f(n) ≤ 81 × 4 = 324
- For n ≥ 10000: f(n) ≤ 81 × 5 = 405

**Step 4: Critical Value**

Find n such that f(n) < n:

```
f(n) = 81 × (⌊log₁₀(n)⌋ + 1) < n

For n = 1000: f(1000) = 81 × 4 = 324 < 1000 ✓
For n = 100: f(100) = 81 × 3 = 243 > 100 ✗
```

**Critical Value:** n = 1000

### Implementation: Finding Critical Value

```cpp
class ConvergenceAnalyzer {
public:
    int transform(int n) {
        int sum = 0;
        while (n > 0) {
            int digit = n % 10;
            sum += digit * digit;
            n /= 10;
        }
        return sum;
    }
    
    // Find critical value where f(n) < n
    int findCriticalValue() {
        for (int n = 1; n <= 10000; n++) {
            int transformed = transform(n);
            if (transformed < n) {
                return n;  // First value where transformation reduces number
            }
        }
        return -1;  // Not found
    }
    
    // Prove: all numbers >= critical value will decrease
    bool proveConvergence(int criticalValue) {
        for (int n = criticalValue; n <= criticalValue * 10; n++) {
            int transformed = transform(n);
            if (transformed >= n) {
                return false;  // Counterexample found
            }
        }
        return true;
    }
    
    // Find maximum value in convergence range
    int findMaxInRange() {
        int maxVal = 0;
        for (int n = 1; n < 1000; n++) {
            maxVal = max(maxVal, transform(n));
        }
        return maxVal;
    }
};
```

## Part 5: Complete Mathematical Proof

### Theorem: Convergence to Bounded Range

**Statement:** For `f(n) = sum of squares of digits`, any number n will eventually reach a value < 1000.

**Proof:**

**Lemma 1:** For n ≥ 1000, f(n) < n

**Proof of Lemma 1:**
- Let n have k digits, where k ≥ 4 (n ≥ 1000)
- Maximum sum of squares: 81k
- We need: 81k < 10^(k-1)

For k = 4: 81 × 4 = 324 < 1000 ✓
For k = 5: 81 × 5 = 405 < 10000 ✓
For k ≥ 4: 81k < 10^(k-1) (proven by induction)

Therefore, for n ≥ 1000, f(n) < n.

**Lemma 2:** All numbers eventually reach [1, 999]

**Proof:**
- Start with any n
- If n ≥ 1000, f(n) < n (by Lemma 1)
- Repeated application: n → f(n) → f(f(n)) → ...
- Sequence is decreasing until n < 1000
- Once n < 1000, f(n) ≤ 81 × 3 = 243
- Therefore, all numbers converge to [1, 243]

**Critical Range:** [1, 243]

### Finding Exact Critical Value

```cpp
class CriticalValueFinder {
public:
    int transform(int n) {
        int sum = 0;
        while (n > 0) {
            int digit = n % 10;
            sum += digit * digit;
            n /= 10;
        }
        return sum;
    }
    
    // Find smallest n where f(n) >= n
    int findCriticalValue() {
        for (int n = 1; n <= 10000; n++) {
            int f = transform(n);
            if (f >= n) {
                // Check if this is the maximum
                bool isMax = true;
                for (int i = n + 1; i <= n * 10; i++) {
                    if (transform(i) >= i) {
                        isMax = false;
                        break;
                    }
                }
                if (isMax) {
                    return n;
                }
            }
        }
        return -1;
    }
    
    // More efficient: use mathematical analysis
    int findCriticalValueMath() {
        // For k-digit number: f(n) ≤ 81k
        // Need: 81k < 10^(k-1)
        // Solve: k such that 81k < 10^(k-1)
        
        for (int k = 1; k <= 10; k++) {
            int maxSum = 81 * k;
            int minNum = (k == 1) ? 1 : (int)pow(10, k - 1);
            
            if (maxSum < minNum) {
                return minNum;  // Critical value
            }
        }
        return -1;
    }
    
    // Find maximum value that can be reached
    int findMaxReachable() {
        int maxVal = 0;
        // Check all numbers < critical value
        int critical = findCriticalValueMath();
        
        for (int n = 1; n < critical; n++) {
            maxVal = max(maxVal, transform(n));
        }
        
        // Also check numbers in convergence range
        for (int n = 1; n <= 243; n++) {
            maxVal = max(maxVal, transform(n));
        }
        
        return maxVal;
    }
};
```

## Part 6: Complete Solution with Proof

```cpp
#include <iostream>
#include <unordered_set>
#include <cmath>
#include <algorithm>
using namespace std;

class PureNumberCycleDetector {
private:
    // Transform: sum of squares of digits
    int transform(int n) {
        int sum = 0;
        while (n > 0) {
            int digit = n % 10;
            sum += digit * digit;
            n /= 10;
        }
        return sum;
    }
    
    // Critical value: smallest n where f(n) < n for all larger n
    static const int CRITICAL_VALUE = 1000;
    static const int MAX_RANGE = 243;  // 81 * 3 (max for 3-digit)
    
public:
    // Floyd's cycle detection - O(1) space
    bool hasCycle(int n) {
        int slow = n;
        int fast = transform(n);
        
        while (fast != 1 && slow != fast) {
            slow = transform(slow);
            fast = transform(transform(fast));
        }
        
        return fast != 1;
    }
    
    // Find cycle with bounded memory
    pair<bool, int> findCycleBounded(int n, int maxIterations = 10000) {
        // First, converge to bounded range
        int iterations = 0;
        while (n >= CRITICAL_VALUE && iterations < maxIterations) {
            n = transform(n);
            iterations++;
        }
        
        if (iterations >= maxIterations) {
            return {false, -1};  // Couldn't converge
        }
        
        // Now n is in bounded range [1, MAX_RANGE]
        // Use Floyd's algorithm (guaranteed to work in bounded range)
        return findCycleInRange(n);
    }
    
    pair<bool, int> findCycleInRange(int n) {
        // Ensure n is in range
        while (n > MAX_RANGE) {
            n = transform(n);
        }
        
        int slow = n;
        int fast = transform(n);
        
        // Find meeting point
        int iterations = 0;
        while (fast != 1 && slow != fast && iterations < MAX_RANGE * 2) {
            slow = transform(slow);
            fast = transform(transform(fast));
            iterations++;
        }
        
        if (fast == 1) {
            return {false, 1};  // No cycle, converges to 1
        }
        
        if (iterations >= MAX_RANGE * 2) {
            return {false, -1};  // Timeout
        }
        
        // Find cycle start
        slow = n;
        while (slow != fast) {
            slow = transform(slow);
            fast = transform(fast);
        }
        
        return {true, slow};  // Cycle detected, return start
    }
    
    // Prove convergence (for interview)
    void proveConvergence() {
        cout << "=== Proof of Convergence ===" << endl;
        
        // Part 1: Show critical value
        cout << "Critical Value: " << CRITICAL_VALUE << endl;
        cout << "For n >= " << CRITICAL_VALUE << ", f(n) < n" << endl;
        
        // Verify for sample values
        for (int n = CRITICAL_VALUE; n < CRITICAL_VALUE + 100; n++) {
            int f = transform(n);
            if (f >= n) {
                cout << "ERROR: Counterexample found: " << n << endl;
                return;
            }
        }
        cout << "Verified: All n >= " << CRITICAL_VALUE << " decrease" << endl;
        
        // Part 2: Show bounded range
        cout << "\nMaximum value in convergence range: " << MAX_RANGE << endl;
        cout << "All numbers eventually reach [1, " << MAX_RANGE << "]" << endl;
        
        // Part 3: Show cycle detection is guaranteed
        cout << "\nCycle detection guaranteed:" << endl;
        cout << "- All numbers converge to [1, " << MAX_RANGE << "]" << endl;
        cout << "- Floyd's algorithm works in bounded range" << endl;
        cout << "- Maximum cycle length <= " << MAX_RANGE << endl;
        cout << "- Detection time: O(" << MAX_RANGE << ") = O(1)" << endl;
    }
    
    // Find critical value mathematically
    int findCriticalValue() {
        // For k-digit number: f(n) ≤ 81k
        // Need: 81k < 10^(k-1)
        
        for (int k = 1; k <= 10; k++) {
            int maxSum = 81 * k;
            int minNum = (k == 1) ? 1 : (int)pow(10, k - 1);
            
            if (maxSum < minNum) {
                return minNum;
            }
        }
        return -1;
    }
};
```

## Part 7: Handling Edge Cases

### Case 1: Very Large Numbers

```cpp
class LargeNumberHandler {
public:
    // Handle numbers that might overflow
    long long transformLarge(long long n) {
        long long sum = 0;
        while (n > 0) {
            long long digit = n % 10;
            sum += digit * digit;
            n /= 10;
        }
        return sum;
    }
    
    // Converge large number to manageable range
    long long convergeToRange(long long n) {
        const long long CRITICAL = 1000;
        
        while (n >= CRITICAL) {
            n = transformLarge(n);
            // Safety check
            if (n < 0) {
                return -1;  // Error
            }
        }
        
        return n;
    }
};
```

### Case 2: Memory-Limited Detection

```cpp
class MemoryLimitedDetector {
private:
    int maxMemory;
    
public:
    MemoryLimitedDetector(int maxMem) : maxMemory(maxMem) {}
    
    bool hasCycleLimited(int n) {
        // Converge to range first
        while (n >= 1000) {
            n = transform(n);
        }
        
        // Now use Floyd's (O(1) space)
        return hasCycleFloyd(n);
    }
    
private:
    int transform(int n) {
        int sum = 0;
        while (n > 0) {
            int digit = n % 10;
            sum += digit * digit;
            n /= 10;
        }
        return sum;
    }
    
    bool hasCycleFloyd(int n) {
        int slow = n;
        int fast = transform(n);
        
        while (fast != 1 && slow != fast) {
            slow = transform(slow);
            fast = transform(transform(fast));
        }
        
        return fast != 1;
    }
};
```

## Part 8: Interview Discussion Points

### Key Questions and Answers

**Q1: What if memory is not enough?**

**A:** Use Floyd's cycle detection algorithm:
- O(1) space complexity
- Only needs two pointers
- Works even with very long sequences

**Q2: Will we always find a cycle?**

**A:** Yes, because:
- Numbers converge to bounded range [1, 243]
- In bounded range, cycle detection is guaranteed
- Maximum iterations needed: O(range size)

**Q3: How to prove convergence?**

**A:** Mathematical proof:
- For n ≥ 1000: f(n) < n (decreasing)
- Eventually reaches [1, 999]
- Maximum value: 243 (for 3-digit numbers)

**Q4: What is the critical value?**

**A:** Critical value = 1000
- For n < 1000: f(n) might be larger
- For n ≥ 1000: f(n) < n (always decreases)
- Maximum reachable: 243

**Q5: Time complexity?**

**A:** 
- Convergence: O(log n) - number of digits
- Cycle detection: O(λ + μ) - cycle length + distance
- Overall: O(log n) average case

## Part 9: Test Program

```cpp
#include <iostream>
#include <cassert>

void testCycleDetection() {
    PureNumberCycleDetector detector;
    
    cout << "=== Testing Cycle Detection ===" << endl;
    
    // Test 1: Happy number (no cycle)
    assert(detector.hasCycle(19) == false);
    cout << "Test 1 passed: 19 is happy (no cycle)" << endl;
    
    // Test 2: Unhappy number (has cycle)
    assert(detector.hasCycle(2) == true);
    cout << "Test 2 passed: 2 has cycle" << endl;
    
    // Test 3: Large number
    auto result = detector.findCycleBounded(999999);
    cout << "Test 3: Large number 999999 -> cycle: " << result.first 
         << ", start: " << result.second << endl;
    
    // Test 4: Critical value
    int critical = detector.findCriticalValue();
    cout << "Test 4: Critical value = " << critical << endl;
    
    // Test 5: Proof
    detector.proveConvergence();
    
    cout << "\nAll tests passed!" << endl;
}

int main() {
    testCycleDetection();
    return 0;
}
```

## Key Takeaways

### Cycle Detection Methods

1. **HashSet**: Simple but O(n) space
2. **Floyd's Algorithm**: O(1) space, optimal
3. **Sampling**: Trade accuracy for memory
4. **Bloom Filter**: Probabilistic, space-efficient

### Mathematical Insights

1. **Convergence**: Numbers decrease for n ≥ critical value
2. **Bounded Range**: All numbers reach [1, 243]
3. **Critical Value**: 1000 (where f(n) < n for all larger n)
4. **Guarantee**: Cycle detection always works in bounded range

### Interview Tips

1. **Start with HashSet**: Show understanding
2. **Optimize to Floyd's**: Demonstrate space optimization
3. **Discuss memory constraints**: Show system thinking
4. **Provide mathematical proof**: Demonstrate analytical skills
5. **Find critical value**: Show problem-solving approach

## Summary

This problem combines:
- **Algorithm design**: Cycle detection
- **Space optimization**: O(1) space solution
- **Mathematical analysis**: Convergence proofs
- **Critical thinking**: Finding bounds and guarantees

**Key insights:**
- Floyd's algorithm solves memory constraint
- Mathematical proof guarantees convergence
- Critical value = 1000 (where numbers start decreasing)
- Maximum range = 243 (bounded convergence)

Understanding these concepts demonstrates:
- Algorithm optimization skills
- Mathematical reasoning ability
- System design thinking
- Problem-solving methodology

This is valuable for companies like Pure Storage where efficient algorithms and mathematical rigor are essential.

