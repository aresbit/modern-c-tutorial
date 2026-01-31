# 第八章：内存管理 —— 线性逻辑的实践

## 8.1 资源的生命周期

王小波在《黄金时代》里写道："后来我才知道，生活就是个缓慢受锤的过程，人一天天老下去，奢望也一天天消失。"内存也是如此——从分配的那一刻起，它就在走向被释放的命运。

在C语言中，内存管理是手动的。没有垃圾回收器（GC）来拯救我们，每个`malloc`都必须有对应的`free`。这看似负担，实则是**控制**——对资源生命周期的精确控制。

```c
void *p = malloc(100);  // 出生
if (p == NULL) {
    // 处理分配失败
    return -1;
}

// 使用 p...
strcpy(p, "hello");

free(p);  // 死亡
p = NULL; // 避免野指针
```

## 8.2 堆与栈：两种不同的生命

C程序中有两种主要的内存分配方式：

**栈分配**：自动、快速、大小受限

```c
void foo(void) {
    int local[1000];  // 在栈上，函数返回时自动释放
    // 使用 local...
}  // local 在这里死亡
```

**堆分配**：手动、灵活、持久

```c
void *p = malloc(1000);  // 在堆上
// p 在所有函数返回后仍然存在
free(p);  // 必须显式释放
```

理解何时使用哪种分配方式是C程序设计的核心技能。

## 8.3 所有权：谁负责释放？

**所有权**（ownership）是内存管理的关键概念。Rust语言将其作为核心特性，而C语言中我们手动维护：

```c
// 情况1：调用者拥有所有权（必须free）
char *read_file(const char *path);

// 情况2：函数拥有传入的所有权（接管并可能释放）
void list_add(List *list, void *item);  // list现在拥有item

// 情况3：借用（不转移所有权）
void print_string(const char *s);  // s仍然属于调用者
```

清晰的**所有权约定**是避免内存泄漏和双重释放的关键。

## 8.4 线性类型：使用即消耗

在类型理论中，**线性类型**（linear types）要求每个值必须且只能使用一次。这正好对应C语言中内存管理的安全模型：

```c
// 理想化的"线性指针"
LinearPtr p = linear_malloc(100);  // 获得所有权
use(p);       // 使用 p，但 p 仍然有效（在这个模型中）
// linear_free(p);  // 必须且只能释放一次
```

虽然C语言没有真正的线性类型，但我们可以用**约定**和**静态分析**来模拟：

```c
// 标记所有权转移
#define MOVE(ptr) ({ typeof(ptr) _tmp = ptr; ptr = NULL; _tmp; })

char *s = malloc(100);
strcpy(s, "hello");
char *t = MOVE(s);  // s 变为 NULL，所有权转移到 t
// 现在 s 不能再使用，任何使用都会被工具检测到
free(t);
```

## 8.5 智能指针模式

我们可以用结构体封装指针，添加自动清理功能：

```c
typedef struct {
    void *ptr;
    void (*destructor)(void*);
} SmartPtr;

SmartPtr smart_alloc(size_t size) {
    return (SmartPtr){malloc(size), free};
}

void smart_free(SmartPtr *sp) {
    if (sp->ptr) {
        sp->destructor(sp->ptr);
        sp->ptr = NULL;
    }
}

// 使用GNU C的cleanup属性
void auto_free(void *p) {
    SmartPtr *sp = p;
    smart_free(sp);
}

#define AUTO __attribute__((cleanup(auto_free)))

void example(void) {
    AUTO SmartPtr sp = smart_alloc(100);
    // 使用 sp.ptr...
}  // sp 在这里自动释放
```

## 8.6 内存池：批量管理

对于大量小对象的分配，**内存池**（memory pool）比单独malloc/free更高效：

```c
typedef struct Pool {
    char *buffer;
    size_t size;
    size_t used;
    struct Pool *next;  // 链表支持扩展
} Pool;

Pool *pool_create(size_t initial_size) {
    Pool *p = malloc(sizeof(Pool));
    p->buffer = malloc(initial_size);
    p->size = initial_size;
    p->used = 0;
    p->next = NULL;
    return p;
}

void *pool_alloc(Pool *p, size_t size) {
    // 对齐到8字节
    size = (size + 7) & ~7;

    if (p->used + size > p->size) {
        if (p->next) {
            return pool_alloc(p->next, size);
        }
        // 创建新池（通常是当前大小的2倍）
        p->next = pool_create(p->size * 2);
        return pool_alloc(p->next, size);
    }

    void *result = p->buffer + p->used;
    p->used += size;
    return result;
}

void pool_destroy(Pool *p) {
    while (p) {
        Pool *next = p->next;
        free(p->buffer);
        free(p);
        p = next;
    }
}

// 使用
Pool *pool = pool_create(1024);
char *s1 = pool_alloc(pool, 100);
char *s2 = pool_alloc(pool, 200);
// 不需要单独释放 s1, s2
pool_destroy(pool);  // 一次性释放所有
```

内存池的优势：
- 分配是O(1)，只需移动指针
- 释放是O(1)（一次性）
- 减少内存碎片
- 提高缓存局部性

## 8.7 垃圾回收的模拟

虽然C没有内置GC，但我们可以实现**引用计数**：

