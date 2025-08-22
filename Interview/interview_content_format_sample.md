# 🎯 C Programming Interview Preparation

## �� **Quick Navigation**
- [Common Questions](#common-questions)
- [Problem-Solving Examples](#problem-solving-examples)
- [Practice Problems](#practice-problems)
- [Resources](#resources)

## �� **Quick Reference: Key Concepts**
- **Pointers & Memory**: Address arithmetic, pointer arithmetic, void pointers
- **Volatile & Const**: When to use, common misconceptions
- **Memory Management**: Stack vs heap, memory leaks, fragmentation
- **Embedded C**: Bit manipulation, register access, optimization

## �� **Common Interview Questions**

### **Question 1: Explain the difference between volatile and const**
**Why this matters**: Understanding these qualifiers is crucial for embedded systems.

**Answer Structure**:
- `const`: Prevents modification, enables compiler optimization
- `volatile`: Prevents optimization, ensures memory access
- Combined usage: `const volatile` for read-only hardware registers

**Example**:
```c
// Read-only hardware register
const volatile uint32_t* const STATUS_REG = (uint32_t*)0x40000000;

// Compiler cannot optimize away this read
uint32_t status = *STATUS_REG;
```

**Follow-up Questions**:
- When would you use `const volatile`?
- How does this affect compiler optimization?

### **Question 2: Implement a function to reverse bits in a byte**
**Problem**: Write a function that reverses the bit order of an 8-bit value.

**Solution Approach**:
1. Use bit shifting and masking
2. Consider lookup table for optimization
3. Handle edge cases

**Solution**:
```c
uint8_t reverse_bits(uint8_t byte) {
    uint8_t result = 0;
    for (int i = 0; i < 8; i++) {
        result = (result << 1) | (byte & 1);
        byte >>= 1;
    }
    return result;
}
```

**Optimization**: Use lookup table for O(1) performance
**Follow-up**: How would you optimize this for 32-bit values?

## 🧪 **Practice Problems**

### **Problem 1: Memory Layout Analysis**
**Scenario**: Given this structure, what is the memory layout and size?
```c
struct example {
    char a;
    int b;
    char c;
};
```

**Expected Answer**: 
- Size: 12 bytes (due to padding)
- Layout: a(1) + padding(3) + b(4) + c(1) + padding(3)

**Key Points**: Structure alignment, padding rules, memory efficiency

### **Problem 2: Interrupt Safety**
**Scenario**: Is this code safe to call from an interrupt handler?
```c
void isr_function(void) {
    static int counter = 0;
    counter++;
    // ... more code
}
```

**Analysis**: 
- ✅ Safe: Static variable, no function calls
- ⚠️ Consider: Volatile if shared with main loop
- ❌ Avoid: Dynamic allocation, complex operations

## ✅ **Self-Assessment Checklist**

### **Basic Understanding** ✅
- [ ] Explain pointer arithmetic
- [ ] Describe memory layout of structures
- [ ] Understand volatile and const qualifiers

### **Problem Solving** ✅
- [ ] Can implement bit manipulation functions
- [ ] Can analyze memory usage and optimization
- [ ] Can identify potential issues in code

### **Advanced Concepts** ✅
- [ ] Understand compiler optimization effects
- [ ] Can design memory-efficient data structures
- [ ] Can implement interrupt-safe code

## 🔗 **Related Topics**
- [Memory Management](../Embedded_C/Memory_Management.md)
- [Type Qualifiers](../Embedded_C/Type_Qualifiers.md)
- [Bit Manipulation](../Embedded_C/Bit_Manipulation.md)

## 📚 **Additional Resources**
- **Books**: "C Programming: A Modern Approach" by K.N. King
- **Online**: [Embedded.com C Programming](https://www.embedded.com/)
- **Practice**: [LeetCode Embedded Problems](https://leetcode.com/)