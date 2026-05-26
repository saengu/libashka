# libashka Design Specification

**Date**: 2026-05-25
**Status**: Draft
**Project**: libashka — Babashka as an embeddable shared library

## 1. Overview

libashka compiles Babashka into a C-linkable shared dynamic library using GraalVM Native Image. Host applications (C, Zig, Rust, Swift, Go) embed a full Clojure/SCI interpreter via a clean C ABI, with per-thread Isolate-based execution contexts, bidirectional host-script callbacks, and full Pod support.

### 1.1 Output Artifacts

| Platform | Shared Library |
|----------|---------------|
| Linux x86_64 / arm64 | `libbabashka.so` |
| macOS x86_64 / arm64 | `libbabashka.dylib` |
| Windows x86_64 | `libbabashka.dll` |

Public header: `include/libbabashka.h`

### 1.2 Key Design Decisions

- **Integration**: Git submodule of babashka (not a fork), inheriting its entire build pipeline
- **Isolate model**: Per-thread independent GraalVM Isolates. Each Isolate owns a private SCI context and host-fn registry — no cross-Isolate shared state
- **Isolate lifecycle**: C host creates Isolates via `libashka_create_context`, passes `graal_isolatet_t*` to every subsequent call. GraalVM routes each `@CEntryPoint` invocation to the correct Isolate automatically
- **Data exchange**: JSON strings across the FFI boundary via `cheshire.core`
- **Callbacks**: `@CFunctionPointer` + `@InvokeCFunctionPointer`. `libashka_register_host_fn(ctx, "log", "debug", fn)` dynamically interns a SCI var so scripts use natural syntax `(log/debug "hello")`. JSON-in / JSON-out under the hood.
- **Build flow**: `babashka uberjar → native-image --shared → libbabashka.so`
- **Pod support**: Full — preserves `babashka.pods` namespace and subprocess management, with mandatory child-process cleanup on context destroy

## 2. Directory Structure

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
│   │       ├── core_test.clj       ← 上下文创建/销毁/求值/子进程清理
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

## 3. C ABI Specification (`libbabashka.h`)

### 3.1 Types

```c
/* Opaque context handle — wraps a GraalVM graal_isolatet_t*.
   The C host passes this pointer to every API call so that
   GraalVM can route the invocation to the correct Isolate. */
typedef struct libashka_ctx libashka_ctx_t;

/* Value type tags */
typedef enum {
    LIBASHKA_NIL     = 0,
    LIBASHKA_INT64   = 1,
    LIBASHKA_FLOAT64 = 2,
    LIBASHKA_BOOLEAN = 3,
    LIBASHKA_STRING  = 4,
    LIBASHKA_JSON    = 5,   /* Composite types serialized as JSON */
    LIBASHKA_BINARY  = 6    /* Reserved: binary serialization (Transit, MessagePack) */
} libashka_type_t;

/* Tagged union for return values */
typedef struct {
    libashka_type_t type;
    union {
        int64_t  i64;
        double   f64;
        int32_t  boolean;
        char    *string;    /* Caller must free via libashka_free_value */
        struct {
            uint8_t *data;
            size_t   len;   /* Caller must free via libashka_free_value */
        } binary;
    } value;
} libashka_val_t;

/* Host callback signature: JSON args in, JSON result out */
typedef char *(*libashka_host_fn_t)(const char *json_args);
```

### 3.2 Capability Flags

```c
/* Bitmask controlling which features are available to a context.
   Host applications use these to sandbox untrusted scripts. */
typedef enum {
    LIBASHKA_CAP_POD         = 1 << 0,  /* Pod loading and external subprocesses */
    LIBASHKA_CAP_SHELL       = 1 << 1,  /* shell/sh execution */
    LIBASHKA_CAP_FILE_READ   = 1 << 2,  /* File system reads */
    LIBASHKA_CAP_FILE_WRITE  = 1 << 3,  /* File system writes */
    LIBASHKA_CAP_NETWORK     = 1 << 4,  /* Network access (sockets, HTTP) */
    LIBASHKA_CAP_ALL         = 0xFFFFFFFF
} libashka_capability_t;
```

### 3.3 API Functions

