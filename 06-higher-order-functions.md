# 第六章：函数指针与高阶函数 —— 一等公民的幻觉

## 6.1 编程语言的等级制度

王小波在《沉默的大多数》里说："所谓弱势群体，就是有些话没有说出来的人。"在编程语言的世界里，函数在C语言中就是这样的"弱势群体"——它们不是一等公民。

什么是一等公民（first-class citizen）？在编程语言理论中，一个实体如果是"一等的"，意味着它可以：
1. 被赋值给变量
2. 作为参数传递给函数
3. 作为函数的返回值
4. 在运行时创建

在JavaScript或Python中，函数是一等的：

```javascript
// JavaScript
const add = (a, b) => a + b;
const apply = (f, x, y) => f(x, y);
const result = apply(add, 2, 3);  // 5
```

在C语言中，函数**不是**一等的——我们不能在运行时创建新函数，也不能返回一个函数。但是，函数**指针**给了我们一个近似：

```c
int add(int a, int b) { return a + b; }

int apply(int (*f)(int, int), int x, int y) {
    return f(x, y);
}

int result = apply(add, 2, 3);  // 5
```

注意这里的区别：`add`不是一个值，而是一个函数。`apply`的第一个参数是`add`的地址（自动转换）。

## 6.2 函数指针的语法迷宫

C语言的函数指针语法 notoriously 难读。让我们分解一下：

```c
int (*fp)(int, int);  // fp 是一个指向函数的指针
                      // 该函数接受两个int，返回int

int *fp(int, int);    // fp 是一个函数
                      // 该函数接受两个int，返回int*
                      // （完全不同！）

const int (*fp)(int);     // 返回const int
int (*fp)(int, ...)       // 变参函数
int (*fp)(int) const;     // 错误！函数本身不能是const
```

`typedef`是救命稻草：

```c
typedef int (*BinaryIntOp)(int, int);

BinaryIntOp fp = add;  // 清晰多了
```

## 6.3 高阶函数的实践

让我们实现一些经典的函数式编程工具：

```c
#include <stdio.h>
#include <stdlib.h>

typedef int (*IntPred)(int);
typedef int (*IntOp)(int);
typedef int (*IntReduce)(int, int);

// filter：保留满足条件的元素
int filter(int *in, int n, int *out, IntPred pred) {
    int count = 0;
    for (int i = 0; i < n; i++) {
        if (pred(in[i])) {
            out[count++] = in[i];
        }
    }
    return count;  // 返回输出数组的大小
}

// map：对每个元素应用函数
void map(int *in, int n, int *out, IntOp op) {
    for (int i = 0; i < n; i++) {
        out[i] = op(in[i]);
    }
}

// reduce：折叠数组为一个值
int reduce(int *in, int n, int init, IntReduce op) {
    int result = init;
    for (int i = 0; i < n; i++) {
        result = op(result, in[i]);
    }
    return result;
}

// 使用示例
int is_even(int x) { return x % 2 == 0; }
int square(int x) { return x * x; }
int add(int a, int b) { return a + b; }
int mul(int a, int b) { return a * b; }

int main(void) {
    int nums[] = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // 筛选偶数
    int evens[10];
    int n_evens = filter(nums, 10, evens, is_even);

    // 平方
    int squares[10];
    map(nums, 10, squares, square);

    // 求和与求积
    int sum = reduce(nums, 10, 0, add);
    int product = reduce(nums, 10, 1, mul);

    printf("Sum = %d, Product = %d\n", sum, product);

    return 0;
}
```

这是MapReduce的雏形——Google著名的分布式计算框架的思想根源，其实可以追溯到函数式编程的基本原语。

## 6.4 闭包的缺失与模拟

函数式语言中的函数可以"捕获"外部变量，这称为**闭包**（closure）：

```javascript
// JavaScript
function makeAdder(n) {
    return function(x) { return x + n; };  // 捕获了 n
}

const add5 = makeAdder(5);
add5(10);  // 15
```

C语言的函数不能捕获变量——它们只有全局状态和参数。但我们可以用**显式环境**来模拟闭包：

