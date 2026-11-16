# C Essentials

> **Tier:** T2 (working knowledge) · **Related:** [C++ fundamentals](02-cpp-fundamentals.md), [C++ OOP](03-cpp-oop.md), [Programming fundamentals](../../01-cs-fundamentals/01-programming-fundamentals.md), [Operating systems](../../01-cs-fundamentals/05-operating-systems.md), [C/C++ rapid-fire](99-interview-questions.md)
> **On your resume:** C in your skills list. Pointers and manual memory are also the foundation of every C++ question in your OOP and DSA rounds.

## Quick revision
- Build pipeline: **preprocess** (`#include`, `#define`) -> **compile** (C to assembly) -> **assemble** (object file `.o`) -> **link** (resolve symbols, produce the executable).
- `undefined reference to 'foo'` is a **linker** error (declared, never defined or not linked). Syntax and type errors come from the compiler.
- `sizeof(char)` is 1 by definition. Typical 64-bit sizes: `int` 4, `long` 8 on Linux/macOS but 4 on Windows, `long long` 8, any pointer 8. Use `<stdint.h>` (`int32_t`, `uint64_t`) when width matters.
- `&x` takes an address, `*p` dereferences. `p + 1` moves `sizeof(*p)` bytes, and `a[i]` means exactly `*(a + i)`.
- Arrays **decay** to a pointer to their first element in most expressions, including function arguments, so `sizeof(arr)` inside a function gives the pointer size. Pass the length separately.
- A C string is a `char` array ending in `'\0'`. `strlen` does not count the terminator, so allocate `strlen(s) + 1`. String literals are read-only.
- `strcpy`, `strcat`, `sprintf` and `gets` (removed in C11) never check the destination size: classic buffer overflows. Prefer `snprintf` and `fgets`.
- `malloc` gives uninitialized memory, `calloc` zeroes it, `realloc` may move the block (assign the result to a temporary), `free(NULL)` is a no-op.
- Leak: the last pointer to a block is lost. Dangling pointer: points to freed or dead memory. Double free: undefined behavior that corrupts the heap.
- `struct` members are laid out in declaration order with padding; a `union` overlays all members in one block sized for the largest.
- `static` at file scope = internal linkage (private to the `.c` file); inside a function = keeps its value between calls. `const int *p` = read-only data, `int *const p` = pointer that cannot be reseated (read right to left).
- Macros are text substitution: parenthesize every argument and the whole body, never pass arguments with side effects. Include guards (`#ifndef X_H` / `#define X_H` / `#endif`) or `#pragma once` stop a header being processed twice.
- C passes everything **by value**. To let a function change your variable, pass its address.
- Undefined behavior (UB): out-of-bounds access, use after free, signed overflow, uninitialized reads, null dereference, modifying a string literal.

## 1. Program structure and the build pipeline

Every C program has exactly one `main`; its return value becomes the process exit status (0 means success).

```c
#include <stdio.h>          /* preprocessor pastes the declaration of printf */
#define GREETING "Hello"    /* preprocessor text replacement */

int main(void) {
    printf("%s, %s!\n", GREETING, "world");
    return 0;
}
```

| Stage | What happens | gcc command | Output |
|---|---|---|---|
| Preprocess | Expands `#include` and macros, evaluates `#if`/`#ifdef`, strips comments | `gcc -E main.c -o main.i` | Plain C text |
| Compile | Parses, type-checks, optimizes, emits assembly | `gcc -S main.i` | `main.s` |
| Assemble | Assembly to machine code plus a symbol table | `gcc -c main.s` | `main.o` |
| Link | Joins object files and libraries, resolves calls and globals | `gcc main.o util.o -o app` | Executable |

Knowing the stages lets you explain errors: a missing semicolon is a compile error, while `undefined reference to 'parse_config'` means the linker saw a call but no definition (you forgot to compile `config.c` or to link a library with `-l`). A good development build: `gcc -std=c11 -Wall -Wextra -g -fsanitize=address,undefined main.c -o main`.

## 2. Types and sizes

The standard guarantees only minimum ranges and the ordering `char <= short <= int <= long <= long long`. Real sizes depend on the platform.