**Lifecycle:**
```c
/* Create a new Isolate with all capabilities enabled.
   Returns the Isolate thread pointer that must be passed to every
   subsequent call targeting this context. Returns NULL on failure. */
libashka_ctx_t *libashka_create_context(void);

/* Create a new Isolate with restricted capabilities.
   The capabilities bitmask is evaluated at SCI init time and
   cannot be changed for the lifetime of the context. */
libashka_ctx_t *libashka_create_context_with_capabilities(uint32_t capabilities);

/* Destroy the Isolate. Before teardown this function:
   1. Sends SIGTERM to all Pod subprocesses spawned by this context
   2. Waits up to 3 seconds, then sends SIGKILL to any survivors
   3. Calls waitpid() to reap every child
   4. Frees all associated memory
   Passing NULL is a no-op. */
void            libashka_destroy_context(libashka_ctx_t *ctx);

/* Reset the SCI state within an existing Isolate without destroying it.
   Clears all user-defined vars, host-fn registrations, and Pod subprocesses
   (killed with the same policy as destroy_context), then re-initialises
   SCI to its factory state. Useful for context reuse / pooling. */
void            libashka_reset_context(libashka_ctx_t *ctx);
```

**Evaluation:**
```c
/* Evaluate a Clojure expression string.
   Returns a tagged union. On success the value carries the result;
   on error the value carries a LIBASHKA_JSON string {"error": "..."}. */
libashka_val_t  libashka_eval_string(libashka_ctx_t *ctx, const char *code);

/* Read and evaluate a Clojure source file.
   Returns the value of the last expression in the file. */
libashka_val_t  libashka_eval_file(libashka_ctx_t *ctx, const char *path);
```

**Bidirectional calls:**
```c
/* Host → Script: call a function defined in a script namespace.
   json_args is a JSON-encoded argument vector, e.g. "[1, \"hello\"]". */
libashka_val_t  libashka_call_script_fn(libashka_ctx_t *ctx,
                                        const char *ns, const char *fn_name,
                                        const char *json_args);

/* Script → Host: register a C function so that scripts can call it.
   Dynamically creates a SCI var in the given namespace so scripts
   can call it with natural Clojure syntax. Example:
     libashka_register_host_fn(ctx, "log", "debug", my_logger);
   Afterwards the script can write:
     (log/debug "hello")
   Arguments are serialized to JSON, the C function receives a JSON
   string and returns a JSON string that is parsed back into Clojure. */
void            libashka_register_host_fn(libashka_ctx_t *ctx,
                                          const char *ns, const char *fn_name,
                                          libashka_host_fn_t fn_ptr);
```

**Memory management:**
```c
/* Free any heap-allocated memory inside a libashka_val_t.
   Must be called once for every non-NIL value returned by
   eval_string, eval_file, and call_script_fn. */
void            libashka_free_value(libashka_val_t *val);
```

### 3.4 Design Rationale

- **Tagged union for primitives, JSON for composites**: Avoids complex structure mapping across FFI. int64/float64/bool/nil pass directly; vectors, maps, sets serialize to JSON strings with `LIBASHKA_JSON` tag.
- **LIBASHKA_BINARY (future)**: Reserved tag for binary serialization formats (Transit, MessagePack, Protocol Buffers). No implementation in v1, but the enum value and union field are carved out now so the ABI does not change later.
- **String ownership**: All returned strings are allocated on the C heap via `UnmanagedMemory.malloc()`. The caller must call `libashka_free_value` to release.
- **Error model**: All errors caught internally, returned as `{"error": "stacktrace..."}` JSON via `LIBASHKA_JSON` type. No exceptions leak across the FFI boundary.
- **Per-thread Isolate**: `libashka_ctx_t` is the `graal_isolatet_t*` returned by GraalVM's own isolate-creation machinery. Every `@CEntryPoint` receives it as `IsolateThread`, and GraalVM routes execution to the correct Isolate automatically. No cross-Isolate shared state.
- **Capability model**: Host applications that embed untrusted scripts can restrict dangerous operations at context-creation time. Default (`libashka_create_context`) grants all capabilities for backward-compatibility convenience.

## 4. Java Layer

### 4.1 `LibashkaStructs.java` — C Type Mappings

Uses GraalVM `@CStruct`, `@CFieldGroup`, and `@CField` annotations to map the `libashka_val_t` C struct into Java-accessible memory layouts. This file is pure boilerplate — no business logic. Includes the `binary` field group (`data` + `len`) for forward compatibility.

### 4.2 `LibashkaAPI.java` — C Entry Points

