# libashka 设计规格说明书

**日期**: 2026-05-25
**状态**: 草案
**项目**: libashka — 将 Babashka 编译为可嵌入共享动态库

## 1. 概述

libashka 使用 GraalVM Native Image 将 Babashka 编译为 C 可链接的共享动态库。宿主应用（C、Zig、Rust、Swift、Go）通过简洁的 C ABI 嵌入完整的 Clojure/SCI 解释器，支持每线程独立 Isolate 执行上下文、双向宿主-脚本回调，以及完整的 Pod 功能。

### 1.1 输出产物

| 平台 | 共享库 |
|------|--------|
| Linux x86_64 / arm64 | `libbabashka.so` |
| macOS x86_64 / arm64 | `libbabashka.dylib` |
| Windows x86_64 | `libbabashka.dll` |

公开头文件：`include/libbabashka.h`

### 1.2 关键设计决策

- **集成方式**：Git submodule 引用 babashka（非 fork），完整继承其构建体系
- **Isolate 模型**：每线程独立 GraalVM Isolate。每个 Isolate 拥有私有的 SCI 上下文和宿主函数注册表，不跨 Isolate 共享状态
- **Isolate 生命周期**：C 宿主通过 `libashka_create_context` 创建 Isolate，后续每次调用都传入 `graal_isolatet_t*`。GraalVM 自动将 `@CEntryPoint` 调用路由到正确的 Isolate
- **数据交换**：通过 `cheshire.core` 以 JSON 字符串跨 FFI 边界传递
- **回调机制**：`@CFunctionPointer` + `@InvokeCFunctionPointer`。`libashka_register_host_fn(ctx, "log", "debug", fn)` 动态在 SCI 中注入 var，脚本直接使用自然语法 `(log/debug "hello")`。底层自动完成 JSON 序列化/反序列化
- **构建流程**：`babashka uberjar → native-image --shared → libbabashka.so`
- **Pod 支持**：完整保留 `babashka.pods` 命名空间及子进程管理，上下文销毁时强制清理子进程
- **能力模型**：宿主可通过 capability 位掩码在执行上下文级别限制 Pod、Shell、文件、网络访问

## 2. 目录结构

```
libashka/
├── babashka/                       ← git submodule（不修改）
├── deps.edn                        ← :local/root "babashka"
│
├── src/java/libashka/
│   ├── LibashkaStructs.java        ← @CStruct 映射 libashka_val_t
│   └── LibashkaAPI.java            ← @CEntryPoint 导出函数
│
├── src/clojure/libashka/
│   ├── core.clj                    ← SCI 上下文生命周期管理 + Pod 子进程追踪
│   ├── serialization.clj           ← JSON 序列化/反序列化（cheshire）
│   ├── callbacks.clj               ← 宿主回调注册与调用调度
│   └── build.clj                   ← native-image 构建脚本
│
├── include/
│   └── libbabashka.h               ← 公开 C ABI 头文件
│
├── test/
│   ├── clojure/
│   │   └── libashka/
│   │       ├── core_test.clj       ← 上下文创建/销毁/重置/子进程清理
│   │       ├── serialization_test.clj
│   │       ├── callbacks_test.clj
│   │       └── pod_test.clj        ← Pod 功能测试
│   └── integration/
│       ├── c/
│       │   ├── main.c              ← C 集成测试（含 Pod 场景）
│       │   └── Makefile
│       ├── zig/
│       │   ├── main.zig            ← Zig 集成测试（含 Pod 场景）
│       │   └── build.zig
│       └── rust/
│           ├── main.rs             ← Rust 集成测试（含 Pod 场景）
│           └── Cargo.toml
│
├── script/
│   └── build                       ← 构建入口：uberjar → native-image --shared
│
├── Makefile                        ← 顶层快捷命令
├── CLAUDE.md                       ← 项目说明 + TDD 协议
└── DESIGN.md                       ← 原始架构设计文档
```

### 2.1 文件职责说明

