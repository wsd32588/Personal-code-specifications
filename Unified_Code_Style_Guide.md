# 个人项目代码规范手册

**C · C++ · Go · Rust**

可读性优先 · 显式优于隐式 · 不过度抽象

---

## 0. 如何使用本手册

本手册分两层：

- **第一部分（§1）** 是语言无关的通用原则，适用于所有项目的所有语言。
- **第二部分（§2–§7）** 是各语言的具体规则，只在用到该语言时查阅。

> 💡 遇到"应该怎么写"的疑问，先查通用原则。通用原则没覆盖再查对应语言章节。

## 设计哲学

本规范的核心目标：**降低认知负担，降低未来维护成本。**

| 追求 | 不追求 |
|------|--------|
| 降低认知负担 | 极限代码高尔夫 |
| 降低未来维护成本 | 过度抽象 |
| 显式优于隐式 | 模板/宏炫技 |
| 稳定优于炫技 | 为"高级感"而复杂化 |
| 工程可维护性优先于短期速度 | 过早泛化 |

**跨语言一致性**是本规范的设计意图。C、C++、Go、Rust 的具体写法虽不同，但错误处理、模块化、防御式编程等工程思想是统一的——语言只是实现细节，工程思想才是主体。

## 破例原则

规则服务于可维护性。如果违反某条规则能显著提升**正确性、可读性、性能（且已测量）、平台兼容性**之一，则允许破例，但必须在代码注释或 ADR 中说明原因。

> 规则是经验，不是法律。

## ADR 模板（前瞻）

当项目出现需要记录决策的场景（架构选型、API 设计权衡、破例理由），建议用 ADR（Architecture Decision Record）存档，格式如下：

```markdown
# ADR-NNNN: 标题

## 背景
为什么需要做这个决策？

## 选择
选了什么方案？为什么？

## 后果
优点、缺点、兼容性影响。
```

人最容易忘的不是"怎么写"，而是"为什么当初这么设计"。
---

# 1. 通用原则（语言无关）

## 1.1 优先级排序

当几个目标冲突时，按以下顺序取舍：

| # | 原则 | 说明 |
|---|------|------|
| ① | 可读性 | 代码首先是给未来的自己看的。 |
| ② | 可维护性 | 三个月后还能快速定位和修改。 |
| ③ | 正确性 | 显式的错误处理 > 祈祷不出错。 |
| ④ | 性能 | 先写对，再在有测量依据时优化。 |
| ⑤ | 字符数 | 短不是美德，清晰才是。 |

## 1.2 命名：语义完整，拒绝缩写

名字是最廉价的文档。缩写把认知负担转移给读者（未来的自己）。

| 不推荐（✗） | 推荐（✓） |
|-------------|-----------|
| `ctx, cfg, buf, tmp, fp, nr` | `render_context, editor_config` |
| `proc(), init2(), handle()` | `text_buffer, file_ptr, row_count` |
| | `process_input(), init_editor()` |

> 💡 例外：循环变量 i/j/k，数学公式中的 x/y/n，以及语言社区有强约定的缩写（如 Go 的 `err`）。

## 1.3 函数：单一职责，显式依赖

一个函数做一件事。名字能用一句话说清它做了什么。所有依赖通过参数显式传入。

```c
// 不推荐（✗）：三件事混在一起
void process(void) {
    read_input();   // 读
    update_state(); // 更新
    render();       // 渲染
}

// 推荐（✓）：各自独立，主循环组合
void main_loop(App *app) {
    int key = input_read_key();
    app_handle_key(app, key);
    render_frame(app);
}
```

## 1.4 注释：解释"为什么"，不重复"做什么"

```c
// 不推荐（✗）：废话注释
i++;        // i 加 1
fp = NULL;  // 置空 fp

// 推荐（✓）：解释为什么这么处理
// Ctrl-H (0x08) 和 DEL (0x7f) 在不同
// 终端仿真器里都映射为退格，统一处理。
case 0x08: case 0x7f:
    editor_delete_char(e); break;
```

## 1.5 模块：单一职责，单向依赖

一个文件/模块只负责一类事情。依赖关系必须是有向无环图（DAG），禁止循环依赖。

```
// 好的依赖方向（从上到下）
main  →  app, render, input
render  →  buffer
input   →  app
buffer  →  （无依赖）

// 禁止: render → input → render  （循环）
```