| Type | Typical size (64-bit) | Notes |
|---|---|---|
| `char` | 1 | Whether plain `char` is signed is implementation-defined |
| `short` / `int` | 2 / 4 | `int` is at least 16 bits, 32 on every mainstream platform |
| `long` | 8 on Linux/macOS, 4 on Windows | LP64 vs LLP64 data models |
| `long long` | 8 | At least 64 bits |
| `float` / `double` | 4 / 8 | IEEE 754 in practice |
| any pointer | 8 (4 on 32-bit) | Size of the address, not of the data |
| `size_t` | 8 | Unsigned result type of `sizeof`; print with `%zu` |

```c
unsigned int u = 1;
int i = -1;
if (i < u) puts("less");     /* never prints: i converts to unsigned 4294967295 */
double avg = 7 / 2;          /* 3.0: integer division happens before the conversion */
```

## 3. Pointers

A pointer stores a memory address. Its type says what lives there, which tells the compiler how many bytes to read and how far `p + 1` moves.

### 3.1 Basics and arithmetic

```c
int x = 10;
int *p = &x;                        /* p holds the address of x */
*p = 20;                            /* write through p: x is now 20 */
int *q, r;                          /* careful: q is int*, r is a plain int */

int arr[5] = {10, 20, 30, 40, 50};
int *s = arr;                       /* decays to &arr[0] */
printf("%d\n", *(s + 2));           /* 30: moved 2 * sizeof(int) = 8 bytes */
printf("%d\n", s[3]);               /* 40: s[3] is *(s + 3) */
printf("%td\n", &arr[4] - &arr[1]); /* 3: difference in elements (ptrdiff_t) */
```

Arithmetic is valid only inside one array (plus the one-past-the-end position). An array is not a pointer: `sizeof(arr)` is 20 and `arr = s;` does not compile.

### 3.2 Pointer to pointer

You need `T **` when a function must change the caller's pointer itself (allocate it, advance it, reset it). `char **argv` and dynamic 2D arrays use the same idea.

```c
int make_counter(int **out) {
    *out = malloc(sizeof **out);    /* sizeof **out == sizeof(int), survives type changes */
    if (*out == NULL) return -1;
    **out = 0;
    return 0;
}

int *counter = NULL;
if (make_counter(&counter) == 0) {  /* the callee sets OUR pointer */
    (*counter)++;
    free(counter);
}
```

### 3.3 Function pointers

A function pointer stores a function's address. It enables callbacks, dispatch tables and plug-in behavior: C's version of polymorphism.

```c
#include <stdio.h>
#include <stdlib.h>

int add(int a, int b) { return a + b; }
int mul(int a, int b) { return a * b; }
typedef int (*BinOp)(int, int);          /* pointer to a function (int, int) -> int */

int cmp_int(const void *a, const void *b) {
    int x = *(const int *)a, y = *(const int *)b;
    return (x > y) - (x < y);            /* avoids the overflow that x - y can cause */
}

int main(void) {
    BinOp op = add;
    printf("%d\n", op(2, 3));            /* 5 */
    op = mul;
    printf("%d\n", op(2, 3));            /* 6 */

    int nums[] = {3, 1, 2};
    qsort(nums, 3, sizeof nums[0], cmp_int);          /* qsort calls back into cmp_int */
    printf("%d %d %d\n", nums[0], nums[1], nums[2]);  /* 1 2 3 */
    return 0;
}
```

### 3.4 void pointers and NULL
- `void *` is a generic object pointer. In C any object pointer converts to and from it without a cast (C++ requires one), which is why `malloc` returns `void *`. You cannot dereference it or do arithmetic on it in standard C (GCC allows arithmetic as an extension): cast to a concrete type first, as `cmp_int` does.
- `NULL` is a null pointer constant (often `((void *)0)`). Dereferencing it is UB, usually a segmentation fault. Initialize pointers to `NULL` and check before use.

## 4. Array decay and sizeof

```c
#include <stdio.h>

void print_size(int arr[]) {                   /* the parameter is really int *arr */
    printf("%zu\n", sizeof(arr));              /* 8: size of a pointer */
}

int main(void) {
    int a[10];
    printf("%zu\n", sizeof(a));                /* 40 */
    printf("%zu\n", sizeof(a) / sizeof(a[0])); /* 10: element-count idiom */
    print_size(a);                             /* 8 */
    return 0;
}
```

- Decay does not happen for `sizeof a`, for `&a`, or when a string literal initializes a char array. `a` and `&a` have the same address but different types: `a + 1` moves 4 bytes, `&a + 1` moves 40.
- Because the size is lost, C APIs take pointer-plus-length pairs: `int sum(const int *a, size_t n)`. GCC warns about the `sizeof` trap above by default.
- 2D arrays are contiguous and row-major; a function needs every dimension except the first: `void show(int m[][4], int rows)`.