| 文件 | 职责 | 所属层 |
|------|------|--------|
| `LibashkaStructs.java` | GraalVM `@CStruct` 注解，将 `libashka_val_t` C 结构体映射为 Java 可操作的内存布局 | 类型系统 |
| `LibashkaAPI.java` | 9 个 `@CEntryPoint` 方法：上下文生命周期、上下文重置、能力限制创建、求值、回调注册、内存释放 | C ABI 边界 |
| `core.clj` | Isolate 本地的 SCI 上下文管理、能力过滤、Pod 子进程 PID 追踪与销毁清理 | 核心逻辑 |
| `serialization.clj` | JSON ↔ Clojure 数据结构转换、SCI 特殊类型安全处理、错误 JSON 包装 | 数据层 |
| `callbacks.clj` | 宿主函数注册表、C 函数指针 → SCI 可调用 fn 的映射、命名空间注入 | 双向调用 |
| `build.clj` | 调用 babashka 构建链 + `native-image --shared`，输出共享库 | 构建 |

## 3. C ABI 接口规范 (`libbabashka.h`)

### 3.1 类型定义

```c
/* 不透明上下文句柄 —— 封装 GraalVM graal_isolatet_t*。
   C 宿主在每次 API 调用中传入此指针，
   GraalVM 据此将调用路由到正确的 Isolate。 */
typedef struct libashka_ctx libashka_ctx_t;

/* 值类型标签 */
typedef enum {
    LIBASHKA_NIL     = 0,   /* 空值 */
    LIBASHKA_INT64   = 1,   /* 64 位有符号整数 */
    LIBASHKA_FLOAT64 = 2,   /* 64 位浮点数 */
    LIBASHKA_BOOLEAN = 3,   /* 布尔值 */
    LIBASHKA_STRING  = 4,   /* UTF-8 字符串 */
    LIBASHKA_JSON    = 5,   /* 复合类型序列化为 JSON */
    LIBASHKA_BINARY  = 6    /* 预留：二进制序列化（Transit / MessagePack） */
} libashka_type_t;

/* 带标签的联合体返回值 */
typedef struct {
    libashka_type_t type;
    union {
        int64_t  i64;       /* type == LIBASHKA_INT64 时有效 */
        double   f64;       /* type == LIBASHKA_FLOAT64 时有效 */
        int32_t  boolean;   /* type == LIBASHKA_BOOLEAN 时有效 */
        char    *string;    /* type == LIBASHKA_STRING 或 LIBASHKA_JSON 时有效 */
        struct {
            uint8_t *data;
            size_t   len;   /* type == LIBASHKA_BINARY 时有效 */
        } binary;
    } value;
} libashka_val_t;

/* 宿主回调函数签名：JSON 参数入，JSON 结果出 */
typedef char *(*libashka_host_fn_t)(const char *json_args);
```

### 3.2 能力位掩码

```c
/* 控制执行上下文可用功能的位掩码。
   宿主应用以此沙箱化不可信脚本。 */
typedef enum {
    LIBASHKA_CAP_POD         = 1 << 0,  /* Pod 加载与外部子进程 */
    LIBASHKA_CAP_SHELL       = 1 << 1,  /* shell/sh 执行 */
    LIBASHKA_CAP_FILE_READ   = 1 << 2,  /* 文件系统读取 */
    LIBASHKA_CAP_FILE_WRITE  = 1 << 3,  /* 文件系统写入 */
    LIBASHKA_CAP_NETWORK     = 1 << 4,  /* 网络访问（socket、HTTP） */
    LIBASHKA_CAP_ALL         = 0xFFFFFFFF
} libashka_capability_t;
```

### 3.3 API 函数清单

**生命周期管理：**
```c
/* 创建具备全部能力的 Isolate。
   返回的 Isolate thread 指针后续每次调用均需传入。
   失败返回 NULL。 */
libashka_ctx_t *libashka_create_context(void);

/* 创建受限能力的 Isolate。
   能力位掩码在 SCI 初始化时评估，上下文生命周期内不可更改。 */
libashka_ctx_t *libashka_create_context_with_capabilities(uint32_t capabilities);

/* 销毁 Isolate。销毁前执行以下清理步骤：
   1. 向此上下文创建的所有 Pod 子进程发送 SIGTERM
   2. 等待最多 3 秒，向未退出的进程发送 SIGKILL
   3. 对每个子进程调用 waitpid() 回收僵尸
   4. 释放所有关联内存
   传入 NULL 为无操作。 */
void            libashka_destroy_context(libashka_ctx_t *ctx);

/* 在现有 Isolate 内重置 SCI 状态，不销毁 Isolate。
   清空所有用户自定义 var、宿主函数注册、Pod 子进程
   （清理策略同 destroy_context），
   然后将 SCI 重新初始化到出厂状态。
   适用于上下文复用 / 池化场景。 */
void            libashka_reset_context(libashka_ctx_t *ctx);
```