All `@CEntryPoint` methods receive `IsolateThread` as the first parameter. GraalVM uses this pointer to route the call to the correct Isolate's heap and static state.

| Method | C Symbol | Responsibility |
|--------|----------|---------------|
| `createContext` | `libashka_create_context` | Create Isolate, init SCI with `LIBASHKA_CAP_ALL`, return `IsolateThread` |
| `createContextWithCapabilities` | `libashka_create_context_with_capabilities` | Create Isolate, parse capability mask, init SCI with restricted namespaces |
| `destroyContext` | `libashka_destroy_context` | Kill Pod subprocesses, tear down SCI, detach Isolate |
| `resetContext` | `libashka_reset_context` | Kill Pod subprocesses, clear SCI user state, re-init to factory state |
| `evalString` | `libashka_eval_string` | C string → Java → delegate to Clojure → JSON → `LibashkaVal` |
| `evalFile` | `libashka_eval_file` | Read file, eval contents, return result |
| `registerHostFn` | `libashka_register_host_fn` | Wrap C fn pointer as `@CFunctionPointer`, dynamically intern a SCI var so scripts call it natively (e.g. `(log/debug "msg")`) |
| `callScriptFn` | `libashka_call_script_fn` | Resolve SCI ns/fn, parse JSON args, call, serialize result |
| `unregisterHostFn` | — (internal) | Remove a previously registered host function, un-intern its SCI var |
| `freeValue` | `libashka_free_value` | `UnmanagedMemory.free()` on string or binary fields |

### 4.3 Callback Interface

```java
@CFunctionPointer
interface LibashkaHostFn extends PointerBase {
    @InvokeCFunctionPointer
    CCharPointer invoke(CCharPointer jsonArgs);
}
```

When a script calls a registered host function, SCI invokes `LibashkaHostFn.invoke()`, which GraalVM routes to the original C function pointer via `@InvokeCFunctionPointer`.

### 4.4 Context Storage (Isolate-Local)

Each Isolate owns a **private** `Map<String, Object>` stored via a `ThreadLocal`-style mechanism backed by `IsolateThread`. There is no global `ConcurrentHashMap` — every Isolate's heap is independent by GraalVM design, so static fields naturally hold Isolate-private values.

The per-Isolate state object contains:
- `sci.ctx` — the SCI interpreter instance
- `host-fns` — an `Atom<Map<String, LibashkaHostFn>>` for host callback registration
- `pod-pids` — an `Atom<Set<Long>>` tracking PIDs of spawned Pod subprocesses
- `capabilities` — the `uint32_t` capability bitmask this context was created with

When capabilities restrict Pod usage (`LIBASHKA_CAP_POD` absent), `babashka.pods` namespace is omitted from SCI init and pod-pids tracking is disabled.

### 4.5 Error Handling

All exceptions caught at the Java layer:
```java
try {
    // ... eval logic ...
} catch (Exception e) {
    StringWriter sw = new StringWriter();
    e.printStackTrace(new PrintWriter(sw));
    return buildJsonError(sw.toString());
}
```
Returned as `libashka_val_t` with type `LIBASHKA_JSON` and value `{"error": "..."}`.

## 5. Clojure Layer

### 5.1 `libashka.core` — Context Management

Core namespace handling Isolate/SCI lifecycle:

- `create-context [capabilities]` → Creates new SCI context via `sci/init`, selectively including namespaces based on the capability mask. Returns the context state map.
- `eval-string [ctx code]` → Calls `sci/eval-string` within the context, wraps result in `{:result ...}` JSON. Catches all exceptions and returns `{:error ...}` JSON.
- `destroy-context [ctx]` → (1) Iterates `pod-pids` atom, sends SIGTERM → 3s wait → SIGKILL → `waitpid` for each PID. (2) Clears SCI context. (3) Returns nil.
- `reset-context [ctx]` → Same Pod cleanup as destroy, then clears all user-defined vars and host-fn registrations, re-initialises SCI namespaces to factory state.

SCI initialization respects the capability mask:
- `LIBASHKA_CAP_POD` absent → omit `babashka.pods`, `clojure.java.shell`
- `LIBASHKA_CAP_SHELL` absent → omit `babashka.tasks` shell helpers
- `LIBASHKA_CAP_FILE_READ` / `LIBASHKA_CAP_FILE_WRITE` absent → restrict `clojure.java.io`
- `LIBASHKA_CAP_NETWORK` absent → omit socket / HTTP namespaces

