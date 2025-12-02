---
layout: post
title: "Pure Storage Interview: Virtual Table (VTable) Implementation"
date: 2025-11-16
categories: [Interview Preparation, Pure Storage, Technical Interview, Data Structures, C++]
excerpt: "Complete explanation of Virtual Method Tables (vtables) in C++: memory layout, inheritance, multiple inheritance, and implementation details. Covers how vtables enable dynamic polymorphism and virtual function dispatch."
---

## Problem Overview

**Pure Storage Interview: Virtual Table (VTable)**

This is a fundamental C++ interview problem that tests your understanding of:
1. Virtual method tables (vtables) and their structure
2. Object memory layout with virtual functions
3. Dynamic polymorphism implementation
4. Multiple inheritance and vtable organization
5. Virtual function dispatch mechanism

### What is a Virtual Table?

A **Virtual Method Table (VTable)** is a data structure used by C++ compilers to support dynamic polymorphism (runtime method resolution). It contains function pointers to virtual functions, allowing the correct function to be called based on the object's actual type at runtime.

## Part 1: Basic Virtual Table Concept

### Simple Inheritance Example

```cpp
#include <iostream>
using namespace std;

class Base {
public:
    virtual void func1() {
        cout << "Base::func1()" << endl;
    }
    virtual void func2() {
        cout << "Base::func2()" << endl;
    }
    virtual ~Base() {
        cout << "Base destructor" << endl;
    }
};

class Derived : public Base {
public:
    void func1() override {
        cout << "Derived::func1()" << endl;
    }
    // func2() not overridden, uses Base::func2()
    virtual void func3() {
        cout << "Derived::func3()" << endl;
    }
    ~Derived() override {
        cout << "Derived destructor" << endl;
    }
};
```

### Memory Layout

**Base Class Object:**
```
[VTable Pointer]  -> Points to Base's vtable
[Base data members]
```

**Base's VTable:**
```
[0] -> Base::func1()
[1] -> Base::func2()
[2] -> Base::~Base()
```

**Derived Class Object:**
```
[VTable Pointer]  -> Points to Derived's vtable
[Base data members]
[Derived data members]
```

**Derived's VTable:**
```
[0] -> Derived::func1()  // Overridden
[1] -> Base::func2()     // Inherited
[2] -> Derived::~Derived() // Overridden
[3] -> Derived::func3()  // New virtual function
```

### How Virtual Function Calls Work

```cpp
void demonstrateVTable(Base* ptr) {
    ptr->func1();  // Calls Derived::func1() if ptr points to Derived
    ptr->func2();  // Calls Base::func2()
}
```

**Dispatch Process:**
1. Get object's vtable pointer (first 8 bytes on 64-bit)
2. Look up function pointer at index in vtable
3. Call the function through the pointer

## Part 2: Manual VTable Implementation

### Understanding the Mechanism

Let's implement a simplified vtable to understand how it works:

```cpp
#include <iostream>
#include <vector>
using namespace std;

// Function pointer type
typedef void (*FuncPtr)();

// Simplified VTable structure
struct VTable {
    FuncPtr* functions;
    size_t size;
    
    VTable(size_t s) : size(s) {
        functions = new FuncPtr[s];
    }
    
    ~VTable() {
        delete[] functions;
    }
};

// Base class with manual vtable
class BaseManual {
private:
    VTable* vtable;
    
public:
    BaseManual() {
        vtable = new VTable(3);
        vtable->functions[0] = reinterpret_cast<FuncPtr>(&BaseManual::func1_impl);
        vtable->functions[1] = reinterpret_cast<FuncPtr>(&BaseManual::func2_impl);
        vtable->functions[2] = reinterpret_cast<FuncPtr>(&BaseManual::destructor_impl);
    }
    
    virtual ~BaseManual() {
        delete vtable;
    }
    
    void func1() {
        // Simulate virtual call: this->vtable->functions[0]()
        cout << "BaseManual::func1()" << endl;
    }
    
    void func2() {
        cout << "BaseManual::func2()" << endl;
    }
    
private:
    static void func1_impl() {
        cout << "BaseManual::func1_impl()" << endl;
    }
    
    static void func2_impl() {
        cout << "BaseManual::func2_impl()" << endl;
    }
    
    static void destructor_impl() {
        cout << "BaseManual destructor" << endl;
    }
};
```