**脚本求值：**
```c
/* 在当前上下文中求值 Clojure 表达式字符串。
   成功返回对应类型的 tagged union，失败返回 type=LIBASHKA_JSON 的 {"error": "..."}。 */
libashka_val_t  libashka_eval_string(libashka_ctx_t *ctx, const char *code);

/* 读取并求值 Clojure 源文件。
   返回文件中最后一个表达式的值。 */
libashka_val_t  libashka_eval_file(libashka_ctx_t *ctx, const char *path);
```

**双向调用：**
```c
/* 宿主 → 脚本：调用脚本命名空间中的函数。
   json_args 是 JSON 编码的参数数组，如 "[1, \"hello\"]"。 */
libashka_val_t  libashka_call_script_fn(libashka_ctx_t *ctx,
                                        const char *ns, const char *fn_name,
                                        const char *json_args);

/* 脚本 → 宿主：注册 C 函数供脚本调用。
   在 SCI 中动态创建命名空间并注入 var，脚本可直接使用自然语法调用。
   示例：
     libashka_register_host_fn(ctx, "log", "debug", my_logger);
   注册后脚本可直接写：
     (log/debug "hello")
   参数自动序列化为 JSON，C 函数接收 JSON 字符串并返回 JSON 字符串，
   返回值自动解析为 Clojure 数据。 */
void            libashka_register_host_fn(libashka_ctx_t *ctx,
                                          const char *ns, const char *fn_name,
                                          libashka_host_fn_t fn_ptr);
```

**内存管理：**
```c
/* 释放 libashka_val_t 中的堆分配内存。
   eval_string、eval_file、call_script_fn 返回的每个非 NIL 值必须调用一次此函数。 */
void            libashka_free_value(libashka_val_t *val);
```

### 3.4 设计原理

- **基本类型直接传值，复合类型走 JSON**：避免跨 FFI 边界的复杂结构体映射。int64/float64/bool/nil 直接在 union 中传递；vector、map、set 序列化为 JSON 字符串，标记为 `LIBASHKA_JSON` 类型。
- **LIBASHKA_BINARY（预留）**：为二进制序列化格式（Transit、MessagePack、Protocol Buffers）预留的类型标签。v1 不实现，但枚举值和 union 字段提前占位，确保 ABI 向后兼容。
- **字符串所有权归被调用方**：所有返回字符串通过 `UnmanagedMemory.malloc()` 在 C 堆上分配，不受 JVM GC 管理。调用者必须调用 `libashka_free_value` 释放，否则内存泄漏。
- **错误模型**：所有异常在内部捕获，以 `{"error": "..."}` JSON 格式返回。异常绝不跨 FFI 边界泄漏。
- **每线程 Isolate**：`libashka_ctx_t` 就是 GraalVM 自身 isolate 创建机制返回的 `graal_isolatet_t*`。每个 `@CEntryPoint` 将其作为 `IsolateThread` 参数接收，GraalVM 自动将执行路由到正确的 Isolate。无跨 Isolate 共享状态。
- **能力模型**：嵌入不可信脚本的宿主应用可在上下文创建时限制危险操作。默认 `libashka_create_context` 授予全部能力，保证向后兼容。

## 4. Java 层设计

### 4.1 `LibashkaStructs.java` — C 类型映射

使用 GraalVM 的 `@CStruct`、`@CFieldGroup`、`@CField` 注解，将 C 的 `libashka_val_t` 结构体映射为 Java 可直接操作的内存布局。包含 `binary` 字段组（`data` + `len`）以支持前向兼容。此文件为纯样板代码，不含业务逻辑。

### 4.2 `LibashkaAPI.java` — C 入口点

所有 `@CEntryPoint` 方法以 `IsolateThread` 作为第一个参数。GraalVM 使用此指针将调用路由到正确 Isolate 的堆和静态状态。

