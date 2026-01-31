# 第四章：组合的艺术 —— Product Types

## 4.1 笛卡尔积的朴素之美

王小波讲过一个关于集合论的故事：据说罗素发现集合论悖论的时候，正在思考"所有不包含自身的集合的集合"。这个故事告诉我们，即使是简单的组合，也可能产生复杂的后果。

在类型理论中，**积类型**（Product Type）对应数学中的笛卡尔积。如果有类型A和类型B，它们的积是：

$$A \times B = \{(a, b) \mid a \in A, b \in B\}$$

在C语言里，积类型就是`struct`：

```c
struct Point {
    double x;
    double y;
};  // struct Point ≅ double × double

struct Person {
    char name[100];
    int age;
    double height;
};  // struct Person ≅ String × int × double
```

一个`struct Point`值是一个对`(x, y)`，同时包含x坐标和y坐标。这就是"积"的含义——**同时具有**。

## 4.2 结构体的内存布局

结构体在内存中是连续存放的（可能有填充）：

```c
#include <stdio.h>

struct Point {
    double x;  // 8 bytes, offset 0
    double y;  // 8 bytes, offset 8
};

struct Padded {
    char c;    // 1 byte, offset 0
    // 7 bytes padding
    double d;  // 8 bytes, offset 8
};

int main(void) {
    printf("sizeof(struct Point) = %zu\n", sizeof(struct Point));
    printf("sizeof(struct Padded) = %zu\n", sizeof(struct Padded));

    struct Padded p;
    printf("\n偏移量:\n");
    printf("offsetof(c) = %zu\n", (size_t)((char*)&p.c - (char*)&p));
    printf("offsetof(d) = %zu\n", (size_t)((char*)&p.d - (char*)&p));

    return 0;
}
```

输出：
```
sizeof(struct Point) = 16
sizeof(struct Padded) = 16

偏移量:
offsetof(c) = 0
offsetof(d) = 8
```

注意`struct Padded`有7字节的填充。这是因为`double`需要8字节对齐。我们可以重新排列字段来减少填充：

```c
struct Compact {
    double d;  // offset 0
    char c;    // offset 8
    // 7 bytes padding at end
};  // sizeof = 16，还是16，但至少不会更差
```

或者使用`#pragma pack`（编译器扩展）来禁用填充，但这可能影响性能。

## 4.3 访问器与不变式

在函数式编程中，我们使用记录（record）和访问器（accessor）函数：

```ocaml
type point = { x : float; y : float }
let get_x p = p.x
```

在C语言中，我们可以模拟这种模式：

```c
typedef struct {
    double x;
    double y;
} Point;

// 访问器（内联以提高效率）
static inline double point_x(const Point *p) { return p->x; }
static inline double point_y(const Point *p) { return p->y; }

// 构造函数
Point point_make(double x, double y) {
    return (Point){x, y};
}

// 派生值（计算属性）
static inline double point_norm_sq(const Point *p) {
    return p->x * p->x + p->y * p->y;
}
```

使用访问器而不是直接访问字段的好处是：**我们可以改变内部表示而不影响客户端代码**。

## 4.4 嵌套结构：积类型的积

结构体可以嵌套，这对应于积类型的笛卡尔积：

```c
struct Date {
    int year;
    int month;
    int day;
};

struct Time {
    int hour;
    int minute;
    int second;
};

struct DateTime {
    struct Date date;
    struct Time time;
};

// struct DateTime ≅ struct Date × struct Time
//                   ≅ (int × int × int) × (int × int × int)
//                   ≅ int^6
```

注意`struct DateTime`里嵌套的`struct Date`和`struct Time`是完整的值，不是指针。这意味着`sizeof(struct DateTime)`是六个`int`的大小（可能有填充）。

## 4.5 不透明类型与抽象

C语言允许我们隐藏结构体的定义，只暴露接口。这就是**不透明类型**（opaque type）：

```c
// 在头文件 point.h 中
typedef struct Point Point;

Point *point_create(double x, double y);
void point_destroy(Point *p);
double point_get_x(const Point *p);
double point_get_y(const Point *p);
void point_translate(Point *p, double dx, double dy);

// 在实现文件 point.c 中
struct Point {
    double x;
    double y;
};

Point *point_create(double x, double y) {
    Point *p = malloc(sizeof(Point));
    if (p) {
        p->x = x;
        p->y = y;
    }
    return p;
}

// ... 其他实现
```

