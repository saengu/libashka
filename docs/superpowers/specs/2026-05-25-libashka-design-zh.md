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
- **Isolate 模型**：每线程独立 GraalVM Isolate，通过 `ConcurrentHashMap` 管理
- **数据交换**：通过 `cheshire.core` 以 JSON 字符串跨 FFI 边界传递
- **回调机制**：`@CFunctionPointer` + `@InvokeCFunctionPointer`，JSON 进 / JSON 出
- **构建流程**：`babashka uberjar → native-image --shared → libbabashka.so`
- **Pod 支持**：完整保留 `babashka.pods` 命名空间及子进程管理能力

## 2. 目录结构

```
libashka/
├── babashka/                       ← git submodule（不修改）
├── deps.edn                        ← :local/root "babashka"
│
├── src/java/libashka/
│   ├── LibashkaStructs.java        ← @CStruct 映射 libashka_val_t
│   └── LibashkaAPI.java            ← @CEntryPoint 导出函数（6个核心 + free）
│
├── src/clojure/libashka/
│   ├── core.clj                    ← SCI 上下文生命周期管理
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
│   │       ├── core_test.clj       ← 上下文创建/销毁/求值
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
| `LibashkaAPI.java` | 7 个 `@CEntryPoint` 方法：上下文生命周期、求值、回调注册、内存释放 | C ABI 边界 |
| `core.clj` | 线程安全的上下文注册表、Isolate 创建/销毁、SCI 初始化 | 核心逻辑 |
| `serialization.clj` | JSON ↔ Clojure 数据结构转换、SCI 特殊类型安全处理、错误 JSON 包装 | 数据层 |
| `callbacks.clj` | 宿主函数注册表、C 函数指针 → SCI 可调用 fn 的映射、命名空间注入 | 双向调用 |
| `build.clj` | 调用 babashka 构建链 + `native-image --shared`，输出共享库 | 构建 |

## 3. C ABI 接口规范 (`libbabashka.h`)

### 3.1 类型定义

```c
/* 不透明上下文句柄 —— 每线程/Isolate 一个 */
typedef struct libashka_ctx libashka_ctx_t;

/* 值类型标签 */
typedef enum {
    LIBASHKA_NIL     = 0,   /* 空值 */
    LIBASHKA_INT64   = 1,   /* 64 位有符号整数 */
    LIBASHKA_FLOAT64 = 2,   /* 64 位浮点数 */
    LIBASHKA_BOOLEAN = 3,   /* 布尔值 */
    LIBASHKA_STRING  = 4,   /* UTF-8 字符串 */
    LIBASHKA_JSON    = 5    /* 复合类型（vector/map/set）序列化为 JSON */
} libashka_type_t;

/* 带标签的联合体返回值 */
typedef struct {
    libashka_type_t type;
    union {
        int64_t  i64;       /* type == LIBASHKA_INT64 时有效 */
        double   f64;       /* type == LIBASHKA_FLOAT64 时有效 */
        int32_t  boolean;   /* type == LIBASHKA_BOOLEAN 时有效 */
        char    *string;    /* type == LIBASHKA_STRING 或 LIBASHKA_JSON 时有效 */
    } value;
} libashka_val_t;

/* 宿主回调函数签名：JSON 参数入，JSON 结果出 */
typedef char *(*libashka_host_fn_t)(const char *json_args);
```

### 3.2 API 函数清单

**生命周期管理：**
```c
libashka_ctx_t *libashka_create_context(void);
void            libashka_destroy_context(libashka_ctx_t *ctx);
```
- `create_context`：创建新的 GraalVM Isolate 和 SCI 上下文，返回不透明句柄。失败返回 NULL。
- `destroy_context`：销毁 Isolate，释放所有关联资源。传入 NULL 为无操作。

**脚本求值：**
```c
libashka_val_t  libashka_eval_string(libashka_ctx_t *ctx, const char *code);
libashka_val_t  libashka_eval_file(libashka_ctx_t *ctx, const char *path);
```
- `eval_string`：在当前上下文中求值 Clojure 表达式字符串。成功返回 `{"result": <value>}` 的 JSON，失败返回 `{"error": "<stacktrace>"}`。
- `eval_file`：读取文件内容并求值。文件不存在时返回错误 JSON。

**双向调用：**
```c
/* 宿主 → 脚本：调用脚本中已定义的函数 */
libashka_val_t  libashka_call_script_fn(libashka_ctx_t *ctx,
                                        const char *ns,
                                        const char *fn_name,
                                        const char *json_args);