## 1.6 不写"高手感"代码

优先选择平凡但清晰的实现。炫技代码的维护成本由未来的自己承担。

```c
// 不推荐（✗）：聪明但难懂
int is_pow2 = n && !(n & (n-1));
x ^= y ^= x ^= y; // 无临时变量交换

// 推荐（✓）：平凡且清晰
int is_pow2 = (n > 0) && ((n & (n-1)) == 0);
int tmp = x; x = y; y = tmp;
```

## 1.7 显式优于隐式

任何"这个值从哪冒出来的"的疑问，都是隐式性造成的。以下场景必须显式：

| 场景 | 要求 |
|------|------|
| 类型转换 | 禁止隐式窄化转换。double/size_t 转 int 等必须显式 cast，并前置 assert 确认安全。 |
| 控制流 | if / else / for 的花括号不省略，哪怕只有一行。省略大括号在 diff 中容易引入隐患。 |
| 全局状态 | 函数内部不静默依赖全局变量。依赖必须通过参数传入，或在注释中明确声明。 |
| 函数副作用 | 有副作用的函数在名字或注释中体现。`editor_insert_and_mark_dirty` 好过 `editor_insert`。 |

```c
// 不推荐（✗）：隐式窄化，可能截断
size_t len = buffer_length();
int n = len;  // 悄悄截断

// 推荐（✓）：显式转换 + 前置检查
size_t len = buffer_length();
assert(len <= INT_MAX);
int n = (int)len;  // 明确知道这里安全
```

```c
// 不推荐（✗）：省略大括号，diff 陷阱
if (is_error)
    log_error();
    return -1;  // 实际不在 if 里！

// 推荐（✓）：始终有花括号
if (is_error) {
    log_error();
    return -1;
}
```

## 1.8 防御性编程

写代码时假设调用方会传错参数、依赖会失效、外部数据是有毒的。

### assert 检查不变量（invariant）

assert 保护"理论上绝不可能发生"的前提条件，针对编程错误，不针对运行时输入。

```c
void buffer_append(Buffer *buf, const char *src, int len) {
    assert(buf != NULL);  // 调用方不应传 NULL——这是编程错误
    assert(src != NULL);
    assert(len >= 0);
    // ... 正式逻辑
}
// assert 在 release 构建（-DNDEBUG）下会被剥离
// 因此不能用 assert 做运行时输入校验
```

### 外部输入必须显式校验

用户、网络、文件、环境变量——所有外部来源的数据在使用前必须校验，不信任任何假设。

```c
// 不推荐（✗）：直接使用外部数据
int port = atoi(argv[1]);
bind(sock, port);  // port 可能是 -1 或 99999

// 推荐（✓）：校验后再用
int port = atoi(argv[1]);
if (port < 1 || port > 65535) {
    fprintf(stderr, "invalid port\n");
    exit(1);
}
bind(sock, port);
```

> 💡 两类错误不可混用：assert 针对编程错误（invariant 被违反），显式校验针对运行时的外部输入。

---

# 2. C 规范

## 2.1 命名约定

| 类别 | 格式 | 示例 | 说明 |
|------|------|------|------|
| 类型 / struct | PascalCase | `Editor`, `RenderBuffer` | typedef struct 统一用 PascalCase |
| 函数 | snake_case | `editor_insert_char()` | 格式：模块\_动作\_对象 |
| 变量 | snake_case | `cursor_x`, `row_count` | 完整语义，不缩写 |
| 全局变量 | g\_ 前缀 | `g_editor` | 标记全局状态，提示谨慎修改 |
| 常量 / 宏 | UPPER_SNAKE | `TAB_SIZE`, `MAX_ROWS` | 全大写 |
| 枚举成员 | PREFIX_UPPER | `KEY_ARROW_UP` | 带模块前缀避免污染 |

## 2.2 文件组织

### #include 顺序（各组间空一行）

```c
#include "editor.h"    // 1. 本模块头文件（强制自洽检查）

#include <stdio.h>     // 2. 标准库（字母序）
#include <stdlib.h>
#include <string.h>

#include "buffer.h"    // 3. 项目内其他模块（字母序）
#include "render.h"
```

### 头文件只暴露接口

```c
// 不推荐（✗）：暴露实现细节
// editor.h
static int internal_counter;
void _helper(void); // 下划线开头也不应在头文件

// 推荐（✓）：只放公开接口
// editor.h
#pragma once
typedef struct Editor Editor;
void editor_insert_char(Editor *e, char c);
void editor_delete_char(Editor *e);
```

