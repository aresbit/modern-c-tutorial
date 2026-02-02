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

### CPS与函数指针的深度结合

**Continuation Passing Style (CPS)** 是一种编程风格，其特点是函数接受一个额外的延续参数（continuation），并通过调用该延续而不是返回到调用者来完成计算。

在C语言中，函数指针使CPS风格成为可能。让我们通过一个完整的链表反转示例来理解：

```c
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>

/* 整数链表类型 - 使用const确保不可变性 */
typedef struct __int_list const *const int_list_t;
static int_list_t const Nil = NULL; /* 空链表 */

/* 单链表元素 */
typedef struct __int_list {
    int_list_t next;
    int32_t const val;
} node_t;

/* CPS结果指针：指向存储计算结果的位置 */
typedef void *const CPS_Result;

/* 延续类型定义 */
typedef void (*MakeListCallback)(int_list_t list, CPS_Result res);
typedef void (*ReversedListCallback)(int_list_t rlist, CPS_Result res);
typedef void (*VoidMappable)(int32_t const val);

/* 函数声明 */
void make_list(uint32_t const arr_size,
               int32_t const array[],
               int_list_t lst,
               MakeListCallback const cb,
               CPS_Result res);

void reverse(int_list_t list,
             int_list_t rlist,
             ReversedListCallback const cb,
             CPS_Result res);

void list2array(int_list_t list, CPS_Result res);
void void_map_array(VoidMappable const cb,
                    uint32_t const size,
                    int32_t const *const arr);

/* 反转链表并存入数组 */
void reverse_toarray(int_list_t list, CPS_Result res) {
    reverse(list, Nil, list2array, res);
}

static void print_val(int32_t const val) { printf("%d ", val); }

#define ARRAY_SIZE(arr) (sizeof(arr) / sizeof(arr[0]))

int main(int argc, char *argv[]) {
    int32_t arr[] = {99, 95, 90, 85, 80, 20, 75, 15, 10, 5};
    uint32_t const arr_size = ARRAY_SIZE(arr);

    int32_t res[arr_size];
    /* 调用make_list并将reverse_toarray作为"延续"传递 */
    make_list(arr_size, arr, Nil, reverse_toarray, res);

    /* 输出反转后的数组 */
    void_map_array(print_val, arr_size, res);
    printf("\n");
    return 0;
}

/* 从数组构建链表 */
void make_list(uint32_t const arr_size,
               int32_t const arr[],
               int_list_t lst,
               MakeListCallback const cb,
               CPS_Result res) {
    if (!arr_size) {
        cb(lst, res);  // 调用延续，传递结果
        return;
    }
    make_list(arr_size - 1, arr,
        &(node_t){.val = arr[arr_size - 1], .next = lst}, cb, res);
}

/* 将链表转换为数组 */
void list2array(int_list_t list, CPS_Result res) {
    if (Nil == list) return;
    int32_t *array = res;
    array[0] = list->val;
    list2array(list->next, array + 1);
}

/* 反转链表 */
void reverse(int_list_t list,
             int_list_t rlist, /* 已反转的部分 */
             ReversedListCallback const cb,
             CPS_Result res) {
    if (Nil == list) {
        cb(rlist, res);  // 到达末尾，调用延续
        return;
    }
    reverse(list->next, &(node_t){.val = list->val, .next = rlist},
            cb, res);
}

/* 遍历数组并对每个元素执行操作 */
void void_map_array(VoidMappable const cb,
                    uint32_t const size,
                    int32_t const *const arr) {
    if (!size) return;
    cb(arr[0]);
    void_map_array(cb, size - 1, arr + 1);
}
```

### 执行流程分析

上述代码的执行顺序展示了CPS的精髓：

```
main -->
    make_list -->
      reverse_toarray -->
        reverse -->
            list2array -->
                void_map_array
```

关键点：

1. **不可变性**：`typedef struct __int_list const *const int_list_t` 定义了一个const结构体指针，且该指针的内存地址和内含值无法通过局部变量改变，体现了FP（函数式编程）的精神。

2. **回调函数指针**：`MakeListCallback` 是一个函数指针类型，用于接收 `reverse_toarray` 这样的函数。通过 `cb(lst, res)` 将构建好的链表传递给下一个处理步骤。