/* 脚本 → 宿主：注册宿主函数供脚本调用 */
void            libashka_register_host_fn(libashka_ctx_t *ctx,
                                          const char *ns,
                                          const char *fn_name,
                                          libashka_host_fn_t fn_ptr);
```
- `call_script_fn`：在指定命名空间中查找函数，解析 JSON 参数，调用并序列化返回结果。
- `register_host_fn`：将 C 函数指针注册到指定上下文中，脚本可通过 `(ns/fn-name json-str)` 调用。

**内存管理：**
```c
void            libashka_free_value(libashka_val_t *val);
```
- 释放 `libashka_val_t` 中 `string` 字段所占用的 C 堆内存。对非字符串类型为无操作。
- **规则**：每次 `eval_string`、`eval_file`、`call_script_fn` 返回的非 NIL 值都必须调用此函数释放。

### 3.3 设计原理

- **基本类型直接传值，复合类型走 JSON**：避免跨 FFI 边界的复杂结构体映射。int64/float64/bool/nil 直接在 union 中传递；vector、map、set 序列化为 JSON 字符串，标记为 `LIBASHKA_JSON` 类型。
- **字符串所有权归被调用方**：所有返回字符串通过 `UnmanagedMemory.malloc()` 在 C 堆上分配，不受 JVM GC 管理。调用者必须调用 `libashka_free_value` 释放，否则内存泄漏。
- **错误模型**：所有异常在内部捕获，以 `{"error": "..."}` JSON 格式返回。异常绝不跨 FFI 边界泄漏。
- **每线程 Isolate**：每个 `libashka_ctx_t` 封装一个 `graal_isolatet_t*`，绑定到创建它的线程。上下文间完全隔离，无共享可变状态。

## 4. Java 层设计

### 4.1 `LibashkaStructs.java` — C 类型映射

使用 GraalVM 的 `@CStruct` 和 `@CFieldGroup` 注解，将 C 的 `libashka_val_t` 结构体映射为 Java 可直接操作的内存布局。此文件为纯样板代码，不含业务逻辑。

核心注解：
```java
@CStruct("libashka_val_t")
interface LibashkaVal extends PointerBase {
    @CField("type")   int  getType();
    @CField("type")   void setType(int t);
    @CField("value")  ValueUnion getValue();
}

