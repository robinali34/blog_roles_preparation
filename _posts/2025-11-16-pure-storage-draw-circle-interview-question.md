---
layout: post
title: "Pure Storage Interview: Draw a Circle Without sqrt/sin/cos"
date: 2025-11-16
categories: [Interview Preparation, Pure Storage, Technical Interview, Algorithms]
excerpt: "Complete solution for generating all integer coordinates on a circle without using sqrt, sin, cos, or other built-in functions. Uses symmetry and incremental calculation for efficient circle rasterization."
---

## Problem Overview

**Pure Storage Interview Question: Draw a Circle**

Generate all integer coordinates that lie on a circle without using `sqrt()`, `sin()`, `cos()`, or other built-in mathematical functions.

### Constraints

- All points must have integer coordinates
- Cannot use `sqrt()`, `sin()`, `cos()`, or similar functions
- Need to generate all points efficiently

### Key Insight

The problem leverages two important concepts:
1. **Symmetry**: A circle has 8-way symmetry (8 octants), so we only need to calculate points in one octant (0-45 degrees) and reflect them
2. **Incremental calculation**: Instead of computing distances for each point, we can use incremental updates based on the circle equation

## Approach

### Step 1: Understanding Circle Symmetry

A circle centered at the origin has 8-way symmetry:
- If `(x, y)` is on the circle, so are:
  - `(y, x)` - reflection across y=x
  - `(-x, y)` - reflection across y-axis
  - `(x, -y)` - reflection across x-axis
  - `(-y, x)` - 90° rotation
  - `(-x, -y)` - 180° rotation
  - `(y, -x)` - 270° rotation
  - `(-y, -x)` - reflection + rotation

Therefore, we only need to calculate points in the first octant (0° to 45°), then reflect them to get all 8 octants.

### Step 2: Circle Equation

For a circle centered at origin with radius `r`:
```
x² + y² = r²
```

### Step 3: Incremental Calculation in First Octant

In the first octant (0-45 degrees), as we move from left to right:
- `x` increases by 1
- `y` either stays the same or decreases by 1

We need to decide: when `x` increases, should `y` stay the same or decrease?

**Decision rule**: Choose the point that minimizes the error from the true circle.

For point `(x, y)`, we check which is closer to the circle:
- `(x+1, y)` - keeping y the same
- `(x+1, y-1)` - decreasing y by 1

We compare the squared distances from these points to the circle:
- Error for `(x+1, y)`: `|(x+1)² + y² - r²|`
- Error for `(x+1, y-1)`: `|(x+1)² + (y-1)² - r²|`

We choose the point with smaller error.

### Step 4: Midpoint Algorithm

A more elegant approach uses the midpoint between the two candidate points:
- Midpoint: `(x+1, y-0.5)`
- If the midpoint is inside the circle, choose `(x+1, y)`
- If the midpoint is outside the circle, choose `(x+1, y-1)`

We can use the circle equation without floating points:
- Check: `(x+1)² + (y-0.5)² < r²`
- Multiply by 4 to avoid fractions: `4(x+1)² + (2y-1)² < 4r²`

## Complete Solution