```c
typedef struct {
    int (*func)(void *env, int);  // 接受环境参数
    void *env;                     // 捕获的变量
} Closure;

// add_n 的函数部分
int add_n_func(void *env, int x) {
    int n = *(int*)env;
    return x + n;
}

// 创建闭包
Closure make_adder(int n) {
    int *env = malloc(sizeof(int));
    *env = n;
    return (Closure){add_n_func, env};
}

// 调用闭包
int call_closure(Closure *c, int x) {
    return c->func(c->env, x);
}

// 使用
Closure add5 = make_adder(5);
int result = call_closure(&add5, 10);  // 15
free(add5.env);  // 别忘了释放！
```

这很笨拙，但展示了闭包的本质：**函数 + 环境**。在真正的函数式语言中，环境是自动管理的；在C中，我们必须手动管理内存。

## 6.5 回调函数与事件驱动

函数指针最常见的用途之一是**回调**（callback）：

```c
// 通用的排序比较函数类型
typedef int (*Compare)(const void *, const void *);

// qsort 标准库函数
void qsort(void *base, size_t nmemb, size_t size, Compare cmp);

// 使用示例
int cmp_int(const void *a, const void *b) {
    int ia = *(const int*)a;
    int ib = *(const int*)b;
    return (ia > ib) - (ia < ib);  // 返回 -1, 0, 或 1
}

int numbers[] = {3, 1, 4, 1, 5, 9};
qsort(numbers, 6, sizeof(int), cmp_int);
```

回调是**控制反转**（Inversion of Control）的基础。不是我们在调用代码，而是我们把代码交给库，由库在适当的时机调用。

GTK、libevent、Node.js的libuv——这些框架都建立在回调之上。

## 6.6 虚函数表：面向对象的C实现

C++的虚函数是通过**虚函数表**（vtable）实现的，而vtable本质上就是函数指针数组。我们可以在C中手动实现：

```c
#include <stdio.h>

// 前置声明
typedef struct Shape Shape;

// "虚函数表"
typedef struct {
    double (*area)(const Shape *);
    void (*draw)(const Shape *);
    void (*destroy)(Shape *);
} ShapeVTable;

// 基类
struct Shape {
    const ShapeVTable *vtable;
};

// "虚函数调用"宏
#define SHAPE_AREA(s) ((s)->vtable->area(s))
#define SHAPE_DRAW(s) ((s)->vtable->draw(s))
#define SHAPE_DESTROY(s) ((s)->vtable->destroy(s))

// ===== Circle 实现 =====
typedef struct {
    Shape base;
    double radius;
} Circle;

double circle_area(const Shape *s) {
    Circle *c = (Circle*)s;
    return 3.14159 * c->radius * c->radius;
}

void circle_draw(const Shape *s) {
    Circle *c = (Circle*)s;
    printf("Drawing circle with radius %.2f\n", c->radius);
}

void circle_destroy(Shape *s) {
    free(s);
}

static const ShapeVTable circle_vtable = {
    circle_area,
    circle_draw,
    circle_destroy
};

Shape *circle_create(double r) {
    Circle *c = malloc(sizeof(Circle));
    c->base.vtable = &circle_vtable;
    c->radius = r;
    return (Shape*)c;
}

// ===== 使用 =====
int main(void) {
    Shape *s = circle_create(5.0);

    printf("Area: %.2f\n", SHAPE_AREA(s));
    SHAPE_DRAW(s);
    SHAPE_DESTROY(s);

    return 0;
}
```

这就是面向对象的本质：**数据（对象）携带行为（vtable）**。继承就是通过在子结构体中包含父结构体作为第一个字段来实现，这样指针可以安全地转换。

## 6.7 函数的组合

函数式编程中，我们可以组合函数：

```haskell
compose f g x = f (g x)
```

在C中：

```c
typedef int (*UnaryIntOp)(int);

UnaryIntOp compose(UnaryIntOp f, UnaryIntOp g) {
    // 问题：返回什么？
    // 我们不能在运行时创建一个新函数！
}
```