@CFieldGroup("value")
interface ValueUnion extends PointerBase {
    @CField("i64")    long getI64();
    @CField("i64")    void setI64(long v);
    @CField("f64")    double getF64();
    @CField("f64")    void setF64(double v);
    @CField("string") CCharPointer getString();
    @CField("string") void setString(CCharPointer v);
}
```

### 4.2 `LibashkaAPI.java` — C 入口点

7 个 `@CEntryPoint` 方法，每个标注对应的 C 符号名。所有方法以 `IsolateThread` 作为第一个参数（GraalVM 要求）。

| 方法 | 对应 C 符号 | 职责 |
|------|------------|------|
| `createContext` | `libashka_create_context` | 分配 Isolate，初始化 SCI，返回上下文句柄 |
| `destroyContext` | `libashka_destroy_context` | 销毁 SCI，销毁 Isolate，释放资源 |
| `evalString` | `libashka_eval_string` | C 字符串 → Java → 委托 Clojure → JSON → `LibashkaVal` |
| `evalFile` | `libashka_eval_file` | 读文件，求值内容，返回结果 |
| `registerHostFn` | `libashka_register_host_fn` | 将 C 函数指针包装为 `@CFunctionPointer`，存入注册表 |
| `callScriptFn` | `libashka_call_script_fn` | 查找 SCI 命名空间中的 fn，解析 JSON 参数，调用，序列化返回 |
| `freeValue` | `libashka_free_value` | 对字符串字段调用 `UnmanagedMemory.free()` |

### 4.3 回调接口

```java
@CFunctionPointer
interface LibashkaHostFn extends PointerBase {
    @InvokeCFunctionPointer
    CCharPointer invoke(CCharPointer jsonArgs);
}
```

当脚本调用已注册的宿主函数时，SCI 触发 `LibashkaHostFn.invoke()`，GraalVM 通过 `@InvokeCFunctionPointer` 机制将调用路由回原始的 C 函数指针。

### 4.4 上下文存储

```java
private static final ConcurrentHashMap<Integer, SCIState> contexts = new ConcurrentHashMap<>();
private static final AtomicInteger ctxCounter = new AtomicInteger(0);
```

- `ConcurrentHashMap` 保证构造时即线程安全
- `AtomicInteger` 生成无重复的整数 ID，强制转换为 `libashka_ctx_t*` 返回
- 每个条目存储：`sci.ctx` 实例、宿主函数注册表 atom、Isolate 引用

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
| `create-context` | 无 | `int` 上下文 ID | 创建新 SCI 上下文，注册到全局表 |
| `eval-string` | `ctx-id, code` | JSON 字符串 | 在当前上下文中求值，包装为 `{:result ...}` 或 `{:error ...}` |
| `destroy-context` | `ctx-id` | nil | 从注册表移除，允许 Isolate GC |
| `ctx-registry` | — | `ConcurrentHashMap` | 全局上下文注册表（defonce） |
| `ctx-counter` | — | `AtomicInteger` | 全局 ID 计数器 |

SCI 初始化包含所有 babashka 内置命名空间，确保完整功能对等（包括 `babashka.pods`）。

### 5.2 `libashka.serialization` — JSON 序列化

| 函数 | 说明 |
|------|------|
| `->json [v]` | `cheshire.core/generate-string`，Clojure 值 → JSON 字符串 |
| `<-json [s]` | `cheshire.core/parse-string` + keyword keys，JSON 字符串 → Clojure 数据结构 |
| `->json-safe [v]` | 后序遍历值树，将不可序列化的 SCI 类型（deftype、proxy 实例）转为字符串表示 |

### 5.3 `libashka.callbacks` — 宿主函数注册与调度

| 函数 | 说明 |
|------|------|
| `register-host-fn [ctx-id ns fn-name fn-ptr]` | 将 `LibashkaHostFn` 指针存入上下文的宿主函数注册表 atom |
| `host-fn-namespaces [ctx-id]` | 返回 SCI 命名空间定义，注入 `host.lib/invoke` 供脚本调用 |

脚本侧调用流程：
```clojure
;; 脚本中调用宿主函数
(host.lib/invoke "my.ns" "my-fn" "{\"key\": \"value\"}")
  → (查找注册表) → LibashkaHostFn.invoke(json) → C 宿主函数
```

### 5.4 完整数据流

```
宿主 (C)                     Java (@CEntryPoint)               Clojure
───────                      ───────────────────               ───────
libashka_create_context  →   createContext()              →    core/create-context
                                  ↓                               ↓
                              创建 Isolate                    sci/init + ctx-registry
                                  ↓
                              返回 ctx 句柄（int → libashka_ctx_t*）

libashka_eval_string     →   evalString()                 →    core/eval-string
                                  ↓                               ↓
                              C string → Java String           sci/eval-string
                                  ↓                               ↓
                              ← JSON 字符串                 ←    ser/->json
                                  ↓
                              构造 LibashkaVal → 返回给 C

libashka_register_host_fn → registerHostFn()             →    cb/register-host-fn
                                  ↓                               ↓
                              包装 fnPtr 为 LibashkaHostFn      存入 ctx 的 :host-fns atom
                                  ↓                               ↓
                              返回 void                     注入 SCI 命名空间 host.lib/invoke
                                                                  ↓
(脚本中调用 host.lib/invoke)  → (SCI 触发) → .invoke(fnPtr) →  C 宿主函数执行
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
Zig:  zig build-exe test/integration/zig/main.zig -l babashka
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

```
        ┌─────────────┐
        │  集成测试     │  ← C / Zig / Rust，dlopen 真实 .so
        │  (15-20%)    │     含 Pod 端到端场景
        ├─────────────┤
        │  Java 层测试  │  ← 通过 Clojure 测试 + 集成测试间接验证
        │  (20-25%)    │     @CEntryPoint 的正确性
        ├─────────────┤
        │  Clojure 测试 │  ← 核心逻辑：上下文管理、序列化、回调
        │  (55-60%)    │     纯 JVM，无需 native-image 编译
        └─────────────┘
```

### 7.2 Clojure 单元测试（`test/clojure/libashka/`）

