---
layout: post
title: "Pure Storage Interview: Custom Iterator Implementation"
date: 2025-11-16
categories: [Interview Preparation, Pure Storage, Technical Interview, C++, Design Patterns]
excerpt: "Complete implementation of custom iterators in C++: Sorted Iterator and Filter Iterator. Covers iterator concepts, design patterns, and efficient implementations with detailed explanations."
---

## Problem Overview

**Pure Storage Interview: Custom Iterator Implementation**

This problem tests your understanding of:
1. C++ iterator concepts and requirements
2. Iterator design patterns
3. Lazy evaluation vs. eager evaluation
4. Memory efficiency considerations
5. Template programming and generic design

### Problem Statement

Implement two custom iterators:

1. **Sorted Iterator**: Given an unsorted iterator, create an iterator that yields elements in sorted order
2. **Filter Iterator**: Given a predicate function and an iterator, create an iterator that yields only elements satisfying the predicate

### Key Considerations

- **Lazy vs. Eager**: Should we pre-compute or compute on-demand?
- **Memory efficiency**: Minimize memory usage
- **Iterator requirements**: Must satisfy C++ iterator concepts
- **Performance**: Balance between setup time and iteration time

## Part 1: Iterator Concepts in C++

### Iterator Categories

C++ iterators have different categories with increasing capabilities:

1. **Input Iterator**: Can read and advance (single-pass)
2. **Forward Iterator**: Can read, advance, and compare (multi-pass)
3. **Bidirectional Iterator**: Can move backward
4. **Random Access Iterator**: Can jump to any position

### Iterator Requirements

A valid iterator must support:
- `*it` - Dereference
- `++it` or `it++` - Increment
- `it1 == it2` - Equality comparison
- `it1 != it2` - Inequality comparison

For our implementations, we'll create **Forward Iterators**.

## Part 2: Sorted Iterator Implementation

### Approach 1: Eager Evaluation (Pre-sort)

Load all elements, sort them, then iterate:

```cpp
#include <vector>
#include <algorithm>
#include <iterator>
#include <functional>
using namespace std;

template<typename Iterator>
class SortedIterator {
private:
    vector<typename iterator_traits<Iterator>::value_type> sorted_data;
    typename vector<typename iterator_traits<Iterator>::value_type>::iterator current;
    
public:
    using value_type = typename iterator_traits<Iterator>::value_type;
    using iterator_category = forward_iterator_tag;
    using difference_type = ptrdiff_t;
    using pointer = value_type*;
    using reference = value_type&;
    
    SortedIterator(Iterator begin, Iterator end) {
        // Copy and sort all elements
        copy(begin, end, back_inserter(sorted_data));
        sort(sorted_data.begin(), sorted_data.end());
        current = sorted_data.begin();
    }
    
    // Copy constructor
    SortedIterator(const SortedIterator& other) 
        : sorted_data(other.sorted_data), current(sorted_data.begin() + (other.current - other.sorted_data.begin())) {}
    
    // Dereference
    value_type operator*() const {
        return *current;
    }
    
    // Pre-increment
    SortedIterator& operator++() {
        ++current;
        return *this;
    }
    
    // Post-increment
    SortedIterator operator++(int) {
        SortedIterator temp = *this;
        ++current;
        return temp;
    }
    
    // Equality
    bool operator==(const SortedIterator& other) const {
        return current == sorted_data.end() && other.current == other.sorted_data.end();
    }
    
    // Inequality
    bool operator!=(const SortedIterator& other) const {
        return !(*this == other);
    }
    
    // Check if at end
    bool isEnd() const {
        return current == sorted_data.end();
    }
};

// Helper function to create sorted iterator range
template<typename Iterator>
pair<SortedIterator<Iterator>, SortedIterator<Iterator>> make_sorted_range(Iterator begin, Iterator end) {
    SortedIterator<Iterator> start(begin, end);
    SortedIterator<Iterator> finish(begin, end);
    // Move finish to end
    while (!finish.isEnd()) {
        ++finish;
    }
    return {start, finish};
}
```

**Pros:**
- Simple implementation
- Fast iteration (O(1) per element)
- Supports multiple passes

**Cons:**
- High memory usage (stores all elements)
- Slow initialization (O(n log n))
- Not suitable for large datasets