```cpp
#include <vector>
#include <set>
#include <iostream>
using namespace std;

class CircleDrawer {
public:
    // Generate all integer points on a circle of radius r
    vector<pair<int, int>> drawCircle(int r) {
        vector<pair<int, int>> points;
        set<pair<int, int>> pointSet; // To avoid duplicates
        
        if (r <= 0) return points;
        
        // Start from (0, r) and move to (r/sqrt(2), r/sqrt(2))
        // In first octant: x goes from 0 to approximately r/sqrt(2)
        int x = 0;
        int y = r;
        
        // Decision parameter for midpoint algorithm
        int d = 1 - r; // Initial decision parameter
        
        // Generate points in first octant (0-45 degrees)
        while (x <= y) {
            // Add all 8 symmetric points
            addSymmetricPoints(x, y, points, pointSet);
            
            // Update decision parameter
            if (d < 0) {
                // Midpoint is inside circle, choose (x+1, y)
                d = d + 2 * x + 3;
                x++;
            } else {
                // Midpoint is outside circle, choose (x+1, y-1)
                d = d + 2 * (x - y) + 5;
                x++;
                y--;
            }
        }
        
        return points;
    }
    
private:
    void addSymmetricPoints(int x, int y, 
                            vector<pair<int, int>>& points,
                            set<pair<int, int>>& pointSet) {
        // Generate all 8 symmetric points
        vector<pair<int, int>> symmetric = {
            {x, y},   // First octant
            {y, x},   // Second octant (swap)
            {-x, y},  // Third octant
            {-y, x},  // Fourth octant
            {-x, -y}, // Fifth octant
            {-y, -x}, // Sixth octant
            {x, -y},  // Seventh octant
            {y, -x}   // Eighth octant
        };
        
        for (auto& p : symmetric) {
            if (pointSet.find(p) == pointSet.end()) {
                pointSet.insert(p);
                points.push_back(p);
            }
        }
    }
    
public:
    // Alternative implementation using explicit circle equation check
    vector<pair<int, int>> drawCircleExplicit(int r) {
        vector<pair<int, int>> points;
        set<pair<int, int>> pointSet;
        
        if (r <= 0) return points;
        
        int rSquared = r * r;
        int x = 0;
        int y = r;
        
        // Generate points in first octant
        while (x <= y) {
            // Add all 8 symmetric points
            addSymmetricPoints(x, y, points, pointSet);
            
            // Calculate next x
            int nextX = x + 1;
            
            // Check which y value is closer to the circle
            int y1 = y;      // Keep y same
            int y2 = y - 1;  // Decrease y
            
            // Calculate squared distances from circle
            int dist1 = abs((nextX * nextX + y1 * y1) - rSquared);
            int dist2 = abs((nextX * nextX + y2 * y2) - rSquared);
            
            // Choose the point closer to circle
            if (dist1 <= dist2) {
                x = nextX;
                // y stays the same
            } else {
                x = nextX;
                y = y2;
            }
        }
        
        return points;
    }
    
    // Most efficient: Bresenham's circle algorithm
    vector<pair<int, int>> drawCircleBresenham(int r) {
        vector<pair<int, int>> points;
        set<pair<int, int>> pointSet;
        
        if (r <= 0) return points;
        
        int x = 0;
        int y = r;
        int d = 3 - 2 * r; // Decision parameter
        
        while (x <= y) {
            addSymmetricPoints(x, y, points, pointSet);
            
            if (d < 0) {
                // Move horizontally
                d = d + 4 * x + 6;
                x++;
            } else {
                // Move diagonally
                d = d + 4 * (x - y) + 10;
                x++;
                y--;
            }
        }
        
        return points;
    }
};
```

## Detailed Explanation

### Algorithm Walkthrough

Let's trace through the algorithm for `r = 5`:

**Initialization:**
- `x = 0`, `y = 5`
- `d = 1 - 5 = -4` (decision parameter)

**Iteration 1:**
- Add point `(0, 5)` and its 7 symmetric points
- `d = -4 < 0`, so choose `(x+1, y)` = `(1, 5)`
- Update: `d = -4 + 2*0 + 3 = -1`
- `x = 1`, `y = 5`

**Iteration 2:**
- Add point `(1, 5)` and symmetric points
- `d = -1 < 0`, so choose `(2, 5)`
- Update: `d = -1 + 2*1 + 3 = 4`
- `x = 2`, `y = 5`

**Iteration 3:**
- Add point `(2, 5)` and symmetric points
- `d = 4 >= 0`, so choose `(3, 4)` (diagonal move)
- Update: `d = 4 + 2*(2-5) + 5 = 4 + 2*(-3) + 5 = 3`
- `x = 3`, `y = 4`

**Iteration 4:**
- Add point `(3, 4)` and symmetric points
- `d = 3 >= 0`, so choose `(4, 3)`
- Update: `d = 3 + 2*(3-4) + 5 = 6`
- `x = 4`, `y = 3`