## 2.3 错误处理

### 基本模式：返回值 + 上下文信息

```c
// 函数返回 0 表示成功，-1 表示失败
// 失败时用 perror() 或 fprintf(stderr) 提供上下文
int editor_open(Editor *e, const char *path) {
    FILE *fp = fopen(path, "r");
    if (!fp) {
        fprintf(stderr, "editor_open: cannot open '%s': %s\n",
                path, strerror(errno));
        return -1;
    }
    // ...
    fclose(fp);
    return 0;
}
```

### 多资源释放：goto cleanup 统一出口

```c
int load_file(const char *path, char **out, size_t *out_len) {
    FILE *fp   = NULL;
    char *buf  = NULL;
    int   rc   = -1;

    fp = fopen(path, "rb");
    if (!fp) { perror("load_file: fopen"); goto done; }

    fseek(fp, 0, SEEK_END);
    long len = ftell(fp);
    rewind(fp);

    buf = malloc((size_t)len + 1);
    if (!buf) { perror("load_file: malloc"); goto done; }

    if (fread(buf, 1, (size_t)len, fp) != (size_t)len) {
        fprintf(stderr, "load_file: short read\n"); goto done;
    }
    buf[len] = '\0';
    *out = buf; *out_len = (size_t)len;
    rc = 0;    // 成功

done:
    if (fp)        fclose(fp);
    if (rc && buf) free(buf);   // 失败时才释放
    return rc;
}
```

> 💡 goto 在 C 错误处理里是正当用法，不是坏习惯。它比多层嵌套的 if-else 清晰得多。

### 不忽略返回值

```c
// 不推荐（✗）：返回值丢弃
malloc(size);         // 返回值丢弃
fread(buf, 1, n, fp); // 不检查读了多少

// 推荐（✓）
void *p = malloc(size);
if (!p) { perror("malloc"); exit(1); }

size_t n = fread(buf, 1, len, fp);
if (n != len) { /* 处理短读 */ }
```

## 2.4 结构体规范

```c
// 不推荐（✗）：混杂了多种状态
typedef struct {
    int cx, cy;          // 光标
    struct termios orig; // 终端
    char *filename;      // 文件
    char **rows; int n;
} E; // 什么都管，什么都不专

// 推荐（✓）：各司其职
typedef struct {
    int cursor_x, cursor_y;
    int screen_rows, screen_cols;
    int is_modified;
    EditorRow *rows;
    int row_count;
    char *filename;
} Editor;

typedef struct { struct termios orig; } Terminal;
```

```c
// 不推荐（✗）：依赖 calloc 副作用做初始化
Editor *e = calloc(1, sizeof(Editor));
// 读者需要知道 calloc 保证零初始化才能理解语义

// 推荐（✓）：显式零初始化，意图明确
Editor e = {0};
// 或堆分配后立刻显式初始化
Editor *e = malloc(sizeof(Editor));
if (e) *e = (Editor){0};
```

## 2.5 C 专项反模式

- 用 `void *` 做泛型时，必须有类型 tag 或文档说明实际类型。
- 字符串操作优先 snprintf，禁止 `sprintf` / `gets` / `strcpy` 到固定缓冲区。
- 指针运算后必须检查是否越界，不依赖"刚好不越界"。

---

# 3. C++ 规范

## 3.1 命名约定

| 类别 | 格式 | 示例 | 说明 |
|------|------|------|------|
| 类 / 结构体 | PascalCase | `PacketParser`, `RingBuffer` | |
| 函数 / 方法 | snake_case | `parse_header()`, `send_packet()` | 和 C 一致 |
| 成员变量 | snake_case + \_ | `capacity_`, `head_ptr_` | 尾下划线标记成员 |
| 静态常量 | kPascalCase | `kMaxRetries`, `kDefaultPort` | Google 风格 |
| 宏 | UPPER_SNAKE | `NDEBUG`, `MAX_BUF` | 尽量用 constexpr 替代 |
| 命名空间 | lower_snake | `net::tcp`, `core::buffer` | 全小写 |

## 3.2 错误处理策略

原则：错误码为主，exception 仅用于真正无法恢复的情况。

### 推荐：返回 std::expected / std::optional（C++23 / C++17）

