# 第十章：构建更大的程序 —— 模块与抽象

## 10.1 从代码到系统

王小波在《黄金时代》的结尾写道："后来我才知道，生活就是个缓慢受锤的过程，人一天天老下去，奢望也一天天消失，最后变得像挨了锤的牛一样。可是我过二十一岁生日时没有预见到这一点。我觉得自己会永远生猛下去，什么也锤不了我。"

写小程序和写大系统，就像年轻和年老的区别。小程序可以"生猛"——一股脑写下来，直接运行。但大系统需要**架构**、**纪律**和**抽象**。

本章讨论如何用C语言构建可维护的大型程序。

## 10.2 翻译单元与链接

C程序的构建单元是**翻译单元**（translation unit），通常对应一个`.c`文件及其包含的头文件：

```c
// math_utils.c  -> 编译 -> math_utils.o
#include "math_utils.h"
#include <math.h>

double square(double x) {
    return x * x;
}

static double helper(double x) {  // static = 内部链接
    return x * 0.5;
}
```

**链接**（linking）将多个目标文件组合成可执行文件：

```bash
gcc main.c math_utils.c -o program
# 或分开编译
gcc -c main.c          # main.o
gcc -c math_utils.c    # math_utils.o
gcc main.o math_utils.o -o program  # 链接
```

理解**链接规则**至关重要：

```c
// 外部链接：其他文件可见
int global_var;
void global_func(void);

// 内部链接：仅本文件可见
static int internal_var;
static void internal_func(void);

// 无链接：局部变量
void foo(void) {
    int local_var;
}
```

## 10.3 头文件的艺术

头文件是**接口契约**。好的头文件应该：

1. **自包含**：包含它所需的一切
2. **有包含保护**：防止重复包含
3. **最小化依赖**：前向声明优于包含

```c
// 好的头文件示例：renderer.h
#ifndef RENDERER_H
#define RENDERER_H

#include <stddef.h>   // size_t
#include <stdbool.h>  // bool

// 前向声明，避免包含具体定义
typedef struct Renderer Renderer;
typedef struct Texture Texture;
typedef struct Shader Shader;

// 初始化/清理
Renderer *renderer_create(void);
void renderer_destroy(Renderer *r);

// 资源管理（所有权转移给Renderer）
Texture *texture_load(Renderer *r, const char *path);
Shader *shader_compile(Renderer *r, const char *source);

// 渲染命令
void renderer_clear(Renderer *r, float r, float g, float b);
void renderer_draw(Renderer *r, Texture *tex, float x, float y);

// 查询
bool renderer_is_initialized(const Renderer *r);
size_t renderer_get_draw_calls(const Renderer *r);

#endif
```

## 10.4 命名空间模拟

C没有命名空间，但我们可以通过**命名约定**模拟：

```c
// 模块名作为前缀
// string_buffer.h
#ifndef STRING_BUFFER_H
#define STRING_BUFFER_H

typedef struct StrBuf StrBuf;

StrBuf *strbuf_new(void);
StrBuf *strbuf_new_with_capacity(size_t cap);
void strbuf_free(StrBuf *sb);

void strbuf_append(StrBuf *sb, const char *s);
void strbuf_append_char(StrBuf *sb, char c);
void strbuf_append_fmt(StrBuf *sb, const char *fmt, ...);

const char *strbuf_as_str(const StrBuf *sb);
size_t strbuf_len(const StrBuf *sb);
void strbuf_clear(StrBuf *sb);

#endif
```

这种`module_action`的命名约定有效避免了命名冲突。

## 10.5 错误处理策略

大型程序需要一致的错误处理。C语言有几种模式：

### 模式1：返回错误码

```c
typedef enum {
    ERR_OK = 0,
    ERR_NOMEM,
    ERR_NOT_FOUND,
    ERR_INVALID_ARG
} Error;

Error do_something(int x);
```

### 模式2：返回结果结构

```c
typedef struct {
    Error error;
    union {
        int value;
        const char *msg;
    };
} Result;

Result divide(int a, int b);
```

### 模式3：异常模拟（setjmp/longjmp）

```c
#include <setjmp.h>

jmp_buf error_handler;

void might_fail(void) {
    if (something_wrong) {
        longjmp(error_handler, ERR_SOMETHING);
    }
}

int main(void) {
    Error err = setjmp(error_handler);
    if (err != 0) {
        printf("Error: %d\n", err);
        return 1;
    }

    might_fail();  // 如果出错，跳回setjmp处
    return 0;
}
```

`setjmp/longjmp`类似于异常，但不调用析构函数，容易泄漏资源。