## 5. C strings and buffer overflows

```c
char s[] = "hi";                     /* 3 bytes {'h','i','\0'}: a writable copy */
const char *t = "hi";                /* points into read-only storage: writing is UB */
printf("%zu %zu\n", sizeof s, strlen(s));    /* 3 2 */

char buf[8];
strcpy(buf, "definitely too long");          /* overflow: UB, can overwrite the return address */
snprintf(buf, sizeof buf, "%s", input);      /* safe: truncates and always terminates */
fgets(line, sizeof line, stdin);             /* safe line input (keeps the '\n') */

size_t n = strlen(src) + 1;                  /* +1 for the terminator */
char *copy = malloc(n);
if (copy) memcpy(copy, src, n);
```

- `strncpy(dst, src, n)` does **not** add `'\0'` when `src` has `n` or more characters; terminate manually.
- Compare contents with `strcmp(a, b) == 0`. `a == b` compares addresses.
- Stack canaries, ASLR and non-executable stacks make overflow exploits harder, but the overflow itself is still UB.

## 6. Dynamic memory

| Function | What it does | Notes |
|---|---|---|
| `malloc(bytes)` | Allocates uninitialized memory | Returns `NULL` on failure |
| `calloc(count, size)` | Allocates `count * size` bytes, all zero | Handy for arrays and counters |
| `realloc(p, bytes)` | Resizes a block, possibly moving it | Contents preserved; on failure returns `NULL` and `p` stays valid |
| `free(p)` | Returns the block to the allocator | `free(NULL)` is fine; freeing twice is UB |

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    size_t cap = 4, len = 0;
    int *v = malloc(cap * sizeof *v);
    if (!v) return 1;

    for (int i = 0; i < 10; i++) {
        if (len == cap) {
            cap *= 2;
            int *tmp = realloc(v, cap * sizeof *v);   /* never assign straight back to v */
            if (!tmp) { free(v); return 1; }          /* v is still valid on failure */
            v = tmp;
        }
        v[len++] = i * i;
    }
    printf("%d\n", v[9]);   /* 81 */
    free(v);
    v = NULL;               /* no dangling pointer left behind */
    return 0;
}
```

| Bug | Typical cause | Symptom |
|---|---|---|
| Leak | Overwriting or losing the last pointer, early `return` without `free` | Memory grows over time |
| Dangling pointer | Use after `free`, returning the address of a local | Garbage values, crashes far from the cause |
| Double free | Two `free` calls on one block | Heap corruption, abort |
| Uninitialized read | Reading `malloc`ed memory or locals before writing | Different results per run |

Find these with `-g -fsanitize=address` (reports overflows, use after free and leaks with stack traces) or `valgrind --leak-check=full ./app` on Linux.

## 7. struct, union, enum, typedef

```c
typedef enum { RED, GREEN = 5, BLUE } Color;   /* RED = 0, GREEN = 5, BLUE = 6 */

typedef struct {
    char tag;           /* 1 byte, then 3 bytes of padding so id is 4-byte aligned */
    int id;
    Color color;
} Item;                 /* sizeof(Item) is typically 12, not 9 */

typedef union {
    int i;
    float f;
    unsigned char bytes[4];
} Value;                /* all members share the same 4 bytes */

Item it = { .tag = 'a', .id = 7, .color = BLUE };   /* designated initializers (C99) */
Item *ip = &it;
ip->id = 8;             /* ip->id is (*ip).id */
```

- Padding keeps members aligned; ordering members from largest to smallest usually shrinks a struct.
- A union holds one meaningful member at a time. The common pattern is a **tagged union**: a struct with an `enum kind` plus a union of payloads.
- Struct assignment and pass-by-value copy every member. Pointer members are copied as addresses (a shallow copy), so pass big structs as `const Item *`. `typedef` only creates an alias, not a new type.

## 8. static and const

```c
static int request_count = 0;      /* file scope: invisible to other .c files (same for static functions) */