3. **延迟计算**：指针在这里不仅是内存地址，更是"接下来做什么"的承诺。每个函数通过函数指针指定计算完成后的下一步操作。

4. **递归与延续**：在 `reverse` 函数中，递归的终止条件是当 `list` 为 `Nil` 时，此时调用延续 `cb(rlist, res)` 将结果传递下去。

### 指针与CPS的哲学联系

从更深层次看，CPS中的函数指针与数据指针有相似的抽象本质：

| 指针类型 | 存储的内容 | 代表的能力 |
|---------|-----------|-----------|
| `int *p` | 内存地址 | 将来访问该位置的能力 |
| `void (*cb)(int)` | 代码地址 | 将来执行该计算的能力 |
| `void *ctx` | 上下文指针 | 将来访问相关数据的能力 |

CPS风格通过函数指针将这种"延迟能力"发挥到极致——程序不再通过返回值传递数据，而是通过调用延续来传递控制流和数据。

这种风格的优势：
- **显式控制流**：没有隐式的返回，每个步骤都清晰可见
- **易于组合**：可以将多个延续串联起来，形成处理管道
- **状态隔离**：每个函数只使用局部变量，避免了副作用

虽然CPS在C语言中不如在函数式语言中那样自然，但理解这种风格能帮助我们更好地掌握指针作为"计算延续"的本质。

## 5.6 函数指针：一等函数的幻觉与CPS实现

C语言不支持一等函数（first-class functions），但函数指针给了我们一个近似。更重要的是，函数指针是实现CPS（Continuation Passing Style）的基础。

### 基础示例

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

### 函数指针的类型语法

函数指针的类型语法很奇特，需要仔细理解：

```c
int (*fp)(int, int);     // fp 是指向函数的指针（接受两个int，返回int）
int *fp(int, int);       // fp 是返回int*的函数（不同！）
int (*fp[10])(int);      // fp 是函数指针数组（10个元素，每个指向接受int返回int的函数）
int (*(*fp)[10])(int);   // fp 是指向函数指针数组的指针
```

**记忆口诀**：`(*fp)` 表示 `fp` 是一个指针。如果去掉 `(*fp)` 后剩下 `int (int, int)`，那就是一个函数类型；如果去掉后剩下 `int *`，那就是返回指针的函数。

可以使用 `typedef` 来简化：

```c
typedef int (*IntFunc)(int);           // IntFunc是函数指针类型
typedef void (*Callback)(void *ctx);   // 带上下文的回调类型
```

C23引入了 `typeof`，可以更方便地声明：

```c
typeof(int (*)(int, int)) fp;          // fp的类型是"指向接受两个int返回int的函数的指针"
```

### 函数指针与CPS的实战模式

在实际应用中，函数指针与CPS结合有以下几种常见模式：

**模式1：错误处理延续**

```c
typedef void (*SuccessCallback)(void *result, void *ctx);
typedef void (*ErrorCallback)(int error_code, const char *msg, void *ctx);

void read_file(const char *path,
               SuccessCallback on_success,
               ErrorCallback on_error,
               void *ctx) {
    FILE *f = fopen(path, "r");
    if (!f) {
        on_error(errno, strerror(errno), ctx);  // 调用错误延续
        return;
    }
    // ... 读取文件 ...
    on_success(buffer, ctx);  // 调用成功延续
}
```

**模式2：状态机驱动**

```c
typedef struct State State;
typedef State *(*StateFunc)(State *s, Event e);

struct State {
    StateFunc handler;
    void *data;
};

// 状态转换通过返回新的状态函数指针实现
State *state_idle(State *s, Event e) {
    if (e.type == EV_START) {
        return state_running(s, e);  // 转换到running状态
    }
    return s;  // 保持当前状态
}
```

**模式3：异步回调链**

```c
typedef void (*AsyncCallback)(void *result, void (*next)(void *), void *ctx);

// 链式异步操作
void async_step1(void *input, AsyncCallback cb, void *ctx);
void async_step2(void *input, AsyncCallback cb, void *ctx);
void async_step3(void *input, AsyncCallback cb, void *ctx);

// 通过函数指针将多个异步步骤串联
void run_pipeline(void *initial_data) {
    async_step1(initial_data, async_step2,
        async_step2, async_step3,
        async_step3, final_handler,
        NULL
    );
}
```

