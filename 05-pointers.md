# 第五章：指针 —— 计算的延续

## 5.1 地址与引用

王小波在《红拂夜奔》里写道："一个人只拥有此生此世是不够的，他还应该拥有诗意的世界。"指针就是C语言的诗意世界——它让我们能够引用（refer to）而不拥有，指向（point to）而不复制。

在最基础的层面上，指针就是一个内存地址。但这个简单的概念背后，藏着计算理论中最深刻的思想之一：**延续**（continuation）。

```c
int x = 42;
int *p = &x;  // p 指向 x，或者说，p 是 x 的延续
```

在这里，`&x`取得`x`的地址，`p`存储了这个地址。通过`*p`，我们可以访问（或者修改）`x`的值。

## 5.2 指针的本质：延迟的计算

从计算理论的角度看，指针是一种**延迟**（suspension）或**延续**。当我们有一个指向某处的指针时，我们拥有的是**将来访问该位置的能力**，而不是该位置的当前值。

这和函数式编程中的延迟求值（lazy evaluation）有相似之处：

```haskell
-- Haskell 中的延迟值
x :: Int
x = expensiveComputation  -- 直到需要时才计算
```

```c
// C 中的延迟访问（通过指针）
int *p = &some_variable;  // 现在不读取，保留读取的能力
// ... 稍后 ...
int value = *p;  // 现在才真正访问
```

当然，C语言的指针不是纯粹的延迟值——它可能有副作用（因为`some_variable`可能在别处被修改）。但这不妨碍我们用函数式的视角理解它。

## 5.3 指针与内存模型

让我们画一张C语言的内存模型图：

```
栈区 (Stack)          堆区 (Heap)
┌──────────┐         ┌──────────┐
│   x: 42  │         │          │
│   &x ───────┐      │          │
│   p: ───────┼──────┼────┐     │
└──────────┘  │      │    │     │
              │      │    ▼     │
              └──────┼──> [42]   │
                     │          │
                     └──────────┘
```

变量`x`存储在栈上，值为42。`p`也存储在栈上，值为`x`的地址。通过`*p`，我们"跳转"到`x`的位置。

这种间接性是强大的。它允许我们：
- 共享数据而不复制
- 在函数间传递引用
- 构建递归数据结构
- 实现多态（通过`void*`）

## 5.4 指针的代数：指针运算

指针可以进行有限的运算：

```c
int arr[5] = {10, 20, 30, 40, 50};
int *p = &arr[0];  // 指向第一个元素

p + 1;   // 指向第二个元素（不是地址+1！）
p + 2;   // 指向第三个元素
*p;      // 解引用，得到 10
*(p+1);  // 得到 20
```

指针运算的单位是**目标类型的大小**，不是字节。`p + 1`实际上给地址增加了`sizeof(int)`。

这对应于数组的索引：

```c
arr[i]    ≡    *(arr + i)    ≡    *(i + arr)    ≡    i[arr]
```

是的，`i[arr]`是合法的C代码！因为加法是可交换的，而`a[b]`等价于`*(a+b)`。

## 5.5 指针作为计算的延续

现在让我们深入"延续"这个概念。在计算理论中，一个延续表示"程序剩余要完成的工作"。

考虑一个表达式求值：

```c
int result = (a + b) * c;
```

求值过程是：
1. 计算`a + b`，得到一个临时值`t`
2. 计算`t * c`，得到最终结果

第二步就是第一步的**延续**。在CPS（Continuation Passing Style）中，我们显式传递延续：

```c
// 直接风格
int add(int a, int b) { return a + b; }
int mul(int a, int b) { return a * b; }
int result = mul(add(a, b), c);

// CPS 风格（概念上）
void add_cps(int a, int b, void (*k)(int)) {
    k(a + b);  // 将结果传递给延续
}

void mul_cps(int a, int b, void (*k)(int)) {
    k(a * b);
}

// 使用
add_cps(a, b, lambda(int t) {
    mul_cps(t, c, lambda(int result) {
        // 最终结果在这里
    });
});
```

C语言没有lambda，但我们可以用函数指针模拟：

```c
typedef void (*IntCallback)(int);

void add_then(int a, int b, IntCallback cont) {
    cont(a + b);
}

void with_result(int result) {
    printf("结果是: %d\n", result);
}

// 使用
add_then(2, 3, with_result);
```

指针在这里扮演了关键角色——它让我们可以传递"接下来做什么"的信息。

## 5.6 函数指针：一等函数的幻觉

C语言不支持一等函数（first-class functions），但函数指针给了我们一个近似：

```c
#include <stdio.h>

// 函数指针类型：接受两个int，返回int
typedef int (*BinaryOp)(int, int);

int add(int a, int b) { return a + b; }
int mul(int a, int b) { return a * b; }

// 高阶函数：接受一个函数作为参数
int fold(int *arr, int n, int init, BinaryOp op) {
    int result = init;
    for (int i = 0; i < n; i++) {
        result = op(result, arr[i]);
    }
    return result;
}

int main(void) {
    int nums[] = {1, 2, 3, 4, 5};

    int sum = fold(nums, 5, 0, add);
    int product = fold(nums, 5, 1, mul);

    printf("Sum = %d\n", sum);
    printf("Product = %d\n", product);

    return 0;
}
```