int next_id(void) {
    static int id = 0;             /* initialized once, lives for the whole program */
    return ++id;                   /* 1, 2, 3, ... across calls */
}
```

| Declaration | `*p = 1` | `p = &y` | Read as |
|---|---|---|---|
| `const int *p` (same as `int const *p`) | error | OK | pointer to const int |
| `int *const p = &x` | OK | error | const pointer to int |
| `const int *const p = &x` | error | error | const pointer to const int |

A `const char *` parameter (`size_t my_strlen(const char *s)`) promises callers you will not modify their data, and lets them pass string literals.

## 9. Macros vs inline functions

```c
#define SQUARE(x) ((x) * (x))
#define BAD_SQUARE(x) x * x
static inline int square(int x) { return x * x; }

int a = BAD_SQUARE(1 + 2);   /* expands to 1 + 2 * 1 + 2 = 5, not 9 */
int i = 3;
int b = SQUARE(i++);         /* ((i++) * (i++)): two unsequenced modifications, UB */
int c = square(i++);         /* fine: the argument is evaluated once */
```

| | Macro | `inline` function |
|---|---|---|
| Type checking | none (text substitution) | full |
| Argument evaluation | possibly several times | exactly once |
| Scope and debugging | global, invisible in the debugger | normal function rules |
| Works for any type | yes | no (one signature in C) |

Macros remain the right tool for include guards, conditional compilation (`#ifdef DEBUG`) and `__FILE__`/`__LINE__` logging. `inline` is only a hint; the optimizer decides.

## 10. Header files and include guards

```c
/* point.h: declarations only */
#ifndef POINT_H
#define POINT_H

typedef struct { int x, y; } Point;
int manhattan(Point a, Point b);
extern int point_count;          /* declared here, defined in exactly one .c file */

#endif
```

```c
/* point.c: definitions */
#include <stdlib.h>
#include "point.h"

int point_count = 0;
int manhattan(Point a, Point b) { return abs(a.x - b.x) + abs(a.y - b.y); }
```

Defining a non-static function or global inside a header that two `.c` files include causes a "multiple definition" link error. `#include <x.h>` searches system paths; `#include "x.h"` searches the current directory first.

## 11. Pass-by-value and simulating pass-by-reference

```c
void swap_wrong(int a, int b) { int t = a; a = b; b = t; }      /* swaps local copies */
void swap(int *a, int *b) { int t = *a; *a = *b; *b = t; }

int x = 1, y = 2;
swap_wrong(x, y);   /* x = 1, y = 2 */
swap(&x, &y);       /* x = 2, y = 1 */

int divmod(int a, int b, int *rem) { *rem = a % b; return a / b; }   /* two results */
```

## 12. Undefined behavior

UB means the standard places no requirements on the result. Optimizers assume UB never happens, so "it works on my machine" proves nothing.

```c
int a[3]; a[3] = 1;                       /* out-of-bounds write */
int *p = malloc(sizeof *p); free(p); *p = 1;   /* use after free */
int x; printf("%d", x);                   /* reading an uninitialized local */
int big = INT_MAX; big = big + 1;         /* signed overflow (unsigned wraps, which is defined) */
int i = 0; i = i++ + 1;                   /* unsequenced modification: UB in C (C++17 defined this case) */
char *s = "hi"; s[0] = 'H';               /* modifying a string literal */
```

Related terms: **implementation-defined** (documented per platform, such as `sizeof(int)`) and **unspecified** (one of several outcomes, such as the evaluation order of function arguments). Catch UB with warnings, sanitizers and Valgrind.

## Common pitfalls
- Using `sizeof(arr)` on an array parameter, or forgetting the `+ 1` for `'\0'`.
- `p = realloc(p, n)`: on failure the only pointer to the old block is lost (a leak).
- Comparing strings with `==`, or using `strncpy` and assuming termination.
- `if (x = 5)` instead of `==`, `scanf("%d", n)` without `&`, missing `break` in a `switch`.
- Storing `getchar()` in a `char`: it returns `int` so that `EOF` stays distinguishable.
- Returning the address of a local, keeping a pointer after `free`, mixing signed and unsigned in comparisons.

## How to talk about this in an interview
- Be honest about where C sits for you: it underpins the C++ you used for OOP and DSA at university, while your daily work is Rails and TypeScript. Interviewers mainly use C questions to test whether you understand memory.
- Show the habit, not just the syntax: "For every allocation I ask who owns it and where it is freed." The same thinking applies to closing DB connections, files and sockets in web code.
- A useful bridge if asked why C matters to a web developer: MRI Ruby and many native gems (such as `pg`) are written in C, so a segfault in production usually means a native extension.
- > **Fill in:** which C or low-level assignments did you do or help students with as a TA (string functions, a `malloc`-based linked list, file handling)?