## Part 3: Multiple Inheritance and VTable

### Multiple Inheritance Example

```cpp
class Base1 {
public:
    virtual void func1() {
        cout << "Base1::func1()" << endl;
    }
    virtual ~Base1() = default;
    int base1_data = 1;
};

class Base2 {
public:
    virtual void func2() {
        cout << "Base2::func2()" << endl;
    }
    virtual ~Base2() = default;
    int base2_data = 2;
};

class Derived : public Base1, public Base2 {
public:
    void func1() override {
        cout << "Derived::func1()" << endl;
    }
    void func2() override {
        cout << "Derived::func2()" << endl;
    }
    virtual void func3() {
        cout << "Derived::func3()" << endl;
    }
    ~Derived() override = default;
    int derived_data = 3;
};
```

### Memory Layout with Multiple Inheritance

**Derived Object Layout:**
```
[VTable Pointer 1]  -> Points to Derived's Base1 vtable
[Base1 data members] (base1_data)
[VTable Pointer 2]  -> Points to Derived's Base2 vtable
[Base2 data members] (base2_data)
[Derived data members] (derived_data)
```

**VTable Structure:**
- **VTable 1 (for Base1):**
  ```
  [0] -> Derived::func1()  // Overridden
  [1] -> Derived::~Derived()
  ```

- **VTable 2 (for Base2):**
  ```
  [0] -> Derived::func2()  // Overridden
  [1] -> Derived::~Derived()
  [2] -> Derived::func3()  // New function
  ```

### Pointer Adjustment

```cpp
void demonstrateMultipleInheritance() {
    Derived d;
    
    Base1* b1 = &d;  // No adjustment needed
    Base2* b2 = &d;  // Pointer adjusted by compiler
    
    // b2 points to Base2 subobject, not start of Derived
    // Compiler adjusts: b2 = (Base2*)((char*)&d + offsetof(Base2))
}
```

## Part 4: Virtual Inheritance and VTable

### Diamond Problem

```cpp
class Base {
public:
    virtual void func() {
        cout << "Base::func()" << endl;
    }
    int base_data = 0;
};

class Left : public virtual Base {
public:
    void func() override {
        cout << "Left::func()" << endl;
    }
    int left_data = 1;
};

class Right : public virtual Base {
public:
    void func() override {
        cout << "Right::func()" << endl;
    }
    int right_data = 2;
};

class Bottom : public Left, public Right {
public:
    void func() override {
        cout << "Bottom::func()" << endl;
    }
    int bottom_data = 3;
};
```

### Virtual Inheritance Memory Layout

**Bottom Object:**
```
[VTable Pointer Left]  -> Left's vtable
[Left data members]
[VTable Pointer Right] -> Right's vtable
[Right data members]
[VTable Pointer Base]  -> Base's vtable (shared)
[Base data members]     -> Single copy (shared)
[Bottom data members]
```

## Part 5: Complete VTable Implementation Example

### Demonstrating VTable Behavior

```cpp
#include <iostream>
#include <cstddef>
using namespace std;

class Base {
public:
    virtual void func1() {
        cout << "Base::func1()" << endl;
    }
    virtual void func2() {
        cout << "Base::func2()" << endl;
    }
    virtual ~Base() {
        cout << "Base destructor" << endl;
    }
    int base_data = 10;
};

class Derived : public Base {
public:
    void func1() override {
        cout << "Derived::func1()" << endl;
    }
    virtual void func3() {
        cout << "Derived::func3()" << endl;
    }
    ~Derived() override {
        cout << "Derived destructor" << endl;
    }
    int derived_data = 20;
};

// Helper to inspect vtable
void inspectVTable(Base* obj) {
    // Get vtable pointer (first 8 bytes on 64-bit)
    void** vtable = *(void***)obj;
    
    cout << "VTable address: " << vtable << endl;
    cout << "VTable entries:" << endl;
    
    // Print first few entries (simplified - actual size varies)
    for (int i = 0; i < 5; i++) {
        if (vtable[i] != nullptr) {
            cout << "  [" << i << "] = " << vtable[i] << endl;
        }
    }
}

void demonstratePolymorphism() {
    Base* base = new Base();
    Base* derived = new Derived();
    
    cout << "=== Base Object ===" << endl;
    inspectVTable(base);
    base->func1();
    base->func2();
    
    cout << "\n=== Derived Object ===" << endl;
    inspectVTable(derived);
    derived->func1();  // Calls Derived::func1()
    derived->func2();  // Calls Base::func2()
    
    // Try to call func3 (not in Base interface)
    // derived->func3();  // Compile error
    
    delete base;
    delete derived;
}
```

