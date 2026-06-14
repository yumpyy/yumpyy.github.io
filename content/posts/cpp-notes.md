---
title: "C++ Notes"
date: 2024-06-01
lastmod: 2024-06-01
draft: false
tags: ["notes", "cpp"]
description: "Personal notes on C++ fundamentals — statements, variables, functions, and more."
---

**C++ fundamentals**: C++ is a statically-typed, compiled programming language that supports procedural, object-oriented, and generic programming paradigms. Key concepts include statements (instructions that cause program actions), objects (memory regions storing values), and functions (reusable units of statements). Every C++ program requires a `main()` function as its entry point.

## Build Configurations

- For debugging: `-ggdb` flag (gcc)
- For release builds: `-O2 -DNDEBUG`

The debugging flag produces a larger executable compared to release builds.

## Statements and Structure of a Program

A statement is an instruction that causes the program to perform an action.

Types of statements:
- Declaration statements
- Jump statements
- Expression statements
- Compound statements
- Selection statements (conditional)
- Iteration statements (loops)
- Try blocks

### Functions and the main function

Functions are units of statements. Every C++ program must have a `main()` function.

```cpp
int main() {
  // instructions here
  return 0;
}
```

`return 0` tells the Operating System that everything went fine.

## Objects and Variables

An object is a region in memory that can store a value. An object with a name is called a variable.

### Variable instantiation

Creating an object and assigning it an address is called variable instantiation.

```
data-type variable-name;
```

eg. `int x;`, `double width;`

### Variable assignment and initialization

**Assignment** is defining a variable and providing it a value in two separate statements:
```cpp
int x;
x = 1;
```

**Initialization** is defining a variable and providing a value in the same statement.

Types of initialization:
- `int a;` — no initialization
- `int a = 5;` — copy initialization
- `int a(5);` — direct initialization
- `int a {5};` — direct list initialization (C++11)
- `int a = {5};` — copy list initialization
- `int a {};` — value initialization

List initialization prevents narrowing conversions:
```cpp
int a {4.5};  // Error: narrowing conversion
```

### Unused initialized variable warnings

Use `[[maybe_unused]]` attribute (C++17):
```cpp
[[maybe_unused]] double pi { 3.14159 };
```

## iostream: cout, cin, endl

```cpp
#include <iostream>

int main() {
  std::cout << "hello" << std::endl;
  int x;
  std::cin >> x;
  return 0;
}
```

- `std::endl` prints newline and flushes buffer (slow)
- `\n` only prints newline (preferred)

## Keywords and Naming Identifiers

C++ has 92 reserved keywords.

Identifier rules:
- Can't be a keyword
- Can only contain letters, numbers, underscore
- Must begin with a letter
- Case sensitive

## Functions

### Function return values

```cpp
int add(int x, int y) {
  return x + y;
}
```

### Void functions

```cpp
void hello() {
  std::cout << "hello";
}
```

### Forward declarations

```cpp
int add(int x, int y); // declaration

int main() {
  add(3, 4);
}

int add(int x, int y) { // definition
  return x + y;
}
```

## Namespaces

Using directive (`using namespace std;`) should be avoided to prevent naming conflicts. Use `std::` prefix instead.

## Header Files

- Use angled brackets (`<iostream>`) for system headers
- Use double quotes (`"myheader.h"`) for your own headers
- Always use header guards (`#ifndef` / `#define` / `#endif` or `#pragma once`)

## Data Types

### Signed integers

| Type | Minimum Size |
|------|-------------|
| short | 16 bits |
| int | 16 bits |
| long | 32 bits |
| long long | 64 bits |

### Unsigned integers

Range: 0 to (2^n)-1. Avoid using unless necessary — easy to overflow.

### Floating point numbers

- `float`: 6-9 digits precision
- `double`: 15-18 digits precision (default for literals)

Use `f` suffix for float literals: `5.0f`

### Boolean

```cpp
bool b {true};  // stored as 1
bool b {};      // default false
```

Use `std::boolalpha` to print "true"/"false" instead of 1/0.

### Chars

```cpp
char ch {'a'};  // stored as ASCII value 97
```

Chars are integral types — can be converted to int via `static_cast<int>(ch)`.

## Type Conversion

- **Implicit**: compiler does it automatically (may have narrowing issues)
- **Explicit**: use `static_cast<target_type>(expression)`

```cpp
static_cast<int>(ch);  // char to int
```

## Constants

```cpp
const double pi { 3.14159 };       // compile-time constant
constexpr double gravity { 9.8 };  // evaluated at compile time
```

## Related Reading

- [Matra](/projects/matra/): Custom tokenizer for Indic languages
- [Desi Maximalism](/projects/desi-maximalism/): Text-to-image LoRA for South Asian aesthetics
- [Connman systemd-resolved](/posts/connman-systemd-resolved/): Linux network configuration
- [Gemini Gemtext](/posts/gemini-gemtext/): Alternative protocol reference