| 测试文件 | 测试内容 |
|----------|---------|
| `core_test.clj` | `create-context` / `destroy-context` 生命周期；`eval-string` 基本求值；异常捕获为错误 JSON；多上下文隔离验证 |
| `serialization_test.clj` | Clojure ↔ JSON 双向转换；SCI 特殊类型安全处理；nil/数字/字符串/嵌套结构正确性 |
| `callbacks_test.clj` | 宿主函数注册；脚本调宿主流；参数传递；返回值回传；未注册函数错误处理 |
| `pod_test.clj` | `babashka.pods` 命名空间可用性；Pod 加载；Pod 子进程生命周期 |

### 7.3 集成测试（`test/integration/{c,zig,rust}/`）

每种语言的集成测试覆盖相同 10 个场景：

| # | 测试场景 | 验证点 |
|---|---------|--------|
| 1 | `test_create_destroy` | 上下文创建返回非 NULL，销毁不崩溃 |
| 2 | `test_eval_simple` | `(+ 1 2)` 返回 `{"result": 3}` |
| 3 | `test_eval_error` | 语法错误返回 `{"error": "..."}` |
| 4 | `test_multi_contexts` | 两个独立上下文，变量互不污染 |
| 5 | `test_host_callback` | 注册 C 函数，脚本调用返回正确结果 |
| 6 | `test_call_script_fn` | 宿主调用脚本定义的函数 |
| 7 | `test_pod_load` | 加载外部 Pod，调用 Pod 函数 |
| 8 | `test_pod_subprocess` | Pod 启动子进程并正常通信 |
| 9 | `test_memory_leak` | 循环 create/destroy 100 次，内存不增长 |
| 10 | `test_thread_isolation` | 多线程各持独立 ctx，并发求值互不干扰 |

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
8. GREEN: 确保 babashka.pods 命名空间正确注入 → 通过
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

- `babashka.pods` 命名空间必须在所有 SCI 上下文中完整可用
- Pod 通过 stdin/stdout 与父进程通信
- 共享库模式下，父进程即宿主应用，必须保留 Pod 子进程管理能力
- 外部 Pod 通过 `babashka.pods/load-pod` 加载必须正常工作

### 8.2 实现方案

- 完整继承 babashka 现有的 Pod 基础设施，不做修改
- 通过 native-image 参数启用进程管理：
  - `--allow-incomplete-classpath`
  - `--report-unsupported-elements-at-runtime`
- SCI 上下文初始化时在命名空间映射中保留 `babashka.pods`
- Pod 子进程继承宿主进程环境变量，无需特殊隔离

## 9. 线程安全与隔离

### 9.1 每线程 Isolate 模型

每次调用 `libashka_create_context` 创建一个新的 GraalVM Isolate：

- Isolate 之间不共享任何可变状态
- 每个 Isolate 拥有独立的 SCI 上下文、堆内存和宿主函数注册表
- Isolate 绑定到创建它的线程（GraalVM 约束）

### 9.2 线程安全保证

| 组件 | 机制 | 保证 |
|------|------|------|
| 上下文注册表 | `ConcurrentHashMap` | 多线程并发 `createContext` / `destroyContext` 安全 |
| 上下文 ID 生成 | `AtomicInteger` | 无重复 ID |
| 宿主函数注册表 | 每上下文独立的 `atom` | 每 Isolate 内无锁变更 |
| 全局状态 | 无（注册表除外） | 无竞争条件 |

### 9.3 约束

- Isolate 不可跨线程共享 —— 每个线程必须创建自己的上下文
- GraalVM Isolate 创建有固定内存开销（每个约数 MB）
- 并发 Isolate 数量受可用内存限制

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
| Pod 子进程在共享库上下文中的行为差异 | 中 | 中 | 三种宿主语言的 Pod 集成测试全覆盖 |
| Isolate 内存开销不适合高并发场景 | 低 | 低 | 文档明确说明约束；按需创建 Isolate |
| Feature flag 组合爆炸 | 低 | 低 | 默认全开，按需裁剪 |

### 11.2 后续规划（本阶段不实现）

- GitHub Actions 矩阵构建（所有平台 × 架构组合）
- 预编译二进制发布（GitHub Releases）
- 语言专用 binding 库（`zig-libashka`、`rust-libashka` crate）
- Isolate 池化（降低高频 create/destroy 场景的分配开销）
- 性能基准测试套件
