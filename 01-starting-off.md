# 第一章：开始 —— 类型是世界的分类法

## 1.1 图书馆里的分类学

想象一下，你走进一座图书馆。王小波说过，他在插队的时候见过一座没有分类的图书馆——所有的书都堆在地上，要找到一本《霍乱时期的爱情》，可能需要翻遍整座楼。这就是没有类型的世界。

类型系统就是图书馆的分类法。它告诉我们：这本书是小说，那本是历史，那边的是数学。更重要的是，它告诉我们什么可以做，什么不可以做——你不会用一本诗集去垫桌脚（好吧，也许你会，但图书管理员会不高兴）。

C语言的类型系统像是一个严谨的德国图书馆管理员。它说话直接，不太会委婉。当你试图把一个`double`塞进`int`的时候，它不会温柔地提醒你，而是直接截断——就像把一本太宽的书硬塞进太窄的书架，多余的部分就被切掉了。

```c
#include <stdio.h>

int main(void) {
    double pi = 3.14159;
    int approx_pi = pi;  // 小数部分？什么小数部分？

    printf("pi 约等于 %d，如果你生活在整数的世界里\n", approx_pi);
    return 0;
}
```

运行结果：
```
pi 约等于 3，如果你生活在整数的世界里
```

这就是C语言的态度：你要转换类型？可以，但这是你的责任。

## 1.2 原子类型：世界的积木

在C语言中，有一些类型是原子的——就像元素周期表里的元素，它们不是由其他类型组合而成的。

```c
char c = 'A';        // 字符，本质上是个小整数
short s = 42;        // 短整数，可能比int小
int i = 1000000;     // 整数，计算的主力
long l = 1000000000L; // 长整数，当int不够用时

float f = 3.14f;     // 单精度浮点数
double d = 2.71828;  // 双精度浮点数，更精确

_Bool b = 1;         // 布尔值，C99引入
void *p = NULL;      // 空指针，指向虚无
```

但这里有个有趣的事情：在C语言里，类型的大小不是固定的。`int`可能是16位、32位或64位，取决于你的编译器和平台。这就像在不同国家，"一杯咖啡"的量可能大不相同。

王小波说："参差多态乃是幸福的本源。"但在C语言里，参差多态有时候是bug的本源。所以C99给了我们`<stdint.h>`：

```c
#include <stdint.h>

int32_t exactly_32_bits = 42;    // 一定是32位
uint64_t big_positive = 1000000; // 一定是64位，无符号
int8_t tiny = -128;              // 一定是8位
```

这就像是在国际会议上使用公制单位——没有歧义，皆大欢喜。

## 1.3 类型即证明

现在让我们深入一点，进入类型理论的领域。

在数学上，类型可以看作是一个集合。`int`是所有32位（或 whatever）整数的集合，`char`是所有8位值的集合。一个类型系统实际上是在说：这个变量只能取这个集合里的值。

但类型系统比集合更强大，因为它还告诉我们**能对这些值做什么操作**。你可以对两个`int`做加法，但不能对两个`void*`直接做加法（好吧，在GNU C里你可以，但这是扩展）。

这是Haskell Curry和William Howard发现的一个深刻联系：**类型即证明，程序即证明**。每一个类型正确的程序，都对应着一个逻辑证明。这被称为Curry-Howard同构。

```c
// 这个函数的类型是：int -> int
// 用逻辑的语言说："假设有一个整数，那么可以推出一个整数"
int add_one(int x) {
    return x + 1;
}

// 这个函数的类型是：(int, int) -> int
// 逻辑上："假设有两个整数，那么可以推出一个整数"
int add(int x, int y) {
    return x + y;
}
```

C语言的类型系统相比Haskell或OCaml来说是简陋的。它不会帮你证明指针不会为空，不会证明数组不会越界。但它给了我们工具，让我们可以自己建立这些保证——这就是为什么我们要学习结构体、联合体和指针。

## 1.4 第一个程序：Hello, Type

让我们写一个在运行时查询类型大小的程序，这是对C语言"反射"能力的一次窥探：

```c
#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>

int main(void) {
    printf("在这座图书馆里，每本书的尺寸如下：\n\n");

    printf("char:       %zu 字节\n", sizeof(char));
    printf("short:      %zu 字节\n", sizeof(short));
    printf("int:        %zu 字节\n", sizeof(int));
    printf("long:       %zu 字节\n", sizeof(long));
    printf("long long:  %zu 字节\n", sizeof(long long));
    printf("float:      %zu 字节\n", sizeof(float));
    printf("double:     %zu 字节\n", sizeof(double));
    printf("pointer:    %zu 字节\n", sizeof(void*));
    printf("bool:       %zu 字节\n", sizeof(bool));

    printf("\n精确大小的类型：\n");
    printf("int8_t:     %zu 字节\n", sizeof(int8_t));
    printf("int32_t:    %zu 字节\n", sizeof(int32_t));
    printf("int64_t:    %zu 字节\n", sizeof(int64_t));

    return 0;
}
```

运行结果（在64位Linux上）：
```
在这座图书馆里，每本书的尺寸如下：

char:       1 字节
short:      2 字节
int:        4 字节
long:       8 字节
long long:  8 字节
float:      4 字节
double:     8 字节
pointer:    8 字节
bool:       1 字节

精确大小的类型：
int8_t:     1 字节
int32_t:    4 字节
int64_t:    8 字节
```

注意`sizeof`返回的是`size_t`类型——这是一个无符号整数类型，专门用来表示大小。C语言的类型系统就是这样层层递进的，每一种类型都有其特定的用途。

## 1.5 练习

1. 在你的平台上运行上面的程序，记录下每种类型的大小。它们和本书中的例子相同吗？

2. 尝试以下代码，观察类型转换的行为：

```c
#include <stdio.h>

int main(void) {
    int a = 300;
    char b = a;  // 会发生什么？

    printf("a = %d, b = %d\n", a, (int)b);

    unsigned int c = -1;
    printf("c 作为无符号数是：%u\n", c);
    printf("c 作为有符号数是：%d\n", (int)c);

    return 0;
}
```

3. 思考：为什么`sizeof(char)`总是1？这是C语言标准的规定，还是某种必然？

## 1.6 延伸阅读

- The C Programming Language (K&R), Chapter 2
- C99 Standard, Section 6.2.5 (Types)
- Pierce, Benjamin C. "Types and Programming Languages", Chapter 1

---

> "在我看来，大千世界芸芸众生，无不在做白日梦。生活在世界上的每个人都在说'我想成为...'，但很少有人真的成为。C语言的好处是，当你写下`int x = 42;`，你就真的拥有一个整数42。这不是梦，这是现实。"