```cpp
#include <expected>  // C++23
#include <string>

struct ParseError { std::string message; int code; };

std::expected<Packet, ParseError>
parse_packet(std::span<const uint8_t> data) {
    if (data.size() < HEADER_SIZE)
        return std::unexpected(ParseError{"too short", -1});

    Packet pkt;
    // ... 解析 ...
    return pkt;
}

// 调用方：强制处理错误，不能忽略
auto result = parse_packet(buf);
if (!result) {
    log_error(result.error().message);
    return result.error().code;
}
process(result.value());
```

### C++17 兼容：返回错误码 + 输出参数

```cpp
enum class NetError { Ok = 0, Timeout, Disconnected, InvalidData };

NetError read_frame(Socket &sock, Frame *out_frame) {
    // ...
    if (timeout) return NetError::Timeout;
    *out_frame = parsed;
    return NetError::Ok;
}

// 调用方
Frame frame;
if (auto err = read_frame(sock, &frame); err != NetError::Ok) {
    handle_error(err);
    return err;
}
```

### exception 的合法使用场景

- 构造函数中资源获取失败（RAII 无法返回错误码）。
- 深层调用栈的致命错误，且调用链上每一层都无法合理处理。
- 第三方库抛出异常时的边界捕获（转换为错误码后继续处理）。

```cpp
// 构造函数中合理使用 exception
class MmapFile {
public:
    explicit MmapFile(const std::string &path) {
        fd_ = open(path.c_str(), O_RDONLY);
        if (fd_ < 0)
            throw std::system_error(errno, std::generic_category(),
                                    "MmapFile: open " + path);
        // mmap...
    }
    ~MmapFile() { if (fd_ >= 0) close(fd_); }  // RAII 保证清理
};
```

## 3.3 RAII 与资源管理

所有资源（内存、文件、锁、socket）必须由 RAII 对象管理，禁止裸 new/delete 出现在业务逻辑中。

```cpp
// 不推荐（✗）：裸指针，容易泄漏
Packet *pkt = new Packet();
if (parse_fail) return -1; // 泄漏!
process(pkt);
delete pkt;

// 推荐（✓）：unique_ptr 自动管理
auto pkt = std::make_unique<Packet>();
if (parse_fail) return -1; // 自动释放
process(*pkt);
```

## 3.4 现代 C++ 使用原则

- 优先 constexpr 而非宏常量；优先 enum class 而非 enum。
- range-for 优先于下标循环，除非确实需要下标。
- `std::string_view` 用于只读字符串参数，避免不必要的拷贝。
- 模板仅在确实需要泛型时使用，不为了"面向对象感"而模板化。
- `auto` 用于类型明显的场合（迭代器、make_unique 返回值），不滥用。

## 3.5 C++ 专项反模式

```cpp
// 不推荐（✗）：用继承模拟接口层次，过度设计
class IRenderer { virtual void render() = 0; };
class IBufferedRenderer : public IRenderer {...};
class ConcreteRenderer : public IBufferedRenderer {...};
// 三层继承，实际只有一种实现

// 推荐（✓）：直接实现，必要时再抽象
class Renderer {
public:
    void render();
private:
    Buffer buffer_;
};
```

```cpp
// 不推荐（✗）：对返回值 std::move，阻断 NRVO
Buffer make_buffer() {
    Buffer buf;
    // ...
    return std::move(buf);  // 错！禁用了编译器优化
}

// 推荐（✓）：直接返回，让编译器做 NRVO
Buffer make_buffer() {
    Buffer buf;
    // ...
    return buf;  // 编译器会自动优化（零拷贝）
}
```

不需要的特殊成员函数用 `= delete` 显式禁用，不要依赖"不声明就不会被调用"的侥幸：

```cpp
class NonCopyable {
public:
    NonCopyable() = default;
    ~NonCopyable() = default;
    NonCopyable(const NonCopyable &)            = delete;  // 明确禁止拷贝
    NonCopyable &operator=(const NonCopyable &) = delete;
    NonCopyable(NonCopyable &&)                 = default; // 允许移动
    NonCopyable &operator=(NonCopyable &&)      = default;
};
```

---

# 4. Go 规范

## 4.1 命名约定