| 方法 | 对应 C 符号 | 职责 |
|------|------------|------|
| `createContext` | `libashka_create_context` | 创建 Isolate，以 `LIBASHKA_CAP_ALL` 初始化 SCI，返回 `IsolateThread` |
| `createContextWithCapabilities` | `libashka_create_context_with_capabilities` | 创建 Isolate，解析能力掩码，以受限命名空间初始化 SCI |
| `destroyContext` | `libashka_destroy_context` | 清理 Pod 子进程、销毁 SCI、分离 Isolate |
| `resetContext` | `libashka_reset_context` | 清理 Pod 子进程、清空 SCI 用户状态、重置到出厂状态 |
| `evalString` | `libashka_eval_string` | C 字符串 → Java → 委托 Clojure → JSON → `LibashkaVal` |
| `evalFile` | `libashka_eval_file` | 读文件，求值内容，返回结果 |
| `registerHostFn` | `libashka_register_host_fn` | 将 C 函数指针包装为 `@CFunctionPointer`，存入 Isolate 本地注册表 |
| `callScriptFn` | `libashka_call_script_fn` | 查找 SCI 命名空间中的 fn，解析 JSON 参数，调用，序列化返回 |
| `freeValue` | `libashka_free_value` | 对字符串或二进制字段调用 `UnmanagedMemory.free()` |

### 4.3 回调接口

```java
@CFunctionPointer
interface LibashkaHostFn extends PointerBase {
    @InvokeCFunctionPointer
    CCharPointer invoke(CCharPointer jsonArgs);
}
```

当脚本调用已注册的宿主函数时，SCI 触发 `LibashkaHostFn.invoke()`，GraalVM 通过 `@InvokeCFunctionPointer` 机制将调用路由回原始的 C 函数指针。

### 4.4 上下文存储（Isolate 本地）

每个 Isolate 拥有**私有的** `Map<String, Object>`，通过基于 `IsolateThread` 的 `ThreadLocal` 风格机制存储。不存在全局 `ConcurrentHashMap`——GraalVM 设计中每个 Isolate 的堆天然独立，因此 Java 的静态字段自然持有 Isolate 私有的值。

每个 Isolate 的状态对象包含：
- `sci.ctx` — SCI 解释器实例
- `host-fns` — `Atom<Map<String, LibashkaHostFn>>`，宿主回调注册表
- `pod-pids` — `Atom<Set<Long>>`，追踪该上下文创建的所有 Pod 子进程 PID
- `capabilities` — 上下文创建时的 `uint32_t` 能力位掩码

当 capability 限制 Pod 使用时（`LIBASHKA_CAP_POD` 缺失），`babashka.pods` 命名空间不注入 SCI，跳过 pod-pids 追踪。

### 4.5 错误处理

所有异常在 Java 层捕获，不允许向 C 侧泄漏：

```java
try {
    // ... eval 逻辑 ...
} catch (Exception e) {
    StringWriter sw = new StringWriter();
    e.printStackTrace(new PrintWriter(sw));
    return buildJsonError(sw.toString());
}
```

返回 `libashka_val_t`，type=`LIBASHKA_JSON`，value.string=`{"error": "..."}`。

## 5. Clojure 层设计

### 5.1 `libashka.core` — 上下文管理

核心命名空间，管理 Isolate/SCI 生命周期：

| 函数 | 参数 | 返回值 | 说明 |
|------|------|--------|------|
| `create-context` | `capabilities` | 上下文状态 map | 创建新 SCI 上下文，根据 capability 掩码选择性注入命名空间 |
| `eval-string` | `ctx, code` | JSON 字符串 | 调用 `sci/eval-string`，包装为 `{:result ...}` 或 `{:error ...}` |
| `destroy-context` | `ctx` | nil | (1) 遍历 `pod-pids` → SIGTERM → 3s → SIGKILL → waitpid；(2) 清空 SCI；(3) 返回 nil |
| `reset-context` | `ctx` | nil | 与 destroy 相同的 Pod 清理逻辑，然后清空所有用户 var 和宿主函数注册，重新初始化 SCI 命名空间到出厂状态 |

SCI 初始化根据 capability 掩码进行能力过滤：
- `LIBASHKA_CAP_POD` 缺失 → 省略 `babashka.pods`、`clojure.java.shell`
- `LIBASHKA_CAP_SHELL` 缺失 → 省略 shell 辅助命名空间
- `LIBASHKA_CAP_FILE_READ` / `LIBASHKA_CAP_FILE_WRITE` 缺失 → 限制 `clojure.java.io`
- `LIBASHKA_CAP_NETWORK` 缺失 → 省略 socket / HTTP 命名空间