## Part 6: VTable Structure Details

### Typical VTable Layout

```cpp
// Simplified representation
struct VTable {
    // Offset to top (for multiple inheritance)
    ptrdiff_t offset_to_top;
    
    // Type info pointer (for RTTI)
    const type_info* type_info;
    
    // Virtual function pointers
    void (*func1)();
    void (*func2)();
    void (*destructor)();
    // ... more functions
};
```

### Virtual Function Call Mechanism

```cpp
// What compiler generates for: obj->virtualFunc()
void callVirtualFunction(Base* obj, int vtable_index) {
    // 1. Get vtable pointer
    void** vtable = *(void***)obj;
    
    // 2. Get function pointer at index
    void (*func)() = (void(*)())vtable[vtable_index];
    
    // 3. Adjust 'this' pointer if needed (for multiple inheritance)
    // Base* adjusted_this = adjustPointer(obj, vtable[-1]);
    
    // 4. Call function
    func();
}
```

## Part 7: Performance Considerations

### VTable Overhead

1. **Memory Overhead:**
   - One vtable pointer per object (8 bytes on 64-bit)
   - One vtable per class (shared among all instances)

2. **Time Overhead:**
   - Extra indirection: vtable lookup + function pointer call
   - Typically 1-2 extra memory accesses
   - Modern CPUs handle this efficiently with branch prediction

### Optimization Techniques

```cpp
// Compiler optimizations:
// 1. Devirtualization: If type is known at compile time
void knownType() {
    Derived d;
    d.func1();  // Direct call, no vtable lookup
}

// 2. Inline virtual functions (if final)
class FinalDerived final : public Base {
    void func1() override final {
        // Can be inlined
    }
};
```

## Part 8: Common Interview Questions

### Question 1: What happens without virtual?

```cpp
class BaseNoVirtual {
public:
    void func() {
        cout << "BaseNoVirtual::func()" << endl;
    }
};

class DerivedNoVirtual : public BaseNoVirtual {
public:
    void func() {
        cout << "DerivedNoVirtual::func()" << endl;
    }
};

void testNoVirtual() {
    BaseNoVirtual* ptr = new DerivedNoVirtual();
    ptr->func();  // Calls BaseNoVirtual::func() - NO polymorphism!
    delete ptr;
}
```

**Answer:** Without `virtual`, function calls are resolved at compile time based on pointer type, not object type. No vtable is created.

### Question 2: Pure Virtual Functions

```cpp
class AbstractBase {
public:
    virtual void func() = 0;  // Pure virtual
    virtual ~AbstractBase() = default;
};

class Concrete : public AbstractBase {
public:
    void func() override {
        cout << "Concrete::func()" << endl;
    }
};
```

**VTable for AbstractBase:**
- Contains a null pointer or special marker for pure virtual function
- Cannot instantiate AbstractBase directly

### Question 3: Virtual Destructors

```cpp
class BaseNoVirtualDtor {
public:
    ~BaseNoVirtualDtor() {
        cout << "Base destructor" << endl;
    }
};

class DerivedNoVirtualDtor : public BaseNoVirtualDtor {
public:
    ~DerivedNoVirtualDtor() {
        cout << "Derived destructor" << endl;
    }
};

void testDestructor() {
    BaseNoVirtualDtor* ptr = new DerivedNoVirtualDtor();
    delete ptr;  // Only Base destructor called - MEMORY LEAK!
}
```

**Answer:** Always use virtual destructors in base classes to ensure proper cleanup.

## Part 9: Complete Example with Analysis

