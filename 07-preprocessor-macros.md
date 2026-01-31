# 第七章：预处理宏的函数式本质 —— 编译期的λ演算

## 7.1 宏：被误解的巨人

C语言的预处理宏常常被人诟病。它们被视为危险的遗产，是通向未定义行为的快速通道。但在这层粗糙的外表下，宏隐藏着一种惊人的力量：**它们是编译期的函数式编程语言**。

王小波说过："猪的一生是悲惨的，但如果它能思考，它可能会觉得自己的一生很幸福。"很多C程序员就像这样的猪——他们使用宏，却从不理解宏的真正本质。

让我们从基础开始。宏是**文本替换**机制：

```c
#define SQUARE(x) ((x) * (x))

int y = SQUARE(5);      // 展开为 ((5) * (5))
int z = SQUARE(a + b);  // 展开为 ((a + b) * (a + b))
```

注意括号的重要性。没有它们：

```c
#define BAD_SQUARE(x) x * x

BAD_SQUARE(a + b);  // 展开为 a + b * a + b = a + (b*a) + b ≠ (a+b)^2
```

## 7.2 宏作为编译期函数

宏可以接受"参数"并"返回"结果。虽然这些都在编译时发生，但其结构与运行时函数惊人地相似：

```c
// 运行时函数
int add(int a, int b) { return a + b; }

// 编译期宏
#define ADD(a, b) ((a) + (b))
```

但宏更强大——它们可以操作**代码本身**：

```c
#define MAX(a, b) ((a) > (b) ? (a) : (b))

// 使用
int m = MAX(x++, y++);  // 展开为 ((x++) > (y++) ? (x++) : (y++))
                        // 有副作用的参数可能被求值两次！
```

这是宏的主要陷阱：它们不知道"求值"的概念，只知道文本替换。

## 7.3 多行宏与表达式语句

复杂的宏可以跨越多行，使用`\`续行：

```c
#define SWAP(type, a, b) \
    do {                 \
        type temp = (a); \
        (a) = (b);       \
        (b) = temp;      \
    } while(0)

// 使用
SWAP(int, x, y);
```

`do { ... } while(0)`是一个惯用法，它确保宏在任何上下文中都能正确工作（比如`if`语句的分支）。

GNU C提供了**表达式语句**扩展，让宏可以"返回值"：

```c
#define MAX_WITH_TYPE(type, a, b) \
    ({                            \
        type _a = (a);            \
        type _b = (b);            \
        _a > _b ? _a : _b;        \
    })

int m = MAX_WITH_TYPE(int, x++, y++);  // 每个参数只求值一次
```

注意局部变量名前的下划线——这是为了避免与宏参数名冲突。

## 7.4 条件编译：编译期的模式匹配

`#if`、`#ifdef`等指令让我们可以在编译期进行条件判断：

```c
#ifdef DEBUG
    #define DEBUG_PRINT(fmt, ...) fprintf(stderr, fmt, ##__VA_ARGS__)
#else
    #define DEBUG_PRINT(fmt, ...) ((void)0)
#endif
```

`##__VA_ARGS__`是GCC扩展，处理可变参数前面的逗号问题（C++20和C23有更标准的方案）。

我们可以用条件编译实现类似函数式语言中的**编译期配置**：

```c
// 选择整数类型
#if PLATFORM_BITS == 64
typedef int64_t Int;
#else
typedef int32_t Int;
#endif

// 静态断言（C11 标准）
_Static_assert(sizeof(Int) >= 4, "Int must be at least 32 bits");
```

## 7.5 X宏：数据即代码

**X宏**（X-Macros）是一种强大的技术，它把数据定义为宏，然后用不同的方式展开：

```c
// 定义所有错误码
#define ERROR_CODES(X) \
    X(OK, 0, "Success") \
    X(NOT_FOUND, 404, "Resource not found") \
    X(PERMISSION_DENIED, 403, "Access denied") \
    X(INTERNAL_ERROR, 500, "Internal server error")

// 展开为枚举
#define DEFINE_ENUM(name, code, msg) name = code,
enum ErrorCode {
    ERROR_CODES(DEFINE_ENUM)
};
#undef DEFINE_ENUM

// 展开为错误消息数组
#define DEFINE_MSG(name, code, msg) [code] = msg,
const char *error_messages[] = {
    ERROR_CODES(DEFINE_MSG)
};
#undef DEFINE_MSG

// 展开为switch case
#define DEFINE_CASE(name, code, msg) case code: return msg;
const char *error_to_string(enum ErrorCode e) {
    switch (e) {
        ERROR_CODES(DEFINE_CASE)
        default: return "Unknown error";
    }
}
#undef DEFINE_CASE
```

这是**代码生成**的经典例子——我们在一个地方定义数据，然后在多个地方使用它。这比手动保持多个列表同步要安全得多。

## 7.6 递归宏？

标准C宏不能递归——如果宏在展开时引用了自己，它不会被再次展开：

```c
#define RECURSIVE RECURSIVE  // 不会无限展开
```

但我们可以用**间接**来模拟递归。这里是一个经典的技巧：

```c
#define EMPTY()
#define DEFER(id) id EMPTY()
#define OBSTRUCT(...) __VA_ARGS__ DEFER(EMPTY)()
#define EXPAND(...) __VA_ARGS__

#define A() 123

// A() 直接展开为 123
A()  // 123

// DEFER(A)() 延迟展开
DEFER(A)()  // A () - 注意A和()之间有空格，所以没展开

// 需要再次展开
EXPAND(DEFER(A)())  // 123
```

利用这种技术，一些疯狂的程序员实现了真正的**递归宏**。例如，Boost.Preprocessor库（用于C++）使用类似技术实现了循环、条件等。

让我们看一个简化的例子——计算参数个数：