### 5.2 `libashka.serialization` — JSON 序列化

| 函数 | 说明 |
|------|------|
| `->json [v]` | `cheshire.core/generate-string`，Clojure 值 → JSON 字符串 |
| `<-json [s]` | `cheshire.core/parse-string` + keyword keys，JSON 字符串 → Clojure 数据结构 |
| `->json-safe [v]` | 后序遍历值树，将不可序列化的 SCI 类型（deftype、proxy 实例）转为字符串表示 |

### 5.3 `libashka.callbacks` — 宿主函数注册与调度

将 C 函数指针桥接为 SCI 可调用的 var。

**`register-host-fn [ctx ns fn-name fn-ptr]`**：
1. 将 `LibashkaHostFn` C 函数指针包装为一个 Clojure 函数，该包装函数 (a) 将所有接收的参数序列化为 JSON 数组，(b) 调用 C 指针，(c) 将返回的 JSON 字符串解析为 Clojure 数据
2. 如果 SCI 命名空间 `ns` 不存在则自动创建
3. 在该命名空间中 intern 名为 `fn-name` 的 var，绑定到包装函数
4. 注册完成后，脚本直接使用自然 Clojure 语法调用宿主函数——无需显式 dispatch 字符串

**示例 — C 侧：**
```c
char *my_logger(const char *json_args) {
    printf("[HOST] %s\n", json_args);
    return strdup("true");
}
libashka_register_host_fn(ctx, "log", "debug", my_logger);
```

**示例 — 脚本侧：**
```clojure
(log/debug "hello")         ;; → C 接收 "[\"hello\"]"，返回 true
(log/debug "x" 42)          ;; → C 接收 "[\"x\",42]"
```

**`host/invoke` 底层原语** — 系统提供内置的 `host/invoke` 函数作为动态代理包装器内部使用的底层调度机制。也可直接调用以支持动态场景：
```clojure
(host/invoke "log" "debug" "hello")  ;; 等价于 (log/debug "hello")
```

**`unregister-host-fn [ctx ns fn-name]`** — 移除之前注册的宿主函数，un-intern 其 SCI var。

> 旧设计使用 `host.lib/invoke`，去掉 `lib` 子段以简化命名。系统内置命名空间现在统一为 `host`。

### 5.4 完整数据流

```
宿主 (C)                     Java (@CEntryPoint)               Clojure
───────                      ───────────────────               ───────
libashka_create_context  →   createContext()              →    core/create-context
                                  ↓                               ↓
                              创建 Isolate                    sci/init + cap 过滤
                                  ↓
                              返回 IsolateThread（即 ctx 句柄）

libashka_eval_string     →   evalString(thread, ...)      →    core/eval-string
                                  ↓                               ↓
                              路由到正确 Isolate              sci/eval-string
                                  ↓                               ↓
                              ← JSON 字符串                 ←    ser/->json
                                  ↓
                              构造 LibashkaVal → 返回给 C

libashka_register_host_fn → registerHostFn(thread, ...) →    cb/register-host-fn
                                  ↓                               ↓
                              路由到正确 Isolate              包装 fnPtr 为 wrapper fn
                                  ↓                               ↓
                              存储 LibashkaHostFn              在 SCI ns 中 intern var
                                                                   ↓
(脚本直接调用 (log/debug "hello")) → wrapper fn → .invoke(fnPtr) → C 宿主函数执行
                                        ↑
                                   参数序列化为 JSON
                                   ["hello"]
                                        ↓
                                   返回值解析为 Clojure 数据
                                   C 返回 "true" → true

libashka_destroy_context  →  destroyContext(thread)      →    core/destroy-context
                                  ↓                               ↓
                              1. SIGTERM → SIGKILL Pod 进程    遍历 pod-pids，批量清理
                              2. 等待所有子进程退出             waitpid 回收
                              3. 销毁 SCI 上下文
                              4. 分离 Isolate
```

## 6. 构建流程

### 6.1 三阶段构建

**阶段一：Uberjar（复用 babashka 构建基础设施）**

```
babashka/script/uberjar
  → Leiningen + feature flag profiles
  → reify2 字节码动态生成
  → 产出: babashka-standalone.jar
```

**阶段二：Native Image 编译为共享库**