这实现了**信息隐藏**——客户端代码无法直接访问`Point`的内部字段，只能通过提供的函数操作它。

这在语义上类似于ML语言中的抽象类型（abstract type）：

```ocaml
module Point : sig
  type t  (* 抽象类型 *)
  val create : float -> float -> t
  val get_x : t -> float
  (* ... *)
end = struct
  type t = { x : float; y : float }
  (* ... *)
end
```

## 4.6 灵活数组成员：变长积类型

C99引入了一个有趣的特性：**灵活数组成员**（flexible array member）：

```c
struct Buffer {
    size_t len;
    char data[];  // 柔性数组，不占空间，必须在最后
};

// 创建变长buffer
struct Buffer *make_buffer(size_t len) {
    struct Buffer *b = malloc(sizeof(struct Buffer) + len);
    if (b) {
        b->len = len;
        memset(b->data, 0, len);
    }
    return b;
}
```

这实现了**依赖积类型**（dependent product type）的一个近似：缓冲区的大小是值（`len`）依赖的。

在类型理论中，依赖积写作$(x: A) \times B(x)$，表示第二个分量的类型依赖于第一个分量的值。C语言当然没有真正的依赖类型，但灵活数组成员给了我们一个实用的近似。

## 4.7 代数视角：和类型与积类型的交互

现在我们可以同时运用和类型与积类型了：

```c
// 二叉树：每个节点要么是一个叶子（值），要么是一个分支（左子树 + 右子树）
typedef struct Tree Tree;

struct Tree {
    enum { LEAF, BRANCH } tag;
    union {
        int value;           // 当 tag == LEAF
        struct {             // 当 tag == BRANCH
            Tree *left;
            Tree *right;
        } children;
    } data;
};
```

这棵树的类型可以写作：

$$Tree = int + (Tree \times Tree)$$

这是递归类型方程。在数学上，它的解是无限树类型。

## 4.8 内存对齐的艺术

对齐（alignment）是C语言中一个重要的概念。每个类型都有一个对齐要求，这个要求决定了它的地址必须是某个数的倍数。

```c
#include <stdalign.h>

int main(void) {
    printf("alignof(char)   = %zu\n", alignof(char));
    printf("alignof(short)  = %zu\n", alignof(short));
    printf("alignof(int)    = %zu\n", alignof(int));
    printf("alignof(double) = %zu\n", alignof(double));

    return 0;
}
```

为什么对齐很重要？因为许多处理器在非对齐访问上性能很差，甚至直接崩溃。C语言通过填充（padding）来保证对齐。

但有时候我们想要控制布局——比如硬件寄存器映射或网络协议：

```c
#include <stdint.h>

// 假设一个32位控制寄存器的布局
struct ControlReg {
    uint32_t enable : 1;    // 位 0
    uint32_t mode : 3;      // 位 1-3
    uint32_t reserved : 4;  // 位 4-7
    uint32_t divisor : 8;   // 位 8-15
    uint32_t : 16;          // 位 16-31，未使用
};
```

**位域**（bit-field）让我们可以精确控制内存布局。注意`struct ControlReg`仍然占用4字节，但我们能单独访问其中的字段。

## 4.9 练习

1. 实现一个复数类型`Complex`，提供加、减、乘、除运算。思考：应该使用`struct`还是两个独立的`double`参数？

2. 设计一个矩阵类型。支持以下两种表示：
   - 行主序的二维数组
   - 带有stride的灵活布局

   如何用不透明类型隐藏具体实现？

3. 解释以下现象：

```c
struct Empty {};
struct WithEmpty {
    struct Empty e;
    int x;
};

// sizeof(struct Empty) 是多少？
// sizeof(struct WithEmpty) 是多少？
```

4. 用Tagged Union和struct实现一个简单的表达式求值器，支持整数、加法和乘法。

## 4.10 延伸阅读

- K&R, Chapter 6 (Structures)
- C99 Standard, Section 6.7.2.1
- The Lost Art of Structure Packing: https://www.catb.org/esr/structure-packing/

---

> "一个人倘若需要从思想中得到快乐，那么他的第一个欲望就是学习。结构体是C语言给学习者的礼物——它简单到可以在五分钟内理解，又深刻到足以构建整个操作系统。从`int`到`struct`，再到`union`的组合，我们看到了类型代数的基本构件。这就像是从积木块到宏伟建筑的过程，每一块都平凡无奇，但组合起来就有了无限可能。"