**Iteration 5:**
- Add point `(4, 3)` and symmetric points
- `d = 6 >= 0`, so choose `(5, 2)`
- Update: `d = 6 + 2*(4-3) + 5 = 13`
- `x = 5`, `y = 2`

**Iteration 6:**
- Add point `(5, 2)` and symmetric points
- `d = 13 >= 0`, so choose `(6, 1)`
- Update: `d = 13 + 2*(5-2) + 5 = 24`
- `x = 6`, `y = 1`

**Iteration 7:**
- Check: `x = 6 > y = 1`, so stop

### Why This Works

1. **Symmetry**: By calculating only the first octant, we reduce computation by 8x

2. **Incremental Updates**: Instead of computing `x² + y²` for each point, we update the decision parameter incrementally:
   - When moving horizontally: `d_new = d_old + 2x + 3`
   - When moving diagonally: `d_new = d_old + 2(x - y) + 5`

3. **No Floating Point**: All calculations use integers, avoiding precision issues

4. **Efficiency**: Time complexity is O(r) since we only iterate through one octant

## Visual Example

For a circle with radius 5, the first octant points are:
```
(0, 5)
(1, 5)
(2, 5)
(3, 4)
(4, 3)
(5, 2)
```

After applying 8-way symmetry, we get all points on the circle.

## Complete Test Program

```cpp
#include <iostream>
#include <vector>
#include <set>
#include <algorithm>
using namespace std;

int main() {
    CircleDrawer drawer;
    
    // Test with radius 5
    int radius = 5;
    vector<pair<int, int>> points = drawer.drawCircle(radius);
    
    cout << "Points on circle with radius " << radius << ":\n";
    cout << "Total points: " << points.size() << "\n\n";
    
    // Sort for better visualization
    sort(points.begin(), points.end(), [](const pair<int, int>& a, const pair<int, int>& b) {
        if (a.first != b.first) return a.first < b.first;
        return a.second < b.second;
    });
    
    // Print points
    for (auto& p : points) {
        cout << "(" << p.first << ", " << p.second << ")\n";
    }
    
    // Verify: check that all points satisfy x² + y² = r²
    int rSquared = radius * radius;
    cout << "\nVerification:\n";
    bool allValid = true;
    for (auto& p : points) {
        int distSq = p.first * p.first + p.second * p.second;
        if (distSq != rSquared) {
            cout << "ERROR: (" << p.first << ", " << p.second 
                 << ") has distance² = " << distSq << ", expected " << rSquared << "\n";
            allValid = false;
        }
    }
    if (allValid) {
        cout << "All points are valid!\n";
    }
    
    return 0;
}
```

## Key Points for Interview

1. **Why avoid sqrt/sin/cos?**
   - Performance: Floating-point operations are expensive
   - Precision: Integer arithmetic is exact
   - Efficiency: Incremental updates are faster than recalculating

2. **Symmetry optimization:**
   - Reduces computation by 8x
   - Only calculate one octant, then reflect

3. **Incremental calculation:**
   - Use decision parameter to avoid recalculating circle equation
   - Update decision parameter based on previous value

4. **Edge cases:**
   - `r = 0`: Only point `(0, 0)`
   - `r = 1`: 4 points `(0, ±1)`, `(±1, 0)`
   - Negative radius: Invalid input

## Time and Space Complexity

- **Time Complexity:** O(r) - We iterate through approximately r/√2 points in the first octant
- **Space Complexity:** O(r) - We store all points on the circle (approximately 8r points)

## Comparison with Naive Approach

**Naive approach** (checking all points in bounding box):
- Time: O(r²) - Check all points in `[-r, r] × [-r, r]`
- Space: O(r²)

**Optimized approach** (using symmetry):
- Time: O(r) - Only calculate first octant
- Space: O(r) - Store only unique points

## Summary

This problem demonstrates:
1. **Geometric algorithms**: Circle rasterization
2. **Optimization techniques**: Using symmetry to reduce computation
3. **Integer arithmetic**: Avoiding floating-point operations
4. **Incremental algorithms**: Efficient updates using previous calculations

The key insight is leveraging the 8-way symmetry of circles and using incremental decision parameters to avoid expensive calculations.