```bash
native-image \
  -jar babashka-standalone.jar \
  -cp src/java:src/clojure \
  -H:Name=libbabashka \
  --shared \
  --no-fallback \
  -H:+JNI \
  -H:+ForeignAPISupport \
  -H:Path=target/ \
  --features=babashka.impl.CharsetsFeature \
  --allow-incomplete-classpath \
  --report-unsupported-elements-at-runtime \
  -H:IncludeResources=BABASHKA_VERSION \
  -H:IncludeResources=SCI_VERSION \
  [所有 babashka 原有的 native-image 参数...]

产出:
  target/libbabashka.so    (Linux)
  target/libbabashka.dylib (macOS)
  target/libbabashka.dll   (Windows)
```

**阶段三：集成测试编译**

```
C:    gcc -ldl test/integration/c/main.c -o test/c_runner
Zig:  zig build-exe test/integration/zig/main.zig
Rust: cargo build --manifest-path test/integration/rust/Cargo.toml
```

### 6.2 `script/build` 构建脚本

Bash 脚本编排阶段一和二：

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
PROJECT_ROOT="$(dirname "$SCRIPT_DIR")"
BABASHKA_DIR="$PROJECT_ROOT/babashka"

# 阶段 1: 构建 babashka uberjar
cd "$BABASHKA_DIR"
./script/uberjar
UBERJAR=$(ls -t target/babashka-*-standalone.jar | head -1)

# 阶段 2: Native Image 编译为共享库
cd "$PROJECT_ROOT"
native-image \
  -jar "$UBERJAR" \
  -cp "$PROJECT_ROOT/src/java:$PROJECT_ROOT/src/clojure" \
  -H:Name=libbabashka \
  --shared \
  --no-fallback \
  -H:Path="$PROJECT_ROOT/target" \
  [其余参数...]

echo "构建完成: target/libbabashka.$(uname -s | tr '[:upper:]' '[:lower:]' | sed 's/darwin/dylib/' | sed 's/linux/so/')"
```

### 6.3 关键环境变量

| 变量 | 作用 | 默认值 |
|------|------|--------|
| `BABASHKA_FEATURE_*` | 控制可选模块编译（xml、yaml、jdbc 等） | 全部启用 |
| `BABASHKA_DISABLE_SIGNAL_HANDLERS` | 禁止共享库接管宿主进程信号 | `true` |
| `GRAALVM_HOME` | GraalVM 安装路径 | 系统 PATH 中查找 |
| `BABASHKA_LEAN` | 禁用所有可选功能 | `false` |

### 6.4 顶层 `Makefile`

```makefile
.PHONY: build test test-clj test-c test-zig test-rust clean

build:
	./script/build

test: test-clj
	$(MAKE) -C test/integration/c test
	cd test/integration/zig && zig build test
	cd test/integration/rust && cargo test

test-clj:
	clojure -M:test

clean:
	rm -rf target/
