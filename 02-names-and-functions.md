# 第二章：名字和函数 —— 从表达式到语句

## 2.1 命名的暴政

王小波讲过一个故事：他在插队的时候，村里有个人叫"二嘎子"。不是因为他排行第二，也不是因为他嘎（淘气），只是因为他爹随口这么一叫，他就背了一辈子。

命名是一种权力。当你给一个值命名，你就创造了一个可以被引用、被操作、被传递的实体。在C语言里，我们用变量来给值命名。

```c
int x = 42;  // 从此，42 在这块作用域里叫做 x
```

但名字是有范围的。二嘎子出了村，可能没人认识他；同样，`x`出了它所在的代码块，就烟消云散了。

```c
#include <stdio.h>

int main(void) {
    int x = 10;

    {
        int x = 20;  // 这是一个新的 x，和外面的 x 无关
        printf("内部的 x = %d\n", x);
    }  // 内部的 x 死在这里

    printf("外部的 x = %d\n", x);  // 还是 10

    return 0;
}
```

这叫做**作用域**（scope）。C语言使用词法作用域，这意味着一个名字的有效性由它在代码中的位置决定，而不是由运行时的调用链决定。

## 2.2 函数：命名的计算

如果变量是给值命名，那么函数就是给计算命名。

```c
// 计算平方
int square(int x) {
    return x * x;
}

// 计算绝对值
int abs(int x) {
    return x < 0 ? -x : x;
}

// 计算最大公约数（欧几里得算法）
int gcd(int a, int b) {
    while (b != 0) {
        int temp = b;
        b = a % b;
        a = temp;
    }
    return a;
}
```

在函数式语言里，函数是表达式——它们计算一个值，仅此而已。但在C语言里，函数包含**语句**。语句不像表达式那样返回值，它们执行动作。

这是C语言和函数式语言的根本区别之一。在Haskell里，你写：

```haskell
factorial n = if n == 0 then 1 else n * factorial (n - 1)
```

这是一个表达式。在C语言里，你写：

```c
int factorial(int n) {
    if (n == 0) {
        return 1;  // 这是一个语句
    } else {
        return n * factorial(n - 1);  // 这也是一个语句
    }
}
```

`return`是一个语句，它做了两件事：终止函数的执行，并带出一个值。这就像一个人说完最后一句话，然后摔门而去。

## 2.3 表达式 vs 语句

C语言既有表达式，也有语句。理解它们的区别至关重要。

**表达式**计算一个值：
- `42` 是表达式，值是42
- `x + y` 是表达式，值是两数之和
- `square(5)` 是表达式，值是25
- `x = 42` 也是表达式！它的值是42，副作用是把x设为42

**语句**执行动作：
- `int x = 42;` 是声明语句
- `x = 42;` 是表达式语句
- `if (cond) { ... }` 是选择语句
- `while (cond) { ... }` 是循环语句
- `return expr;` 是返回语句

C语言的一个特点是表达式可以变成语句，只需加上分号：

```c
42;           // 合法但无意义的语句
x + y;        // 计算了值但丢弃了它
printf("hi"); // 有用的语句，副作用是输出
```

反过来，语句不能变成表达式（除了GNU C的扩展）。这意味着你不能这样写：

```c
int z = if (x > y) { x } else { y };  // 错误！if 不是表达式
```

而你必须这样写：

```c
int z;
if (x > y) {
    z = x;
} else {
    z = y;
}
```

或者用条件运算符（C语言里唯一的"三元表达式"）：

```c
int z = (x > y) ? x : y;  // 条件表达式
```

## 2.4 函数的数学本质

从数学的角度看，一个函数`f: A -> B`是从集合A到集合B的映射。C语言的函数类型与之类似，但不完全相同。

```c
// 类型签名：int -> int
int succ(int x);

// 类型签名：(int, int) -> int
int add(int x, int y);

// 类型签名：() -> int
int get_random(void);  // void 表示"没有参数"

// 类型签名：(int*) -> void
void print_int(int *p);  // void 返回类型表示"不返回值"
```

注意C语言没有直接表达"函数接受一个对，返回一个整数"的语法——`add`在数学上是`int × int -> int`，但在C语法里是`(int, int) -> int`。这个区别看起来微小，但在谈论高阶函数时变得重要。

## 2.5 副作用：函数式与命令式的分水岭

纯函数（数学意义上的）没有副作用：给定相同的输入，总是产生相同的输出，且不影响外部世界。

C语言的函数通常有副作用。它们可能：
- 修改全局变量
- 修改参数指向的内存
- 执行I/O操作
- 调用其他有副作用的函数

```c
static int counter = 0;

// 这个函数有副作用：修改了 counter
int next_id(void) {
    return ++counter;
}

// 这个函数也有副作用：修改了传入的指针指向的值
void increment(int *p) {
    *p = *p + 1;
}

// 这个函数有副作用：输出到屏幕
void greet(const char *name) {
    printf("Hello, %s!\n", name);
}
```

副作用不是坏事——没有副作用，程序就无法与外界交互。但副作用使得推理程序行为变得更困难。这也是为什么函数式编程强调"隔离副作用"的原因。

## 2.6 递归：函数调用自己

C语言支持递归，这给了它图灵完备的计算能力。

```c
// 递归计算阶乘
int factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}

// 递归计算斐波那契数（低效但优雅）
int fib(int n) {
    if (n <= 1) return n;
    return fib(n - 1) + fib(n - 2);
}

// 尾递归优化的例子（大多数C编译器会优化这个）
int factorial_tail(int n, int acc) {
    if (n <= 1) return acc;
    return factorial_tail(n - 1, n * acc);
}

// 包装函数，提供友好的接口
int factorial_fast(int n) {
    return factorial_tail(n, 1);
}
```

递归是函数式编程的核心，但在C语言里要谨慎使用。每次函数调用都会消耗栈空间，太深的递归会导致**栈溢出**（stack overflow）。

## 2.7 练习

1. 写一个函数，用迭代（而非递归）计算第n个斐波那契数。

2. 下面的代码输出什么？为什么？

```c
#include <stdio.h>

int foo(int x) {
    printf("foo(%d)\n", x);
    return x;
}

int main(void) {
    int a = foo(1) + foo(2) * foo(3);
    printf("结果：%d\n", a);
    return 0;
}
```

注意：函数调用的求值顺序在C语言中是**未指定**的（unspecified）！

3. 研究`?:`运算符的短路行为：

```c
int min = (a < b) ? a : b;
```

和`if-else`相比，这个表达式有什么优势和劣势？

4. 写一个函数`void swap(int *a, int *b)`，交换两个整数变量的值。为什么我们需要指针？

## 2.8 延伸阅读

- K&R, Chapter 4 (Functions)
- C99 Standard, Section 6.9 (External definitions)
- Structure and Interpretation of Computer Programs, Chapter 1

---

> "在我周围有一种热乎乎的气氛，像桑拿浴室一样，仿佛每个人都在关心着别人。C语言的函数不是这样——如果你不显式地把参数传给它，它就不知道外面发生了什么。这种冷漠是一种美德，它让程序更容易理解。"
