---
layout: post
title: "Pure Storage Interview: Valid Square Problem (LeetCode 593)"
date: 2025-11-16
categories: [Interview Preparation, Pure Storage, Technical Interview, Algorithms]
excerpt: "Complete breakdown of the Pure Storage interview question: Valid Square (LeetCode 593) with solutions from O(n^4) to O(n^2) optimization, including C++ implementations and detailed explanations."
---

## Problem Overview

**LeetCode 593: Valid Square**

This is a classic interview problem from Pure Storage that tests your ability to:
1. Solve a geometric problem efficiently
2. Optimize solutions through multiple iterations
3. Handle edge cases and avoid floating-point precision issues
4. Understand duplicate counting and how to handle it

### Problem Statement

**Part 1:** Given four points in a 2D plane, determine if they form a valid square.

**Part 2 (Follow-up):** Given n points, count how many valid squares can be formed.

## Part 1: Validating Four Points Form a Square

### Approach

A valid square has these properties:
- All four sides are equal in length
- Both diagonals are equal in length
- The four sides and two diagonals form exactly 2 distinct distances (4 sides + 2 diagonals, but sides = diagonals in a square)

**Key Insight:** Instead of using `sqrt()` to calculate distances (which involves expensive floating-point operations and potential precision issues), we can compare squared distances using integer arithmetic.

### Solution: O(1) - Four Points

```cpp
#include <vector>
#include <set>
#include <algorithm>
using namespace std;

class Solution {
public:
    bool validSquare(vector<int>& p1, vector<int>& p2, 
                    vector<int>& p3, vector<int>& p4) {
        // Calculate squared distances between all pairs
        set<int> distances;
        
        vector<vector<int>> points = {p1, p2, p3, p4};
        
        for (int i = 0; i < 4; i++) {
            for (int j = i + 1; j < 4; j++) {
                int dx = points[i][0] - points[j][0];
                int dy = points[i][1] - points[j][1];
                int distSq = dx * dx + dy * dy;
                
                // If any distance is 0, points are not distinct
                if (distSq == 0) return false;
                
                distances.insert(distSq);
            }
        }
        
        // A square must have exactly 2 distinct distances:
        // 4 equal sides and 2 equal diagonals
        return distances.size() == 2;
    }
};
```

### Explanation

1. **Calculate squared distances:** We compute the squared distance between all pairs of points (6 pairs total: 4C2 = 6).

2. **Check for duplicate points:** If any squared distance is 0, two points are the same, so it can't be a square.

3. **Validate square property:** A valid square has exactly 2 distinct squared distances:
   - 4 sides with the same squared length
   - 2 diagonals with the same squared length (which equals 2 × side²)

### Why Avoid sqrt()?

- **Performance:** Floating-point operations are expensive
- **Precision:** Avoid floating-point precision errors
- **Integer comparison:** Squared distances are integers (when points have integer coordinates), making comparison exact and faster

### Corner Cases

1. **Duplicate points:** Two or more points are the same
2. **Degenerate cases:** Points forming a line or other shapes
3. **Integer coordinates:** Since input uses integers, we don't need to worry about precision issues that would arise with floating-point coordinates

## Part 2: Counting Valid Squares from N Points

### Follow-up Problem

Given n points, count how many distinct valid squares can be formed.

### Approach 1: O(n⁴) - Brute Force

Try all combinations of 4 points and check if they form a square.

```cpp
int countSquaresBruteForce(vector<vector<int>>& points) {
    int n = points.size();
    int count = 0;
    
    for (int i = 0; i < n; i++) {
        for (int j = i + 1; j < n; j++) {
            for (int k = j + 1; k < n; k++) {
                for (int l = k + 1; l < n; l++) {
                    vector<int> p1 = points[i];
                    vector<int> p2 = points[j];
                    vector<int> p3 = points[k];
                    vector<int> p4 = points[l];
                    
                    if (validSquare(p1, p2, p3, p4)) {
                        count++;
                    }
                }
            }
        }
    }
    
    return count;
}
```

**Time Complexity:** O(n⁴)  
**Space Complexity:** O(1)

### Approach 2: O(n³) - Optimized with Hash Set

Fix 3 points, calculate the 4th point that would complete a square, and check if it exists.