```

## 7. 测试策略

### 7.1 三层测试金字塔

| 层 | 位置 | 范围 | 占比 |
|----|------|------|------|
| Clojure 单元测试 | `test/clojure/libashka/` | 上下文生命周期、序列化、回调、Pod 命名空间、能力过滤 | 55-60% |
| 集成测试 | `test/integration/{c,zig,rust}/` | dlopen 真实 .so，端到端场景 | 15-20% |
| Java 层 | 通过 Clojure + 集成测试间接验证 | @CEntryPoint 正确性 | 20-25% |

### 7.2 Clojure 单元测试（`test/clojure/libashka/`）

| 测试文件 | 测试内容 |
|----------|---------|
| `core_test.clj` | 带/不带 capability 的 `create-context`；`destroy-context` 的 Pod 清理；`reset-context`；`eval-string` 基本求值；异常捕获；多上下文隔离 |
| `serialization_test.clj` | Clojure ↔ JSON 双向转换；SCI 特殊类型安全处理；nil/数字/字符串/嵌套结构正确性 |
| `callbacks_test.clj` | 宿主函数注册；脚本调宿主流；参数传递；返回值回传；未注册函数错误处理 |
| `pod_test.clj` | `babashka.pods` 命名空间可用性；Pod 加载；Pod 子进程生命周期；destroy/reset 时的子进程清理；`LIBASHKA_CAP_POD` 缺失时的行为 |

### 7.3 集成测试（`test/integration/{c,zig,rust}/`）

每种语言的集成测试覆盖 11 个场景：

| # | 测试场景 | 验证点 |
|---|---------|--------|
| 1 | `test_create_destroy` | 上下文创建返回非 NULL，销毁不崩溃 |
| 2 | `test_create_with_capabilities` | 受限上下文创建，验证受限操作被拒绝 |
| 3 | `test_eval_simple` | `(+ 1 2)` 返回 `{"result": 3}` |
| 4 | `test_eval_error` | 语法错误返回 `{"error": "..."}` |
| 5 | `test_multi_contexts` | 两个独立上下文，变量互不污染 |
| 6 | `test_host_callback` | 注册 C 函数，脚本调用返回正确结果 |
| 7 | `test_call_script_fn` | 宿主调用脚本定义的函数 |
| 8 | `test_pod_load_and_cleanup` | 加载外部 Pod，调用函数，验证 destroy 后子进程被杀 |
| 9 | `test_reset_context` | 重置上下文，验证状态清空，验证 Pod 进程被杀 |
| 10 | `test_memory_leak` | 循环 create/destroy 100 次，内存不增长 |
| 11 | `test_thread_isolation` | 多线程各持独立 ctx，并发求值互不干扰 |

### 7.4 TDD 执行顺序

**第一阶段 — Clojure 层（纯 JVM，不依赖 native-image）：**

```
1. RED:   编写 core_test.clj → 失败（文件/函数不存在）
2. GREEN: 实现 libashka.core → 通过
3. RED:   编写 serialization_test.clj → 失败
4. GREEN: 实现 libashka.serialization → 通过
5. RED:   编写 callbacks_test.clj → 失败
6. GREEN: 实现 libashka.callbacks → 通过
7. RED:   编写 pod_test.clj → 失败
8. GREEN: 确保 babashka.pods 正确注入，Pod 子进程清理功能正常 → 通过
```

**第二阶段 — Java @CEntryPoint 层：**

- 在 JVM 上通过 Clojure 测试间接验证（调用相同的代码路径）
- 完整的 @CEntryPoint 验证需要 native-image 编译（第三阶段）

**第三阶段 — 集成测试（需先编译 .so）：**

```
1. BUILD: script/build → libbabashka.so
2. RED:   编写 main.c → 编译通过（header 存在），运行测试失败（.so 未链接）
3. GREEN: 编译 main.c + dlopen .so → 全部场景通过
4. 同样流程 for Zig, Rust
```

## 8. Pod 支持

### 8.1 需求

- `babashka.pods` 命名空间必须在所有 SCI 上下文中完整可用（除非被 capability 限制）
- Pod 通过 stdin/stdout 与父进程通信
- 共享库模式下，父进程即宿主应用，必须保留 Pod 子进程管理能力
- 外部 Pod 通过 `babashka.pods/load-pod` 加载必须正常工作

### 8.2 实现方案

- 完整继承 babashka 现有的 Pod 基础设施，不做修改
- 通过 native-image 参数启用进程管理：
  - `--allow-incomplete-classpath`
  - `--report-unsupported-elements-at-runtime`
- SCI 上下文初始化时在命名空间映射中保留 `babashka.pods`（除非 `LIBASHKA_CAP_POD` 缺失）

### 8.3 Pod 进程生命周期与清理

每个上下文创建的 Pod 子进程被记录在 **每个上下文的 PID 集合**（Isolate 本地状态中的 `pod-pids` atom）中。在 `destroy_context` 或 `reset_context` 时：

1. 遍历所有已追踪的 PID
2. 向每个子进程发送 `SIGTERM`
3. 等待最多 3 秒以便子进程优雅退出
4. 向未退出的进程发送 `SIGKILL`
5. 对每个 PID 调用 `waitpid()` 回收僵尸进程
6. 清空 PID 集合

这防止了长时间运行的宿主应用在反复创建和销毁上下文时积累孤儿/僵尸 Pod 进程。

## 9. 线程安全与隔离

### 9.1 每线程 Isolate 模型

每次调用 `libashka_create_context` 返回一个 GraalVM `IsolateThread` 指针：

- Isolate 之间**不共享任何可变状态**——各自拥有独立的堆、SCI 上下文和宿主函数注册表
- GraalVM 的 `--shared` 模式内置了隔离创建函数（`graal_create_isolate`），C 宿主调用这些函数
- Isolate 绑定到创建它的线程（GraalVM 约束）

### 9.2 实际工作原理

在 `--shared` 模式下，GraalVM 的 `@CEntryPoint` 机制工作原理：

1. 编译出的 `.so` 导出 `graal_create_isolate()`——C 宿主首次调用创建初始 Isolate
2. 每个 `@CEntryPoint` 方法的第一个参数 `IsolateThread` 就是 GraalVM 用来路由执行到正确 Isolate 堆和静态变量的指针
3. **不需要全局 ConcurrentHashMap 跨隔离区查找**——Java 的静态字段在 Isolate 间天然隔离。每个 Isolate 的静态 `SCIState` 自动就是该 Isolate 的正确状态
4. C 宿主将 `graal_isolatet_t*`（别名 `libashka_ctx_t*`）传入每次调用；GraalVM 透明切换上下文

### 9.3 约束

- Isolate 不可跨线程共享——每个线程必须创建自己的上下文
- GraalVM Isolate 创建有固定内存开销（每个约数 MB）
- 并发 Isolate 数量受可用内存限制
- 高频 create/destroy 场景请使用 `libashka_reset_context` 复用已有 Isolate

## 10. 依赖关系

### 10.1 构建时依赖

| 依赖 | 用途 | 版本要求 |
|------|------|---------|
| GraalVM | native-image 编译器 | JDK 21+，含 `native-image` |
| Leiningen | babashka uberjar 构建 | 继承自 babashka |
| Clojure CLI | deps.edn / tools.deps | 继承自 babashka |
| GCC | C 集成测试编译 | 任意 |
| Zig | Zig 集成测试编译 | 0.11+ |
| Rust | Rust 集成测试编译 | stable |

### 10.2 运行时依赖

**无。** 编译后的 `.so` / `.dylib` / `.dll` 是独立的原生二进制文件。宿主应用链接共享库即可，部署时不需要 JVM 或 Clojure 运行时。

### 10.3 deps.edn 配置

```clojure
{:paths ["src/clojure"]
 :deps {org.babashka/babashka {:local/root "babashka"}
        cheshire/cheshire {:mvn/version "5.13.0"}
        org.clojure/clojure {:mvn/version "1.12.0"}}
 :aliases {:build {:main-opts ["-m" "libashka.build"]}
           :test  {:extra-paths ["test/clojure"]
                   :extra-deps {io.github.cognitect-labs/test-runner
                                {:git/url "https://github.com/cognitect-labs/test-runner"
                                 :git/tag "v0.5.17"
                                 :git/sha "9be3aee"}}}}}