| 类别 | 格式 | 示例 | 说明 |
|------|------|------|------|
| 导出标识符 | PascalCase | `PacketParser`, `ReadFrame` | 首字母大写 = 公开 |
| 非导出标识符 | camelCase | `packetParser`, `readFrame` | 首字母小写 = 包私有 |
| 接口 | PascalCase + -er | `Reader`, `Stringer`, `Closer` | 单方法接口用 -er 后缀 |
| 错误变量 | Err + PascalCase | `ErrTimeout`, `ErrNotFound` | 导出错误用 Err 前缀 |
| 常量 | PascalCase | `MaxRetries`, `DefaultPort` | |
| 包名 | lowercase | `net`, `bufio`, `httputil` | 全小写，无下划线 |

> 💡 Go 社区接受短变量名（buf, n, err, ok），但仅限于作用域极短的局部变量。跨越多行的变量仍需完整名字。

## 4.2 错误处理

### 基本惯用法：if err != nil 立即处理

```go
func readConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        // 用 fmt.Errorf + %w 包装，保留原始错误链
        return nil, fmt.Errorf("readConfig: %w", err)
    }

    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        return nil, fmt.Errorf("readConfig: unmarshal: %w", err)
    }
    return &cfg, nil
}
```

### 错误检查：errors.Is / errors.As

```go
// 判断错误类型，不要直接比较 error.Error() 字符串
cfg, err := readConfig("config.json")
if err != nil {
    if errors.Is(err, os.ErrNotExist) {
        log.Println("config not found, using defaults")
        cfg = defaultConfig()
    } else {
        return fmt.Errorf("startup: %w", err)
    }
}

// 提取具体错误类型
var netErr *net.OpError
if errors.As(err, &netErr) {
    log.Printf("network error on %s: %v", netErr.Net, netErr)
}
```

### 自定义错误类型

```go
// 需要携带上下文信息时，定义专属错误类型
type ParseError struct {
    Line    int
    Column  int
    Message string
}

func (e *ParseError) Error() string {
    return fmt.Sprintf("parse error at %d:%d: %s", e.Line, e.Column, e.Message)
}
```

### panic 的合法使用场景

- 程序员错误（不变量被违反）：如 nil 指针被传入明确要求非 nil 的函数。
- 初始化阶段的不可恢复错误（init() 函数中）。
- **禁止**在常规业务逻辑中用 panic 代替错误返回。

## 4.3 接口设计

Go 接口应该小而精。依赖接口而非具体类型，但不要为了"面向接口"而提前抽象。

```go
// 不推荐（✗）：过大的接口，难以实现和 mock
type Storage interface {
    Read(id string) ([]byte, error)
    Write(id string, data []byte) error
    Delete(id string) error
    List(prefix string) ([]string, error)
    Stats() StorageStats
    Flush() error
}

// 推荐（✓）：小接口，按需组合
type Reader interface {
    Read(id string) ([]byte, error)
}
type Writer interface {
    Write(id string, data []byte) error
}
// 需要读写时：io.ReadWriter 模式
type ReadWriter interface {
    Reader; Writer
}
```

## 4.4 并发规范

- goroutine 必须有明确的退出机制（context.Context 取消 或 channel close）。
- 共享状态优先用 channel 传递，次选 sync.Mutex，禁止裸共享内存。
- 启动 goroutine 前确认谁负责等待它结束（sync.WaitGroup 或 errgroup）。
- 库代码默认同步——将并发留给调用方。让调用方通过 `go func()` 决定是否并发，而不是在库内部偷偷启动 goroutine。

```go
func worker(ctx context.Context, jobs <-chan Job, wg *sync.WaitGroup) {
    defer wg.Done()
    for {
        select {
        case <-ctx.Done():
            return  // 优雅退出
        case job, ok := <-jobs:
            if !ok { return }  // channel 已关闭
            process(job)
        }
    }
}
```

```go
// 不推荐（✗）：库函数内部偷启动 goroutine，调用方失控
func (c *Client) Send(data []byte) {
    go c.sendAsync(data)  // 调用方无法等待或取消
}

// 推荐（✓）：同步函数，并发由调用方决定
func (c *Client) Send(data []byte) error {
    return c.sendSync(data)
}
// 调用方自己决定要不要并发：
// go client.Send(data)
```

## 4.5 Go 专项反模式

```go
// 不推荐（✗）：忽略错误，假装没发生
data, _ := os.ReadFile(path)
result, _ := strconv.Atoi(s)

// 推荐（✓）：错误必须处理或明确记录忽略原因
data, err := os.ReadFile(path)
if err != nil { return fmt.Errorf(...) }

// 如果确实要忽略，注释说明原因
n, _ = fmt.Fprintln(os.Stderr, msg) // best-effort
```