### 5.2 `libashka.serialization` — JSON Serialization

- `->json [v]` → `cheshire.core/generate-string`
- `<-json [s]` → `cheshire.core/parse-string` with keyword keys
- `->json-safe [v]` → Post-walks the value tree, converting non-serializable SCI types (IReified, deftype instances) to their string representations

### 5.3 `libashka.callbacks` — Host Function Registry

Provides the bridge from C function pointers to callable SCI vars.

**`register-host-fn [ctx ns fn-name fn-ptr]`**:
1. Wraps the `LibashkaHostFn` C function pointer in a Clojure function that (a) serialises all received arguments to a JSON array via `->json`, (b) invokes the C pointer, (c) parses the returned JSON string back into Clojure data.
2. Creates the SCI namespace `ns` if it does not already exist.
3. Interns a var named `fn-name` in that namespace, bound to the wrapper function.
4. After registration completes, scripts call the host function with natural Clojure syntax — no explicit dispatch string needed.

**Example — C side:**
```c
char *my_logger(const char *json_args) {
    printf("[HOST] %s\n", json_args);
    return strdup("true");
}
libashka_register_host_fn(ctx, "log", "debug", my_logger);
```

**Example — Script side:**
```clojure
(log/debug "hello")         ;; → C receives "[\"hello\"]", returns true
(log/debug "x" 42)          ;; → C receives "[\"x\",42]"
```

**`host/invoke` primitive** — The system provides a built-in `host/invoke` function as the underlying dispatch mechanism that the dynamic proxy wrappers use internally. It is also directly callable for dynamic use cases:
```clojure
(host/invoke "log" "debug" "hello")  ;; equivalent to (log/debug "hello")
```

**`unregister-host-fn [ctx ns fn-name]`** — Removes a previously registered host function and un-interns its SCI var.

> The old design used `host.lib/invoke`; the `lib` sub-segment was dropped for conciseness. The built-in system namespace is now simply `host`.

### 5.4 Data Flow

```
Host (C)                    Java (@CEntryPoint)              Clojure
------                      -------------------              --------
libashka_create_context  →  createContext()             →    core/create-context
                                 ↓                              ↓
                            创建 Isolate                   sci/init + cap filtering
                                 ↓
                            返回 IsolateThread (即 ctx 句柄)

libashka_eval_string     →  evalString(thread, ...)     →    core/eval-string
                                 ↓                              ↓
                            路由到正确 Isolate              sci/eval-string
                                 ↓                              ↓
                            ← JSON 字符串                ←    ser/->json
                                 ↓
                            构造 LibashkaVal → 返回给 C

libashka_register_host_fn → registerHostFn(thread, ...) →    cb/register-host-fn
                                 ↓                              ↓
                            路由到正确 Isolate              包装 fnPtr 为 wrapper fn
                                 ↓                              ↓
                            存储 LibashkaHostFn              在 SCI ns 中 intern var
                                                                  ↓
(脚本直接调用 (log/debug "hello")) → wrapper fn → .invoke(fnPtr) → C 宿主函数
                                       ↑
                                 参数序列化为 JSON
                                 ["hello"]
                                       ↓
                                 返回值解析为 Clojure 数据
                                 C 返回 "true" → true

libashka_destroy_context  →  destroyContext(thread)     →    core/destroy-context
                                 ↓                              ↓
                            1. SIGTERM → SIGKILL Pod 进程    遍历 pod-pids，批量清理
                            2. 等待所有子进程退出             waitpid 回收
                            3. 销毁 SCI 上下文
                            4. 分离 Isolate
```

## 6. Build Pipeline

### 6.1 Three Stages

**Stage 1: Uberjar** (reuses babashka build infrastructure)
```
babashka/script/uberjar
  → Leiningen with feature-flag profiles
  → reify2 bytecode generation
  → Output: babashka-standalone.jar
```

**Stage 2: Native Image** (shared library compilation)
```
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
  [all existing babashka native-image flags...]

Output: target/libbabashka.so (Linux)
        target/libbabashka.dylib (macOS)
        target/libbabashka.dll  (Windows)
```

