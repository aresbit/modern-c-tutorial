# 第三章：情况分析 —— Sum Types 的艺术

## 3.1 分类学的困境

王小波在《黄金时代》里写道："那一天我二十一岁，在我一生的黄金时代，我有好多奢望。我想爱，想吃，还想在一瞬间变成天上半明半暗的云。"

注意这里的结构：想爱，**或者**想吃，**或者**想变成云。王小波在这一刻可以是这三种状态中的**一种**，但不能同时是三种。这就是Sum Type的本质——"或者"的关系。

在数学上，如果有集合A和集合B，它们的**和集**（sum，也叫不交并，disjoint union）是：

$$A + B = \{(a, 0) \mid a \in A\} \cup \{(b, 1) \mid b \in B\}$$

每个元素都带有一个"标签"（0或1），告诉我们它来自A还是B。

在OCaml或Haskell里，我们可以这样写：

```ocaml
type shape =
  | Circle of float           (* 半径 *)
  | Rectangle of float * float (* 宽 * 高 *)
  | Triangle of float * float  (* 底 * 高 *)
```

一个`shape`要么是`Circle`，要么是`Rectangle`，要么是`Triangle`——不能同时是两个。

## 3.2 C语言的Union：没有标签的Sum Type

C语言有`union`，它让不同的类型共享同一块内存：

```c
union Number {
    int i;
    float f;
    double d;
};

union Number n;
n.i = 42;       // 现在n是一个整数
n.f = 3.14f;    // 现在n是一个float，之前的整数内容被覆盖
```

但这里有个问题：`union`没有标签。当你查看一个`union`时，你不知道它当前存的是什么类型。这就像是一个没有封面标签的文件袋——里面可能装着合同，也可能装着购物清单。

如果你读错了类型，结果就是**未定义行为**（undefined behavior）——这可能是C语言里最可怕的两个词。

```c
union Number n;
n.i = 42;

// 糟糕！我们存储的是整数，但读取为float
float wrong = n.f;  // 未定义行为！结果可能是任何值
```

## 3.3 Tagged Union：给Sum Type加上标签

解决方法很简单：我们自己加标签。这就是**Tagged Union**（带标签的联合体）：

```c
#include <stdio.h>
#include <math.h>

typedef enum {
    SHAPE_CIRCLE,
    SHAPE_RECTANGLE,
    SHAPE_TRIANGLE
} ShapeTag;

typedef struct {
    ShapeTag tag;
    union {
        struct { double radius; } circle;
        struct { double width, height; } rectangle;
        struct { double base, height; } triangle;
    } data;
} Shape;

// 构造函数
Shape make_circle(double radius) {
    Shape s = {SHAPE_CIRCLE, {.circle = {radius}}};
    return s;
}

Shape make_rectangle(double w, double h) {
    Shape s = {SHAPE_RECTANGLE, {.rectangle = {w, h}}};
    return s;
}

Shape make_triangle(double b, double h) {
    Shape s = {SHAPE_TRIANGLE, {.triangle = {b, h}}};
    return s;
}

// 计算面积（模式匹配的手动实现）
double area(const Shape *s) {
    switch (s->tag) {
        case SHAPE_CIRCLE:
            return M_PI * s->data.circle.radius * s->data.circle.radius;
        case SHAPE_RECTANGLE:
            return s->data.rectangle.width * s->data.rectangle.height;
        case SHAPE_TRIANGLE:
            return 0.5 * s->data.triangle.base * s->data.triangle.height;
        default:
            return 0;  // 不应该到达这里
    }
}

int main(void) {
    Shape shapes[] = {
        make_circle(5.0),
        make_rectangle(4.0, 6.0),
        make_triangle(3.0, 8.0)
    };

    for (int i = 0; i < 3; i++) {
        printf("形状 %d 的面积 = %.2f\n", i, area(&shapes[i]));
    }

    return 0;
}
```

这个模式非常重要，它在C代码中无处不在。GTK、GLib、Linux内核——到处都能看到Tagged Union的身影。

## 3.4 Option类型：可空值的函数式表达

在函数式语言里，`Option`（或`Maybe`）类型表示"可能有值，可能没有"：

```ocaml
type 'a option = None | Some of 'a
```

在C语言里，我们通常用指针和`NULL`来实现类似的概念。但Tagged Union给我们一个更安全的替代：

```c
typedef enum {
    OPT_NONE,
    OPT_SOME
} OptionTag;

typedef struct {
    OptionTag tag;
    int value;  // 只有当 tag == OPT_SOME 时有效
} IntOption;

IntOption some(int x) {
    return (IntOption){OPT_SOME, x};
}

IntOption none(void) {
    return (IntOption){OPT_NONE, 0};
}

// 安全地获取值，带默认值
int get_or_default(IntOption opt, int default_val) {
    return (opt.tag == OPT_SOME) ? opt.value : default_val;
}

// 函数式编程中的 map 操作
IntOption option_map(IntOption opt, int (*f)(int)) {
    return (opt.tag == OPT_SOME) ? some(f(opt.value)) : none();
}
```