### Approach 2: Lazy Evaluation (Heap-based)

Use a priority queue to lazily yield sorted elements:

```cpp
#include <queue>
#include <functional>
#include <iterator>

template<typename Iterator>
class LazySortedIterator {
private:
    using value_type = typename iterator_traits<Iterator>::value_type;
    
    struct Element {
        value_type value;
        Iterator it;
        
        bool operator>(const Element& other) const {
            return value > other.value;
        }
    };
    
    priority_queue<Element, vector<Element>, greater<Element>> min_heap;
    bool is_end;
    
public:
    using iterator_category = input_iterator_tag;
    using difference_type = ptrdiff_t;
    using pointer = value_type*;
    using reference = value_type&;
    
    LazySortedIterator(Iterator begin, Iterator end) : is_end(false) {
        // Initialize heap with first element from each "chunk"
        // For simplicity, we'll use a different approach
        // This is a simplified version - full implementation would be more complex
    }
    
    value_type operator*() const {
        return min_heap.top().value;
    }
    
    LazySortedIterator& operator++() {
        if (min_heap.empty()) {
            is_end = true;
            return *this;
        }
        
        Element top = min_heap.top();
        min_heap.pop();
        
        // Advance the iterator and add next element if available
        // Implementation depends on specific requirements
        
        return *this;
    }
    
    bool operator==(const LazySortedIterator& other) const {
        return is_end && other.is_end;
    }
    
    bool operator!=(const LazySortedIterator& other) const {
        return !(*this == other);
    }
};
```

**Note:** Full lazy implementation is complex. For interview, eager approach is usually preferred.

### Complete Sorted Iterator (Production-Ready)

```cpp
#include <vector>
#include <algorithm>
#include <iterator>
#include <memory>

template<typename Iterator, typename Compare = less<typename iterator_traits<Iterator>::value_type>>
class SortedIterator {
public:
    using value_type = typename iterator_traits<Iterator>::value_type;
    using iterator_category = forward_iterator_tag;
    using difference_type = ptrdiff_t;
    using pointer = value_type*;
    using reference = value_type&;
    
private:
    shared_ptr<vector<value_type>> sorted_data;
    typename vector<value_type>::iterator current;
    
public:
    // Constructor: takes iterator range
    SortedIterator(Iterator begin, Iterator end, Compare comp = Compare()) {
        sorted_data = make_shared<vector<value_type>>();
        copy(begin, end, back_inserter(*sorted_data));
        sort(sorted_data->begin(), sorted_data->end(), comp);
        current = sorted_data->begin();
    }
    
    // End iterator constructor
    SortedIterator(shared_ptr<vector<value_type>> data) 
        : sorted_data(data), current(data->end()) {}
    
    // Copy constructor
    SortedIterator(const SortedIterator& other) 
        : sorted_data(other.sorted_data), 
          current(other.sorted_data->begin() + (other.current - other.sorted_data->begin())) {}
    
    // Assignment operator
    SortedIterator& operator=(const SortedIterator& other) {
        if (this != &other) {
            sorted_data = other.sorted_data;
            current = other.sorted_data->begin() + (other.current - other.sorted_data->begin());
        }
        return *this;
    }
    
    // Dereference
    reference operator*() const {
        return *current;
    }
    
    pointer operator->() const {
        return &(*current);
    }
    
    // Pre-increment
    SortedIterator& operator++() {
        ++current;
        return *this;
    }
    
    // Post-increment
    SortedIterator operator++(int) {
        SortedIterator temp = *this;
        ++current;
        return temp;
    }
    
    // Equality
    bool operator==(const SortedIterator& other) const {
        return current == sorted_data->end() && 
               other.current == other.sorted_data->end();
    }
    
    // Inequality
    bool operator!=(const SortedIterator& other) const {
        return !(*this == other);
    }
    
    // Get end iterator
    static SortedIterator end(shared_ptr<vector<value_type>> data) {
        return SortedIterator(data);
    }
};

// Helper function
template<typename Iterator, typename Compare = less<typename iterator_traits<Iterator>::value_type>>
pair<SortedIterator<Iterator, Compare>, SortedIterator<Iterator, Compare>> 
make_sorted_range(Iterator begin, Iterator end, Compare comp = Compare()) {
    auto data = make_shared<vector<typename iterator_traits<Iterator>::value_type>>();
    copy(begin, end, back_inserter(*data));
    sort(data->begin(), data->end(), comp);
    
    SortedIterator<Iterator, Compare> start(begin, end, comp);
    return {start, SortedIterator<Iterator, Compare>::end(data)};
}
```