**Stage 3: Integration test compilation**
```
C:    gcc -ldl test/integration/c/main.c -o test/c_runner
Zig:  zig build-exe test/integration/zig/main.zig
Rust: cargo build --manifest-path test/integration/rust/Cargo.toml
```

### 6.2 `script/build`

Bash script orchestrating stages 1-2. Key environment variables:

- `BABASHKA_FEATURE_*` — Feature flags (all enabled by default, matching full babashka)
- `BABASHKA_DISABLE_SIGNAL_HANDLERS=true` — Prevents the shared library from intercepting host process signals
- `GRAALVM_HOME` — Path to GraalVM installation

### 6.3 Top-level `Makefile`

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

## 7. Testing Strategy

### 7.1 Three-Layer Test Pyramid

| Layer | Location | Scope | % of tests |
|-------|----------|-------|------------|
| Clojure unit tests | `test/clojure/libashka/` | Context lifecycle, serialization, callbacks, Pod namespace, capability filtering | 55-60% |
| Integration tests | `test/integration/{c,zig,rust}/` | dlopen real .so, end-to-end scenarios | 15-20% |
| Java layer | (validated indirectly via Clojure + integration) | @CEntryPoint correctness | 20-25% |

### 7.2 Clojure Test Files

| File | What it tests |
|------|--------------|
| `core_test.clj` | `create-context` with/without capability mask, `destroy-context` Pod cleanup, `reset-context`, `eval-string` basic evaluation, exception capture, multi-context isolation |
| `serialization_test.clj` | Clojure ↔ JSON round-trip, SCI special type safety, nil/number/string/nested structure correctness |
| `callbacks_test.clj` | Host fn registration, script-calls-host flow, argument passing, return value propagation, unregistered-fn error |
| `pod_test.clj` | `babashka.pods` namespace availability, Pod loading, Pod subprocess lifecycle, subprocess cleanup on destroy/reset, Pod namespace omitted when `LIBASHKA_CAP_POD` is absent |

### 7.3 Integration Test Scenarios (all languages)

Each integration test suite (C, Zig, Rust) covers:

1. `test_create_destroy` — Context creation/destruction, non-NULL verification
2. `test_create_with_capabilities` — Restricted context creation, verify restricted operations fail
3. `test_eval_simple` — `(+ 1 2)` → 3
4. `test_eval_error` — Syntax error → `{"error": "..."}`
5. `test_multi_contexts` — Two independent contexts, no variable cross-contamination
6. `test_host_callback` — Register C fn, call from script, verify result
7. `test_call_script_fn` — Host calls script-defined function
8. `test_pod_load_and_cleanup` — Load external Pod, call Pod function, verify subprocess is killed on destroy
9. `test_reset_context` — Reset context, verify state cleared, verify Pod processes killed
10. `test_memory_leak` — Loop create/destroy, verify no memory leak
11. `test_thread_isolation` — Multi-threaded, each with own context, verify isolation

### 7.4 TDD Execution Order

**Phase 1 — Clojure layer (pure JVM, no native-image dependency):**
1. RED: Write `core_test.clj` → fail (file/fn does not exist)
2. GREEN: Implement `libashka.core` → pass
3. RED: Write `serialization_test.clj` → fail
4. GREEN: Implement `libashka.serialization` → pass
5. RED: Write `callbacks_test.clj` → fail
6. GREEN: Implement `libashka.callbacks` → pass
7. RED: Write `pod_test.clj` → fail
8. GREEN: Ensure `babashka.pods` namespace correctly injected, Pod subprocess cleanup functional → pass

**Phase 2 — Java @CEntryPoint layer:**
- Validated on JVM via Clojure tests that exercise the same code paths
- Full @CEntryPoint validation requires native-image compilation (Phase 3)

**Phase 3 — Integration tests (requires compiled .so):**
1. Build: `script/build` → `libbabashka.so`
2. RED: Write `main.c` → compilation against header succeeds, runtime tests fail until .so is linked
3. GREEN: Compile `main.c`, dlopen `.so`, run all scenarios → pass
4. Repeat for Zig and Rust

## 8. Pod Support

### 8.1 Requirements

- `babashka.pods` namespace must be fully available in every SCI context (unless restricted by capability flags)
- Pods communicate over stdin/stdout with their parent process
- In shared-library mode, the parent process is the host application — Pod subprocess management must be preserved
- External Pod loading via `babashka.pods/load-pod` must work

### 8.2 Implementation