```cpp
#include <iostream>
#include <iomanip>
using namespace std;

class Base {
public:
    virtual void vfunc1() { cout << "Base::vfunc1" << endl; }
    virtual void vfunc2() { cout << "Base::vfunc2" << endl; }
    virtual ~Base() { cout << "Base::~Base" << endl; }
    int data1 = 100;
};

class Derived : public Base {
public:
    void vfunc1() override { cout << "Derived::vfunc1" << endl; }
    virtual void vfunc3() { cout << "Derived::vfunc3" << endl; }
    ~Derived() override { cout << "Derived::~Derived" << endl; }
    int data2 = 200;
};

void analyzeMemoryLayout() {
    Base b;
    Derived d;
    
    cout << "=== Base Object ===" << endl;
    cout << "Size: " << sizeof(b) << " bytes" << endl;
    cout << "Address: " << &b << endl;
    cout << "VTable ptr: " << *(void**)&b << endl;
    cout << "Data: " << b.data1 << endl;
    
    cout << "\n=== Derived Object ===" << endl;
    cout << "Size: " << sizeof(d) << " bytes" << endl;
    cout << "Address: " << &d << endl;
    cout << "VTable ptr: " << *(void**)&d << endl;
    cout << "Base data: " << d.data1 << endl;
    cout << "Derived data: " << d.data2 << endl;
    
    cout << "\n=== Polymorphism Test ===" << endl;
    Base* ptr = &d;
    ptr->vfunc1();  // Derived::vfunc1
    ptr->vfunc2();  // Base::vfunc2
}

int main() {
    analyzeMemoryLayout();
    return 0;
}
```

## Part 10: VTable Implementation Details

### How Compiler Generates VTable

```cpp
// Pseudo-code of what compiler does:

// 1. For each class with virtual functions, create vtable
void* Base_vtable[] = {
    (void*)&Base::vfunc1,
    (void*)&Base::vfunc2,
    (void*)&Base::~Base
};

void* Derived_vtable[] = {
    (void*)&Derived::vfunc1,  // Overridden
    (void*)&Base::vfunc2,      // Inherited
    (void*)&Derived::~Derived, // Overridden
    (void*)&Derived::vfunc3    // New
};

// 2. Each object gets vtable pointer
struct Base_object {
    void** vtable = Base_vtable;
    int data1;
};

struct Derived_object {
    void** vtable = Derived_vtable;
    int data1;  // Base part
    int data2;  // Derived part
};

// 3. Virtual function call becomes:
// obj->vfunc1() becomes:
//   (*(obj->vtable))[0]()
```

## Key Takeaways

### VTable Fundamentals

1. **Purpose**: Enable runtime polymorphism (dynamic dispatch)
2. **Structure**: Array of function pointers
3. **Location**: One pointer per object, one table per class
4. **Cost**: Small memory and time overhead

### Memory Layout Rules

1. **VTable pointer**: First member (8 bytes on 64-bit)
2. **Base classes**: Laid out in declaration order
3. **Multiple inheritance**: Multiple vtable pointers
4. **Virtual inheritance**: Additional vtable for shared base

### Best Practices

1. ✅ **Always use virtual destructors** in base classes
2. ✅ **Use `override` keyword** for clarity
3. ✅ **Use `final`** when inheritance shouldn't continue
4. ❌ **Don't call virtual functions** in constructors/destructors
5. ❌ **Avoid deep inheritance hierarchies** (performance)

### Interview Tips

1. **Draw memory layouts**: Show object and vtable structure
2. **Explain dispatch**: How virtual function calls work
3. **Discuss trade-offs**: Performance vs. flexibility
4. **Multiple inheritance**: Explain pointer adjustment
5. **Virtual inheritance**: Explain diamond problem solution

## Summary

Virtual Tables (vtables) are fundamental to C++ polymorphism:

- **Structure**: Array of function pointers per class
- **Storage**: One pointer per object, shared table per class
- **Purpose**: Enable runtime method resolution
- **Overhead**: Minimal (one indirection per virtual call)
- **Complexity**: Increases with multiple/virtual inheritance

Understanding vtables is essential for:
- Writing efficient C++ code
- Debugging polymorphic behavior
- Designing class hierarchies
- Optimizing performance-critical code

This knowledge is crucial for systems programming roles at companies like Pure Storage, where performance and memory efficiency matter.