### 模式4：错误回调

```c
typedef void (*ErrorHandler)(const char *context, Error err);

void set_error_handler(ErrorHandler handler);
```

适合库代码，让用户决定如何处理错误。

## 10.6 模块系统的设计

让我们设计一个简单的模块系统：

```c
// module.h
#ifndef MODULE_H
#define MODULE_H

typedef struct Module Module;
typedef struct ModuleRegistry ModuleRegistry;

ModuleRegistry *mod_registry_create(void);
void mod_registry_destroy(ModuleRegistry *reg);

// 注册模块（在main之前自动执行）
typedef void (*ModuleInitFunc)(ModuleRegistry *);

#define REGISTER_MODULE(name, init_func) \
    __attribute__((constructor)) \
    static void register_##name(void) { \
        mod_register(#name, init_func); \
    }

void mod_register(const char *name, ModuleInitFunc init);
Module *mod_get(ModuleRegistry *reg, const char *name);

// 模块生命周期
int mod_init(Module *m, ModuleRegistry *reg);
void mod_shutdown(Module *m);
bool mod_is_initialized(const Module *m);

#endif
```

这允许插件式的架构：

```c
// database_module.c
#include "module.h"

static void db_init(ModuleRegistry *reg) {
    // 初始化数据库模块
}

REGISTER_MODULE(database, db_init);
```

## 10.7 面向对象与虚表

C没有类，但我们可以用结构体和函数指针实现**面向对象**：

```c
// shape.h
typedef struct Shape Shape;
typedef struct ShapeOps ShapeOps;

struct ShapeOps {
    double (*area)(const Shape *);
    void (*draw)(const Shape *, void *renderer);
    void (*destroy)(Shape *);
};

struct Shape {
    const ShapeOps *ops;
};

static inline double shape_area(const Shape *s) {
    return s->ops->area(s);
}

static inline void shape_draw(const Shape *s, void *r) {
    s->ops->draw(s, r);
}

static inline void shape_destroy(Shape *s) {
    s->ops->destroy(s);
}
```

子"类"实现：

```c
// circle.c
typedef struct {
    Shape base;  // 必须放在第一位，用于转换
    double radius;
} Circle;

static double circle_area(const Shape *s) {
    Circle *c = (Circle*)s;
    return M_PI * c->radius * c->radius;
}

static const ShapeOps circle_ops = {
    circle_area,
    circle_draw,
    circle_destroy
};

Shape *circle_create(double r) {
    Circle *c = malloc(sizeof(Circle));
    c->base.ops = &circle_ops;
    c->radius = r;
    return (Shape*)c;
}
```

这是**子类型多态**的完整实现。

## 10.8 构建系统简介

手动编译大型项目是不现实的。我们使用**构建系统**：

### Makefile

```makefile
CC = gcc
CFLAGS = -Wall -Wextra -std=c11 -O2
SRCS = main.c math_utils.c string_buffer.c renderer.c
OBJS = $(SRCS:.c=.o)
TARGET = program

.PHONY: all clean

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $(OBJS) -o $@ $(LDFLAGS)

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJS) $(TARGET)

# 依赖生成
-include $(SRCS:.c=.d)
%.d: %.c
	$(CC) -MM $(CFLAGS) $< > $@
```

### CMake（现代选择）

```cmake
cmake_minimum_required(VERSION 3.10)
project(MyProject C)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

add_library(math_utils math_utils.c)
add_library(renderer renderer.c)

target_link_libraries(renderer PRIVATE math_utils)

add_executable(program main.c)
target_link_libraries(program PRIVATE renderer)

# 测试
enable_testing()
add_subdirectory(tests)
```

## 10.9 测试策略

C语言测试通常用**单元测试框架**或**断言**：

```c
// 简单测试宏
#define TEST(name) static void test_##name(void)
#define ASSERT(cond) \
    do { \
        if (!(cond)) { \
            fprintf(stderr, "FAIL: %s at %s:%d\n", #cond, __FILE__, __LINE__); \
            abort(); \
        } \
    } while(0)
#define ASSERT_EQ(a, b) ASSERT((a) == (b))

TEST(vector_create) {
    Vector *v = vector_new();
    ASSERT(v != NULL);
    ASSERT_EQ(vector_size(v), 0);
    vector_free(v);
}

int main(void) {
    test_vector_create();
    printf("All tests passed!\n");
    return 0;
}
```

或使用框架如**Check**、**Unity**或**cmocka**。

## 10.10 文档与注释

**文档注释**使用Doxygen风格：