```c
#define COUNT(...) COUNT_IMPL(__VA_ARGS__, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1, 0)
#define COUNT_IMPL(_10, _9, _8, _7, _6, _5, _4, _3, _2, _1, N, ...) N

COUNT(a)        // 1
COUNT(a, b)     // 2
COUNT(a, b, c)  // 3
```

原理：`__VA_ARGS__`会把参数推到左边，`N`最终落在第11个位置。

## 7.7 泛型编程：_Generic

C11引入了`_Generic`，让我们可以根据类型选择表达式：

```c
#define MAX(x, y) _Generic((x), \
    int: max_int,              \
    double: max_double,        \
    const char*: max_string    \
)(x, y)

int max_int(int a, int b) { return a > b ? a : b; }
double max_double(double a, double b) { return a > b ? a : b; }
const char* max_string(const char* a, const char* b) {
    return strcmp(a, b) > 0 ? a : b;
}

// 使用
int m1 = MAX(1, 2);              // 调用 max_int
double m2 = MAX(1.5, 2.5);       // 调用 max_double
const char *m3 = MAX("a", "b");  // 调用 max_string
```

这是C语言向**ad-hoc多态**（即函数重载）的妥协。它在标准库中也有应用，比如`tgmath.h`中的类型通用数学函数。

## 7.8 容器泛型：模拟模板

我们可以用宏模拟C++的模板，生成类型特定的代码：

```c
// 在 vector.h 中
#ifndef VECTOR_H
#define VECTOR_H

#define DEFINE_VECTOR(T, Name) \
    typedef struct {          \
        T *data;              \
        size_t size;          \
        size_t capacity;      \
    } Name;                   \
                              \
    static inline void Name##_init(Name *v) { \
        v->data = NULL;       \
        v->size = 0;          \
        v->capacity = 0;      \
    }                         \
                              \
    static inline void Name##_push(Name *v, T val) { \
        if (v->size >= v->capacity) {               \
            v->capacity = v->capacity ? v->capacity * 2 : 4; \
            v->data = realloc(v->data, v->capacity * sizeof(T)); \
        }                                            \
        v->data[v->size++] = val;                   \
    }                                                \
                              \
    static inline T Name##_get(const Name *v, size_t i) { \
        return v->data[i];   \
    }                        \
                              \
    static inline void Name##_free(Name *v) { \
        free(v->data);       \
        v->data = NULL;      \
        v->size = 0;         \
        v->capacity = 0;     \
    }

#endif
```

使用：

```c
#include "vector.h"

DEFINE_VECTOR(int, IntVector)
DEFINE_VECTOR(double, DoubleVector)

IntVector iv;
IntVector_init(&iv);
IntVector_push(&iv, 42);
IntVector_free(&iv);
```

每个`DEFINE_VECTOR`展开为完整的类型定义和函数实现。`static inline`确保不会在多个翻译单元中产生冲突。

## 7.9 宏的函数式本质

现在让我们从理论高度看待宏。

λ演算是函数式编程的理论基础，包含三个要素：
1. **变量引用**
2. **函数抽象**（λx.M）
3. **函数应用**（M N）

C宏提供了：
1. **参数**（变量）
2. **宏定义**（类似于抽象）
3. **宏展开**（类似于应用）

关键区别在于：
- λ演算是**惰性的**——参数在需要时才求值
- 宏展开是**及早的**——参数在展开前被文本替换

但这种"及早文本替换"实际上给了宏一种独特的力量：它们可以操作**未求值的代码**。这是**元编程**（metaprogramming）的本质。

在Lisp家族中，宏是代码生成代码的机制。C宏虽然简陋，但在同一光谱上。

## 7.10 最佳实践与警告

宏是强大的工具，但也是危险的工具。一些准则：

1. **优先使用内联函数**：如果宏可以用函数实现，用函数
2. **括号包围一切**：`#define SQUARE(x) ((x) * (x))`
3. **避免副作用**：`MAX(x++, y++)` 是危险的
4. **使用`do { ... } while(0)`**：对于多语句宏
5. **大写命名**：区分宏和函数
6. **考虑用`#undef`**：不要让宏定义泄露到不该去的地方

## 7.11 练习

1. 实现一个`FOR_EACH`宏，对列表中的每个元素应用操作：

```c
int arr[] = {1, 2, 3, 4, 5};
FOR_EACH(int, x, arr, 5, {
    printf("%d\n", x);
});
```

2. 用X宏实现一个状态机，自动生成状态枚举、转换表和调试字符串。

3. 使用`_Generic`实现类型安全的`println`宏，根据参数类型自动选择格式字符串。

4. 研究`__COUNTER__`宏（GCC/Clang扩展）。如何用它生成唯一的标识符？

5. 解释为什么以下代码会失败：

```c
#define CONCAT(a, b) a ## b
#define MAKE_VAR(prefix) CONCAT(prefix, __LINE__)

int MAKE_VAR(x);  // 结果是什么？为什么不是 x_当前行号？
```

（提示：宏展开的顺序规则）

## 7.12 延伸阅读

- C Standard, Section 6.10 (Preprocessor)
- Expert C Programming, Chapter 3 (Unscrambling Declarations in C)
- The Art of Readable Code, Chapter 8 (Testing)
- https://github.com/pfultz2/Cloak/wiki/Is-the-C-preprocessor-Turing-complete%3F

---

> "我觉得自己会永远生猛下去，什么也锤不了我。"这是王小波笔下的王二。C语言程序员也需要这种精神——面对宏这个复杂且有时危险的语言特性，我们不能退缩。宏是C语言元编程的入口，是编译期的函数式编程。掌握它，我们就能在编译时生成代码、检查约束、优化性能。这是C语言给予勇敢者的奖赏。