## Part 3: Filter Iterator Implementation

### Basic Filter Iterator

```cpp
#include <functional>
#include <iterator>

template<typename Iterator, typename Predicate>
class FilterIterator {
public:
    using value_type = typename iterator_traits<Iterator>::value_type;
    using iterator_category = forward_iterator_tag;
    using difference_type = ptrdiff_t;
    using pointer = value_type*;
    using reference = value_type&;
    
private:
    Iterator current;
    Iterator end_it;
    Predicate pred;
    
    // Advance to next element satisfying predicate
    void advanceToNext() {
        while (current != end_it && !pred(*current)) {
            ++current;
        }
    }
    
public:
    FilterIterator(Iterator begin, Iterator end, Predicate p) 
        : current(begin), end_it(end), pred(p) {
        advanceToNext();  // Find first matching element
    }
    
    // Copy constructor
    FilterIterator(const FilterIterator& other) 
        : current(other.current), end_it(other.end_it), pred(other.pred) {}
    
    // Dereference
    reference operator*() const {
        return *current;
    }
    
    pointer operator->() const {
        return &(*current);
    }
    
    // Pre-increment
    FilterIterator& operator++() {
        ++current;
        advanceToNext();
        return *this;
    }
    
    // Post-increment
    FilterIterator operator++(int) {
        FilterIterator temp = *this;
        ++current;
        advanceToNext();
        return temp;
    }
    
    // Equality
    bool operator==(const FilterIterator& other) const {
        return current == end_it && other.current == other.end_it;
    }
    
    // Inequality
    bool operator!=(const FilterIterator& other) const {
        return !(*this == other);
    }
    
    // Check if at end
    bool isEnd() const {
        return current == end_it;
    }
};

// Helper function
template<typename Iterator, typename Predicate>
pair<FilterIterator<Iterator, Predicate>, FilterIterator<Iterator, Predicate>>
make_filter_range(Iterator begin, Iterator end, Predicate pred) {
    FilterIterator<Iterator, Predicate> start(begin, end, pred);
    FilterIterator<Iterator, Predicate> finish(end, end, pred);
    return {start, finish};
}
```

### Enhanced Filter Iterator with End Handling

```cpp
template<typename Iterator, typename Predicate>
class FilterIterator {
public:
    using value_type = typename iterator_traits<Iterator>::value_type;
    using iterator_category = forward_iterator_tag;
    using difference_type = ptrdiff_t;
    using pointer = value_type*;
    using reference = value_type&;
    
private:
    Iterator current;
    Iterator end_it;
    Predicate pred;
    bool is_end;
    
    void advanceToNext() {
        while (current != end_it && !pred(*current)) {
            ++current;
        }
        is_end = (current == end_it);
    }
    
public:
    FilterIterator(Iterator begin, Iterator end, Predicate p) 
        : current(begin), end_it(end), pred(p), is_end(false) {
        advanceToNext();
    }
    
    // End iterator constructor
    FilterIterator(Iterator end) 
        : current(end), end_it(end), is_end(true) {}
    
    FilterIterator(const FilterIterator& other) 
        : current(other.current), end_it(other.end_it), 
          pred(other.pred), is_end(other.is_end) {}
    
    reference operator*() const {
        if (is_end) {
            throw runtime_error("Dereferencing end iterator");
        }
        return *current;
    }
    
    pointer operator->() const {
        return &(*current);
    }
    
    FilterIterator& operator++() {
        if (!is_end) {
            ++current;
            advanceToNext();
        }
        return *this;
    }
    
    FilterIterator operator++(int) {
        FilterIterator temp = *this;
        ++(*this);
        return temp;
    }
    
    bool operator==(const FilterIterator& other) const {
        return is_end && other.is_end;
    }
    
    bool operator!=(const FilterIterator& other) const {
        return !(*this == other);
    }
};
```