```c
typedef struct {
    int count;
    void (*destructor)(void*);
    char data[];  // 柔性数组
} RefCounted;

void *rc_alloc(size_t size, void (*dtor)(void*)) {
    RefCounted *rc = malloc(sizeof(RefCounted) + size);
    rc->count = 1;
    rc->destructor = dtor;
    return rc->data;
}

RefCounted *get_header(void *p) {
    return (RefCounted*)((char*)p - offsetof(RefCounted, data));
}

void *retain(void *p) {
    if (p) get_header(p)->count++;
    return p;
}

void release(void *p) {
    if (!p) return;
    RefCounted *rc = get_header(p);
    if (--rc->count == 0) {
        if (rc->destructor) rc->destructor(p);
        free(rc);
    }
}

// 使用
void *obj = rc_alloc(100, NULL);
void *ref1 = retain(obj);  // count = 2
void *ref2 = retain(obj);  // count = 3
release(ref1);  // count = 2
release(ref2);  // count = 1
release(obj);   // count = 0，释放
```

引用计数简单直观，但无法处理**循环引用**（A引用B，B引用A）。解决这个问题需要完整的GC算法（如标记-清除）。

## 8.8 逃逸分析

**逃逸分析**（escape analysis）是确定对象生命周期的编译技术。在C中，我们手动做这个分析：

```c
// 对象不逃逸，可以用栈分配
void local_buffer(void) {
    char buf[1024];  // 栈上，高效
    // 使用 buf，但不返回或存储到全局
}

// 对象逃逸，必须用堆分配
char *return_buffer(void) {
    char *buf = malloc(1024);  // 堆上
    // 填充 buf...
    return buf;  // 逃逸了！
}
```

## 8.9 RAII：资源获取即初始化

C++的RAII惯用法在C中也可以模拟：

```c
// 文件句柄包装
typedef struct {
    FILE *fp;
} File;

File file_open(const char *path, const char *mode) {
    return (File){fopen(path, mode)};
}

void file_close(File *f) {
    if (f->fp) {
        fclose(f->fp);
        f->fp = NULL;
    }
}

// 使用 GNU C cleanup
static inline void auto_close(File *f) { file_close(f); }
#define AUTO_FILE __attribute__((cleanup(auto_close)))

void process_file(const char *path) {
    AUTO_FILE File f = file_open(path, "r");
    if (!f.fp) return;  // 自动关闭
    // 处理文件...
}  // f 自动关闭
```

## 8.10 常见内存错误与防御

### 缓冲区溢出

```c
char buf[10];
strcpy(buf, "this is too long");  // 溢出！

// 使用安全版本
strncpy(buf, "this is too long", sizeof(buf) - 1);
buf[sizeof(buf) - 1] = '\0';  // 确保终止
```

### 使用已释放内存

```c
free(p);
// ...
*p = 42;  // 未定义行为！

// 防御：立即置NULL
free(p);
p = NULL;
// *p = 42;  // 现在会段错误，容易发现
```

### 内存泄漏

```c
void leak(void) {
    char *p = malloc(100);
    if (error_condition) {
        return;  // 泄漏！忘记 free(p)
    }
    free(p);
}

// 防御：提前返回时确保清理
void no_leak(void) {
    char *p = malloc(100);
    if (error_condition) {
        free(p);  // 清理
        return;
    }
    // ...
    free(p);
}

// 或更好的：使用 goto 统一清理
good_pattern(void) {
    char *p1 = NULL, *p2 = NULL;
    int result = -1;

    p1 = malloc(100);
    if (!p1) goto cleanup;

    p2 = malloc(200);
    if (!p2) goto cleanup;

    // 使用 p1, p2...
    result = 0;

cleanup:
    free(p2);
    free(p1);
    return result;
}
```

## 8.11 工具辅助

现代工具可以帮助我们检测内存问题：

```bash
# Valgrind - 检测泄漏和非法访问
valgrind --leak-check=full ./program

# AddressSanitizer - 编译时检测
gcc -fsanitize=address -g program.c
./program

# 静态分析
clang --analyze program.c
```

## 8.12 练习

1. 实现一个**arena allocator**，支持分配任意大小的对象，但只能一次性释放整个arena。

2. 写一个**弱引用**系统，允许引用可能被释放的对象，访问时自动检查有效性。

3. 实现**内存跟踪**：在调试模式下记录所有分配/释放，程序退出时报告泄漏。

4. 研究**tcmalloc**或**jemalloc**。它们如何优化多线程程序的内存分配？

5. 用**位图**实现一个固定大小对象的分配器（如分配4KB页）。

## 8.13 延伸阅读

- K&R, Chapter 8 (The UNIX System Interface)
- Advanced Programming in the UNIX Environment, Chapter 7
- "Memory Management: Algorithms and Implementation"
- jemalloc design document

---

> "我活在世上，无非想要明白些道理，遇见些有趣的事。"内存管理可能是C语言中最"无趣"的部分——它琐碎、容易出错、需要大量注意力。但正是在这种琐碎中，我们学会了**责任**和**精确**。每一个`malloc`都有一个对应的`free`，就像每一个开始都有一个结束。这不是悲观，这是对生命（和资源）循环的深刻理解。王小波怀念他的黄金时代，我们也应该珍惜每一次成功管理内存的时刻——因为那是我们作为程序员的成年礼。