这比直接用`NULL`指针安全多了——你不会意外解引用空指针，因为编译器会强迫你检查`tag`。

## 3.5 Result类型：错误处理的显式化

Go语言和Rust都使用显式的错误返回值。我们也可以在C中实现类似`Result<T, E>`的类型：

```c
typedef enum {
    RESULT_OK,
    RESULT_ERR
} ResultTag;

#define DEFINE_RESULT(T, E, Name) \
    typedef struct { \
        ResultTag tag; \
        union { \
            T ok; \
            E err; \
        } data; \
    } Name

// 使用宏定义具体的Result类型
DEFINE_RESULT(int, const char*, IntResult);

IntResult ok_int(int x) {
    return (IntResult){RESULT_OK, {.ok = x}};
}

IntResult err_int(const char *msg) {
    return (IntResult){RESULT_ERR, {.err = msg}};
}

// 使用示例：安全的除法
IntResult safe_divide(int a, int b) {
    if (b == 0) {
        return err_int("Division by zero");
    }
    return ok_int(a / b);
}
```

这种模式强制调用者处理错误：

```c
IntResult result = safe_divide(10, 0);
if (result.tag == RESULT_OK) {
    printf("结果: %d\n", result.data.ok);
} else {
    printf("错误: %s\n", result.data.err);
}
```

这比C语言传统的错误处理（返回特殊值+设置全局`errno`）更安全、更清晰。

## 3.6 Tagged Union的内存布局

让我们看看Tagged Union在内存中是什么样子的：

```c
#include <stdio.h>

typedef struct {
    int tag;
    union {
        char c;
        int i;
        double d;
    } data;
} Tagged;

int main(void) {
    printf("sizeof(Tagged) = %zu\n", sizeof(Tagged));
    printf("sizeof(tag) = %zu\n", sizeof(int));
    printf("sizeof(data) = %zu\n", sizeof(((Tagged*)0)->data));

    Tagged t;
    printf("\n内存布局:\n");
    printf("&t      = %p\n", (void*)&t);
    printf("&t.tag  = %p\n", (void*)&t.tag);
    printf("&t.data = %p\n", (void*)&t.data);

    return 0;
}
```

运行结果（64位系统）：
```
sizeof(Tagged) = 16
sizeof(tag) = 4
sizeof(data) = 8

内存布局:
&t      = 0x7ffeeb2a3a58
&t.tag  = 0x7ffeeb2a3a58
&t.data = 0x7ffeeb2a3a60
```

注意内存对齐：`double`需要8字节对齐，所以虽然`tag`只占4字节，但`data`从偏移8开始，而不是4。

## 3.7 代数数据类型（ADT）的视角

从类型理论的角度看，类型可以构成代数系统：

- **积类型**（Product Type）：`struct { A a; B b; }` 对应 $A \times B$，大小是`sizeof(A) + sizeof(B)`（加对齐）
- **和类型**（Sum Type）：`union { A a; B b; }` 对应 $A + B$，大小是`max(sizeof(A), sizeof(B))`（加标签）

Tagged Union就是 $Tag \times (A + B + C)$ 的实现。

这个视角很重要，因为它告诉我们：**C语言的类型系统虽然简陋，但足以表达现代语言中的代数数据类型**。我们只需要手动实现那些其他语言自动提供的功能（如模式匹配、类型检查）。

## 3.8 练习

1. 实现一个`List`类型，表示整数链表。使用Tagged Union区分空列表和非空列表。

2. 创建一个`JSON`类型的Tagged Union，支持`null`、`boolean`、`number`、`string`和`array`。

3. 比较以下两种表示方式的内存使用：

```c
// 方式1：Tagged Union
typedef struct { int tag; union { int i; double d; } data; } Value1;

// 方式2：同时存储两种值
typedef struct { int i; double d; int is_int; } Value2;
```

哪种方式更省内存？哪种更快？为什么？

4. 研究C11的`_Generic`关键字。如何用它来实现类似OCaml的模式匹配语法？

## 3.9 延伸阅读

- K&R, Section 6.8 (Unions)
- C99 Standard, Section 6.7.2.1 (Structure and union specifiers)
- Pierce, "Types and Programming Languages", Chapter 11 (Simple Extensions)

---

> "生活就是个缓慢受锤的过程，人一天天老下去，奢望也一天天消失，最后变得像挨了锤的牛一样。但Tagged Union不是这样——它永远知道自己是什么，无论是在声明时、赋值时，还是被传递给函数时。这种自我认知是一种奢侈，在C语言的世界里，这种奢侈需要我们手动维护。"