## Part 4: Complete Implementation with Examples

```cpp
#include <iostream>
#include <vector>
#include <list>
#include <algorithm>
#include <functional>
#include <iterator>
#include <memory>
using namespace std;

// Sorted Iterator
template<typename Iterator, typename Compare = less<typename iterator_traits<Iterator>::value_type>>
class SortedIterator {
public:
    using value_type = typename iterator_traits<Iterator>::value_type;
    using iterator_category = forward_iterator_tag;
    using difference_type = ptrdiff_t;
    using pointer = value_type*;
    using reference = value_type&;
    
private:
    shared_ptr<vector<value_type>> sorted_data;
    typename vector<value_type>::iterator current;
    
public:
    SortedIterator(Iterator begin, Iterator end, Compare comp = Compare()) {
        sorted_data = make_shared<vector<value_type>>();
        copy(begin, end, back_inserter(*sorted_data));
        sort(sorted_data->begin(), sorted_data->end(), comp);
        current = sorted_data->begin();
    }
    
    SortedIterator(shared_ptr<vector<value_type>> data) 
        : sorted_data(data), current(data->end()) {}
    
    SortedIterator(const SortedIterator& other) 
        : sorted_data(other.sorted_data), 
          current(other.sorted_data->begin() + (other.current - other.sorted_data->begin())) {}
    
    reference operator*() const { return *current; }
    pointer operator->() const { return &(*current); }
    
    SortedIterator& operator++() {
        ++current;
        return *this;
    }
    
    SortedIterator operator++(int) {
        SortedIterator temp = *this;
        ++current;
        return temp;
    }
    
    bool operator==(const SortedIterator& other) const {
        return current == sorted_data->end() && other.current == other.sorted_data->end();
    }
    
    bool operator!=(const SortedIterator& other) const {
        return !(*this == other);
    }
    
    static SortedIterator end(shared_ptr<vector<value_type>> data) {
        return SortedIterator(data);
    }
};

// Filter Iterator
template<typename Iterator, typename Predicate>
class FilterIterator {
public:
    using value_type = typename iterator_traits<Iterator>::value_type;
    using iterator_category = forward_iterator_tag;
    using difference_type = ptrdiff_t;
    using pointer = value_type*;
    using reference = value_type&;
    
private:
    Iterator current;
    Iterator end_it;
    Predicate pred;
    bool is_end;
    
    void advanceToNext() {
        while (current != end_it && !pred(*current)) {
            ++current;
        }
        is_end = (current == end_it);
    }
    
public:
    FilterIterator(Iterator begin, Iterator end, Predicate p) 
        : current(begin), end_it(end), pred(p), is_end(false) {
        advanceToNext();
    }
    
    FilterIterator(Iterator end) 
        : current(end), end_it(end), is_end(true) {}
    
    FilterIterator(const FilterIterator& other) 
        : current(other.current), end_it(other.end_it), 
          pred(other.pred), is_end(other.is_end) {}
    
    reference operator*() const {
        if (is_end) throw runtime_error("Dereferencing end iterator");
        return *current;
    }
    
    pointer operator->() const { return &(*current); }
    
    FilterIterator& operator++() {
        if (!is_end) {
            ++current;
            advanceToNext();
        }
        return *this;
    }
    
    FilterIterator operator++(int) {
        FilterIterator temp = *this;
        ++(*this);
        return temp;
    }
    
    bool operator==(const FilterIterator& other) const {
        return is_end && other.is_end;
    }
    
    bool operator!=(const FilterIterator& other) const {
        return !(*this == other);
    }
};

// Helper functions
template<typename Iterator, typename Compare = less<typename iterator_traits<Iterator>::value_type>>
pair<SortedIterator<Iterator, Compare>, SortedIterator<Iterator, Compare>> 
make_sorted_range(Iterator begin, Iterator end, Compare comp = Compare()) {
    auto data = make_shared<vector<typename iterator_traits<Iterator>::value_type>>();
    copy(begin, end, back_inserter(*data));
    sort(data->begin(), data->end(), comp);
    
    SortedIterator<Iterator, Compare> start(begin, end, comp);
    return {start, SortedIterator<Iterator, Compare>::end(data)};
}

template<typename Iterator, typename Predicate>
pair<FilterIterator<Iterator, Predicate>, FilterIterator<Iterator, Predicate>>
make_filter_range(Iterator begin, Iterator end, Predicate pred) {
    FilterIterator<Iterator, Predicate> start(begin, end, pred);
    FilterIterator<Iterator, Predicate> finish(end);
    return {start, finish};
}
```