---

# 5. Rust 规范

## 5.1 命名约定

| 类别 | 格式 | 示例 | 说明 |
|------|------|------|------|
| 类型 / 枚举 | PascalCase | `PacketParser`, `NetError` | |
| 函数 / 方法 | snake_case | `parse_header()`, `send()` | |
| 变量 / 参数 | snake_case | `packet_len`, `retry_count` | |
| 常量 | UPPER_SNAKE | `MAX_RETRIES`, `HEADER_SIZE` | const / static |
| 生命周期 | 'short | `'a`, `'buf` | 尽量短且有意义 |
| 模块 | snake_case | `net::tcp`, `core::buffer` | |

## 5.2 错误处理

### 基本模式：Result<T, E> + ? 运算符

```rust
use std::io;
use std::fs;

#[derive(Debug)]
enum AppError {
    Io(io::Error),
    Parse(serde_json::Error),
}

impl From<io::Error>         for AppError { fn from(e: io::Error)         -> Self { AppError::Io(e) } }
impl From<serde_json::Error> for AppError { fn from(e: serde_json::Error) -> Self { AppError::Parse(e) } }

fn load_config(path: &str) -> Result<Config, AppError> {
    let data = fs::read_to_string(path)?;  // io::Error 自动转换
    let cfg  = serde_json::from_str(&data)?; // Parse error 自动转换
    Ok(cfg)
}
```

### unwrap / expect 的使用原则

- 测试代码中可以自由使用 `unwrap()`。
- 生产代码中：`expect("reason")` 优于 `unwrap()`——至少留下崩溃原因。
- 只在逻辑上"绝对不可能失败"时才用 expect，且注释说明理由。
- 禁止在可恢复错误路径上用 unwrap/expect 代替 `?` 传播。

```rust
// 好：expect 带有诊断信息
let listener = TcpListener::bind("0.0.0.0:8080")
    .expect("failed to bind port 8080 — is it already in use?");

// 坏：裸 unwrap，崩溃时无上下文
let listener = TcpListener::bind("0.0.0.0:8080").unwrap();
```

### 错误库选择

- 库 crate：实现 `std::error::Error`，不引入重量级依赖。
- 应用 crate：推荐 `anyhow`（简化错误传播）或 `thiserror`（derive 宏生成 Error 实现）。

## 5.3 所有权与借用规范

- 函数参数：只读传 `&T`，读写传 `&mut T`，需要所有权才传 `T`。
- 不要为了"绕过借用检查器"而引入 `Rc<RefCell<T>>`，先思考是否设计有问题。
- 生命周期标注：只在编译器无法推断时才显式标注，不过度标注。

```rust
// 参数所有权语义要明确
fn count_words(text: &str) -> usize { ... }        // 只读，借用
fn append_line(buf: &mut String, line: &str) { ... } // 读写，可变借用
fn consume_buffer(buf: Vec<u8>) -> Digest { ... }   // 转移所有权，需要消耗
```

## 5.4 Rust 专项反模式

```rust
// 不推荐（✗）：过度 clone 代替思考所有权
fn process(items: Vec<Item>) {
    let copy = items.clone(); // 不必要的深拷贝
    for item in items { use(item); }
}

// 推荐（✓）：传引用，避免不必要 clone
fn process(items: &[Item]) {
    for item in items { use(item); }
}
```

```rust
// 不推荐（✗）：到处 Arc<Mutex<T>>，变成带类型的全局变量
struct App { state: Arc<Mutex<State>> }
// 每处访问都要 .lock().unwrap()

// 推荐（✓）：明确所有权边界，只对真正共享的部分加锁
struct App {
    config: Config,           // 只读，不需要 Mutex
    cache: Arc<Mutex<Cache>>, // 只有 cache 被共享
}
```

### unsafe 使用规范

unsafe 是必要的逃生舱，但必须严格隔离和注释，绝不能污染公共 API。