- Babashka's existing Pod infrastructure is inherited unchanged
- `clojure.java.shell` and process management are enabled via native-image flags:
  - `--allow-incomplete-classpath`
  - `--report-unsupported-elements-at-runtime`
- SCI context initialization includes `babashka.pods` in the namespace map unless `LIBASHKA_CAP_POD` is absent

### 8.3 Pod Process Lifecycle & Cleanup

Every Pod subprocess spawned by a context is tracked in a **per-context PID set** (`pod-pids` atom in the Isolate-local state). On `destroy_context` or `reset_context`:

1. Iterate all tracked PIDs
2. Send `SIGTERM` to each child process
3. Wait up to 3 seconds for graceful shutdown
4. Send `SIGKILL` to any survivors
5. Call `waitpid()` on every PID to reap zombies
6. Clear the PID set

This prevents orphan/zombie Pod processes from accumulating in long-running host applications that repeatedly create and destroy contexts.

## 9. Thread Safety & Isolation

### 9.1 Per-Thread Isolate Model

Each call to `libashka_create_context` returns a GraalVM `IsolateThread` pointer:
- Isolates share **no mutable state** — each has its own heap, SCI context, and host function registry
- GraalVM's `--shared` mode includes built-in isolate-creation functions (`graal_create_isolate`) that the C host uses
- An Isolate is bound to its creating thread (GraalVM constraint)

### 9.2 How It Actually Works

GraalVM's `@CEntryPoint` mechanism works as follows in `--shared` mode:

1. The compiled `.so` exports `graal_create_isolate()` — called once by the C host to create the initial Isolate
2. Every `@CEntryPoint` method's first parameter `IsolateThread` is the pointer that GraalVM uses to route execution to the correct Isolate's heap and static variables
3. **No global ConcurerntHashMap is needed or possible** — static fields in Java are naturally Isolate-private. Each Isolate's static `SCIState` is automatically the correct one for that Isolate
4. The C host passes the `graal_isolatet_t*` (aliased as `libashka_ctx_t*`) to every call; GraalVM switches context transparently

### 9.3 Constraints

- An Isolate cannot be shared across threads — each thread must create its own context
- GraalVM Isolate creation has a fixed memory overhead (~few MB per Isolate)
- The number of concurrent Isolates is limited by available memory
- For high-frequency create/destroy patterns, use `libashka_reset_context` to reuse an Isolate instead of allocating a new one

## 10. Dependencies

### 10.1 Build Dependencies

- **GraalVM** (JDK 21+ with `native-image`)
- **Leiningen** (inherited from babashka build)
- **Clojure CLI** (`deps.edn` / tools.deps)
- **GCC / Zig / Rust toolchains** (for integration tests only)

### 10.2 Runtime Dependencies

- **None** — the compiled `.so`/`.dylib`/`.dll` is a standalone native binary
- Host application links against the shared library; no JVM or Clojure runtime needed at deployment

### 10.3 deps.edn

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

## 11. Open Questions & Risks

### 11.1 Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| Babashka upstream build changes break our pipeline | High | Git submodule pinned to specific SHA; CI catches breakage |
| GraalVM `--shared` conflicts with babashka native-image flags | Medium | libsci already validates `--shared` compatibility; test early |
| Pod subprocess behavior differs in shared-library context | Medium | Dedicated Pod integration tests across all 3 host languages; PID tracking + mandatory cleanup |
| Isolate memory overhead unacceptable for high-concurrency use | Low | Document the constraint; provide `reset_context` for reuse without re-allocation |
| Feature flag combination explosion | Low | Default to all features enabled; capability mask trims dangerous ops, not build flags |
| Host fails to call `libashka_free_value` causing memory leak | Medium | Document ownership rules in header; integration tests verify with valgrind/asan |
| Capability model is too coarse-grained | Low | Start with 5 coarse capabilities; fine-grained policies can be added later without ABI breakage |

### 11.2 Future Considerations (out of scope for initial implementation)

- CI/CD matrix builds for all platform/arch combinations
- Pre-built binary releases (GitHub Releases)
- Language-specific binding libraries (e.g., `zig-libashka`, `rust-libashka` crates)
- Isolate pooling with configurable pool size
- Binary serialization backends (Transit, MessagePack) under `LIBASHKA_BINARY` tag
- Fine-grained capability policies (e.g., filesystem path whitelist, network address allowlist)