函数指针的类型语法很奇特：

```c
int (*fp)(int, int);     // fp 是指向函数的指针
int *fp(int, int);       // fp 是返回指针的函数（不同！）
int (*fp[10])(int);      // fp 是函数指针数组
int (*(*fp)[10])(int);   // fp 是指向函数指针数组的指针
```

可以使用`typedef`来简化，或者使用C23的`typeof`。

## 5.7 空指针与无效指针

`NULL`是一个特殊的指针值，表示"不指向任何地方"。解引用`NULL`是未定义行为（通常导致段错误）。

```c
int *p = NULL;

// 安全的做法
if (p != NULL) {
    *p = 42;  // 只有当 p 有效时才解引用
}

// C11 引入的空指针常量
int *q = nullptr;  // 类型安全的空指针
```

注意区分：
- **空指针**（null pointer）：值为0（或`NULL`或`nullptr`）的指针，明确不指向有效对象
- **野指针**（dangling pointer）：曾经有效但现在已经无效的指针（例如指向已释放的内存）
- **未初始化指针**：声明但没有赋值的指针，包含随机值

后两者比空指针更危险，因为它们可能指向看似有效的内存，导致难以调试的错误。

## 5.8 多级指针与间接性

指针可以指向指针：

```c
int x = 42;
int *p = &x;      // p 指向 int
int **pp = &p;    // pp 指向 int*
int ***ppp = &pp; // ppp 指向 int**

printf("x = %d\n", x);           // 42
printf("*p = %d\n", *p);         // 42
printf("**pp = %d\n", **pp);     // 42
printf("***ppp = %d\n", ***ppp); // 42
```

多级指针常用于：
- 动态二维数组
- 函数需要修改指针参数（例如`realloc`）
- 构建复杂的数据结构

例如，实现一个"可能失败并修改指针"的函数：

```c
int allocate_buffer(char **out_ptr, size_t size) {
    char *p = malloc(size);
    if (p == NULL) {
        return -1;  // 失败
    }
    *out_ptr = p;   // 通过二级指针修改调用者的指针
    return 0;       // 成功
}

// 使用
char *buf;
if (allocate_buffer(&buf, 1024) == 0) {
    // buf 现在指向分配的内存
    free(buf);
}
```

## 5.9 指针与数组：暧昧的关系

在C语言中，数组和指针有暧昧的关系。数组名在大多数表达式中会被转换为指向首元素的指针：

```c
int arr[5] = {1, 2, 3, 4, 5};
int *p = arr;  // arr 退化为 &arr[0]

// 以下等价
arr[2]    // 数组索引
*(arr+2)  // 指针运算
p[2]      // 指针也可以索引！
*(p+2)    // 同上
2[arr]    // 也合法，因为 a[b] ≡ *(a+b) ≡ *(b+a) ≡ b[a]
```

但数组和指针有本质区别：

```c
sizeof(arr);  // 20（5个int）
sizeof(p);    // 8（指针大小，64位系统）

&arr;         // int (*)[5]，指向整个数组的指针
&p;          // int **，指向指针的指针
```

理解这种区别对避免常见的数组-指针混淆错误至关重要。

## 5.10 练习

1. 解释以下声明的含义：

```c
int *a[10];     // ?
int (*b)[10];   // ?
int (*c[10])(int);  // ?
int (*d)(int, int (*[])(int));  // ?
```

2. 实现一个通用的`map`函数，接受一个数组、数组大小、元素大小、一个转换函数，将转换应用到每个元素。

3. 用函数指针实现一个简单的虚函数表（vtable）机制，模拟面向对象的多态。

4. 解释为什么以下代码可能失败：

```c
void get_buffer(char *buf) {
    buf = malloc(100);  // 这能工作吗？
}

// 对比
void get_buffer2(char **buf) {
    *buf = malloc(100);  // 这能工作
}
```

5. 研究C11的`_Generic`关键字。如何用它来实现类型安全的`max`宏，处理不同指针类型？

## 5.11 延伸阅读

- K&R, Chapter 5 (Pointers and Arrays)
- Expert C Programming, Chapter 4 (The Shocking Truth about C Arrays and Pointers)
- Pierce, "Types and Programming Languages", Chapter 5 (The Untyped Lambda-Calculus)

---

> "指针是C语言的灵魂，也是它的诅咒。它让我们可以直接操纵内存，构建高效的数据结构；也让我们可以搬起石头砸自己的脚，在运行时制造出各种噩梦。但如果我们把指针看作'延续'——一种延迟的计算，一种未来的承诺——它就变得不那么可怕了。就像王小波说的：'生活就是个缓慢受锤的过程'，指针也是个缓慢学习的过程。但一旦掌握，你就拥有了直接对话计算机硬件的能力。"