```rust
// unsafe 必须：
// 1. 封装在小而独立的模块或函数中
// 2. 配有 SAFETY 注释，解释为何在此处是安全的
// 3. 公开 API 必须是 safe 的，unsafe 不泄漏到外部

pub fn split_at(slice: &[u8], mid: usize) -> (&[u8], &[u8]) {
    assert!(mid <= slice.len());
    // SAFETY: mid <= slice.len() 已由 assert 保证，
    //         两个子切片不重叠，指针有效。
    unsafe {
        (slice.get_unchecked(..mid),
         slice.get_unchecked(mid..))
    }
}
```

```rust
// 不推荐（✗）：unsafe 泄漏到公开接口，调用方无法安全使用
pub unsafe fn read_raw(ptr: *const u8, len: usize) -> &[u8] {
    std::slice::from_raw_parts(ptr, len)
}

// 推荐（✓）：封装 unsafe，对外暴露 safe 接口
pub fn read_checked(ptr: *const u8, len: usize) -> Option<&[u8]> {
    if ptr.is_null() || len == 0 { return None; }
    // SAFETY: 已检查非空，调用方保证 ptr+len 有效。
    Some(unsafe { std::slice::from_raw_parts(ptr, len) })
}
```

---

# 6. 测试与调试规范

## 6.1 测试策略

个人项目测试的核心原则：轻量但不缺席。两条铁律：

1. 修 bug 之前，先写能复现该 bug 的测试。修完后测试应当通过。
2. 测试函数名语义化，格式：`test_<被测函数>_<场景>`。

```c
// 不推荐（✗）：名字无意义
void test_buffer_1(void) { ... }

// 推荐（✓）：名字即文档
void test_buffer_append_exceeds_capacity(void) { ... }
```

### 各语言测试惯用法

```c
// C：最小化测试框架，用 assert 即可
void test_buffer_append_basic(void) {
    Buffer buf = {0};
    buffer_append(&buf, "hello", 5);
    assert(buf.length == 5);
    assert(memcmp(buf.data, "hello", 5) == 0);
    buffer_free(&buf);
}
```

```cpp
// C++：Google Test，TEST 宏组织用例
#include <gtest/gtest.h>

TEST(BufferTest, AppendExceedsCapacity) {
    Buffer buf;
    buf_init(&buf);
    for (int i = 0; i < CAPACITY + 1; i++) {
        buf_append(&buf, "x", 1);
    }
    EXPECT_TRUE(buf_overflow(&buf));
    buf_free(&buf);
}
```

```go
// Go：标准库 testing，表驱动测试
func TestParsePort(t *testing.T) {
    tests := []struct {
        input   string
        want    int
        wantErr bool
    }{
        {"8080", 8080, false},
        {"-1",   0,    true},
        {"99999",0,    true},
    }
    for _, tt := range tests {
        got, err := parsePort(tt.input)
        if (err != nil) != tt.wantErr {
            t.Errorf("parsePort(%q) error = %v, wantErr %v", tt.input, err, tt.wantErr)
        }
        if !tt.wantErr && got != tt.want {
            t.Errorf("parsePort(%q) = %d, want %d", tt.input, got, tt.want)
        }
    }
}
```

```rust
// Rust：内置测试模块
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_parse_packet_truncated_header() {
        let short_data = vec![0u8; HEADER_SIZE - 1];
        let result = parse_packet(&short_data);
        assert!(result.is_err());
    }
}
```

## 6.2 日志与调试输出

调试完不要删 log，把 printf 调试升级为结构化日志，下次直接复用。

| 语言 | 推荐方式 | 说明 |
|------|---------|------|
| C | `fprintf(stderr, ...)` | 调试输出到 stderr，不污染 stdout；用宏封装 DEBUG_LOG |
| C++ | `std::cerr` / spdlog | 生产代码用 spdlog，支持级别过滤和结构化输出 |
| Go | `log/slog` (1.21+) | 结构化日志，支持 JSON 输出；slog.Info/Warn/Error |
| Rust | `tracing` / `log` crate | tracing 支持 span 上下文，适合异步场景 |

```c
// C：DEBUG_LOG 宏，release 构建自动消除
#ifdef DEBUG
#  define DEBUG_LOG(fmt, ...) \
     fprintf(stderr, "[%s:%d] " fmt "\n", __FILE__, __LINE__, ##__VA_ARGS__)
#else
#  define DEBUG_LOG(fmt, ...) ((void)0)
#endif
```

```go
// Go：slog 结构化日志
slog.Info("packet received",
    "src", conn.RemoteAddr(),
    "size", len(data),
    "seq", pkt.SeqNum,
)
// 输出：time=... level=INFO msg="packet received" src=1.2.3.4:5678 size=64 seq=42
```