### 函数指针与数据指针的统一视角

从抽象的角度看，函数指针和数据指针都是"地址"，都代表一种"间接访问"的能力：

| 特性 | 数据指针 (`T *`) | 函数指针 (`R (*)(Args...)`) |
|-----|-----------------|---------------------------|
| 存储内容 | 数据对象的地址 | 代码（函数）的地址 |
| 解引用 | `*p` 获取数据 | `fp()` 执行函数 |
| 算术运算 | 支持（`p+1`） | 不支持（未定义行为） |
| 转换为 `void*` | 可以 | 不可以（C标准未定义） |
| 作为延续 | 指向未来数据的承诺 | 指向未来计算的承诺 |

理解这种统一性，有助于我们在设计API时灵活运用指针，无论是传递数据还是传递行为。

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

6. **CPS实践**：将以下直接风格的递归阶乘函数改写为CPS风格：

```c
// 直接风格
int factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}

// 改写为CPS风格，使用函数指针传递延续
typedef void (*FactCallback)(int result, void *ctx);
void factorial_cps(int n, FactCallback cb, void *ctx);
```

7. **延续组合**：实现一个`compose`函数，将两个CPS风格的函数组合成一个：

```c
typedef void (*CPSFunc)(int x, void (*k)(int), void *ctx);

// 返回一个新的CPS函数，先执行f，再将结果传给g
CPSFunc compose(CPSFunc f, CPSFunc g);
```

8. **函数指针数组**：实现一个简单的计算器，使用函数指针数组来根据操作符选择对应的运算函数：

```c
typedef int (*OpFunc)(int, int);

// 支持 +, -, *, / 四种运算
// 使用函数指针数组实现O(1)的操作符查找
int calculate(char op, int a, int b);
```

9. **异步回调链**：实现一个简化的文件处理流水线，使用CPS风格串联多个处理步骤：

```c
typedef void (*FileCallback)(const char *content, void (*next)(const char *), void *ctx);

// 步骤：读取文件 -> 去除空白 -> 转换为大写 -> 写入文件
void pipeline(const char *input_path, const char *output_path);
```

10. **不可变链表与CPS**：参考本章5.5节的链表反转示例，实现一个使用CPS风格的`filter`函数，根据条件过滤链表元素：

```c
typedef bool (*PredFunc)(int32_t val, void *ctx);
typedef void (*FilterCallback)(int_list_t result, CPS_Result res);

// 使用CPS风格：过滤完成后调用cb传递结果
void filter(int_list_t list, PredFunc pred, void *ctx,
            FilterCallback cb, CPS_Result res);
```

## 5.11 延伸阅读

- K&R, Chapter 5 (Pointers and Arrays)
- Expert C Programming, Chapter 4 (The Shocking Truth about C Arrays and Pointers)
- Pierce, "Types and Programming Languages", Chapter 5 (The Untyped Lambda-Calculus)
- [Functional Programming 风格的 C 语言实现](https://hackmd.io/@sysprog/c-functional-programming) - 深入理解CPS与函数指针的结合
- [Why Functional Programming Matters](http://www.cs.kent.ac.uk/people/staff/dat/miranda/whyfp90.pdf) - 理解函数式编程的核心思想
- Linux Kernel源码中的`container_of`宏实现 - 理解指针运算的高级技巧

---

> "指针是C语言的灵魂，也是它的诅咒。它让我们能够直接操纵内存，构建高效的数据结构；也让我们可以搬起石头砸自己的脚，在运行时制造出各种噩梦。但如果我们把指针看作'延续'——一种延迟的计算，一种未来的承诺——它就变得不那么可怕了。就像王小波说的：'生活就是个缓慢受锤的过程'，指针也是个缓慢学习的过程。但一旦掌握，你就拥有了直接对话计算机硬件的能力。"
>
> 当我们将函数指针与CPS风格结合，C语言也能展现出函数式编程的优雅。指针不再只是内存地址，而是计算的延续、未来的承诺。这种视角的转变，或许能让我们在 imperative 与 functional 之间找到一种平衡。"