```cpp
int countSquaresOptimized(vector<vector<int>>& points) {
    int n = points.size();
    unordered_set<string> pointSet;
    
    // Store all points in a hash set for O(1) lookup
    for (auto& p : points) {
        pointSet.insert(to_string(p[0]) + "," + to_string(p[1]));
    }
    
    int count = 0;
    
    for (int i = 0; i < n; i++) {
        for (int j = i + 1; j < n; j++) {
            for (int k = j + 1; k < n; k++) {
                // Given 3 points, calculate the 4th point
                vector<int> p1 = points[i];
                vector<int> p2 = points[j];
                vector<int> p3 = points[k];
                
                // Calculate possible 4th points
                // For a square, if we have 3 points, the 4th point is uniquely determined
                vector<vector<int>> candidates = calculateFourthPoint(p1, p2, p3);
                
                for (auto& candidate : candidates) {
                    string key = to_string(candidate[0]) + "," + to_string(candidate[1]);
                    if (pointSet.count(key)) {
                        count++;
                    }
                }
            }
        }
    }
    
    // Each square is counted multiple times (once for each of its 4 vertices)
    // Divide by 4 to get the correct count
    return count / 4;
}

vector<vector<int>> calculateFourthPoint(vector<int>& p1, vector<int>& p2, vector<int>& p3) {
    vector<vector<int>> candidates;
    
    // Calculate vectors
    int dx1 = p2[0] - p1[0], dy1 = p2[1] - p1[1];
    int dx2 = p3[0] - p1[0], dy2 = p3[1] - p1[1];
    
    // Case 1: p1-p2 and p1-p3 are adjacent sides
    int x4_1 = p2[0] + p3[0] - p1[0];
    int y4_1 = p2[1] + p3[1] - p1[1];
    candidates.push_back({x4_1, y4_1});
    
    // Case 2: p1-p2 and p1-p3 form a diagonal and side
    // More cases needed for complete solution...
    
    return candidates;
}
```

**Time Complexity:** O(n³)  
**Space Complexity:** O(n)

**Duplicate Counting:** Each square is counted 4 times (once for each vertex as the "first" point), so we divide by 4.

### Approach 3: O(n²) - Diagonal-Based Method

Use the property that in a square, the two diagonals are equal and perpendicular, and their midpoints coincide.

```cpp
int countSquaresOptimal(vector<vector<int>>& points) {
    int n = points.size();
    if (n < 4) return 0;
    
    // Map: (midpoint, diagonal_length_squared) -> list of point pairs
    map<pair<pair<int, int>, int>, vector<pair<int, int>>> diagonalMap;
    
    // Consider all pairs as potential diagonals
    for (int i = 0; i < n; i++) {
        for (int j = i + 1; j < n; j++) {
            int dx = points[i][0] - points[j][0];
            int dy = points[i][1] - points[j][1];
            int distSq = dx * dx + dy * dy;
            
            // Calculate midpoint
            int midX = points[i][0] + points[j][0];
            int midY = points[i][1] + points[j][1];
            
            // Use midpoint and distance as key
            diagonalMap[{ {midX, midY}, distSq }].push_back({i, j});
        }
    }
    
    int count = 0;
    
    // For each group of diagonals with same midpoint and length
    for (auto& [key, pairs] : diagonalMap) {
        int m = pairs.size();
        // For each pair of diagonals in this group
        for (int i = 0; i < m; i++) {
            for (int j = i + 1; j < m; j++) {
                // Check if these two diagonals form a square
                int p1 = pairs[i].first, p2 = pairs[i].second;
                int p3 = pairs[j].first, p4 = pairs[j].second;
                
                // Ensure all 4 points are distinct
                if (p1 != p3 && p1 != p4 && p2 != p3 && p2 != p4) {
                    // Check if diagonals are perpendicular
                    vector<int> v1 = {points[p1][0] - points[p2][0], 
                                      points[p1][1] - points[p2][1]};
                    vector<int> v2 = {points[p3][0] - points[p4][0], 
                                      points[p3][1] - points[p4][1]};
                    
                    // Dot product should be 0 for perpendicular vectors
                    int dot = v1[0] * v2[0] + v1[1] * v2[1];
                    if (dot == 0) {
                        count++;
                    }
                }
            }
        }
    }
    
    // Each square has 2 diagonals, so we count each square twice
    return count / 2;
}
```