```

## 11. 风险与待确认事项

### 11.1 风险矩阵

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|---------|
| Babashka 上游构建变更破坏我们的流程 | 高 | 中 | Git submodule 锁定特定 commit；CI 自动检测 |
| GraalVM `--shared` 与 babashka 现有 native-image 参数冲突 | 中 | 低 | `libsci` 已验证 `--shared` 兼容性；早期测试 |
| Pod 子进程在共享库上下文中的行为差异 | 中 | 中 | 三种宿主语言的 Pod 集成测试全覆盖；PID 追踪 + 强制清理 |
| Isolate 内存开销不适合高并发场景 | 低 | 低 | 文档明确说明；提供 `reset_context` 复用 |
| Feature flag 组合爆炸 | 低 | 低 | 默认全开，capability 掩码裁剪危险操作，非构建参数 |
| 宿主未调用 `libashka_free_value` 导致内存泄漏 | 中 | 中 | 头文件明确所有权规则；valgrind/asan 集成测试 |
| 能力模型粒度过粗 | 低 | 低 | 初始 5 个粗粒度 cap；后续细粒度策略可在不破坏 ABI 的情况下追加 |

### 11.2 后续规划（本阶段不实现）

- GitHub Actions 矩阵构建（所有平台 × 架构组合）
- 预编译二进制发布（GitHub Releases）
- 语言专用 binding 库（`zig-libashka`、`rust-libashka` crate）
- Isolate 池化，支持可配置的池大小
- 二进制序列化后端（Transit、MessagePack）接入 `LIBASHKA_BINARY` 标签
- 细粒度能力策略（文件系统路径白名单、网络地址白名单）