```c
/**
 * @brief Creates a new dynamic array
 *
 * @param elem_size Size of each element in bytes
 * @param initial_capacity Initial capacity (0 for default)
 * @return Pointer to new array, or NULL on error
 *
 * @note Caller is responsible for freeing with array_free()
 * @see array_free, array_push
 *
 * Example:
 * @code
 * Array *arr = array_new(sizeof(int), 10);
 * int x = 42;
 * array_push(arr, &x);
 * array_free(arr);
 * @endcode
 */
Array *array_new(size_t elem_size, size_t initial_capacity);
```

生成文档：

```bash
doxygen Doxyfile  # 生成 HTML/LaTeX 文档
```

## 10.11 设计模式在C中的实现

### 观察者模式

```c
typedef struct Observer Observer;
typedef struct Subject Subject;

struct Observer {
    void (*notify)(Observer *, Subject *, Event);
    Observer *next;  // 链表
};

struct Subject {
    Observer *observers;
};

void subject_attach(Subject *s, Observer *o);
void subject_detach(Subject *s, Observer *o);
void subject_notify(Subject *s, Event e);
```

### 状态机

```c
// 使用函数指针表
typedef void (*StateFunc)(Context *, Event);

void state_idle(Context *ctx, Event e);
void state_running(Context *ctx, Event e);
void state_paused(Context *ctx, Event e);

static const StateFunc state_table[] = {
    [STATE_IDLE] = state_idle,
    [STATE_RUNNING] = state_running,
    [STATE_PAUSED] = state_paused,
};

// 状态转移
static const State next_state[][EVENT_COUNT] = {
    [STATE_IDLE] = {
        [EVENT_START] = STATE_RUNNING,
    },
    [STATE_RUNNING] = {
        [EVENT_PAUSE] = STATE_PAUSED,
        [EVENT_STOP] = STATE_IDLE,
    },
    // ...
};
```

## 10.12 性能分析

使用**profiler**找出瓶颈：

```bash
# gprof
gcc -pg program.c -o program
./program
gprof ./program gmon.out

# perf (Linux)
perf record ./program
perf report

# Valgrind cachegrind
valgrind --tool=cachegrind ./program
```

手动计时：

```c
#include <time.h>

clock_t start = clock();
// ... 被测代码 ...
clock_t end = clock();
double elapsed = (double)(end - start) / CLOCKS_PER_SEC;
```

## 10.13 练习

1. 设计一个**插件系统**，允许动态加载共享库（`.so`或`.dll`）。定义插件接口和加载机制。

2. 实现一个**引用计数的智能指针**系统，支持`retain`、`release`和自动释放池。

3. 创建一个**日志系统**，支持多个后端（文件、网络、syslog），使用宏提供源文件位置信息。

4. 设计一个**配置系统**，支持INI/JSON/TOML格式，提供类型安全的访问API。

5. 为现有的C库（如SQLite或zlib）编写**绑定生成器**，从头文件自动生成其他语言的绑定。

## 10.14 延伸阅读

- "Large-Scale C++ Software Design"（概念适用于C）
- "C Interfaces and Implementations"
- "The Art of Unix Programming"
- Linux Kernel Coding Style
- FFmpeg或SQLite源代码（优秀的大型C项目）

---

> "我活在世上，无非想要明白些道理，遇见些有趣的事。倘能如我所愿，我的一生就算成功。"构建大型程序也是如此——我们追求的不仅是功能的实现，而是**清晰的设计**、**可维护的结构**和**优雅的抽象**。C语言给了我们极少的语法，却给了我们极大的自由。在这种自由中，架构的艺术得以展现。王小波怀念那只特立独行的猪，我们也应该追求那些特立独行的设计——不是为反叛而反叛，而是为了在约束中创造出真正有价值的东西。

---

## 结语

这本书从一个简单的想法开始：用函数式的视角重新审视C语言。我们从类型出发，走过Sum Types和Product Types的代数世界；我们探索了指针作为计算延续的本质；我们发现了预处理宏的函数式力量；我们学习了内存管理的线性逻辑；最后，我们把这些概念组合成可维护的大型系统。

C语言诞生于1972年，至今已超过50年。它不完美，它危险，它需要程序员承担巨大的责任。但正是这种责任，让C语言程序员成为真正的工程师——我们不只是写代码，我们在管理复杂性，在约束中创造，在危险中舞蹈。

王小波说："一个人只拥有此生此世是不够的。"学习C语言，也是在追求那个"诗意的世界"——在那里，比特服从我们的意志，硅片执行我们的思想。这难道不是一种诗意吗？

愿你在C语言的旅程中找到自己的诗意。