## Part 5: Usage Examples

```cpp
void testSortedIterator() {
    cout << "=== Testing Sorted Iterator ===" << endl;
    
    vector<int> unsorted = {5, 2, 8, 1, 9, 3, 7, 4, 6};
    
    auto [begin, end] = make_sorted_range(unsorted.begin(), unsorted.end());
    
    cout << "Sorted elements: ";
    for (auto it = begin; it != end; ++it) {
        cout << *it << " ";
    }
    cout << endl;
    
    // Using with STL algorithms
    auto [begin2, end2] = make_sorted_range(unsorted.begin(), unsorted.end(), greater<int>());
    cout << "Reverse sorted: ";
    copy(begin2, end2, ostream_iterator<int>(cout, " "));
    cout << endl;
}

void testFilterIterator() {
    cout << "\n=== Testing Filter Iterator ===" << endl;
    
    vector<int> numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // Filter even numbers
    auto isEven = [](int n) { return n % 2 == 0; };
    auto [evenBegin, evenEnd] = make_filter_range(numbers.begin(), numbers.end(), isEven);
    
    cout << "Even numbers: ";
    for (auto it = evenBegin; it != evenEnd; ++it) {
        cout << *it << " ";
    }
    cout << endl;
    
    // Filter numbers greater than 5
    auto greaterThan5 = [](int n) { return n > 5; };
    auto [gt5Begin, gt5End] = make_filter_range(numbers.begin(), numbers.end(), greaterThan5);
    
    cout << "Numbers > 5: ";
    copy(gt5Begin, gt5End, ostream_iterator<int>(cout, " "));
    cout << endl;
    
    // Filter with custom predicate
    auto isPrime = [](int n) {
        if (n < 2) return false;
        for (int i = 2; i * i <= n; i++) {
            if (n % i == 0) return false;
        }
        return true;
    };
    
    auto [primeBegin, primeEnd] = make_filter_range(numbers.begin(), numbers.end(), isPrime);
    cout << "Prime numbers: ";
    copy(primeBegin, primeEnd, ostream_iterator<int>(cout, " "));
    cout << endl;
}

void testChainedIterators() {
    cout << "\n=== Testing Chained Iterators ===" << endl;
    
    vector<int> data = {10, 5, 8, 3, 7, 2, 9, 1, 6, 4};
    
    // First filter, then sort
    auto isOdd = [](int n) { return n % 2 == 1; };
    
    // Filter first
    vector<int> filtered;
    auto [filterBegin, filterEnd] = make_filter_range(data.begin(), data.end(), isOdd);
    copy(filterBegin, filterEnd, back_inserter(filtered));
    
    // Then sort
    auto [sortBegin, sortEnd] = make_sorted_range(filtered.begin(), filtered.end());
    
    cout << "Odd numbers sorted: ";
    copy(sortBegin, sortEnd, ostream_iterator<int>(cout, " "));
    cout << endl;
}

int main() {
    testSortedIterator();
    testFilterIterator();
    testChainedIterators();
    return 0;
}
```

## Part 6: Advanced: Composable Iterator

### Combining Filter and Sort

```cpp
template<typename Iterator, typename Predicate, typename Compare = less<typename iterator_traits<Iterator>::value_type>>
class FilteredSortedIterator {
private:
    using value_type = typename iterator_traits<Iterator>::value_type;
    shared_ptr<vector<value_type>> filtered_sorted_data;
    typename vector<value_type>::iterator current;
    
public:
    FilteredSortedIterator(Iterator begin, Iterator end, Predicate pred, Compare comp = Compare()) {
        filtered_sorted_data = make_shared<vector<value_type>>();
        
        // Filter first
        FilterIterator<Iterator, Predicate> filterIt(begin, end, pred);
        FilterIterator<Iterator, Predicate> filterEnd(end);
        
        copy(filterIt, filterEnd, back_inserter(*filtered_sorted_data));
        
        // Then sort
        sort(filtered_sorted_data->begin(), filtered_sorted_data->end(), comp);
        current = filtered_sorted_data->begin();
    }
    
    value_type operator*() const { return *current; }
    
    FilteredSortedIterator& operator++() {
        ++current;
        return *this;
    }
    
    bool operator==(const FilteredSortedIterator& other) const {
        return current == filtered_sorted_data->end() && 
               other.current == other.filtered_sorted_data->end();
    }
    
    bool operator!=(const FilteredSortedIterator& other) const {
        return !(*this == other);
    }
};
```