## Interview questions

**Q1. What happens when you compile a C program?**
The preprocessor expands includes and macros, the compiler turns C into assembly (type checking and optimization happen here), the assembler produces an object file, and the linker combines object files and libraries into an executable by resolving symbols.
*Follow-up:* What does "undefined reference" mean? The linker found a use of a symbol but no definition: a missing source file or `-l` library.

**Q2. What is the difference between `const int *p` and `int *const p`?**
`const int *p` is a pointer to read-only data: `*p = 1` fails but `p` can point elsewhere. `int *const p` is a read-only pointer: it must be initialized, cannot be reseated, but `*p` can be modified. Reading right to left helps: "p is a const pointer to int".

**Q3. Why does `sizeof` give 40 for an array in `main` but 8 inside a function?**
When passed to a function the array decays to a pointer to its first element, and `int arr[]` in a parameter list really means `int *arr`. So `sizeof` measures the pointer. That is why C functions take a separate length parameter.
*Follow-up:* When does an array not decay? As the operand of `sizeof` or `&`, and when a string literal initializes a char array.

**Q4. Compare `malloc`, `calloc` and `realloc`. What happens if `realloc` fails?**
`malloc` allocates uninitialized bytes, `calloc` allocates and zeroes an array, and `realloc` resizes an existing block, possibly moving it and copying the contents. On failure `realloc` returns `NULL` and leaves the original block valid, so assign the result to a temporary and keep the old pointer for cleanup.

**Q5. What are memory leaks, dangling pointers and double frees? How do you find them?**
A leak loses the last reference to allocated memory. A dangling pointer points to memory that was freed or went out of scope. A double free releases the same block twice and corrupts the allocator. Use AddressSanitizer (`-fsanitize=address`) or Valgrind, and set pointers to `NULL` after freeing.

**Q6. Why is `strcpy` dangerous? What is a buffer overflow?**
`strcpy` copies until the source's `'\0'` without knowing the destination size. If the source is longer, it writes past the buffer: a buffer overflow, which is UB and on the stack can overwrite the return address (a security exploit). Use `snprintf`, or compute sizes and use `memcpy`.

**Q7. What is the difference between a struct and a union? Why is `sizeof` of a struct often bigger than the sum of its members?**
A struct gives each member its own storage; a union overlays all members in the same memory, so only one is meaningful at a time. Structs include padding so each member is aligned to its natural boundary, plus trailing padding so arrays of the struct stay aligned.

**Q8. What does `static` mean in C?**
At file scope it gives a function or global internal linkage, hiding it from other translation units. On a local variable it means static storage duration: initialized once (to 0 by default) and preserved across calls.
*Follow-up:* Is `next_id()` with a static counter thread-safe? No: two threads running `++id` at once is a data race; use a mutex or C11 atomics.

**Q9. Macros vs inline functions: which do you prefer and why?**
Inline functions, for type checking, single evaluation of arguments and normal scoping. Macros are textual, so `SQUARE(i++)` evaluates `i++` twice (UB) and a missing parenthesis changes precedence. Keep macros for include guards, conditional compilation and `__FILE__`/`__LINE__` tricks.

**Q10. C is pass-by-value. How do you write a `swap` function, and how would a function allocate memory for the caller?**
Pass addresses: `void swap(int *a, int *b)` and call `swap(&x, &y)`. To give the caller a newly allocated block, either return the pointer or take a pointer to the caller's pointer (`int **out`) and assign `*out = malloc(...)`.

**Q11. What is undefined behavior? Give examples.**
Code whose result the standard does not define, so anything can happen and optimizers may assume it never occurs. Examples: out-of-bounds access, use after free, signed integer overflow, reading uninitialized variables, null dereference, modifying a string literal, modifying a variable twice without sequencing (`i = i++ + 1`).
*Follow-up:* Is unsigned overflow UB? No, unsigned arithmetic wraps modulo 2^N.

**Q12. Why do headers need include guards, and what goes in a header vs a `.c` file?**
Without guards, a header included twice in one translation unit causes redefinition errors for types. Headers hold declarations (prototypes, types, `extern` globals, macros); `.c` files hold definitions. Defining a non-static function in a header breaks the link step with "multiple definition".