但我们可以创建一个表示组合的数据结构：

```c
typedef struct {
    UnaryIntOp outer;
    UnaryIntOp inner;
} ComposedOp;

int apply_composed(void *env, int x) {
    ComposedOp *op = env;
    return op->outer(op->inner(x));
}

Closure compose(UnaryIntOp f, UnaryIntOp g) {
    ComposedOp *env = malloc(sizeof(ComposedOp));
    env->outer = f;
    env->inner = g;
    return (Closure){apply_composed, env};
}
```

或者，使用宏进行"编译期组合"：

```c
#define COMPOSE(f, g, x) f(g(x))

int result = COMPOSE(square, succ, 5);  // (5+1)^2 = 36
```

## 6.8 惰性求值与Thunk

函数指针可以用来实现**惰性求值**（lazy evaluation）：

```c
typedef struct {
    int (*compute)(void *env);
    void *env;
    int cached;
    int has_cache;
} Thunk;

int force(Thunk *t) {
    if (!t->has_cache) {
        t->cached = t->compute(t->env);
        t->has_cache = 1;
    }
    return t->cached;
}

// 示例：惰性计算斐波那契数
int fib_compute(void *env) {
    int n = *(int*)env;
    if (n <= 1) return n;

    // 递归创建 thunk
    int n1 = n - 1, n2 = n - 2;
    Thunk t1 = {fib_compute, &n1, 0, 0};
    Thunk t2 = {fib_compute, &n2, 0, 0};

    return force(&t1) + force(&t2);
}
```

注意：这个实现有问题（栈上分配的env在函数返回后失效），但它展示了基本思想。完整的实现需要堆分配和引用计数或GC。

## 6.9 尾递归优化

一些C编译器（特别是GCC和Clang）会在优化模式下进行尾调用优化（TCO）。这在配合函数指针时特别有用：

```c
typedef int (*TailCall)(int acc, int n);

int factorial_trampoline(int acc, int n);

TailCall factorial_step(int acc, int n) {
    if (n <= 1) {
        // 基本情况：返回一个特殊标记或直接返回值
        return NULL;
    }
    // 返回下一个" continuation"
    return factorial_trampoline;
}

int factorial_trampoline(int acc, int n) {
    TailCall next = factorial_step(acc, n);
    if (next == NULL) {
        return acc;
    }
    return next(acc * n, n - 1);  // 尾调用
}
```

实际上，更简单的写法是：

```c
int factorial_tail(int acc, int n) {
    if (n <= 1) return acc;
    return factorial_tail(acc * n, n - 1);  // GCC -O2 会优化为循环
}
```

## 6.10 练习

1. 实现`any`和`all`函数，分别检查数组中是否有任意/所有元素满足谓词。

2. 使用函数指针实现一个简单的信号/槽机制（观察者模式）。

3. 解释为什么下面的代码是危险的：

```c
int *bad_closure(void) {
    int local = 42;
    int get_local(void *env, int x) {
        return local + x;  // 捕获局部变量？
    }
    return &get_local;
}
```

（提示：C语言不支持嵌套函数，上面代码使用了GCC扩展。即使它能编译，有什么问题？）

4. 实现一个链表的`fold`（reduce）函数，展示如何用函数指针处理递归数据结构。

5. 研究C++的`std::function`和`std::bind`。如何在C中用函数指针和`void*`环境模拟类似的功能？

## 6.11 延伸阅读

- K&R, Section 5.11 (Pointers to Functions)
- Expert C Programming, Chapter 4
- Object-Oriented Programming With ANSI-C

---

> "人在年轻时，最头疼的一件事就是决定自己这一生要做什么。函数也一样——当它们被定义时，它们的行为就固定了。但函数指针给了我们灵活性，让我们可以延迟这个决定，或者根据不同情况做出不同选择。这就像王小波说的：'一个人活在这世界上，第一要好好做人，第二不要惯坏了别人的坏毛病'。函数指针让我们'好好做人'——写出灵活的代码；而类型系统让我们'不要惯坏坏毛病'——在编译时捕获错误。"