## Part 7: Design Considerations

### Lazy vs. Eager Evaluation

**Eager (Current Implementation):**
- ✅ Simple to implement
- ✅ Fast iteration (O(1))
- ✅ Supports multiple passes
- ❌ High memory usage
- ❌ Slow initialization

**Lazy (Alternative):**
- ✅ Low memory usage
- ✅ Fast initialization
- ❌ Complex implementation
- ❌ Slower iteration
- ❌ Single-pass only

### Memory Efficiency

**Sorted Iterator:**
- Stores all elements: O(n) space
- Can use `shared_ptr` to share data between iterators

**Filter Iterator:**
- No extra storage: O(1) space
- Only advances underlying iterator

### Performance Trade-offs

| Operation | Sorted Iterator | Filter Iterator |
|-----------|----------------|-----------------|
| **Construction** | O(n log n) | O(1) |
| **Next element** | O(1) | O(k) where k = skipped elements |
| **Memory** | O(n) | O(1) |
| **Multiple passes** | ✅ Yes | ✅ Yes |

## Part 8: Interview Discussion Points

### Key Questions

**Q1: Why use iterators instead of containers?**

**A:** 
- **Composability**: Can chain operations
- **Lazy evaluation**: Process on-demand
- **Memory efficiency**: Don't store intermediate results
- **Generic programming**: Work with any iterator type

**Q2: Lazy vs. Eager evaluation?**

**A:**
- **Eager**: Pre-compute everything (current implementation)
  - Better for multiple passes
  - Simpler code
- **Lazy**: Compute on-demand
  - Better for memory
  - More complex

**Q3: How to handle very large datasets?**

**A:**
- **External sorting**: For sorted iterator
- **Streaming**: Process in chunks
- **Lazy evaluation**: Don't load everything
- **Disk-based**: Use temporary files

**Q4: Can we combine filter and sort?**

**A:** Yes, two approaches:
1. Filter first, then sort (current)
2. Sort first, then filter (if sort is cheaper)

**Q5: Thread safety?**

**A:**
- Current implementation: Not thread-safe
- Can add mutex for concurrent access
- Better: Use separate iterators per thread

## Key Takeaways

### Iterator Design Principles

1. **Satisfy iterator concepts**: Implement required operations
2. **Composability**: Work with STL algorithms
3. **Memory efficiency**: Minimize storage when possible
4. **Performance**: Balance setup vs. iteration cost

### Implementation Patterns

1. **Eager evaluation**: Pre-compute for fast iteration
2. **Lazy evaluation**: Compute on-demand for memory efficiency
3. **Composition**: Chain iterators for complex operations
4. **Generic design**: Use templates for flexibility

### Interview Tips

1. **Start simple**: Basic iterator first
2. **Discuss trade-offs**: Memory vs. performance
3. **Show composability**: Work with STL
4. **Handle edge cases**: End iterators, empty ranges
5. **Optimize**: Discuss lazy evaluation, memory efficiency

## Summary

Custom iterator implementation demonstrates:

- **C++ template programming**: Generic, reusable code
- **Iterator concepts**: Understanding C++ iterator requirements
- **Design patterns**: Adapter pattern for iterators
- **Performance trade-offs**: Eager vs. lazy evaluation
- **Composability**: Building complex operations from simple ones

**Key implementations:**
- **Sorted Iterator**: Eager evaluation, O(n log n) setup, O(1) iteration
- **Filter Iterator**: Lazy evaluation, O(1) setup, O(k) iteration

Understanding iterators is crucial for:
- STL algorithm usage
- Generic programming
- Performance optimization
- System design

This knowledge is valuable for C++ positions at companies like Pure Storage, where efficient, generic code is essential.