---

# 7. 项目结构规范

## 7.1 通用目录约定

```
project/
├── src/          — 源代码（C/C++）
├── include/      — 公开头文件（C/C++）
├── tests/        — 测试代码（与 src/ 并列）
├── docs/         — 设计文档、ADR（Architecture Decision Records）
├── scripts/      — 构建辅助脚本
└── README.md     — 项目说明、构建步骤、关键决策
```

> 💡 docs/ 里放设计决策（为什么这么做），不只是使用说明。三个月后"为什么"比"是什么"更难回忆。

## 7.2 C / C++ 项目结构

```
my_tool/
├── CMakeLists.txt
├── include/
│   └── my_tool/     — 公开头文件，带项目名前缀防止冲突
│       ├── buffer.h
│       └── editor.h
├── src/
│   ├── buffer.c
│   ├── editor.c
│   └── main.c
└── tests/
    ├── test_buffer.c
    └── test_editor.c
```

**CMake 最小模板（现代 CMake 风格）：**

```cmake
cmake_minimum_required(VERSION 3.20)
project(my_tool C)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

add_executable(my_tool
    src/main.c src/buffer.c src/editor.c)

target_include_directories(my_tool PRIVATE include)
target_compile_options(my_tool PRIVATE -Wall -Wextra -Wpedantic)

# 测试
enable_testing()
add_executable(test_buffer tests/test_buffer.c src/buffer.c)
target_include_directories(test_buffer PRIVATE include)
add_test(NAME buffer COMMAND test_buffer)
```

## 7.3 Go 项目结构

```
my_service/
├── cmd/
│   └── server/      — 可执行程序入口（main 包）
│       └── main.go
├── internal/        — 项目私有包，外部不可导入
│   ├── parser/
│   └── storage/
├── pkg/             — 可供外部复用的库包（谨慎放内容）
├── docs/
└── go.mod
```

> 💡 不要过早使用 pkg/——个人项目大部分代码放 internal/ 即可。pkg/ 是"我明确打算让别人用"的信号。

## 7.4 Rust 项目结构

```
# 单 crate 项目
my_tool/
├── Cargo.toml
├── src/
│   ├── main.rs      — 入口
│   ├── lib.rs       — 库根（如果同时作为库）
│   ├── parser.rs
│   └── buffer.rs
└── tests/           — 集成测试（与 src/ 并列）
    └── integration_test.rs

# 多 crate workspace（功能模块多时使用）
workspace/
├── Cargo.toml       — [workspace] 声明
├── crates/
│   ├── core/        — 核心逻辑，无外部依赖
│   ├── net/         — 网络层
│   └── cli/         — 命令行入口
└── docs/
```

**workspace Cargo.toml 最小配置：**

```toml
[workspace]
members = ["crates/core", "crates/net", "crates/cli"]
resolver = "2"

[workspace.dependencies]
# 统一管理依赖版本，避免各 crate 版本不一致
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
```

---

# 8. 快速参考卡

## 命名格式对比

| 类别 | C | C++ | Go | Rust |
|------|---|-----|----|------|
| 类型 | PascalCase | PascalCase | PascalCase | PascalCase |
| 函数 | snake_case | snake_case | camelCase / PascalCase | snake_case |
| 变量 | snake_case | snake_case\_ | camelCase | snake_case |
| 全局/静态 | g\_snake | kPascalCase | PascalCase | UPPER_SNAKE |
| 常量/宏 | UPPER_SNAKE | UPPER_SNAKE | PascalCase | UPPER_SNAKE |
| 错误处理 | return -1 / errno | expected\<T,E\> | error interface | Result\<T,E\> |

## 错误处理速查

| 语言 | 正常路径 | 错误路径 | 禁止 |
|------|---------|---------|------|
| C | `return 0;` | `return -1; perror(...)` | 忽略返回值 |
| C++ | `return value;` / `return Expected<v>` | `return Unexpected<e>;` / `throw` | 裸 new/delete 泄漏 |
| Go | `return val, nil` | `return nil, fmt.Errorf("%w", e)` | `_` 忽略 error |
| Rust | `Ok(value)` | `Err(e)` / `?` | unwrap 在生产路径 |

---

> **好代码的标准：** 三个月后的自己还能快速看懂，并且敢于修改。