**Time Complexity:** O(n²)  
**Space Complexity:** O(n²)

**Key Insight:** 
- Two diagonals of a square have the same midpoint and same length
- They are also perpendicular
- We group point pairs by (midpoint, diagonal_length)
- For each group, we check pairs of diagonals to see if they're perpendicular

**Duplicate Counting:** Each square is counted twice (once for each diagonal pair), so we divide by 2.

### Complete O(n²) Solution

Here's a cleaner, more complete implementation:

```cpp
#include <vector>
#include <map>
#include <set>
#include <string>
#include <algorithm>
using namespace std;

class SquareCounter {
public:
    int countSquares(vector<vector<int>>& points) {
        int n = points.size();
        if (n < 4) return 0;
        
        // Store points in a set for O(1) lookup
        set<pair<int, int>> pointSet;
        for (auto& p : points) {
            pointSet.insert({p[0], p[1]});
        }
        
        int count = 0;
        
        // For each pair of points (potential diagonal)
        for (int i = 0; i < n; i++) {
            for (int j = i + 1; j < n; j++) {
                int x1 = points[i][0], y1 = points[i][1];
                int x2 = points[j][0], y2 = points[j][1];
                
                // Calculate the other two points that would form a square
                // with this diagonal
                int dx = x2 - x1, dy = y2 - y1;
                
                // Rotate the vector by 90 degrees to get the other diagonal
                // Two possibilities: rotate clockwise or counterclockwise
                
                // Case 1: Rotate (dx, dy) by 90 degrees to get (dy, -dx)
                int x3 = x1 + dy, y3 = y1 - dx;
                int x4 = x2 + dy, y4 = y2 - dx;
                
                if (pointSet.count({x3, y3}) && pointSet.count({x4, y4})) {
                    count++;
                }
                
                // Case 2: Rotate (dx, dy) by -90 degrees to get (-dy, dx)
                int x5 = x1 - dy, y5 = y1 + dx;
                int x6 = x2 - dy, y6 = y2 + dx;
                
                if (pointSet.count({x5, y5}) && pointSet.count({x6, y6})) {
                    count++;
                }
            }
        }
        
        // Each square is counted 4 times (once for each diagonal direction)
        // and each diagonal is counted twice (once in each direction)
        // So we divide by 8 total
        return count / 8;
    }
};
```

**Explanation:**
- For each pair of points (potential diagonal), we calculate the other two points that would complete a square
- We rotate the diagonal vector by ±90 degrees to find the perpendicular sides
- Each square is counted 8 times total (2 diagonals × 2 directions × 2 rotations), so we divide by 8

## Summary of Optimizations

| Approach | Time Complexity | Key Technique | Duplicate Handling |
|----------|----------------|----------------|---------------------|
| O(n⁴) | O(n⁴) | Brute force all 4-point combinations | None needed |
| O(n³) | O(n³) | Fix 3 points, calculate 4th point | Divide by 4 |
| O(n²) | O(n²) | Use diagonal properties with hash map | Divide by 2 or 8 |

## Key Takeaways

1. **Avoid sqrt():** Use squared distances for integer comparison and better performance
2. **Understand duplicate counting:** Each optimization requires careful handling of how many times each square is counted
3. **Geometric properties:** Leverage properties of squares (equal sides, equal diagonals, perpendicular diagonals, same midpoint)
4. **Hash-based optimization:** Use hash sets/maps to reduce lookup time from O(n) to O(1)

## Bonus: Counting Rectangles

The interviewer also asked about counting rectangles. The approach is similar but uses the midpoint method:

- Two diagonals of a rectangle have the same midpoint
- Unlike squares, rectangles don't require equal diagonal lengths or perpendicular diagonals
- Group point pairs by midpoint, then check if pairs form rectangles

This demonstrates how the same optimization techniques can be applied to related geometric problems.

## Interview Tips

1. **Start with brute force:** Always begin with the simplest O(n⁴) solution
2. **Identify optimizations:** Look for ways to reduce the search space
3. **Handle duplicates carefully:** Think about how many times each solution is counted
4. **Discuss trade-offs:** Talk about time vs. space complexity
5. **Consider edge cases:** Mention duplicate points, collinear points, etc.

